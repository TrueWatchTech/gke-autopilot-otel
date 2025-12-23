# GKE Autopilot Cluster intergration Truewatch

## Prerequisites

[GKE Autopilot mode](https://docs.cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview)

``` txt
gcloud-cli
kubectl
Helm 3.2.0+
```

## Install Datakit by Helm

```bash
helm pull datakit --repo https://pubrepo.guance.com/chartrepo/datakit --untar 
```

Remove roots volume in daemonset yaml

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  labels:
    app: {{ include "datakit.fullname" . }}
  {{- include "datakit.labels" . | nindent 4 }}
  name: {{ include "datakit.fullname" . }}
spec:
  revisionHistoryLimit: 10
  selector:
    matchLabels:
  {{- include "datakit.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      annotations:
        checksum/config: {{ randAlphaNum 5 | quote }}
      {{- with .Values.podAnnotations }}
      {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        app: {{ include "datakit.fullname" . }}
    {{- include "datakit.selectorLabels" . | nindent 8 }}
    spec:
      hostNetwork: false
      dnsPolicy: ClusterFirstWithHostNet
      # ... Other Settings ... #
          securityContext:
            privileged: false
          volumeMounts: [] # remove /run、/var、/sys folder mount
      # ... Other Settings ... #   
      volumes: [] # remove /run、/var、/sys folder mount
      # ... Other Settings ... #   

```

Adjust datakit values

```yaml
## other settings ##
datakit:
  dataway_url: <YOUR_DATAWAY_URL_WITH_TOKEN>
  ## other settings ##
dkconfig: [EXTRA_CONFIG_FILES]
#  - path: "/usr/local/datakit/conf.d/db/mysql.conf"
#    name: mysql.conf
#    value: |
#      # {"version": "1.1.9-rc7.1", "desc": "do NOT edit this line"}
#      [[inputs.mysql]]
#        host = "192.168.0.3"
#        user = "root"
#        pass = "S6QgMvrer2!8xvMD"
#        port = 3306
#        interval = "10s"
#        innodb = true
#        tables = []
#        users = []
#        [inputs.mysql.dbm_metric]
#          enabled = true
#        [inputs.mysql.dbm_sample]
#          enabled = true
#        [inputs.mysql.tags]
#          # some_tag = "some_value"
#          # more_tag = "some_other_value"
## other settings ##
```

## Install Opentelemetry

```bash
  helm install cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --version v1.19.1 \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --set global.leaderElection.namespace=cert-manager

  kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

## Install Otel k8s collector

⚠️ Please remember to replace "<YOUR_DATAKIT_CLUSTER_DNS_WITH_PORT>" in otel-k8s-collector yaml file.

```bash
kubectl apply -f otel-collector.yaml
```

## Install Otel kube state collector

⚠️ Please remember to replace "<YOUR_DATAKIT_CLUSTER_DNS_WITH_PORT>" in otel-kube-collector yaml file.

```bash
helm install kube-state-metrics prometheus-community/kube-state-metrics \       
  --namespace <YOUR_NAMESPACE> \                                 
  --set autosharding.enabled=false \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=128Mi

kubectl apply -f otel-kube-collector.yaml  
```
