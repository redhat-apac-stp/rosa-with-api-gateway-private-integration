# ROSA with AWS API Gateway

This article describe how to setup ROSA and AWS API Gateway using private APIs. The solution relieas upon SSL/TLS certificates for enabling end-to-end HTTPS and uses a split-horizon DNS approach to enable a single registered domain name to be used for both internal and external endpoints so as to minimise the total number of domains and certificates needed.

A diagram of the solution architecture showing all of the major components and flows is presented.

<img src="https://github.com/redhat-apac-stp/rosa-with-aws-api-gateway/blob/main/ROSA%20with%20AWS%20API%20Gateway.jpeg" alt="ROSA with AWS API Gateway" width="970" height="660">

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

Create a ClusterIssuer resource in the openshift-operators namespace that points to a LetsEncrypt endpoint for performing a DNS01 challenge:

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

Click on the URL for the API endpoint.






configure end-to-end connectivity between an application hosted on ROSA and AWS API Gateway using HTTP API private integrations  and SSL/TLS certificates for secure communications between the API Gateway and ROSA ingress end point.

API Gateway supports private integrations via a VPC link that terminates on NLB/ALB endpoints - CLB endpoints are not supported when deploying VPC links. Thus the default ROSA OpenShift Router which deploys a CLB cannot be used for accessing applications running on ROSA via the AWS API Gateway. Furthermore the ingress controller must support proxy protocol version 2 to send origin source and destination address information. Adding additional Ingress Controllers (haproxy) and publishing these via NLB is currently not supported by SRE.

The instructions below will first deploy a non-secured (HTTP) setup to verify connectivity. Subsequently this is upgraded to a secured channel using a SSL/TLS certificate that must be validated by one of the following public Certificate Authorities: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html. Note that self-signed certificates or those issued by private Certificate Authorities are not supported by API Gateway. A diagram depicting the secured channel is as follows.

<img src="https://raw.githubusercontent.com/redhat-apac-stp/rosa-with-api-gateway-private-integration/main/ROSA%20for%20CIMB-Thai%20-%20ROSA%20with%20API%20Gateway.jpeg" alt="My Project GIF">

For the purpose of this setup LetsEncrypt is used as the public Certificate Authority. The wildcard domain name used for the certificate CommonName (CN) and subject altNames is \*.example.com. Change this to a registered domain name under your control and configure a public hosted zone in Route 53 for the base domain name that LetsEncrypt can validate. Note down the auto-generated hosted zone ID for use later.

***

A public ROSA STS cluster can be deployed as per the following: https://mobb.ninja/docs/rosa/sts/

Install the NGINX Ingress Operator (v0.4.0) via OperatorHub from the OpenShift web console. Do not change any of the defaults. For more details please consult the following link: https://github.com/nginxinc/nginx-ingress-operator/blob/master/docs/openshift-installation.md

After installing the NGINX Ingress Operator create a minimal NGINX Ingress Controller instance with a service type of NodePort in the openshift-operators namespace.

	apiVersion: k8s.nginx.org/v1alpha1
	kind: NginxIngressController
	metadata:
	  name: my-nginx-ingress-controller
	  namespace: openshift-operators
	spec:
  	  type: deployment
	  nginxPlus: false
	  image:
	    repository: nginx/nginx-ingress
	    tag: edge-ubi
	    pullPolicy: Always
	  replicas: 1
	  serviceType: NodePort

Modify service/my-nginx-ingress-controller and add the following annotations to the metadata:

	annotations:
	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
	  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

Modify nginxingresscontroller/my-nginx-ingress-controller and change the serviceType to LoadBalancer and add a configMapData stanza to enable proxy protocol:

	apiVersion: k8s.nginx.org/v1alpha1
	kind: NginxIngressController
	metadata:
	  name: my-nginx-ingress-controller
	  namespace: openshift-operators
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

Modify the internal-facing NLB that is created from the AWS web console. There will be two listeners associated with the NLB (TCP:80 and TCP:443) and both will need to have their proxy protocol version 2 attributes enabled.

Deploy the echoserver application into a new namespace.

	oc new-project my-project

Create a service account and associate it with the anyuid SCC policy:

	oc create sa sa-with-anyuid -n my-project
	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid -n my-project

Create the application deployment:

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: echoserver
	  namespace: my-project
	spec:
	  selector:
	    matchLabels:
	      app: echoserver
	  replicas: 2
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

Create the application service:

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

Configure an ingress resource exposing an HTTP route for the FQDN echo.example.com:

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

Confirm all resources are ready and that the NGINX Ingress Controller is managing the ingress instance:

	oc get all -n my-project
	oc describe ingress/echoserver -n my-project

Test connectivity to the internal-facing NLB from any node in the ROSA cluster:

	elb=`oc get svc -n openshift-operators | grep 'nginx-ingress-controller' | awk '{print $4}'`
	host $elb | awk '{print$4}'

	oc debug node/<any node> -- curl <elb ip address> -H 'echo.example.com'
	
The echoserver output displays the real IP address of the user-agent in the x-forwarded-for field, as well as the IP address of the NGINX Ingress Controller pod in the client_address field.

The next steps link the AWS API Gateway to the echoserver host URI via a private HTTP API integration over TCP port 80. Subsequently this will be upgraded to SSL/TLS over TCP port 443.

From the AWS web console select the AWS API Gateway service and first create a VPC link to setup a tunnel into the ROSA VPC. Choose the option to create a VPC link for HTTP APIs and give it a name and select ROSA VPC as the target. Select all of the ROSA VPC subnets but leave all security groups uchecked before pressing the create button.

Whilst the VPC link is being provisioned start creating the HTTP API method type and give it a name before pressing the review and create button (skip all of the other steps as these will be completed post-build).

From the develop menu select routes and create a route with a method of ANY and a route of /{proxy+}.

On the next screen that appears select the ANY method for the /{proxy+} route and select attach an integration. Create a new integration and set the integration target to be a private resource. Select manually and choose ALB/NLB as the target service. Select the correct NLB ARN (internal-facing). Select TCP:80 for the listener. Do not modify any of the advanced settings. Select the VPC link that should have completed provisioning.

From the integrations menu select manage integration and create a new parameter mapping. Set the mapping type to all incoming requests and add a mapping with the following parameter option settings:

	parameter to modify = header.Host
	modification type = Overwrite
	value = echo.example.com

Push the create button and that completes the setup. 

Click on the API URL for the $default stage. It should look something like this: https://0fj4opay08.execute-api.ap-southeast-1.amazonaws.com.

The echoserver response in the web browser indicates that the real IP address of the user-agent is now contained in the forwarded field of the header and the x-forwarded-for field contains the IP address of the ENI that was created by the VPC link in the ROSA VPC. The client_address field should be the same as before. Also note that the field x-forwarded-proto should be http and the field x-forwarded-port should be 80.

***

The next steps upgrade the unencrypted connection between the API Gateway and ROSA to use SSL/TLS.

Install the Cert Manager Operator (v1.5.4) via the OperatorHub from the OpenShift web console. Do not change any of the defaults. For more details please consult the following link: https://cert-manager.io/docs/installation/operator-lifecycle-manager/

These steps assume that a registered domain name which is under your control has been provisioned in a public hosted zone on Route 53. The OIDC Identity Provider created during ROSA installation is used to enable the Cert Manager service account to obtain temporary credentials (using IAM Roles for Service Accounts) for modifying Route 53 records to enable the DNS-01 challenge to be performed by LetsEncrypt.

Use IAM to create a policy named cert-manager-policy with the following set of permissions:

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
		    "Resource": "arn:aws:route53:::hostedzone/<hosted zone id for your domain>"
		}
	    ]
	}

Use IAM to create a role named cert-manager-irsa and link it to the aforementioned policy. Establish a trust relationship with the OIDC Identity Provider that was created during ROSA cluster installation. You can obtain the name of the S3 bucket hosting your JWKS endpoint via the following command:

	rosa describe cluster -c <cluster name>
	
The trust relationship policy should look like the following:

	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Principal": {
		"Federated": "arn:aws:iam::<your AWS account id>:oidc-provider/rh-oidc.s3.us-east-1.amazonaws.com/<S3 bucket hosting your JWKS endpoint>"
	      },
	      "Action": "sts:AssumeRoleWithWebIdentity",
	      "Condition": {
		"StringEquals": {
		  "rh-oidc.s3.us-east-1.amazonaws.com/<S3 bucket hosting your JWKS endpoint>:sub": "system:serviceaccount:openshift-operators:cert-manager"
		}
	      }
	    }
	  ]
	}

Edit the cert-manager service account (sa/cert-manager) in the openshift-operators namespace and add the following annotation:

	annotations:
	  eks.amazonaws.com/role-arn: arn:aws:iam::<your AWS account id>:role/cert-manager-irsa

Delete the cert-manager pod so that it gets recreated with a service account token (aws-iam-token) injected into the pod via the webhook token authenticator. The spec section for the pod should now include the following environment variables and values:

	- name: AWS_ROLE_ARN
	  value: arn:aws:iam::<your AWS account id>:role/cert-manager-irsa
	- name: AWS_WEB_IDENTITY_TOKEN_FILE
	  value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token

Create a ClusterIssuer resource in the openshift-operators that points to the LetsEncrypt production endpoint used for performing DNS-01 challenges:

	apiVersion: cert-manager.io/v1
	kind: ClusterIssuer
	metadata:
	  name: letsencrypt
	spec:
	  acme:
	  solvers:
	    - dns01:
	      route53:
	        hostedZoneID: <hosted zone id for your domain>
	        region: <your AWS regions>
	  server: 'https://acme-v02.api.letsencrypt.org/directory'
	  privateKeySecretRef:
	    name: letsencrypt
	  email: <your email>
			        
Create a certificate in the namespace of the application.

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

In Route 53 verify the creation of a TXT record for the hosted zone of your domain.

Verify the readiness of the certificate:

	oc get certificates -n my-project
	
Verify the presence of the TLS secret that stores the certificate and private key:

	oc get secret -n my-project

Modify the existing ingress so that it upgrades incoming connections to use SSL/TLS:

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

In the API Gateway from the integrations menu select the previously created  ANY /{proxy+} route and select manage integration. Edit integration details and change the listener from TCP:80 to TCP:443. Under the advanced settings enter the FQDN of the host (e.g., echo.example.com) as the secure server name. After saving these changes invoke the API URL for the $default stage as before. This time the field x-forwarded-proto should be https and the field x-forwarded-port should be 443.

***

If you experience any errors please contact the author at jwilms@redhat.com
