# ROSA integration with AWS API Gateway

These instructions describe how to configure end-to-end connectivity between an application hosted on ROSA and AWS API Gateway using private integrations for HTTP APIs and SSL/TLS certificates for protection.

AWS API Gateway supports private integrations via a VPC link that terminates on NLB/ALB endpoints - CLB endpoints are not supported for termination of a VPC link. Thus the default ROSA OpenShift Router which deploys a CLB cannot be used for accessing applications running on ROSA via the AWS API Gateway. Furthermore the ingress controller must support proxy protocol version 2 to send origin source and destination address information.  

The instructions below first deploy a non-secured (HTTP) setup to verify connectivity. Subsequently this is upgraded to a secured (SSL/TLS) configuration using a X509 certificate that must be issued by one of the trusted Certificate Authorities as per the following link: https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html

For the purpose of this setup LetsEncrypt is the chosen certificate issuer. The wildcard domain name to be used for the certificate CommonName (CN) is \*.example.com. Change this to a registered domain name under your control and configure a public hosted zone in AWS Route 53 for the base domain name (example.com). Note down the auto-generated hosted zone ID for use later.

***

A public ROSA STS cluster can be deployed as per the following instructions: https://mobb.ninja/docs/rosa/sts/

Install NGINX Ingress Operator (v0.4.0) via the OperatorHub from the OpenShift web console. Do not change the defaults. For more details please consult the following link: https://github.com/nginxinc/nginx-ingress-operator/blob/master/docs/openshift-installation.md

After installing the NGINX Ingress Operator create a minimal configuration for a new NGINX Ingress Controller instance in the openshift-operators namespace:

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

Modify the service/my-nginx-ingress-controller that is created and add the following annotations to the metadata section:

	annotations:
	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
	  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

Modify the nginxingresscontroller/my-nginx-ingress-controller resource and change the serviceType and add a configMapData stanza:

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

Modify the internal-facing NLB that is created for service/my-nginx-ingress-controller. Enable proxy protocol version 2 for the two listeners configured on the NLB (TCP:80 and TCP:443) from the AWS web console.

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

Verify the readiness of all resources:

	oc get all -n my-project

Configure an ingress resource exposing an HTTP route with a FQDN of echo.example.com that will be managed by the NGINX Ingress Controller:

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

Confirm all resources are ready and that the NGINX Ingress Controller is managing the ingress resource:

	oc get all -n my-project
	oc describe ingress/echoserver -n my-project

Test connectivity to the internal-facing NLB from a node in the VPC hosting the ROSA cluster:

	elb=`oc get svc -n openshift-operators | grep 'nginx-ingress-controller' | awk '{print $4}'`
	host $elb | awk '{print$4}'`

	oc debug node/<any node> -- curl <elb ip> -H 'echo.example.com'
	
The echoserver output displays the real IP address of the user-agent (in the x-forwarded-for field) as well as the IP address of the NGINX Ingress Controller pod (client_address field) requesting the echoserver host URI.

The next steps link the AWS API Gateway to the echoserver host URI via a private HTTP API integration over TCP port 80. Subsequently this will be upgraded to SSL/TLS over TCP port 443.

From the AWS web console select the AWS API Gateway service and first create a VPC link to tunnel into the ROSA VPC. Choose the option to create a VPC link for HTTP APIs and give it a name and select the ROSA VPC. Select all subnets but not any of the security groups before pressing the create button.

Whilst the VPC link is being provisioned start creating the HTTP API type and give it a name before pressing the review and create button (skip all of the other steps as these will be completed post-creation).

From the develop drop down menu that appears select routes and create a route. Set the method to ANY and set the route to /{proxy+}.

On the next screen that appears select the ANY method under the /{proxy+} route and attach an integration to this route. Select create and attach an integration. Set the integration target to be a private resource. Select manual for the method and choose ALB/NLB for the target service. Select the correct NLB (use echo $elb if unsure). Select only the TCP:80 listener. Do not modify any of the advanced settings. Select the VPC link that should have completed provisioning by now.

From the integrations drop down menu select manage integration and create a new parameter mapping. Set the mapping type to all incoming requests and add a new mapping. The details of the mapping are as follows:

	parameter to modify = header.Host
	modification type = Overwrite
	value = echo.example.com

Push the create button and that completes the setup. 

Click on the API URL for the $default stage. It should look something like this: https://0fj4opay08.execute-api.ap-southeast-1.amazonaws.com.

The echoserver response in the web browser indicates that the real IP address of the user-agent is now contained in the forwarded field of the header and the x-forwarded-for field contains the IP address of the ENI for the AWS API Gateway that was created for the VPC link terminating in the ROSA VPC. The client_address field should be the same as before. Also note that the field x-forwarded-proto should be http and the field x-forwarded-port should be 80.

***

The next steps upgrade the unencrypted connection between the AWS API Gateway and ROSA with SSL/TLS.

Install Cert Manager Operator (v1.5.4) via the OperatorHub from the OpenShift web console. Do not change the defaults. For more details please consult the following link: https://cert-manager.io/docs/installation/operator-lifecycle-manager/

These steps assume that a registered domain name which is under your control has been provisioned in AWS Route 53 as described above. Because a DNS-01 challenge is required to validate wildcard certificates this will require access to AWS credentials to modify DNS resource records. Choose an IAM user that has the necessary permissions and then store their access key as a secret:

	oc create secret generic route53-secret --from-literal=aws-secret-access-key=<your AWS secret access key> -n openshift-operators

Create a ClusterIssuer resource in the openshift-operators that points to the ACME v2 production endpoint using a DNS-01 to validate wildcard identifiers:

	apiVersion: cert-manager.io/v1
	kind: ClusterIssuer
	metadata:
	  name: letsencrypt
	spec:
	  acme:
	  solvers:
	    - dns01:
	      route53:
	        accessKeyID: <your AWS access key id>
	        hostedZoneID: <hosted zone id for your domain>
	        region: <your AWS regions>
	        secretAccessKeySecretRef:
	          key: aws-secret-access-key
	          name: route53-secret
	  server: 'https://acme-v02.api.letsencrypt.org/directory'
	  privateKeySecretRef:
	    name: letsencrypt
	  email: <your email>
			        
Create a certificate in the namespace of the application to be protected.

	apiVersion: cert-manager.io/v1
	kind: Certificate
	metadata:
	  name: my-cert
	  namespace: my-project
	spec:
	  commonName: '*.example.com'
	  dnsNames:
	  - '*.example.com'
	  issuerRef:
	    kind: ClusterIssuer
	    name: letsencrypt
	  secretName: example-com-tls

Verify the readiness of the certificate:

	oc get certificates -n my-project
	
Verify the presence of the certificate stored as a TLS secret:

	oc get secret -n my-project

Modify the existing ingress so that it upgrades connections to SSL/TLS for the host echo.example.com using the wildcard certificate stored in a secret:

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

Modify the AWS API Gateway so that it now routes traffic to port 443 using SSL/TLS. Start by selecting integrations for the previously created  ANY /{proxy+} route and select manage integration. Next edit the integration details and change the listener from TCP:80 to TCP:443. Open the advanced settings and enter the FQDN of the host, i.e., echo.example.com for the secure server name field. Click save and then invoke the API URL for the $default stage as before. This time the field x-forwarded-proto should be https and the field x-forwarded-port should be 443.

***
