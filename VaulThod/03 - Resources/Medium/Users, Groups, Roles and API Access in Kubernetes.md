---
tags:
  - RBAC
  - KUBERNETES
source: https://adityasamant.medium.com/users-groups-roles-and-api-access-in-kubernetes-10216cfab335
---




# Users, Groups, Roles and API Access in Kubernetes

![](https://miro.medium.com/v2/resize:fit:700/1*9UloDkRmhthvaKhG3uKu8Q.png) 
This blog post describes the nuances of how  `users`  and  `groups`  are configured in Kubernetes and how the  `role-based access control`  (RBAC) mechanism applies for them.
We will also dive into the usage of the  `kubectl`  command line tool to check API access in Kubernetes. It especially focuses on the difference between the  `--user`  and  ``  options in  `kubectl` 
 *This is a hands-on article. You may choose to follow along with the commands to gain a deeper understanding.* 


# Prerequisites

You need to have a Kubernetes cluster, and the  ` [kubectl](https://kubernetes.io/docs/tasks/tools/#kubectl) `  command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one using  ` [minikube](https://minikube.sigs.k8s.io/docs/start/) ` 
 ` [OpenSSL](https://www.openssl.org/source/) `  command line utility will be used to view the x509 certificates.


# Role-based access control

 `Role-based access control (RBAC)`  is a method of regulating access to computer or network resources based on the roles of individual users within your organisation.\
You may choose to go through the  ` [Kubernetes RBAC documentation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) `  before reading ahead.


# Checking API Access

 `kubectl auth can-i`  command can be used to determine whether a user has permissions to execute a certain action.


## Scenario for the Default Admin User

In the first example, we will work with the default admin user.
Check the contexts available in the  `minikube`  cluster:

```
kubectl config get-contexts minikube
```


You should see an output similar to:

```
CURRENT   NAME       CLUSTER    AUTHINFO   NAMESPACE
*         minikube   minikube   minikube   default
```


Check the name of the admin user that is created with our basic  `minikube`  installation:

```
kubectl config view -o jsonpath='{.contexts[?(@.name=="minikube")].context.user}'
```


You should see an output similar to:

```
minikube
```


 ** *There is no entity or resource named ‘User’ in Kubernetes. A user in Kubernetes is nothing but a key and certificate pair issued by the Kubernetes cluster and presented to the Kubernetes API.* ** 
Check the certificate of the  `minikube`  user:

```
kubectl config view -o jsonpath='{.users[?(@.name=="minikube")].user.client-certificate}'
```


You should see an output similar to:

```
/Users/adityasamant/.minikube/profiles/minikube/client.crt
```


View the Subject of this certificate (use the path generated in the previous command):

```
openssl x509 -in /Users/adityasamant/.minikube/profiles/minikube/client.crt -text -noout | grep Subject | grep -v "Public Key Info"
```


You should see an output similar to:

```
Subject: O=system:masters, CN=minikube-user
```


 ``  is the name of the user and  ``  is the group that this user will belong to. As can be seen above, the  `minikube`  admin user (marked by  `` ) is part of the  `system:masters`  group (marked by  `` 
 ` ** *system:masters* ** `  ** * is a group which is hardcoded into the Kubernetes API server source code as having unrestricted rights to the Kubernetes API server. Any user who is a member of this group has full cluster-admin rights to the cluster. Even if every cluster role and role is deleted from the cluster, users who are members of this group retain full access to the cluster.* ** 
Use the  `kubectl auth can-i`  command to verify a few scenarios.
Check permissions to create pods:

```
kubectl auth can-i create pods
```



```
yes
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments
```



```
yes
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets
```



```
yes
```




## Scenario for a Normal User

Configure a normal user and verify how the  `kubectl auth can-i`  commands can be used to check the access. To do this we need to  [issue a certificate](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)  for the user.
Create a private key and a csr file:

```
openssl genrsa -out jane.key 2048
openssl req -new -key jane.key -out jane.csr -subj "/CN=jane"
```


This will generate a private key named  `jane.key`  and a certificate signing request named  `jane.csr` 
Get the base64 encoded value of the CSR file content:

```
cat jane.csr | base64 | tr -d "\n"
```


Create a CertificateSigningRequest

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: <base64 encoded csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```


Get the CSR:

```
kubectl get csr jane
```



```
NAME   AGE   SIGNERNAME                            REQUESTOR       REQUESTEDDURATION   CONDITION
jane   55s   kubernetes.io/kube-apiserver-client   minikube-user   24h                 Pending
```


Approve the CSR:

```
kubectl certificate approve jane
```



```
kubectl get csr jane
```



```
NAME   AGE     SIGNERNAME                            REQUESTOR       REQUESTEDDURATION   CONDITION
jane   2m17s   kubernetes.io/kube-apiserver-client   minikube-user   24h                 Approved,Issued
```




## Granting permissions via RBAC

Create a  `clusterrole`  granting permissions to only create pods:

```
kubectl create clusterrole createpods --verb=create --resource=pods
```



```
clusterrole.rbac.authorization.k8s.io/createpods created
```


Create a  `clusterrolebinding`  to bind the  `clusterrole`  with user  `jane:` 

```
kubectl create clusterrolebinding createpods --clusterrole=createpods --user=jane
```



```
clusterrolebinding.rbac.authorization.k8s.io/createpods created
```




## Difference between  ``  and ‘user’ options of kubectl

 `kubectl`  has a number of global  `options`  that can be passed as an argument to any  `kubectl`  command. The list can be found with the following command:

```
kubectl options
```


Two of the options are  `--user`  and  `` . It is important to understand the difference between them.

```
--user='':
The name of the kubeconfig user to use
```



```
--as='':
Username to impersonate for the operation. User could be a regular user or a service account in a namespace.
```


 **  ` *--user* `  * option is used when you want to trigger the *  ` *kubectl* `  * command under the context of a user which is configured in the *  ` *kubeconfig* `  * file. This option throws an error if the user is not present in the *  ` *kubeconfig* `  * file.* 
 **  ` ** `  * option is used to impersonate any *  ` ** `  **  ` *serviceaccount* `  **  ` ** `  * can be used for a user irrespective of whether that user is present in the *  ` *kubeconfig* `  * file or not.* 
Let’s put the theory into action with the help of the user we created.


## Using ‘as’ to Check Access

Check permissions to create pods:

```
kubectl auth can-i create pods --as=jane
```



```
yes
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments --as=jane
```



```
no
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets --as=jane
```



```
no
```


Due to the fact that we explicitly assigned the  `clusterrole`  `createpods`  to user  `` , we see that  ``  has access to create pods, but no access to create deployments or delete secrets. Great, this is as expected.


## Using ‘user’ to Check Access

Try the same commands, but this time using the `--user`  option:

```
kubectl auth can-i create pods --user=jane
```



```
error: auth info "jane" does not exist
```


The command leads to an error. This is because we have not configured the user  ``  in the  `kubeconfig`  file.
We will fix this by adding the user  ``  to the  `kubeconfig`  file.


## Adding the user to kubeconfig

Get the user’s certificate:

```
kubectl get csr jane -o jsonpath='{.status.certificate}'| base64 -d > jane.crt
```


Add the new credentials to kubeconfig:

```
kubectl config set-credentials jane --client-key=jane.key --client-certificate=jane.crt --embed-certs=true
```



```
User "jane" set.
```


Add the context to kubeconfig:

```
kubectl config set-context jane --cluster=minikube --user=jane
```



```
Context "jane" created.
```


Let’s try the  `kubectl auth can-i`  command once again to verify the permissions on user jane:
Check permissions to create pods:

```
kubectl auth can-i create pods --user=jane
```



```
yes
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments --user=jane
```



```
no
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets --user=jane
```



```
no
```


Now everything works as expected.
 ** ** **  ` ** ** ** `  ** * option does not check the actual presence of the user in the * **  ` ** *kubeconfig* ** `  ** *. It only checks the explicitly configured roles and bindings that are bound to a user and returns a response based on that.* ** 


## Example for a non-existent user

If the  ``  option is used for a non-existent user, there is no error thrown as shown below.

```
kubectl auth can-i create pods --as=nobody
```



```
no
```




## Scenario for a Custom Admin User

Configure a new admin user and verify the behaviour of the  `kubectl auth can-i`  commands.\
This time we will check the permissions that a  ``  inherits via the  `group`  it is attached to.\
 [Issue a certificate](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)  for the new admin user.
Create a private key and a csr file:

```
openssl genrsa -out poweruser.key 2048
openssl req -new -key poweruser.key -out poweruser.csr -subj "/CN=poweruser/O=system:masters"
```


 ``  is the name of the user and  ``  is the group that this user will belong to.
Create a CertificateSigningRequest:

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: poweruser
spec:
  request: <base64 encoded csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```


The above command throws an error as the  `kube-apiserver`  blocks any  `CertificateSigningRequest`  that attempts to add a user as part of the  `system:masters`  group.
 *Error from server (Forbidden): error when creating “STDIN”: certificatesigningrequests.certificates.k8s.io “poweruser” is forbidden: use of kubernetes.io/kube-apiserver-client signer with system:masters group is not allowed* 


## Granting permissions via RBAC through groups

In order to create a new admin user we will create a custom admin group that replicates the behaviour of the  `system:masters`  group. Let’s call it  `example:masters` 
To do this, create a new  `clusterrolebinding`  as below:

```
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: example-cluster-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: example:masters
EOF
```


The next step is to add the new admin user to the  `example:masters`  group.
Delete the previous files created for  `poweruser` 

```
rm poweruser.key poweruser.csr
```


Create a new private key and a csr file:

```
openssl genrsa -out poweruser.key 2048
openssl req -new -key poweruser.key -out poweruser.csr -subj "/CN=poweruser/O=example:masters"
```


Create a CertificateSigningRequest:

```
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: poweruser
spec:
  request: <base64 encoded csr>
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 86400  # one day
  usages:
  - client auth
EOF
```


Approve the CSR:

```
kubectl certificate approve poweruser
```



```
certificatesigningrequest.certificates.k8s.io/poweruser approved
```


Get the certificate:

```
kubectl get csr poweruser -o jsonpath='{.status.certificate}'| base64 -d > poweruser.crt
```


View the Subject of this certificate:

```
openssl x509 -in poweruser.crt -text -noout | grep Subject | grep -v "Public Key Info"
```



```
Subject: O=example:masters, CN=poweruser
```


The above output shows that the user  `poweruser`  belongs to the  `example:masters`  group.
Add the new admin user to  `kubeconfig` 
Add the new credentials:

```
kubectl config set-credentials poweruser --client-key=poweruser.key --client-certificate=poweruser.crt --embed-certs=true
```



```
User "poweruser" set.
```


Add the context:

```
kubectl config set-context poweruser --cluster=minikube --user=poweruser
```



```
Context "poweruser" created.
```


Try the  `kubectl auth can-i`  command to verify the permissions on the new admin user:
Check permissions to create pods:

```
kubectl auth can-i create pods --user=poweruser
```



```
yes
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments --user=poweruser
```



```
yes
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets --user=poweruser
```



```
yes
```


Try the same commands but with using the  ``  option.
Check permissions to create pods:

```
kubectl auth can-i create pods --as=poweruser
```



```
no
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments --as=poweruser
```



```
no
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets --as=poweruser
```



```
no
```


 **Strange!!!**  The expected output was  ``  for all 3 commands, as  `poweruser`  is an admin user with full access to the cluster.
We can prove this as follows:
Switch the context to work with the  `poweruser` 

```
kubectl config use-context poweruser
```


Create a pod:

```
kubectl run nginx --image=nginx
```



```
pod/nginx created
```


Create the deployment:

```
kubectl create deployment nginx-deploy --image=nginx
```



```
deployment.apps/nginx-deploy created
```


Create and delete a secret:

```
kubectl create secret generic test-secret --from-literal=secret=1234
```



```
secret/test-secret created
```



```
kubectl delete secrets test-secret
```



```
secret "test-secret" deleted
```


My first thought was that this is a defect in Kubernetes.\
I raised issue  [#122579](https://github.com/kubernetes/kubernetes/issues/122579)  to Kubernetes for confirmation.
The reality is that the behaviour is as-expected. The API server has no knowledge of group membership apart from what is encoded directly in the credential or provided by a token webhook. To overcome this you need to pass the group you want to impersonate with the  `--as-group`  flag.


## The ‘as-group’ option of kubectl

Try the same commands but this time append the  `--as-group`  option as well.
Check permissions to create pods:

```
kubectl auth can-i create pods --as=poweruser --as-group=example:masters
```



```
yes
```


Check permissions to create deployments:

```
kubectl auth can-i create deployments --as=poweruser --as-group=example:masters
```



```
yes
```


Check permissions to delete secrets:

```
kubectl auth can-i delete secrets --as=poweruser --as-group=example:masters
```



```
yes
```


Now the results are as expected.


# Summary

 `kubectl auth can-i`  command behaves differently for the  `--user`  and  ``  options.
 `--user`  option checks for the actual presence of the user in the  `kubeconfig`  file and has the ability to check permissions derived from the group of the user.
 ``  option can be used to check permissions for any user irrespective of its presence in the  `kubeconfig`  file. It checks permissions which are  ** *directly bound* **  to the user through RBAC, and does not check permissions that are derived from the user’s group. The API server has no knowledge of group membership apart from whatever is encoded directly in the credential or provided by a token webhook.
 `--as-group`  option should be used to check for permissions that are derived from the user’s group.


# Cleaning up

Delete the resources created during this lab:

```
rm jane*
rm poweruser*
kubectl delete pod nginx
kubectl delete deployment nginx-deploy
```


Optionally, you can delete the entire  `minikube`  cluster:

```
minikube delete --all
```

