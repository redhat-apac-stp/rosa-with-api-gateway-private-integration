# ROSA with AWS API Gateway HTTP APIs

This article describes the integration of ROSA with AWS API Gateway HTTP APIs. The solution leverages a split-horizon DNS approach using wildcard certificates for both private and public application endpoints (cluster API endpoints are not affected). The following diagram depicts the major components and interactions.

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/ROSA%20with%20AWS%20API%20Gateway.png" alt="ROSA with AWS API Gateway">

ROSA can be deployed as either a public or private cluster and these instructions. The diagram above depicts a private ROSA cluster deployed in STS mode based on following these instructions:

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

Add an A record for the wildcard domain \*.example.com in the private hosted zone and configure it as an alias to the NLB.

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

From the OpenShift web console select the run command icon on the top menu bar to open a web terminal (refresh your browser if you do no see the run command icon). Test HTTP access:

	curl echo.example.com
	
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

Switch cert-manager from using the default private DNS servers associated with the ROSA VPC to a public DNS server (e.g., Google DNS).

	oc edit csv/cert-manager.v1.5.4
	
	    spec:
              containers:
              - args:
                - --v=2
                - --cluster-resource-namespace=$(POD_NAMESPACE)
                - --leader-election-namespace=kube-system
                - --dns01-recursive-nameservers="8.8.8.8:53"

Validate that these changes are reflected in the cert-manager pod (check for the presence of AWS_ROLE_ARN, AWS_WEB_IDENTITY_TOKEN_FILE, and dns01-recursive-nameservers).

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

From the OpenShift web console select the run command icon to open a web terminal. Test HTTPS access:

	curl -Lk echo.example.com

Assuming this worked the next steps are to create a private integration with API Gateway.

Create a VPC Link in API Gateway for HTTP APIs. Associate the VPC hosting ROSA along with all subnets but none of the security groups.

Whilst the VPC Link is being created create the HTTP API endpoint. Omit details for the integration, route and stage as these will be configured later.

Create a route for the method ANY with a path name of /{proxy+}. After the route is created configure parameter mappings for all incoming requests as follows:

	header.Host	Overwrite	echo.example.com

Create an integration for the route that was created. The integration target is of type private resource. Select ALB/NLB for the target service along with the ARN of the load balancer fronting the NGINX ingress controller. Select TCP 443 for the listener and enter echo.example.com in the secure server name field. Select the VPC Link that should have been created by now.

Click on the auto-generated URL for the API endpoint which should now display the contents of the echoserver.

As a final (but optional) step a custom domain name can be created for acessing an API endpoint via a user-friendly URL (e.g., https://echo.example.com).

Export the secret containing the wildcard certificate into a file for uploading to AWS Certificate Manager via the certficate import function.

	oc extract secret/example-com-tls --to=/tmp

From API Gateway create a custom domain (e.g., echo.example.com) as a regional endpoint and select the ARN of the certificate that was imported. Note down the regional API Gateway domain name that is auto-generated.

Add an A record alias for the custom domain name resolving to the regional API Gateway domain name in Route 53.

Use curl to invoke the user-friendly URL.

	curl https://echo.example.com

Note that if you wish to use a wildcard custom domain to cover all possible API endpoints, API Gateway requires the certificate covering the wildcard domain to be generated by ACM.

***

If you experience any errors or have questions please contact the author at jwilms@redhat.com
