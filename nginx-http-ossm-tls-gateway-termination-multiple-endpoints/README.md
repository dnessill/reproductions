# nginx-http-ossm
This pipeline creates a **HTTP NGINX Server** behind Red Hat Service Mesh Ingress Gateway with TLS termination.

### Network Flow

<img src="https://github.com/dnessill/reproductions/blob/main/nginx-http-ossm-tls-gateway-termination/nginx-http-ossm-tls.png" width="80%" height="80%">
<br/>
<br/>


### Prerequisite
1. Installing [Red Hat Service Mesh](https://docs.openshift.com/container-platform/latest/service_mesh/v2x/installing-ossm.html).
2. Installing [OpenShift ArgoCD](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html).
3. Adding the **nginx-http-ossm-tls** project to the Service Mesh Member Role ([Solution 4683861](https://access.redhat.com/solutions/4683861)).
<br/>
<br/>


### How to create the nginx-http-ossm ArgoCD Application
1. Download AppProject and Application YAML:
   - reproductions/nginx-http-ossm/argocd/project.yaml
   - reproductions/nginx-http-ossm/argocd/application.yaml
<br/>

2. Create the AppProject:
~~~
$ oc apply -f project.yaml
~~~
<br/>

3. Create the Application:
~~~
$ oc apply -f application.yaml
~~~
<br/>
<br/>


### How to reach the NGINX Servers:
1. Print out the OCP Route:
~~~
$ oc -n istio-system get route
NAME                                             HOST/PORT                                                                                                   PATH   SERVICES               PORT    TERMINATION   WILDCARD
nginx-http-ossm-tls-nginx-gateway-525eca1d5089dbdc   nginx-http-ossm-tls-nginx-gateway-525eca1d5089dbdc-istio-system.apps.dnessill-411-sdn.sandbox2574.opentlc.com          istio-ingressgateway   https         passthrough          None
~~~
   **Note**:
   Starting from Red Hat OpenShift Service Mesh 2.5, automatic route creation, also known as Istio OpenShift Routing (IOR), is a deprecated feature that is disabled by default.
   The route can be created manually with the following command:
   ~~~
   $ oc -n istio-system create route passthrough nginx-passthrough-route --service=istio-ingressgateway --port=https --hostname <hostname>
   ~~~

2. Store the Root CA certificate in the ca.crt file:
~~~
$ oc get cm ca-crt -n nginx-http-ossm-tls -ojsonpath='{.data.ca\.crt}' > ca.crt
~~~

3. Server NGINX-1
~~~
$ curl --cacert ca.crt https://nginx-http-ossm-tls-nginx-gateway-525eca1d5089dbdc-istio-system.apps.dnessill-411-sdn.sandbox2574.opentlc.com
<html>
<h1>Welcome</h1>
<h1>Hi! This is HTTP Server </h1>
</html>
~~~
<br/>
<br/>


### Datails of the deployment:
Since this Application uses a TLS termination on the Ingress Gateway, CA and Server certificates are required.<br/>
To avoid a manual creation of the certificates, an Init Container inside the Pod executes the following actions:
1. Generate the self-signed Root CA certificate.
2. Store the CA certificate in the configmap **ca-crt** so that it can be used in the curl command.
3. Generate and sign the Ingress Gateway certificates using the Root CA and then create the secret **creds-wildcard**.
