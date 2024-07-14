---
createdAt: 2024-05-19T20:09:00
status: TODO
tags:
  - KUBERNETES/POD/SECURITY
---

Le champ `securityContext` d'un pod dans Kubernetes est une partie fondamentale pour la protection d'une application. Il permet aux administrateurs et aux développeurs de définir les paramètres de privilèges et de contrôle d'accès pour les pods et les conteneurs. Cet article offre un aperçu détaillé des meilleures pratiques pour configurer les contextes de sécurité dans Kubernetes avec des extraits de code pour illustrer chaque pratique.

# Understanding Kubernetes Security Contexts

A security context defines privilege and access settings for a pod or container. It can be configured at the pod level, affecting all containers in the pod, or at the individual container level, overriding pod-level settings where necessary.

# Best Practices for Configuring Security Contexts

The following are key best practices for setting security contexts in Kubernetes deployments, each aimed at minimizing the potential attack surface and ensuring the principle of least privilege.

# Exécuter un container avec user non root

```yaml title:"pod.yaml"
apiVersion: v1
kind: Pod
metadata:
	name: app
spec:
	containers:
	- name: app
		image: app:1.0
		securityContext:
			runAsUser: 1000
			runAsGroup: 3000
```

# Désactiver l'escalade de privilèges 

La désactivation de l'escalade de privilèges empêche l'utilisation d'outils tels que `sudo` qui peuvent accorder des droits élevés à des processus.

```yaml title:"pod.yaml"
apiVersion: v1  
kind: Pod  
metadata:  
	name: app 
spec:  
	containers:  
	- name: app  
		image: app:1.0  
		securityContext:  
			allowPrivilegeEscalation: false
```

# 3. Set the Security Capabilities

Limit the Linux capabilities granted to a container to only those that are necessary for the application. This example removes all default capabilities and explicitly adds only the  `NET_BIND_SERVICE`  capability:

```yaml
apiVersion: v1  
kind: Pod  
metadata:  
	name: app 
spec:  
	containers:  
	- name: app  
		image: app:1.0  
		securityContext:
		  capabilities:
		    drop:
		      - ALL
		    add:
		      - NET_BIND_SERVICE
```

# 4. Use Read-Only Root Filesystem

Making the root filesystem read-only where possible will prevent any tampering with critical system files:

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

# 5. Configure Seccomp Profiles

Seccomp (Secure Computing Mode) is a kernel security feature that restricts the set of system calls applications can make. Here’s how to specify a default seccomp profile for your containers:

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault
```

# 6. Use SELinux Options

If SELinux is enforced in your environment, you can specify SELinux options for more granular security control:

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
```

# 7. Define the Pod Security Context

For settings that you want to apply to all containers within a pod, use the pod-level security context. This example sets a common user and group ID for all containers in the pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: app
    image: app:1.0
  - name: sidecar
    image: sidecar:1.0
  securityContext:
    fsGroup: 2000
    runAsUser: 1000
```

This configuration applies the  `fsGroup`  and  `runAsUser`  settings across all containers in the pod, ensuring consistent security settings.

# Conclusion

Implementing robust security contexts in Kubernetes is critical for ensuring that containerized applications are isolated and protected from potential security threats. By adhering to these best practices, organizations can significantly enhance the security of their Kubernetes environments, reduce the risk of unauthorized access, and maintain compliance with security policies.