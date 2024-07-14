---
tags:
  - KUBERNETES
  - K9S
source: https://medium.com/@gavinklfong/8-tips-to-incredibly-boost-the-efficiency-of-command-execution-on-kubernetes-using-k9s-a515a90a3a27
---




# 8 tips to incredibly boost the efficiency of command execution on Kubernetes using k9s



## k9s —awesome text based graphical tool with powerful hotkey design

![](https://miro.medium.com/v2/resize:fit:700/1*LCKauT3SLTeRoG1IKla4mw.png) k9s — Kubernetes CLI (source:  [https://k9scli.io/](https://k9scli.io/) 
Kubernetes (k8s) has become a key infrastructure for its powerful features that makes the service and resource management so much easier. The capabilities of auto-scaling, auto-healing and zero down-time deployment are fantastic.
Those who have experience on k8s spend considerable time on running  **kubectl**  commands.
For example, this command get all pods on app namespace

```
kubectl get pod -n app
```


Likewise, we need to run this command in order to get all services

```
kubectl get service -n app
```


Sometimes, we need to run the same command several times. Even though re-running the same command in the history could speed up a little bit, typing commands to retrieve k8s resources is still unavoidable.
We need a friendly user interface that saves us time from typing commands and presents information in a cohesive way.
 [ ****  ](https://k9scli.io/) is a perfect tool which provides a text based user interface for k8s. It is not to replace  **kubectl** , this tool supports quick navigation and operations on various objects in the cluster.
There are 3 sections in the sample screenshot of k9s:
- Shortcut — list out available hotkeys for navigation
- Main content — list out the the k8s resources
- Breadcrumbs — indicate the k8s resource type

![](https://miro.medium.com/v2/resize:fit:700/0*iWvk84WGFMdHSvDd) k9s layout
Although the k8s dashboard provides a more fancy graphical user interface in a web portal, k9s’s text based interface with shortcut navigation is far more efficient because everything can be done just using the keyboard without the need to use the mouse.
In this article, let’s explore k9s and how it helps us speed up operations on kubernetes clusters.


# Local environment setup

 [ **microk8s**  ](https://microk8s.io/) is not the only choice when it comes to running a local k8s cluster.  [ **minikube**  ](https://minikube.sigs.k8s.io/docs/) is another popular option. Microk8s is my preferred choice because  [the comparison](https://microk8s.io/compare)  between microk8s and minikube clearly shows that microk8s is more lightweight with lesser memory consumption but offers more features.
Installation is simple with these 2 commands:

```
> brew install ubuntu/microk8s/microk8s
> microk8s install
```


You can run the CLI with this syntax:  ** *microk8s kubectl <command>* **  For example, this command get all objects across all namespaces
Alternatively, you can use your existing kubectl CLI. Get the token and connectivity information by running  ** *microk8s config* **  and then append to  **<your home directory>/.kube/config** 


## K9s installation

Simply run this brew command line to install k9s and that’s it.

```
brew install derailed/k9s/k9s
```




# Tip #1 — Access k8s resource by resource type

Press colon “:” key to open a text box
![](https://miro.medium.com/v2/resize:fit:547/0*ENhagtVwqM0Qh_YR) get k8s resource by type
You can input k8s resource type name (e.g. namespace, pod, service, deploy, configmap, etc) to get the list of k8s resources.
Auto completion is an awesome feature, you don’t need to type the whole word. For example, the text box provides the suggestion of “namespace” as I type “na”. Press tab to accept the suggestion
![](https://miro.medium.com/v2/resize:fit:459/0*MGHBf8wghq-5UCkR) auto completion
Likewise,  ** *:deploy* **  shows the deployment resource
![](https://miro.medium.com/v2/resize:fit:700/0*3XvB3xzEN7NTJWTD) k8s deployment
 ** ** **  gets service
![](https://miro.medium.com/v2/resize:fit:700/0*JgC-KBtDuU0UTyJg) k8s service


# Tip #2 — Quick access to k8s resources by namespace

K8s resources are usually grouped into different namespaces for easy management. Sometimes, you are just not sure which namespace you should look for. Getting all k8s resources such as service in all namespaces can be done on k9s using hotkey “0” as shown in the screenshot.
Hotkeys are available if you would like to look into a specific namespace.
![](https://miro.medium.com/v2/resize:fit:700/0*YGWtLg-J9DJHbv8C) shortcut keys to namespaces


# Tip #3 — Viewing k8s resource information

Hotkey “d” is a handy function to get k8s resource in more detail such as  ** *kubectl describe* **  command
![](https://miro.medium.com/v2/resize:fit:700/0*AzbUZ8YWdm_2EQ9m) describe k8s resource
Alternatively, “y” hotkey is a helpful shortcut if you would like to get the raw YAML definition. Then, “c” hotkey can copy the whole content.
![](https://miro.medium.com/v2/resize:fit:700/0*TvcqRCNrXqaCVmpG) get yaml content
Simply press <Esc> whenever you would like to to go back to the previous view.


# Tip #4 — Pod viewing & operations

As I select the  ****  namespace and press enter, a list of 3 pods in  ****  is shown or use  ** ** ** 
![](https://miro.medium.com/v2/resize:fit:700/0*PX_PXv_bJsPxSpgM) k8s pods
Press “l” on either pod view or container view to see system logs. There are several hotkeys such as using head (hotkey: 1) to check the beginning of the logs. By default, tail is used that outputs the most recent system logs
![](https://miro.medium.com/v2/resize:fit:700/0*94i1N2UbIIRNy-dx) pod system logs


## Pod testing

The shortcut (shift-f) for port-forward is awesome. A dialogue is popped up for the list of pre-filled input fields and just pressing enter to confirm the creation.
![](https://miro.medium.com/v2/resize:fit:700/0*L6a2OWt0yVzbqFcr) port forward
“F” icon is shown under the PF column once port forward is enabled.
![](https://miro.medium.com/v2/resize:fit:461/0*bGddWgwXp_gzuHKP) pod with port forward enabled
To remove the port forward, use “f” hotkey to show the list of existing port forward and use control-d to delete the select port forward settings.
![](https://miro.medium.com/v2/resize:fit:700/0*Qx2O87l7ynuVkBfE) port forward list


## Pod operations

In addition to information viewing, you can open a terminal command prompt on the selected pod.
![](https://miro.medium.com/v2/resize:fit:639/0*0gp6ZmoW2LcXhBpr) pod terminal


# Tip #5 — k8s resource operations

Use “e” hotkey to modify the YAML definition of any k8s resources in the VI editor. Once you have saved the change, the updated is effective immediately.
![](https://miro.medium.com/v2/resize:fit:700/0*B7CouFvxj70UiSZr) modify yaml content
The shortcut key to delete (ctrl-d) or kill (ctrl-k) k8s resources such as pod in the example below. We can choose propagation (e.g. background, foreground, etc) and determines whether it is a force deletion.
![](https://miro.medium.com/v2/resize:fit:700/0*CKqFMTVHDIUlhAUQ) Deletion dialogue


# Tip #6 — System log viewing on multiple pods

Instead of viewing system logs of a single pod, checking system logs of all pods in a deployment / service is essential.
I like the log viewing on deployment, the same hotkey “l” shows the system logs of all pods of the deployment / service.
The screenshot below displays logs from 2 different pods of a deployment.
![](https://miro.medium.com/v2/resize:fit:620/0*EM-i4Odx92YN0xeK) 
Just log viewing is not useful without search. Use hotkey “/” to open a text box for keyword search on the system log.
The search for a keyword “starting”, it highlights the log lines of all pods with the keyword.
![](https://miro.medium.com/v2/resize:fit:700/0*Quro1DlIdfMdqz-d) 


# Tip #7 — Deployment scaling

Scaling adjustment is insanely easy. Press “s” on a selected deployment item. You will be prompted to update the replica count.
![](https://miro.medium.com/v2/resize:fit:635/0*FrH72LxqTsCZ7QJ2) Deployment scaling


# Tip #8 — Check reference of ConfigMap / Secret

 ** *:configmap* **  and  ** *:secret* **  shows the list of configmap and secret items respectively.
Viewing and copying item content can be done by selecting the target item and using “c” hotkey. Not to mention the functionality that allows you to edit the content.
![](https://miro.medium.com/v2/resize:fit:700/0*_dGxF6qzANG3LDJq) configmap content
Another cool feature is the hotkey “u” to check the list of k8s resources which are using the the configmap / secret item
![](https://miro.medium.com/v2/resize:fit:700/0*SPASlxg4dKzGhZD9) check “usedby” of configmap


# Final Thoughts

k9s present k8s resources in a well organized text based graphical interface with powerful hotkeys. It greatly reduces the number of keystroke required in order to retrieve information from kubernetes cluster.
A typical get pod command line  `kubectl get pod`  requires 16 keystrokes. By contract, it takes just 5 keystrokes (include ENTER key)  ``  on k9s. Getting yaml definition of a pod could be be much faster with just the use of hotkey instead of typing the whole command line.
Text based graphical interface design of k9s outperforms the fancy HTML graphical interface in terms of responsiveness. Moreover, the use of hotkeys is so much efficient as your hands stay work on the keyboard all the time instead of spending extra time on moving mouse around.
k9s is highly recommended if you are looking for tool that speeds up your operations on k8s cluster.