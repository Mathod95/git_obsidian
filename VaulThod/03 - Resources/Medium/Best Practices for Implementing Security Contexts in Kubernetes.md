---
tags:
  - KUBERNETES/POD
  - SECURITY
source: https://medium.com/@tamerbenhassan/7-best-practices-for-implementing-security-contexts-in-kubernetes-68bcc182f19a
postRelated: obsidian://open?vault=VaulThod&file=01%20-%20Projects%2FBlog%2FKubernetes%2FObjects%2FPods%2FsecurityContext
---
# 7 Best Practices for Implementing Security Contexts in Kubernetes

Kubernetes security contexts are a fundamental part of safeguarding Kubernetes applications. They allow administrators and developers to define privilege and access control settings for pods and containers. This article provides a detailed overview of best practices for configuring security contexts in Kubernetes, complete with code snippets to illustrate each practice.
![](https://miro.medium.com/v2/resize:fit:700/1*DxlHplQwxW0eHDxTpxuWeg.png) 


# Understanding Kubernetes Security Contexts

A security context defines privilege and access settings for a pod or container. It can be configured at the pod level, affecting all containers in the pod, or at the individual container level, overriding pod-level settings where necessary.

# Best Practices for Configuring Security Contexts

The following are key best practices for setting security contexts in Kubernetes deployments, each aimed at minimizing the potential attack surface and ensuring the principle of least privilege.

## 1. Run Containers as a Non-Root User

Running containers as non-root users reduces the risks associated with privilege escalation attacks. Here’s how you can set this up in your pod specification:

```
apiVersion: v1
kind: Pod
metadata:
  name: non-root-pod
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
```


This configuration ensures that the  `myapp`  container runs as user ID 1000 and group ID 3000, without the ability to escalate privileges.


## 2. Disable Privilege Escalation

Disabling privilege escalation prevents the use of tools like  `sudo`  that can grant elevated rights to processes:

```
apiVersion: v1
kind: Pod
metadata:
  name: disable-privilige-escalation
spec:
  containers:
  - name: myapp
    image: myapp:1.0
    securityContext:
      allowPrivilegeEscalation: false
```

## 3. Set the Security Capabilities

Limit the Linux capabilities granted to a container to only those that are necessary for the application. This example removes all default capabilities and explicitly adds only the  `NET_BIND_SERVICE`  capability:

```
securityContext:
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE
```

## 4. Use Read-Only Root Filesystem

Making the root filesystem read-only where possible will prevent any tampering with critical system files:

```
securityContext:
  readOnlyRootFilesystem: true
```

## 5. Configure Seccomp Profiles

Seccomp (Secure Computing Mode) is a kernel security feature that restricts the set of system calls applications can make. Here’s how to specify a default seccomp profile for your containers:

```
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

## 6. Use SELinux Options

If SELinux is enforced in your environment, you can specify SELinux options for more granular security control:

```
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

## 7. Define the Pod Security Context

For settings that you want to apply to all containers within a pod, use the pod-level security context. This example sets a common user and group ID for all containers in the pod:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  securityContext:
    fsGroup: 2000
    runAsUser: 1000
  containers:
  - name: myapp
    image: myapp:1.0
  - name: mysidecar
    image: mysidecar:1.0
```


This configuration applies the  `fsGroup`  and  `runAsUser`  settings across all containers in the pod, ensuring consistent security settings.

# Conclusion

Implementing robust security contexts in Kubernetes is critical for ensuring that containerized applications are isolated and protected from potential security threats. By adhering to these best practices, organizations can significantly enhance the security of their Kubernetes environments, reduce the risk of unauthorized access, and maintain compliance with security policies.