# ldap-identity-provider
This reproduction mean to be a simulation of an LDAP Server used as Identiy Provider for validating access users. In addtional, the LDAP Server stores users and groups so that OpenShift Container Platform can sync those LDAP records with internal OpenShift Container Platform records, enabling you to manage your groups in one place.

This reproduction is composed by the following 3 different GitOps Applications:
   - A Server OpenLDAP pod used as Identity Provider.
   - A Client phpLDAPadmin pod used as tool connected to the LDAP Server.
   - Ldap-sync configuration used to sync users and groups.

In this reproduction the LDAP Server is deployed inside the OpenShift Cluster so that its installation can be automated with ArgoCD.
This means that all the communication between the OCP Cluster and the LDAP Server is internal to the Cluster.


### Network Flow

<img src="https://github.com/dnessill/reproductions/blob/main/nginx-http-ossm-tls-gateway-termination/nginx-http-ossm-tls.png" width="80%" height="80%">
<br/>
<br/>


### Prerequisite
1. Installing [OpenShift ArgoCD](https://docs.openshift.com/container-platform/latest/cicd/gitops/installing-openshift-gitops.html).
2. Understand the [Identity Provider](https://docs.openshift.com/container-platform/4.11/authentication/understanding-identity-provider.html).
3. All the informations about the LDAP Server image and the users configurations hardcoded inside it, are reported at the end of this document.
<br/>
<br/>


### How to set up the LDAP Identity Provider using ArgoCD Application
1. Download AppProject and Application YAML:
   - reproductions/ldap-identity-provider/argocd/project.yaml
   - reproductions/ldap-identity-provider/argocd/application-ldap-sync.yaml
   - reproductions/ldap-identity-provider/argocd/application-openldap.yaml
   - reproductions/ldap-identity-provider/argocd/application-phpldapadmin.yaml
<br/>

2. Configure the LDAP Identity Provider updating:
   - Create a Secret object that contains the bindPassword field:
     ~~~
     oc create secret generic ldap-secret --from-literal=bindPassword=anypassword -n openshift-config
     ~~~
   - Patch the `oauth` CR adding the LDAP Server information:
     ~~~
     oc patch oauth cluster -p='{"spec": {"identityProviders": [{"ldap":{"attributes":{"email":["mail"],"id":["dn"],"name":["cn"],"preferredUsername":["cn"]},"bindDN": "cn=admin,dc=example,dc=com","bindPassword": {"name": "ldap-secret"},"insecure": true,"url": "ldap://openldap.openldap.svc.cluster.local/ou=users,dc=example,dc=com?cn"},"mappingMethod": "claim","name": "ldapidp","type": "LDAP"}]}}' --type=merge
     ~~~
   **Note**:
   Wait for the pods restart in the **openshift-authentication** project.

3. Create the AppProject:
~~~
oc apply -f project.yaml
~~~
<br/>

4. Create al the resources:
~~~
oc apply -f application-ldap-sync.yaml -f application-openldap.yaml -f application-phpldapadmin.yaml
~~~
<br/>
<br/>


### Verify the new Identity Provider access:
1. Log in the Cluster as **admin_gh** user (all the user passwords are **password**):
   ~~~
   oc login -u admin_gh -p password https://<your_api_url>:6443
   Login successful.
   
   You have access to 71 projects, the list has been suppressed. You can list all projects with 'oc projects'
   
   Using project "default".
   ~~~

2. Check synchronized groups and users:
   ~~~
   $ oc get groups
   NAME         USERS
   Admins       admin_gh
   Maintaners   maintainer, developer
   ~~~


### Access the phpLDAPadmin web app:

1. Gather the **phpLDAPadmin** URL:
   ~~~
   oc get route -n phpldapadmin
   ~~~

2. Connect to the web app using your browser:<br/>
   Login DN: **cn=admin,dc=example,dc=com**
   Password: **anypassword**
<img src="https://github.com/dnessill/reproductions/blob/main/ldap-identity-provider/phpldapadmin.png" width="80%" height="80%">
<br/>
<br/>


### Ldap Server details:
The Ldap Server image used in this reproduction is generated starting from the **osixia/openldap** image adding the following users and groups:
- Users:
  - admin_gh
  - maintainer
  - developer

- Groups:
  - Admins
  - Maintaners

The bootstrap ldif file used to hardcoded users and groups is the following:
~~~
dn: ou=Users,dc=example,dc=com
changetype: add
objectclass: organizationalUnit
ou: Users
 
dn: cn=developer,ou=Users,dc=example,dc=com
changetype: add
objectclass: inetOrgPerson
cn: developer
givenname: developer
sn: Developer
displayname: Developer User
mail: developer@gmail.com
userpassword: password
 
dn: cn=maintainer,ou=Users,dc=example,dc=com
changetype: add
objectclass: inetOrgPerson
cn: maintainer
givenname: maintainer
sn: Maintainer
displayname: Maintainer User
mail: maintainer@gmail.com
userpassword: password
 
dn: cn=admin_gh,ou=Users,dc=example,dc=com
changetype: add
objectclass: inetOrgPerson
cn: admin_gh
givenname: admin_gh
sn: AdminGithub
displayname: Admin Github User
mail: admin_gh@gmail.com
userpassword: password
 
dn: ou=Groups,dc=example,dc=com
changetype: add
objectclass: organizationalUnit
ou: Groups
 
dn: cn=Admins,ou=Groups,dc=example,dc=com
changetype: add
cn: Admins
objectclass: groupOfUniqueNames
uniqueMember: cn=admin_gh,ou=Users,dc=example,dc=com
 
dn: cn=Maintaners,ou=Groups,dc=example,dc=com
changetype: add
cn: Maintaners
objectclass: groupOfUniqueNames
uniqueMember: cn=maintainer,ou=Users,dc=example,dc=com
uniqueMember: cn=developer,ou=Users,dc=example,dc=com
~~~

The Dockerfile used to generate the image is the following:
~~~
FROM osixia/openldap

ENV LDAP_ORGANISATION="Test Org" \     
LDAP_DOMAIN="example.com"

COPY bootstrap.ldif /container/service/slapd/assets/config/bootstrap/ldif/50-bootstrap.ldif
~~~

The command used to build the image is the following:
~~~
podman build -t quay.io/dnessill/my-openldap -f Dockerfile
~~~
