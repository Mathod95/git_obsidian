---
createdAt: 2024-05-08T19:32:00
status: TODO
tags:
  - KUBERNETES/PROBES
  - KUBERNETES/PROBES/LIVENESS
  - KUBERNETES/PROBES/READINESS
  - KUBERNETES/PROBES/STARTUP
---
# Les types de probes
## startupProbe
## readinessProbe
## livenessProbe
## Probe attributes

# Probe actions
## Exec
## tcpSockets
## httpGet

## gRPC

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-pod
spec:
  containers:
  - name: my-app-container
    image: my-app-image:latest
    ports:
    - containerPort: 8080
    # Configuration de la sonde de démarrage (startupProbe)
    startupProbe:
      # Délai initial avant le premier démarrage de la sonde
      initialDelaySeconds: 10
      # Période de vérification de la sonde
      periodSeconds: 30
      # Délai maximal d'attente pour l'exécution de la sonde
      timeoutSeconds: 5
      # Nombre de succès requis consécutifs pour considérer la sonde comme réussie
      successThreshold: 1
      # Nombre d'échecs consécutifs autorisés avant de considérer la sonde comme échouée
      failureThreshold: 3
      # Commande à exécuter pour tester le démarrage
      exec:
        command:
        - "sh"
        - "-c"
        - "echo 'Startup check'; exit 0"

    # Configuration de la sonde de readiness (readinessProbe)
    readinessProbe:
      # Délai initial avant le premier démarrage de la sonde
      initialDelaySeconds: 15
      # Période de vérification de la sonde
      periodSeconds: 10
      # Délai maximal d'attente pour l'exécution de la sonde
      timeoutSeconds: 3
      # Nombre de succès requis consécutifs pour considérer la sonde comme réussie
      successThreshold: 1
      # Nombre d'échecs consécutifs autorisés avant de considérer la sonde comme échouée
      failureThreshold: 5
      # Configuration HTTP GET pour tester la readiness
      httpGet:
        path: /healthz
        port: 8080
        scheme: HTTP
        httpHeaders:
        - name: Custom-Header
          value: Custom-Value

    # Configuration de la sonde de liveness (livenessProbe)
    livenessProbe:
      # Délai initial avant le premier démarrage de la sonde
      initialDelaySeconds: 20
      # Période de vérification de la sonde
      periodSeconds: 15
      # Délai maximal d'attente pour l'exécution de la sonde
      timeoutSeconds: 5
      # Nombre de succès requis consécutifs pour considérer la sonde comme réussie
      successThreshold: 1
      # Nombre d'échecs consécutifs autorisés avant de considérer la sonde comme échouée
      failureThreshold: 3
      # Configuration TCP Socket pour tester la liveness
      tcpSocket:
        port: 8080
```

Type of probes

1. **Exec Probe**:
    
    - The `exec` probe type executes a specified command inside the container and considers the probe successful if the command exits with a zero status code.
    - Example:

        ```yaml
        exec:
          command:
          - cat
          - /tmp/health
        ```

2. **HttpGet Probe**:
    
    - The `httpGet` probe type performs an HTTP GET request to a specified endpoint on the container and considers the probe successful if the response status code falls within a specified range (default is 200-399).
    - Example:

        ```yaml
        httpGet:
          path: /healthz
          port: 8080
        ```

3. **TcpSocket Probe**:
    
    - The `tcpSocket` probe type attempts to open a TCP connection to a specified port on the container and considers the probe successful if the connection is established.
    - Example:

        ```yaml
        tcpSocket:
          port: 3306
        ```

# Configurer les Probes

[Probes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#probe-v1-core) ont un certain nombre de champs qui vous pouvez utiliser pour contrôler plus précisément le comportement de la vivacité et de la disponibilité des probes :

- `initialDelaySeconds`: Nombre de secondes après le démarrage du conteneur avant que les liveness et readiness probes ne soient lancées. La valeur par défaut est de 0 seconde. La valeur minimale est 0.
- `periodSeconds`: La fréquence (en secondes) à laquelle la probe doit être effectuée. La valeur par défaut est de 10 secondes. La valeur minimale est de 1.
- `timeoutSeconds`: Nombre de secondes après lequel la probe time out. Valeur par défaut à 1 seconde. La valeur minimale est de 1.
- `successThreshold`: Le minimum de succès consécutifs pour que la probe soit considérée comme réussie après avoir échoué. La valeur par défaut est 1. Doit être 1 pour la liveness probe. La valeur minimale est de 1.
- `failureThreshold`: Quand un Pod démarre et que la probe échoue, Kubernetes va tenter `failureThreshold` fois avant d'abandonner. Abandonner en cas de liveness probe signifie le redémarrage du conteneur. En cas de readiness probe, le Pod sera marqué Unready. La valeur par défaut est 3, la valeur minimum est 1.

[HTTP probes](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.30/#httpgetaction-v1-core) ont des champs supplémentaires qui peuvent être définis sur `httpGet` :

- `host`: Nom de l'hôte auquel se connecter, par défaut l'IP du pod. Vous voulez peut être mettre "Host" en httpHeaders à la place.
- `scheme`: Schéma à utiliser pour se connecter à l'hôte (HTTP ou HTTPS). La valeur par défaut est HTTP.
- `path`: Chemin d'accès sur le serveur HTTP.
- `httpHeaders`: En-têtes personnalisés à définir dans la requête. HTTP permet des en-têtes répétés.
- `port`: Nom ou numéro du port à accéder sur le conteneur. Le numéro doit être dans un intervalle de 1 à 65535.