# ROSA integration with AWS API Gateway

These instructions describe how to configure end-to-end connectivity between an application hosted on ROSA and AWS API Gateway using private integrations for HTTP APIs and SSL/TLS certificates for protection.

AWS API Gateway supports private integrations via a VPC link that terminates on NLB/ALB endpoints - CLB endpoints are not supported for termination of a VPC link. Thus the default ROSA OpenShift Router which deploys a CLB cannot be used for accessing applications running on ROSA via the AWS API Gateway. Furthermore the ingress controller must support proxy protocol version 2 to send origin source and destination address information.  

https://github.com/openshift-cs/managed-openshift/projects/2

The instructions below first deploy a non-secured (HTTP) setup to verify connectivity. Subsequently this is upgraded to a secured (SSL/TLS) configuration using a X509 certificate that must be issued by one of the trusted Certificate Authorities as per the following link:

https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html

For the purpose of this setup LetsEncrypt is the chosen issuer. The wildcard domain name to be used for the certificate CommonName (CN) is \*.example.com. Change this to a registered domain name under your control and configure a public hosted zone in AWS Route 53 for the base domain name (example.com). Note down the auto-generated hosted zone ID for use later.

***

A public ROSA STS cluster can be deployed as per the following instructions:

https://mobb.ninja/docs/rosa/sts/

Install the NGINX Ingress Operator (v0.4.0) via the OperatorHub in the OpenShift web console selecting all defaults. For more details please consult the following link:

https://github.com/nginxinc/nginx-ingress-operator/blob/master/docs/openshift-installation.md

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

From the AWS web console select the AWS API Gateway service and first create a VPC link to the ROSA VPC. Choose the option to create a VPC link for HTTP APIs and give it a name and select the ROSA VPC. Select all subnets but not any of the security groups before pressing the Create button.

Whilst the VPC link is being provisioned start creating the HTTP API type and give it a name before pressing the Review and Create button (skip all of the other steps).

From the develop drow-down that appears select routes and create a route. Set the method to ANY and path to /{proxy+}.

header.Host
Overwrite
echo.example.com

***


 

Note that the load balancer type provisioned by the NGINX Ingress Controller defaults to CLB. Later, this will be switched to a NLB for integration with the AWS API Gateway.

Cert Manager can be installed using by the Cert Manager Operator (v1.5.4) via the OperatorHub web console in OpenShift as per the following instructions:

https://cert-manager.io/docs/installation/operator-lifecycle-manager/

Before creating issuers and certificates first provision an "A" record in AWS Route 53 pointing to the IP address of the Internet-facing load balancer provisioned by the NGINX Ingress Controller. This name is used by LetsEncrypt when issuing the HTTP01 challenge to validate domain ownership. For this setup www.example.com is the FQDN to be used for hosts in the TLS section of the ingress resource created later, as well as the CommonName (CN) for the certificate subject. This name must be changed to a registered domain name that your organisation owns in order for any of this to work.

	elb=`oc get svc -n openshift-operators | grep 'nginx-ingress-controller' | awk '{print $4}`
	host $elb | awk '{print $4}'`
	oc debug node/<select any node hostname> -- curl <nlb ip address> -H 'echo.example.com'

Because proxy protocol is enabled the echoserver application displays the real IP address of the caller (in the field x-forwarded-for of the header) along with the IP address of the NGINX Ingress Controller pod in the client_address field.

***

Create a ClusterIssuer resource in the openshift-operators namespace pointing to the LetsEncrypt CA issuer for production certficates:

	apiVersion: cert-manager.io/v1
	kind: ClusterIssuer
	metadata:
	  name: letsencrypt-production
	  namespace: openshift-operators
	spec:
	  acme:
	    email: admin@example.com
	    preferredChain: ''
	    privateKeySecretRef:
	      name: letsencrypt-production
	    server: 'https://acme-v02.api.letsencrypt.org/directory'
	    solvers:
	      - http01:
	          ingress:
	            class: nginx
	        selector: {}

Create a Certificate resource in the namespace of the application to be secured (e.g., my-projects):

	apiVersion: cert-manager.io/v1
	kind: Certificate
	metadata:
	  name: www-example-com
	  namespace: my-projects
	spec:
	  commonName: www.example.com
	  dnsNames:
	  - www.example.com
	  issuerRef:
	    kind: ClusterIssuer
	    name: letsencrypt-production
	  secretName: example-com-tls

Verify the readiness of the certificate:

	oc get certificates -n my-projects


Later, the ingress resource will be amended to include a TLS section for HTTPS routing and referencing the TLS secret created earlier.

***
  
Install the NGINX Ingress Operator (v0.4.0) from OperatorHub selecting all default options. This should complete in about 2-3 minutes.

Create an instance of the NGINX Ingress Controller from Installed Operators selecting all default options (do not change ServiceType to LoadBalancer).

Switch to the openshift-operators namespace:

	oc project openshift-operators

Verify all NGINX resources are healthy:

	oc get all,nginxingresscontroller

Edit the Service resource fronting the NGINX Ingress Controller (service/my-nginx-ingress-controller) and add the following to the metadata:

	annotations:
    	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
    	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
    	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"

Edit the NGINX Ingress Controller resource (nginxingresscontroller.k8s.nginx.org/my-nginx-ingress-controller) and add the following to the spec:

	configMapData:
	  proxy-protocol: "True"
	  real-ip-header: "proxy_protocol"
	  set-real-ip-from: "0.0.0.0/0"

Also change the value of serviceType from NodePort to LoadBalancer. Save these changes and then wait for a public NLB to be created (later this will be changed to a private NLB):

	oc get all

Verify the status of provisioning the NLB from within the AWS console. This will take 2-3 minutes to complete. Also verify that there are two listeners configured (TCP:80 and TCP:443).

Click on the forwarding rule for the first listener and edit it's attributes. Enable proxy protocol v2 and save. Repeat this step for the next forwarding rule.

To test that everything is working so far return to the CLI and obtain the IP address of the NLB. Add an entry to the local /etc/hosts file resolving the hostname echo.example.com to the public IP address of the NLB.

Curl the endpoint:

	curl -Lkv http://echo.example.com/echo

It is expected that in the Headers Received identifies the real IP address of the caller (x-forwarded-for and x-real-ip). For example:

	CLIENT VALUES:
	client_address=10.128.2.32
	command=GET
	real path=/echo
	query=nil
	request_version=1.1
	request_uri=http://echo.example.com:8080/echo

	SERVER VALUES:
	server_version=nginx: 1.10.0 - lua: 10001

	HEADERS RECEIVED:
	accept=*/*
	connection=close
	host=echo.example.com
	user-agent=curl/7.61.1
	x-forwarded-for=118.200.48.201
	x-forwarded-host=echo.example.com
	x-forwarded-port=443
	x-forwarded-proto=https
	x-real-ip=118.200.48.201
	BODY:
	* Connection #0 to host echo.example.com left intact
	-no body in request-

The test can also be repeated from a web browser enabling inspection of the X.509 certificate.

Change the NLB from public-facing to private-facing. To do first revert the serviceType to NodePort for the NGINX Ingress Controller resource and verify that the existing load balancer is removed from within OpenShift and AWS. If for some reason the load balancer is not removed from AWS then delete it manually (check for tags kubernetes.io/service-name=openshift-ingress/router-default).
     
Editing the Service resource fronting the NGINX Ingress Controller (service/my-nginx-ingress-controller) ensure the annotations now includes the following:

	annotations:
	  service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
	  service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "tcp"
	  service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"
	  service.beta.kubernetes.io/aws-load-balancer-internal: "true"
	
To make this effective edit the the NGINX Ingress Controller and change the serviceType from NodePort to LoadBalancer. Verify an internal-facing NLB is created in AWS (check for scheme=internal in the description section). Edit the attributes of each listener to enable proxy protocol v2. 

Verify that internal access to the NLB works. Spin up a pod using a distribution that includes curl. And then use the IP address of the NLB to test. For example:

	oc run fedora -it --image=fedora
	curl -Lk http://echo.example.com/echo  --resolve echo.example.com:80:${NLB_IP} --resolve echo.example.com:${NLB_IP}

This time the x-forwarded-for and x-real-ip should match the IP address of the node that the pod is running on.

To create an API Gateway private integration go to API Gateway from the AWS console and first create a VPC link. Select VPC link for HTTP APIs. Enter any name and select the VPC hosting the ROSA cluster. Select all of the VPC subnets presented but do not select any security group.

Whilst the VPC link is being created start building the API Gateway instance. Select HTTP API as the type and give it a name and go directly to review and create (other bits will be added post-buid).

From the develop drop-down select routes. For the method select ANY and change the path /{proxy+}. After the route is created, select it and then the attach integration option. Select create and attach integration and the route displayed should be ANY /{proxy+}. For the integration target choose private resource. For the integration details select ALB/NLB and choose the ARN corresponding to the internal-facing NLB created earlier. Choose TCP 80 for the listener and the name of the VPC link that was created in the previous step.

From the integrations drop-down select manage integration. Select create parameter mapping and set the mapping type to all incoming requests and set parameter to modify to header.Host with an overwrite value of echo.example.com.

Select the URL of the API with /echo for the endpoint. You should see the echoserver respond. If not then edit the echoserver ingress resource and remove TLS entries for now (TBD).







