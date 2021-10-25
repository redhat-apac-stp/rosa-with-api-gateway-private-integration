# ROSA with AWS API Gateway

This article describe how to setup ROSA and AWS API Gateway using private APIs. The solution relieas upon SSL/TLS certificates for enabling end-to-end HTTPS and uses a split-horizon DNS approach to enable a single registered domain name to be used for both internal and external endpoints so as to minimise the total number of domains and certificates needed.

A diagram of the solution architecture showing all of the major components and flows is presented.

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/ROSA%20with%20AWS%20API%20Gateway.png" alt="ROSA with AWS API Gateway">

ROSA can be deployed as either a public or private cluster and the instructions will work in either case. The diagram above depicts a private ROSA cluster deployed in STS mode as per the following set of instructions:

https://mobb.ninja/docs/rosa/sts/

Post-installation of ROSA install the following set of operators from OperatorHub in the OpenShift web console:

	NGINX Ingress Operator v0.4.0
	Cert-Manager v1.5.4
	Web Terminal v1.3.0

All operators are installed in the openshift-operators namespace as per default.

The next step require a registered a domain name that will be used for accessing both internal and external endpoints. The domain names used in these instructions are example.com (base domain) and \*.example.com (wilcard domain). Change these to a domain that is registered to you.

Create a public hosted zone in Route 53 with your registered domain name and update your DNS registrar to resolve queries to the name servers that Route 53 has allocated. Note down the hosted zone ID for use later.

Create a private hosted zone in Route 53 with the same domain name and associate it with the VPC hosting ROSA. An additional A record will be added later to this zone aliasing the NLB endpoint fronting the NGINX ingress controller which will be created next.

Create an NGINX ingress controller using the following custom resource in the openshift-operators namespace:

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

Add the following annotations to the my-nginx-ingress-controller service:

	annotations:
	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
	  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

Update the my-nginx-ingress-controller configuration so that it will trigger the creation of an NLB:

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

From the AWS web console modify the NLB that was created by enabling proxy protocol for both listeners (TCP:80 and TCP:443).

Add an additional A record (\*.example.com) to the private hosted zone aliasing the NLB endpoint.

In the next steps the echoserver application is deployed. Create a namespace for hosting this application.

	oc new-project my-project

Create a service account and associate it with the anyuid SCC policy (echoserver runs as user root):

	oc create sa sa-with-anyuid -n my-project
	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid -n my-project

Apply the application deployment:

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

Apply the application service:

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

Create an ingress resource exposing an HTTP route with a fully-qualified domain name of echo.example.com:

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

From the OpenShift web console select the run command icon to open a web terminal. Test access to the internal endpoint:

	curl echo.example.com
	
Assuming this worked the next steps are to generate a TLS certificate and configure an endpoing for HTTPS on the ingress controller. 

To obtain a TLS certificate for the wildcard domain Cert-Manager will need to be able to provision a TXT record in a public hosted zone that LetsEncrypt can externally validate. Cert-Manager will require privileges on Route 53 to perform this action. This will require leveraging the OIDC provider installed by ROSA to generate a signed token that will be exchanged for short-term security credentials by AWS Security Token Service for accessing a role that will be bound to the Cert-Manager service account.

Create a policy named cert-manager-policy in IAM with the following set of permissions. Substitute with the public hosted zone ID created earlier. 

	{
	    "Version": "2012-10-17",
	    "Statement": [
		{
		    "Effect": "Allow",
		    "Action": "route53:GetChange",
		    "Resource": "arn:aws:route53:::change/\*"
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

Create a role named cert-manager-irsa in IAM that has the cert-manager-policy attached. This role also requires the following trust relationship:

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
		"Federated": "arn:aws:iam::<AWS account ID>:<OIDC endpoint URL>"
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

Substitute account ID and OIDC endpoint URL accordingly (the latter can be obtained from Identity Providers under Access Management in IAM).

Add the following annotation to the the cert-manager service account:

	annotations:
	  eks.amazonaws.com/role-arn: arn:aws:iam::<AWS account ID>:role/cert-manager-irsa

Delete the cert-manager pod and inspect the re-created pod to validate the existence of AWS environment variables and token.

Switch cert-manager from using the default private DNS servers associated with the VPC to a public DNS server by adding dns01-recursive-nameservers to the list of arguments.

	oc edit csv/cert-manager.v1.5.4
	
	    spec:
              containers:
              - args:
                - --v=2
                - --cluster-resource-namespace=$(POD_NAMESPACE)
                - --leader-election-namespace=kube-system
                - --dns01-recursive-nameservers="8.8.8.8:53"

Note that this modification should only be made to the cert-manager specification section. Also note the use of well-known Google public DNS servers to prove there is no dependency on AWS for domain name resolution.

Create a ClusterIssuer resource in the openshift-operators namespace that points to a LetsEncrypt endpoint for the DNS01 challenge. Note that this should be a production endpoint as this uses a Certificate Authority from the list that is supported by API Gateway (https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html).

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
			        
Create a Certificate resource in the echoserver application namespace for the wildcard domain:

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

Verify the creation of a TXT record in the public hosted zone in Route 53.

Verify the certificate is ready:

	oc get certificates -n my-project
	
Verify the TLS secret is created:

	oc get secret -n my-project

Update the ingress resource to enable HTTPS using the TLS secret for the wildcard domain:

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

Confirm all resources are up and ready:

	oc get all,ingress -n my-project

From the OpenShift web console select the run command icon to open a web terminal. Test access to the internal endpoint:

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

Use curl to invoke the URL to display the contents of the echoserver:

	curl https://echo.example.com

Note that if you wish to create a wildcard custom domain to cover all possible API endpoints, API Gateway requires the certificate covering the wildcard domain to be generated by ACM.

***

If you experience any errors or have questions please contact the author at jwilms@redhat.com
