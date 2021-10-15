# ROSA integration with AWS API Gateway

These instructions describe how to configure secure end-to-end connectivity from AWS API Gateway to a backend service accessed via an NGINX Ingress Controller using SSL/TLS certificates.

AWS API Gateway supports private integrations via a VPC link that terminates on NLB/ALB endpoints - CLB endpoints are not supported for termination. Thus the default ROSA OpenShift Router which deploys a CLB cannot be used for accessing applications running on ROSA via the AWS API Gateway. The ROSA public roadmap indicates that support for NLB is planned and the need for deploying NGINX Ingress Controller to accomplish what is described below can subsequently be reviewed. 

https://github.com/openshift-cs/managed-openshift/projects/2

The instructions below first deploy a non-secured (HTTP) private integration to verify the overall setup and facilite troubleshooting. Subsequently this is upgraded to a secured (HTTPS) private integration using a public X509 certificate that must be issued by one of the Certificate Authorities that API Gateway trusts as per the following link:

https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-supported-certificate-authorities-for-http-endpoints.html

For the purpose of this setup LetsEncrypt production certificates were used (staging certificates from LetsEncrypt are not a supported by AWS API Gateway).

A public ROSA STS cluster can be deployed as per the following instructions:

https://mobb.ninja/docs/rosa/sts/

NGINX Ingress Controllers can be installed by the NGINX Ingress Operator (v0.4.0) via the OperatorHub web console in OpenShift as per the following instructions:

https://github.com/nginxinc/nginx-ingress-operator/blob/master/docs/openshift-installation.md

After installing the operator create an instance of the NGINX Ingress Controller fronted by a load balancer in a new namespace (e.g., my-nginx-ingress):

	apiVersion: k8s.nginx.org/v1alpha1
	kind: NginxIngressController
	metadata:
	  name: my-nginx-ingress-controller
	  namespace: my-nginx-ingress
	spec:
  	  type: deployment
	  nginxPlus: false
	  image:
	    repository: nginx/nginx-ingress
	    tag: edge-ubi
	    pullPolicy: Always
	  replicas: 1
	  serviceType: LoadBalancer

Note that the load balancer type provisioned by the NGINX Ingress Controller defaults to CLB. Later, this will be switched to a NLB for integration with the AWS API Gateway.

Cert Manager can be installed using by the Cert Manager Operator (v1.5.4) via the OperatorHub web console in OpenShift as per the following instructions:

https://cert-manager.io/docs/installation/operator-lifecycle-manager/

Before creating issuers and certificates first provision an "A" record in AWS Route 53 pointing to the IP address of the Internet-facing load balancer provisioned by the NGINX Ingress Controller. This name is used by LetsEncrypt when issuing the HTTP01 challenge to validate domain ownership. For this setup www.example.com is the FQDN to be used for hosts in the TLS section of the ingress resource created later, as well as the CommonName (CN) for the certificate subject. This name must be changed to a registered domain name that your organisation owns in order for any of this to work.

	elb=`oc get svc -n openshift-operators | grep 'nginx-ingress-controller' | awk '{print $4}`
	host $elb

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

Create a service account for the application and apply the anyuid SCC policy:

	oc create sa sa-with-anyuid -n my-projects
	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid -n my-projects

Create the application deployment:

	apiVersion: apps/v1
	kind: Deployment
	metadata:
	  name: echoserver
	  namespace: my-projects
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
	  namespace: my-projects
	spec:
	  ports:
	    - port: 80
	      targetPort: 8080
	      protocol: TCP
	  type: ClusterIP
	  selector:
	    app: echoserver

Verify the readiness of all resources:

	oc get all -n my-projects

Configure an ingress resource exposing an HTTP route to be managed by the NGINX Ingress Controller:

	apiVersion: networking.k8s.io/v1
	kind: Ingress
	metadata:
	  name: echoserver
	  namespace: my-projects
	spec:
  	ingressClassName: nginx
  	rules:
  	- host: www.example.com
	    http:
	      paths:
	      - backend:
	          service:
	            name: echoserver
	            port:
	              number: 80
	        path: /
	        pathType: Prefix

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







