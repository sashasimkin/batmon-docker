# batmon-docker

Docker build + helm chart for https://github.com/fl4p/batmon-ha

## Example deployment with argocd

### Requirement, MQTT application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mosquitto
  namespace: argocd
  finalizers:
  - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: yourns
  project: default
  source:
    chart: mosquitto
    repoURL: https://bdclark.github.io/helm-charts
    targetRevision: 0.6.1
    helm:
      values: |
        persistence:
          enabled: true
          storageClass: longhorn
          size: 1Gi
        service:
          type: ClusterIP
          ports:
            mqtt:
              enabled: true
              port: 1883
              targetPort: 1883
        config:
          allowAnonymous: true
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

```

### Application:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: batmon-ha
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    server: {{ .Values.spec.destination.server }}
    namespace: batmon-ha
  project: default
  sources:
    - repoURL: https://github.com/sashasimkin/batmon-docker.git
      path: chart
      targetRevision: HEAD
      helm:
        # Same repo as argocd application, values in batmon-ha directory
        valueFiles:
          - values.yaml
          - $argorepo/batmon-ha/values.yaml
          - $argorepo/batmon-ha/values.secret.yaml
        ignoreMissingValueFiles: true
    - repoURL: {{ .Values.spec.source.repoURL }}
      targetRevision: {{ .Values.spec.source.targetRevision }}
      ref: argorepo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### values.yaml

```yaml

image:
  repository: ghcr.io/sashasimkin/batmon-ha
  # pin specific version
  tag: sha-7c78689
  pullPolicy: IfNotPresent

nodeSelector:
  kubernetes.io/hostname: rpi # adjust to the node with Bluetooth

# config becomes /data/options.json inside the container
# Sensitive values (mqtt_password, etc.) should go in values.secret.yaml
config:
  period: 1.0
  devices:
    - address: "AA:BB:CC:DD:EE:FF"
      type: jbd
      alias: my_bms
  mqtt_broker: "mosquitto.yourns.svc.cluster.local"
  mqtt_port: 1883
  mqtt_user: ""
  mqtt_password: ""
```
