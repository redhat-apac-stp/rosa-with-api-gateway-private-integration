ROSA with API Gateway private integration

Follow https://mobb.ninja/docs/rosa/sts/ for installing ROSA and creating a cluster-admin user.

Login to the cluster using the cluster-admin account.

Create a new project for hosting the echoserver service:

	oc new-project my-projects

Generate a self-signed certificate and private key:

	openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout echo.example.com.key -out echo.example.com.crt -subj "/CN=echo.example.com"
  
Create the TLS secret:

	oc create secret tls echo-secret --cert=hello.example.com.crt --key=hello.example.com.key

Install the echoserver resources:

	oc apply -f echoserver.yaml
  
Apply the anyuid SCC to the service account:

	oc adm policy add-scc-to-user anyuid -z sa-with-anyuid
  
Install the NGINX Ingress Operator (0.4.0) from OperatorHub selecting all default options

