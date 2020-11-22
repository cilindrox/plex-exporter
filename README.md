# Plex Prometheus Exporter

This repo installs an instance of Plex Media Server (PMS) [running in
Kubernetes][pms-docker] alongside of the Prometheus stack and a dashboard with
PMS metrics in Grafana.

## Setup

This section goes step-by-step into the cluster setup and configuration. Make
sure you've set up and are running the following:

- Kubernetes >=1.19
- Helm v2

### Install Prometheus and Grafana

The Prometheus and Grafana are installed from the community charts:

```bash
helm init --client-only --skip-repos
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```
> *NOTE*: it's highly recommended to run a ["Tillerless Helm"][tillerless] when
> using Helm v2

```bash
# Assumes tiller is on $PATH, otherwise you'll need to download Helm & Tiller binaries
tiller&

# Helm will use the local running tiller instance.
export HELM_HOST=localhost:44134
```

```bash
# This will create two namespaces, "infra" and "plex" that will be used to
# install the Prometheus stack and PMS respectively.
kubectl apply -f bootstrap

helm upgrade --install prom --namespace infra prometheus-community/prometheus \
  --wait \
  -f prometheus.yaml

# Note: using version 5.x of the chart here, latest with support for Helm 2
# this is not required if using Helm >=3.4
helm upgrade --install grafana --namespace infra grafana/grafana \
  --version 5.8.16 \
  --wait \
  -f grafana.yaml \
  --set adminPassword="$(openssl rand 12 -base64)"
```

### Accessing the Prometheus instance

```bash
export POD_NAME=$(kubectl get pods --namespace infra -l "app=prometheus,component=server" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace infra port-forward $POD_NAME 9090:9090
```

### Accessing the Grafana instance

```bash
export POD_NAME=$(kubectl get pods --namespace infra -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace infra port-forward $POD_NAME 3000:3000
```

## Installing PMS and claiming the server

PMS must be connected to a [plex.tv](https://plex.tv) account. Once the
temporary token has been issued, you can install and claim the server using the
included [kube-plex chart](./charts/kube-plex).

This chart is a copy of [munnerz/kube-plex] with the addition of a `/media`
volume, ie:

```patch
diff --git a/charts/kube-plex/templates/deployment.yaml b/charts/kube-plex/templates/deployment.yaml
index eafe64b..d9a924a 100644
--- a/charts/kube-plex/templates/deployment.yaml
+++ b/charts/kube-plex/templates/deployment.yaml
@@ -154,6 +154,10 @@ spec:
         - mountPath: "/data-{{ .name }}"
           name: "extradata-{{ .name }}"
         {{- end }}
+        {{- if .Values.persistence.hostVolume }}
+        - name: media
+          mountPath: /media
+        {{- end }}
         - name: shared
           mountPath: /shared
         resources:
@@ -199,6 +203,11 @@ spec:
 {{- end }}
       - name: shared
         emptyDir: {}
+      {{- if .Values.persistence.hostVolume }}
+      - name: media
+        hostPath:
+          path: {{ .Values.persistence.hostVolume }}
+      {{ end }}
     {{- with .Values.affinity }}
       affinity:
 {{ toYaml . | indent 8 }}
```

> **NOTE**: the above git patch can be applied to any existing copies of the
> kube-plex chart.

This approach is mainly a workaround for a couple of [existing issues in 
Minikube][minikube-mounts], which results in empty bind mounts when using kvm as
the driver.

```bash
helm diff upgrade --install plex ./charts/kube-plex \
  --namespace plex \
  -f kube-plex.yaml \
  --set 'claimToken=MY_CLAIM' \
  --set 'persistence.hostVolume=/hosthome/music'
```

The same can be accomplished with Helm 3.1+ via [a post-render
hook][helm-post-render] that [applies a kustomization][kustomize-post].


### PMS metrics exporter

A [chart is included](./charts/plex_exporter) for exposing PMS metrics in
prometheus format.

```bash
helm upgrade --install \
  --namespace plex  \
  exporter ./charts/plex_exporter \
  --set 'server.token=MY-TOKEN'
```

Alternatively, the exporter can be added as a sidecar to existing chart, but
since it requires a server token to auth against the api which can only be
obtained once the server is up and running, I went for running it on its own
chart.

[helm-post-render]: https://helm.sh/docs/topics/advanced/#post-rendering
[kustomize-post]: https://github.com/thomastaylor312/advanced-helm-demos/tree/master/post-render
[minikube-mounts]: https://github.com/kubernetes/minikube/issues/3973
[munnerz/kube-plex]: https://github.com/munnerz/kube-plex/tree/master/charts/kube-plex
[pms-docker]: https://github.com/plexinc/pms-docker
[tillerless]: https://rimusz.net/tillerless-helm
