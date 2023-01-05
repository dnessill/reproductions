# nginx-http
This pipeline creates two **HTTP NGINX Servers** exposed by Service Mesh using an OpenShift Route.

### Network Flow

<img src="https://github.com/dnessill/reproductions/blob/main/nginx-http/nginx-http.png" width="80%" height="80%">
<br/>
<br/>


### Prerequisite
1. Installing [Red Hat Service Mesh](https://docs.openshift.com/container-platform/latest/service_mesh/v2x/installing-ossm.html).
2. Installing [OpenShift ArgoCD](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html).
3. Adding the **nginx-http** project to the Service Mesh Member Role ([Solution 4683861](https://access.redhat.com/solutions/4683861)).
<br/>
<br/>


### How to create the NGINX ArgoCD Application
1. Download AppProject and Application YAML:
   - reproductions/nginx-http/argocd/project.yaml
   - reproductions/nginx-http/argocd/application.yaml
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
oc -n istio-system get route
NAME                                        HOST/PORT                                                                                              PATH   SERVICES               PORT          TERMINATION          WILDCARD
nginx-http-nginx-gateway-525eca1d5089dbdc   nginx-http-nginx-gateway-525eca1d5089dbdc-istio-system.apps.dnessill-411-sdn.sandbox2574.opentlc.com          istio-ingressgateway   http2                              None
~~~

2. Server NGINX-1
~~~
curl nginx-http-nginx-gateway-525eca1d5089dbdc-istio-system.apps.dnessill-411-sdn.sandbox2574.opentlc.com/nginx-1.html
<html>
<h1>Welcome</h1>
</br>
<h1>Hi! This is HTTP Server 1 </h1>
</html>
~~~

3. Server NGINX-2
~~~
curl nginx-http-nginx-gateway-525eca1d5089dbdc-istio-system.apps.dnessill-411-sdn.sandbox2574.opentlc.comhostname/nginx-2.html
<html>
<h1>Welcome</h1>
</br>
<h1>Hi! This is HTTP Server 2 </h1>
</html>
~~~
