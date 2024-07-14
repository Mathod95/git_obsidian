---
tags:
  - KUBERNETES/NAMESPACES
source: https://blog.devgenius.io/k8s-troubleshooting-namespace-stuck-in-terminating-state-b91ab6fa8948
postRelated: obsidian://open?vault=Vaulthod&file=_Publish%2FKubernetes%2FNamespaces%2FNamespaces
---
# Hierarchical Namespaces in Kubernetes

## Introduction & Hands-on Demonstration!

![](https://miro.medium.com/v2/resize:fit:700/1*te_im4i-6ayzK6OrhOI3zQ.png) 
 Kubernetes a  `Namespace`  is the most fundamental building block. It helps to organise & isolate resources within a cluster by creating a logical partitions. By separating resources into different namespaces, administrators can enforce security policies, limit resource consumption, and ensure a clean, organised environment.
In order to create a Namespace in Kubernetes, one could follow -
- Either the declarative way -


```
$ echo "apiVersion: v1
  kind: Namespace
  metadata:
      name: <insert-namespace-name-here>" > ns.yaml

$ kubectly apply -f ns.yaml
```


- Or the imperative way -


```
$ kubectl create namespace <insert-namespace-name-here>
```




# Limitations of Kubernetes Namespaces

Namespaces are indeed very useful — You may find yourself creating many namespaces per cluster. However, at scale, numerous namespaces might lead to management issues.
Consider these scenarios —
> 
 ** *Scenario #1 — * ** You might want many namespaces to have similar policies applied to them, such as to allow access by members of the same team. However, since Role Bindings operate at the level of individual namespaces, you will be forced to create such Role Bindings in each namespace individually, which can be tedious and error-prone. The same applies to other policies such as Network Policies and Limit Ranges.
 ** *Scenario #2 —* **  You might want to allow some teams to create namespaces themselves as isolation units for their own services. However, namespace creation is a privileged cluster-level operation, and you typically want to control this privilege very closely.

![](https://miro.medium.com/v2/resize:fit:121/1*JZ4d9Azbao_yw-sfdv4kVA.gif) 


#  **Hierarchical Namespaces** 

These are simple extension to Kubernetes namespaces that addresses some of the shortcomings of namespace we discussed above. It addresses these problems by allowing one to organise their namespaces into trees, ability to create new namespaces within those trees and allowing to apply policies to those trees (or their subtrees).
Something like below where each of these being namespaces -

```
acme-org
└── team-a
     └── service-1
```


By doing this it easy to manage groups of namespaces that share a common concept of ownership. They are especially useful in clusters that are shared by multiple teams, but the owners do not need to be people. For example, you might want to make an Operator an owner of a set of namespaces.
![](https://miro.medium.com/v2/resize:fit:121/1*pwoK921vEIcL7C78jdlpKw.gif) 


# Key Features of HNC

Some of the key features possible through HNC (Hierarchical Namespaces Controller) are -
-  **Namespace hierarchy — ** HNC allows the creation of parent-child relationships between namespaces, enabling a more structured approach to managing resources.
-  **Configuration propagation — ** With HNC, configurations and policies defined in a parent namespace are automatically propagated to its child namespaces.
-  **Access control — ** HNC simplifies the management of Role-Based Access Control (RBAC) in a hierarchical namespace setup, making it easy to enforce security policies across the hierarchy.



# Setting Up HNC

To use HNC, you must first install the HNC extension on your Kubernetes cluster.

```
#Installing the HNC extensions
$ kubectl apply -f https://github.com/kubernetes-sigs/hierarchical-namespaces/releases/download/v1.1.0-rc2/default.yaml

namespace/hnc-system created
customresourcedefinition.apiextensions.k8s.io/hierarchicalresourcequotas.hnc.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/hierarchyconfigurations.hnc.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/hncconfigurations.hnc.x-k8s.io created
customresourcedefinition.apiextensions.k8s.io/subnamespaceanchors.hnc.x-k8s.io created
role.rbac.authorization.k8s.io/hnc-leader-election-role created
clusterrole.rbac.authorization.k8s.io/hnc-admin-role created
clusterrole.rbac.authorization.k8s.io/hnc-manager-role created
rolebinding.rbac.authorization.k8s.io/hnc-leader-election-rolebinding created
clusterrolebinding.rbac.authorization.k8s.io/hnc-manager-rolebinding created
secret/hnc-webhook-server-cert created
service/hnc-controller-manager-metrics-service created
service/hnc-webhook-service created
deployment.apps/hnc-controller-manager created
mutatingwebhookconfiguration.admissionregistration.k8s.io/hnc-mutating-webhook-configuration created
validatingwebhookconfiguration.admissionregistration.k8s.io/hnc-validating-webhook-configuration created
```


If you don’t have krew installed already, install krew followed by next, installing the kubectl plugin for hns —

```
$ (
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)

+-zsh:590> mktemp -d
+-zsh:590> cd /var/folders/2l/fvc8dgl91yl9k6d3nxjhhykm0000gp/T/tmp.2LxeHmdC
+-zsh:591> OS=+-zsh:591> OS=+-zsh:591> tr '[:upper:]' '[:lower:]'
+-zsh:591> uname
+-zsh:591> OS=darwin
+-zsh:592> ARCH=+-zsh:592> uname -m
+-zsh:592> ARCH=+-zsh:592> sed -e s/x86_64/amd64/ -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/'
+-zsh:592> ARCH=amd64
+-zsh:593> KREW=krew-darwin_amd64
+-zsh:594> curl -fsSLO https://github.com/kubernetes-sigs/krew/releases/latest/download/krew-darwin_amd64.tar.gz
+-zsh:595> tar zxvf krew-darwin_amd64.tar.gz
x ./LICENSE
x ./krew-darwin_amd64
+-zsh:596> ./krew-darwin_amd64 install krew
Adding "default" plugin index from https://github.com/kubernetes-sigs/krew-index.git.
Updated the local copy of plugin index.
Installing plugin: krew
Installed plugin: krew
\
 | Use this plugin:
 |  kubectl krew
 | Documentation:
 |  https://krew.sigs.k8s.io/
 | Caveats:
 | \
 |  | krew is now installed! To start using kubectl plugins, you need to add
 |  | krew's installation directory to your PATH:
 |  |
 |  |   * macOS/Linux:
 |  |     - Add the following to your ~/.bashrc or ~/.zshrc:
 |  |         export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
 |  |     - Restart your shell.
 |  |
 |  |   * Windows: Add %USERPROFILE%\.krew\bin to your PATH environment variable
 |  |
 |  | To list krew commands and to get help, run:
 |  |   $ kubectl krew
 |  | For a full list of available plugins, run:
 |  |   $ kubectl krew search
 |  |
 |  | You can find documentation at
 |  |   https://krew.sigs.k8s.io/docs/user-guide/quickstart/.
 | /
/
```



```
$ kubectl krew update && kubectl krew install hns

Updated the local copy of plugin index.
Updated the local copy of plugin index.
Installing plugin: hns

Installed plugin: hns
\
 | Use this plugin:
 |  kubectl hns
 | Documentation:
 |  https://github.com/kubernetes-sigs/hierarchical-namespaces/tree/master/docs/user-guide
 | Caveats:
 | \
 |  | This plugin works best if you have the most recent minor version of HNC on
 |  | your cluster. Get the latest version of HNC, as well as prior versions of
 |  | this plugin, at:
 |  |
 |  |   https://github.com/kubernetes-sigs/hierarchical-namespaces
 |  |
 |  | Watch out for the following common misconceptions when using HNC:
 |  |
 |  | * Not all child namespaces are subnamespaces!
 |  | * Only RBAC Roles and RoleBindings are propagated by default, but you can configure more.
 |  |
 |  | The user guide contains much more information.
 | /
/
WARNING: You installed plugin "hns" from the krew-index plugin repository.
   These plugins are not audited for security by the Krew maintainers.
   Run them at your own risk.
```


![](https://miro.medium.com/v2/resize:fit:121/1*YSHKEA495RuljXr5CdSH8A.gif) 


# Demonstrations



## Setting parent-child relationships & RBAC propagation

Imagine you’ve an org called “ ** *acme-org”* ** . We’ll create a root namespace to represent it -

```
$ kubectl create namespace acme-org
namespace/acme-org created
```


Next, we will create a namespace for a team within that org and call it as “ ** *team-a* ** 

```
$ kubectl create namespace team-a
namespace/team-a created
```


and another namespace for a service owned by that team. By default, there’s no relationship between these namespaces.

```
$ kubectl create namespace service-1
namespace/service-1 created
```


Let’s say that we want to make someone Site Reliability Engineer (SRE) for team-a, so we’ll create an RBAC Role -

```
$ kubectl -n team-a create role team-a-sre --verb=update --resource=deployments
role.rbac.authorization.k8s.io/team-a-sre created
```


and a RoleBinding to that role -

```
$ kubectl -n team-a create rolebinding team-a-sres --role team-a-sre --serviceaccount=team-a:default
rolebinding.rbac.authorization.k8s.io/team-a-sres created 
```


Similarly, let’s say, we might want to have a super-SRE group across the whole org, so create roles and corresponding role binding for the same -

```
$ kubectl -n acme-org create role org-sre --verb=update --resource=deployments
role.rbac.authorization.k8s.io/org-sre created

$ kubectl -n acme-org create rolebinding org-sres --role org-sre --serviceaccount=acme-org:default
rolebinding.rbac.authorization.k8s.io/org-sres created
```


Obviously, none of this affects “service-1”, since that’s a completely independent namespace, and RBAC only applies at the namespace level -

```
$ kubectl -n service-1 get rolebindings
No resources found in service-1 namespace.
```


So this is where the HNC comes in. Let’s make “ **acme-org”**  the parent of “ **team-a”** 

```
$ kubectl hns set team-a --parent acme-org
Setting the parent of team-a to acme-org
Succesfully updated 1 property of the hierarchical configuration of team-a
```


and “ **team-a”**  the parent of “ **service-1 ”** 

```
kubectl hns set service-1 --parent team-a

Setting the parent of service-1 to team-a
Succesfully updated 1 property of the hierarchical configuration of service-1 
```


Let’s now display the hierarchy which ascertains things are as we desired -

```
$ kubectl hns tree acme-org

acme-org
└── team-a
    └── service-1
```


Now, if we check “ **service-1** ” again, we’ll see that the roles & rolebindings from the ancestor namespaces have been propagated to the child namespace -

```
$ kubectl -n service-1 describe roles
Name:         org-sre
Labels:       app.kubernetes.io/managed-by=hnc.x-k8s.io
              hnc.x-k8s.io/inherited-from=acme-org
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  deployments.apps  []                 []              [update]

Name:         team-a-sre
Labels:       app.kubernetes.io/managed-by=hnc.x-k8s.io
              hnc.x-k8s.io/inherited-from=team-a
Annotations:  <none>
PolicyRule:
  Resources         Non-Resource URLs  Resource Names  Verbs
  ---------         -----------------  --------------  -----
  deployments.apps  []                 []              [update]



$ kubectl -n service-1 get rolebindings
NAME          ROLE              AGE
org-sres      Role/org-sre      25s
team-a-sres   Role/team-a-sre   25s
```




## Hierarchical Modification & Subnamespace creation

Let’s create a new namespace called “ ** *team-b”* **  inside “ ** *acme-org”* ** 
This time, instead of creating the namespace ourselves, we’ll let the HNC create it for us. This is useful if you’re administering a subtree and don’t have cluster-wide permissions to create namespaces .

```
$ kubectl hns create team-b -n acme-org
Successfully created "team-b" subnamespace anchor in "acme-org" namespace
```


Following ascertains our desired setup -

```
$ kubectl hns tree acme-org
acme-org
├── team-a
│   └── service-1
└── [s] team-b

[s] indicates subnamespaces
```


Let’s say, “ ** *team-b”* **  is a little weird, they call their SRE’s “wizards” so we’ll set up their roles too -

```
$ kubectl -n team-b create role team-b-wizard --verb=update --resource=deployments
role.rbac.authorization.k8s.io/team-b-wizard created

$ kubectl -n team-b create rolebinding team-b-wizards --role team-b-wizard --serviceaccount=team-b:default
rolebinding.rbac.authorization.k8s.io/team-b-wizards created
```


Now, if we assign the service (service-1) to the new team (team-b) —

```
$ kubectl hns set service-1 --parent team-b
Changing the parent of service-1 from team-a to team-b
Succesfully updated 1 property of the hierarchical configuration of service-1
```


And we can verify that the roles and rolebindings have been updated as well -

```
$ kubectl -n service-1 get roles
NAME         CREATED AT
org-sre      2023-04-09T02:16:51Z
team-a-sre   2023-04-09T02:16:51Z


$ kubectl -n service-1 get rolebindings
NAME          ROLE              AGE
org-sres      Role/org-sre      84s
team-a-sres   Role/team-a-sre   84s
```




## Propagating different resources

HNC doesn’t only work for RBAC. Any Kubernetes resource can be configured to be propagated through the hierarchy, although by default, only RBAC objects are propagated.
let’s say that the workloads in “ ** *service-1”* **  expect a secret called my-creds which is different for each team.
Let’s create those creds in “ ** *team-b”* ** 

```
$ kubectl -n team-b create secret generic my-creds --from-literal=password=iamteamb
secret/my-creds created
```


If you check existing secrets in “ ** *service-1”* **  using the command below, you will find that the secret does not show up in service-1 because we haven’t configured HNC to propagate secrets in HNCConfiguration —

```
$ kubectl -n service-1 get secrets
NAME                  TYPE                                  DATA   AGE
default-token-7qvr8   kubernetes.io/service-account-token   3      40m
```


In order to get this to work, you need to update the HNCConfiguration object, which is a single cluster-wide configuration for HNC as a whole. To do this, simply use the config subcommand —

```
$ kubectl hns config set-resource secrets --mode Propagate
```


Now, we should be able to verify that my-creds was propagated to service-1 —

```
$ kubectl -n service-1 get secrets
NAME                  TYPE                                  DATA   AGE
default-token-7qvr8   kubernetes.io/service-account-token   3      41m
my-creds              Opaque                                1      7sExceptions
```




## Exceptions

Now let’s say your acme-org has a secret that you originally wanted to share with all the teams. We created this Secret as follows —

```
$ kubectl -n acme-org create secret generic my-secret --from-literal=password=iamacme
secret/my-secret created
```


You’ll see that “ ** *my-secret”* **  is propagated to both “ ** *team-a”* **  and “ ** *team-b ”* ** 

```
$ kubectl -n team-a get secrets
NAME                  TYPE                                  DATA   AGE
default-token-n99bx   kubernetes.io/service-account-token   3      45m
my-secret             Opaque                                1      22s


$ kubectl -n team-b get secrets
NAME                  TYPE                                  DATA   AGE
default-token-gs449   kubernetes.io/service-account-token   3      42m
my-secret             Opaque                                1      24s
```


But now we’ve started running an untrusted service in “ ** *team-b”* ** , so we’ve decided not to share that secret with it anymore. We can do this by setting the propagation selectors on the secret —

```
$ kubectl annotate secret my-secret -n acme-org propagate.hnc.x-k8s.io/treeSelect='!team-b'
secret/my-secret annotated
```


Now you’ll see the secret is no longer accessible from “ ** *team-b”* ** . If we add any children below “ ** *team-b”* ** , the secret won’t be propagated to them, either.

```
$ kubectl -n team-b get secrets
NAME                  TYPE                                  DATA   AGE
default-token-gs449   kubernetes.io/service-account-token   3      43m
```




# Parting Thoughts

In this blog, we have explored the power of Hierarchical Namespaces in Kubernetes, which simplifies namespace management and provides a more structured approach to organising resources within a cluster. By implementing the Hierarchical Namespace Controller (HNC), administrators can easily manage parent-child relationships between namespaces, propagate configurations and resources, and enforce access control policies across the hierarchy.
As containerised applications continue to grow in complexity, embracing the concept of hierarchical namespaces can help streamline management and enhance security within your Kubernetes environment. By incorporating HNC into your workflow, you’ll be well-equipped to tackle the challenges of large-scale namespace administration, leading to more efficient and scalable deployments.
As always, staying up-to-date with the latest developments in Kubernetes and its ecosystem is essential for maintaining a robust and efficient container orchestration setup. Keep exploring, learning, and experimenting to unlock the full potential of your Kubernetes infrastructure. Happy containerising!
![](https://miro.medium.com/v2/resize:fit:700/0*HtjY9aXfkHknIs7n.png) 


## 👋 If you find this helpful, please click the clap 👏 button below a few times to show your support for the author 👇



##  [Join FAUN Developer Community & Get Similar Stories in your Inbox Each Week](http://from.faun.to/r/8zxxd) 