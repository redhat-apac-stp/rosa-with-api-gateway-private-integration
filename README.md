ROSA with API Gateway private integration

Follow https://mobb.ninja/docs/rosa/sts/ for guidance on installing ROSA.

As the cluster admin create a new project for hosting the echoserver service:

	oc new-project my-projects

Generate a self-signed certificate and private key:

	openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout echo.example.com.key -out echo.example.com.crt -subj "/CN=echo.example.com"
  
Create the TLS secret:

	oc create secret tls echo-secret --cert=hello.example.com.crt --key=hello.example.com.key

Install the echoserver resources:

	oc apply -f echoserver.yaml
  
Apply the anyuid SCC to the service account:

	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid
  
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







