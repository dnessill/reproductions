# ldap-identity-provider
This pipeline creates the following two objects:
   - A Server OpenLDAP pod used uas Identity Provider.
   - A Client phpLDAPadmin pod used as tool connected to the LDAP Server.

In this reproduction the LDAP Server is deployed inside the OpenShift Cluster so that its installation can be automated with ArgoCD.
This means that all the communication between the OCP Cluster and the LDAP Server is internal to the Cluster.


### Network Flow

<img src="https://github.com/dnessill/reproductions/blob/main/nginx-http-ossm-tls-gateway-termination/nginx-http-ossm-tls.png" width="80%" height="80%">
<br/>
<br/>


### Prerequisite
1. Installing [OpenShift ArgoCD](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html).
2. Understand the [Identity Provider](https://docs.openshift.com/container-platform/4.11/authentication/understanding-identity-provider.html).
3. All the informations about the LDAP Server image and the users configurations hardcoded inside it are reported at the end of this document.
<br/>
<br/>


### How to set up the LDAP Identity Provider using ArgoCD Application
1. Download AppProject and Application YAML:
   - reproductions/ldap-identity-provider/argocd/project.yaml
   - reproductions/ldap-identity-provider/argocd/application.yaml
<br/>

2. Configure the LDAP Identity Provider updating:
   - Create a secret that stores the LDAP Server password:
     ~~~
     $ oc create secret generic ldap-secret --from-literal=bindPassword=anypassword -n ldap-sync
     ~~~
   - Patch the `oauth` CR adding the LDAP Server information:
     ~~~
     $ oc patch oauth cluster -p='{"spec": {"identityProviders": [{"ldap":{"attributes":{"email":["mail"],"id":["dn"],"name":["cn"],"preferredUsername":["cn"]},"bindDN": "cn=admin,dc=example,dc=com","bindPassword": {"name": "ldap-secret"},"insecure": true,"url": "ldap://openldap.openldap.svc.cluster.local/ou=users,dc=example,dc=com?cn"},"mappingMethod": "claim","name": "ldapidp","type": "LDAP"}]}}' --type=merge
     ~~~
   **Note**:
   Wait for the pods restart in the **openshift-authentication** project.

3. Create the AppProject:
~~~
$ oc apply -f project.yaml
~~~
<br/>

4. Create the Application:
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
