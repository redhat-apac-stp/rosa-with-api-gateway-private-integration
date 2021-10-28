# ROSA with AWS API Gateway HTTP APIs

This article describes how to integrate ROSA with AWS API Gateway HTTP APIs that are exposed to multiple end users. It thus becomes imperative to ensure that a shared backend service is able to distinguish between different client sources for the purpose of correct execution and auditing.

Technically this requires implementing a number of supporting technology patterns such as transparent proxying, wildcard certificates and split-horizon DNS to simplify end-to-end communications using a common custom domain name. The following diagram depicts the major components and their interaction.

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/ROSA%20with%20AWS%20API%20Gateway.png">

ROSA can be deployed as either a public or private cluster. The diagram above depicts a private ROSA cluster deployed in STS mode based on following these instructions:

https://mobb.ninja/docs/rosa/sts/

Post-installation the following set of operators from OperatorHub should be installed via the OpenShift web console:

	NGINX Ingress Operator v0.4.0
	Cert-Manager v1.5.4
	Web Terminal v1.3.0

All operators should be installed in the openshift-operators namespace with their default settings.

The next step require a registered a domain that will be used for both private and public endpoints. The domain name used in these instructions is example.com (base domain) and \*.example.com (wilcard domain). Change these to a domain that is registered to you.

Create a public hosted zone in Route 53 for the registered domain and update the name servers in your DNS registrar to point to the name servers that Route 53 has allocated. Note down the hosted zone ID for use later.

Create a private hosted zone in Route 53 for the same domain and associate it with the ROSA VPC. An additional A record will be added later for aliasing the NLB endpoint that exposes the NGINX ingress controller.

Create an NGINX ingress controller in the openshift-operators namespace:

	apiVersion: k8s.nginx.org/v1alpha1
	kind: NginxIngressController
	metadata:
	  name: my-nginx-ingress-controller
	spec:
  	  type: deployment
	  nginxPlus: false
	  image:
	    repository: nginx/nginx-ingress
	    tag: edge-ubi
	    pullPolicy: Always
	  replicas: 1
	  serviceType: NodePort

Add the following annotations to the service created for the my-nginx-ingress-controller:

	annotations:
	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
	  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

Modify the configuration of the my-nginx-ingress-controller so that it will trigger the creation of a NLB:

	apiVersion: k8s.nginx.org/v1alpha1
	kind: NginxIngressController
	metadata:
	  name: my-nginx-ingress-controller
	spec:
  	  type: deployment
	  nginxPlus: false
	  image:
	    repository: nginx/nginx-ingress
	    tag: edge-ubi
	    pullPolicy: Always
	  replicas: 1
	  serviceType: LoadBalancer
	  configMapData:
	    proxy-protocol: "True"
	    real-ip-header: "proxy_protocol"
	    set-real-ip-from: "0.0.0.0/0"	  

From the AWS web console modify the NLB that was created by enabling proxy protocol attributes for both listeners (TCP:80 and TCP:443).

Add an A record for the domain echo.example.com in the private hosted zone and configure it as an alias to the NLB.

Deploy the echoserver application in a new namespace which displays HTTP headers when invoked.

	oc new-project my-project

Create a service account and associate it with the anyuid SCC policy (echoserver runs as user root):

	oc create sa sa-with-anyuid -n my-project
	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid -n my-project

Apply the echoserver deployment manifest:

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: echoserver
	  namespace: my-project
	spec:
	  selector:
	    matchLabels:
	      app: echoserver
	  replicas: 1
	  template:
	    metadata:
	      labels:
	        app: echoserver
	    spec:
	      serviceAccount: sa-with-anyuid
	      serviceAccountName: sa-with-anyuid
	      containers:
	      - image: gcr.io/google_containers/echoserver:1.4
	        imagePullPolicy: Always
	        name: echoserver
	        ports:
	        - containerPort: 8080

Apply the echoserver service manifest:

	apiVersion: v1
	kind: Service
	metadata:
	  name: echoserver
	  namespace: my-project
	spec:
	  ports:
	    - port: 80
	      targetPort: 8080
	      protocol: TCP
	  type: ClusterIP
	  selector:
	    app: echoserver

Create an ingress managed by NGINX that will route HTTP request for echo.example.com to the echoserver service:

	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: echoserver
	  namespace: my-project
	spec:
  	  ingressClassName: nginx
  	  rules:
  	  - host: echo.example.com
	    http:
	      paths:
	      - backend:
	          service:
	            name: echoserver
	            port:
	              number: 80
	        path: /
	        pathType: Prefix

Confirm all resources are up and ready:

	oc get all,ingress -n my-project

From the OpenShift web console select the run command icon on the top menu bar to open a web terminal (refresh your browser if you do no see the run command icon) and test HTTP access:

	curl echo.example.com -w "\n"
	
The output should look something like this:

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/echoserver-http.png">

The client_address field contains the IP address of the NGINX ingress controller pod. The address of the user-agent calling the URL is in the x-forwarded-for field and in this case is the IP address of the node running the web terminal pod. Note that the x-forwarded-port and x-forwarded-proto confirm that this connection is over HTTP port 80 as expected.

Assuming this worked the next steps are to generate a wildcard certificate and enable HTTPS routing on the ingress endpoint. 

LetsEncrypt is a supported certificate authority for AWS API Gateway HTTP APIs as per:

https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html

Cert-manager will be used to acquire a wildcard certficate from LetsEncrypt. Privileges on Route 53 are required for the cert-manager controller to perform the DNS01 challenge operation required by LetsEncrypt.

Create an IAM policy named cert-manager-policy with the following permissions. Substitute with the public hosted zone ID you created earlier. 

	{
	    "Version": "2012-10-17",
	    "Statement": [
		{
		    "Effect": "Allow",
		    "Action": "route53:GetChange",
		    "Resource": "arn:aws:route53:::change/*"
		},
		{
		    "Effect": "Allow",
		    "Action": [
			"route53:ChangeResourceRecordSets",
			"route53:ListResourceRecordSets"
		    ],
		    "Resource": "arn:aws:route53:::hostedzone/<public hosted zone ID>"
		}
	    ]
	}

Create an IAM role named cert-manager-irsa and attach to it the cert-manager-policy. This role also requires a trust relationship with the ROSA OIDC provider:

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
		"Federated": "arn:aws:iam::<AWS account ID>:oidc-provider/<OIDC endpoint URL>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
		"StringEquals": {
		  "<OIDC endpoint URL>:sub": "system:serviceaccount:openshift-operators:cert-manager"
		}
	      }
	    }
	  ]
	}

Substitute with values for your AWS account ID and OIDC endpoint URL. Use the rosa describe cluster command to obtain the OIDC endpoint URL and remove the https:// protocol from the path.

Add the following annotation to the the cert-manager service account (substitute with values for your AWS account ID):

	annotations:
	  eks.amazonaws.com/role-arn: arn:aws:iam::<AWS account ID>:role/cert-manager-irsa

Switch cert-manager to use a public DNS server (e.g., Google DNS) for all name resolution. Make this change at the operator level which will propogate this to the deployment and pod.

	oc edit csv/cert-manager.v1.5.4
	
              - args:
                - --v=2
                - --cluster-resource-namespace=$(POD_NAMESPACE)
                - --leader-election-namespace=kube-system
                - --dns01-recursive-nameservers="8.8.8.8:53"

Validate that all changes are reflected in the cert-manager pod (check for the presence of AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE, and dns01-recursive-nameservers).

Create a ClusterIssuer in the openshift-operators namespace that links to the LetsEncrypt provider endpoint for completing the DNS01 challenge. Note that this should be a production endpoint as certificates issued by a staging endpoint are not supported by AWS API Gateway. For a complete list of supported certificate issuing authorities that AWS API Gateway supports see here: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html

	apiVersion: cert-manager.io/v1
	kind: ClusterIssuer
	metadata:
	  name: letsencrypt
	spec:
	  acme:
	    solvers:
	      - dns01:
	        route53:
	          hostedZoneID: <public hosted zone ID>
	          region: <your AWS regions>
	    server: 'https://acme-v02.api.letsencrypt.org/directory'
	    privateKeySecretRef:
	      name: letsencrypt
	    email: <email address>
			        
Create a wildcard Certificate resource in the echoserver application namespace:

	apiVersion: cert-manager.io/v1
	kind: Certificate
	metadata:
	  name: example-com-tls
	  namespace: my-project
	spec:
	  commonName: '*.example.com'
	  dnsNames:
	  - '*.example.com'
	  issuerRef:
	    kind: ClusterIssuer
	    name: letsencrypt
	  secretName: example-com-tls
	  privateKey:
	    rotationPolicy: Always

Verify the creation of a TXT record in the Route 53 public hosted zone for the domain.

Verify the certificate is ready (this may take a few minutes and do be aware of LetsEncrypt rate limits for the production endpoint):

	oc get certificates -n my-project
	
You can also validate all your certificates for a specific domain via the following URL:

	https://crt.sh/?q=example.com
	
Verify the TLS secret is created in the application namespace:

	oc get secret -n my-project

Update the ingress endpoint to support HTTPS routing:

	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: echoserver
	  namespace: my-project
	spec:
  	  ingressClassName: nginx
	  tls:
	  - hosts:
	    - echo.example.com
	    secretName: example-com-tls
  	  rules:
  	  - host: echo.example.com
	      http:
	        paths:
	        - backend:
	            service:
	              name: echoserver
	              port:
	                number: 80
	          path: /
	          pathType: Prefix

Confirm all resources are up and that the ingress is listeneing on both ports 80 and 443:

	oc get all,ingress -n my-project

From the OpenShift web console select the run command icon to open a web terminal and test HTTPS access:

	curl -Lk echo.example.com -w "\n"
	
The output should look something like this:

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/echoserver-https.png">

The client_address field contains the IP address of the NGINX ingress controller pod. The address of the user-agent calling the URL is in the x-forwarded-for field and in this case is the IP address of the node running the web terminal pod. Note that the x-forwarded-port and x-forwarded-proto confirm that this connection is over HTTPS port 443 as expected.

The next steps are to create a private AWS API Gateway HTTP API integration to the application endpoint on ROSA.

Create a VPC Link in API Gateway for HTTP APIs. Associate this link with the ROSA VPC and select all of the subnets hosting the NLB fronting the NGINX ingress controller. Do not select any security group.

Whilst the VPC Link is being provisioned, create a default HTTP API endpoint. Integration, route and stage will be configured separately.

Create a default route for the ANY method with a path name of "/{proxy+}". 

Create a default integration target of private resource for the route and associate it with the NLB fronting the NGINX ingress controller on port 443. Under advanced settings configure the secure server name to echo.example.com. Select the provisioned VPC link.

Under manage integration create a new parameter mapping that applies to all incoming requests and configure this with the following options:

	header.Host	Overwrite	echo.example.com

Click on the auto-generated API endpoint URL provisioned by AWS API Gateway which will open a tab in your web browser.

The output should look something like this:

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/echoserver-apigateway.png">

The client_address field contains the IP address of the NGINX ingress controller pod. The address of the user-agent calling the URL is in the forwarded=for field and in this case is a public IP address. The x-forwarded-for field now contains the private IP address of the ENI of the VPC link created by AWS API Gateway in the ROSA VPC. Note that the x-forwarded-port and x-forwarded-proto confirm that this connection is over HTTPS port 443 as expected.

As a final (but optional) step a custom domain name can be used to acess the API endpoint via a user-friendly URL (e.g., https://echo.example.com).

Export the wildcard certificate into a file in preparation for uploading to AWS Certificate Manager.

	oc extract secret/example-com-tls --to=/tmp --confirm

From ACM create a new certificate using the import option. Copy and paste the certificate, key and chain from the files that were generated.

From AWS API Gateway create a custom domain (e.g., echo.example.com) protected by the certificate that was imported into ACM. Next configure an API mapping for the custom domain with the following options:

	<API endpoint name>	$default	<leave path blank>

From Route 53 add an A record alias in the public hosted zone for echo.example.com to the regional API Gateway domain name auto-generated by AWS API Gateway.

Enter the URL https://echo.example.com in your web browser which should display the following:

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/echoserver-customdomain.png">

All of the HTTP headers remains the same except for the name of the URL which is now uniform across all private and public endpoints thus simplifying the communication flow and tracing. 

To enable different groups of users to access a shared backend service via a common API, this can be accomplished by creating another custom domain name specific to the user group. When the custom domain URL is invoked the custom domain name will show up in HTTP request headers injected by AWS API Gateway; thus enabling the backend service to filter incoming traffic. Other permutations are also possible such as creating separate API endpoints for each user group mapping to seperate VPC links and filtering based on the IP address of the ENI in the HTTP request header.

***

If you experience any errors or have questions please contact the author at jwilms@redhat.com
