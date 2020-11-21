# plex-exporter

This is a chart for [othalla/plex_exporter] that exposes PMS metrics in
prometheus format.

## Installation

```bash
helm upgrade --install \
  --namespace plex  \
  exporter ./plex_exporter \
  --set 'server.token=MY-TOKEN'
```

> **NOTE**: A server token is needed in order to communicate with the PMS API,
> follow [these steps][x-plex-token] in order to get your plex authentication
> token.

### Building the Docker image

This chart includes a [`Dockerfile`](./Dockerfile) for `othalla/plex_exporter`,
image can be built the via the following:

```bash
docker build --rm -t plex-exporter:0.0.1 .
```

If using minikube, you can make it available to the kubelet Docker daemon via:

```bash
eval $(minikube docker-env)
```

## Notes

This chart skips the ingress resource, mostly because it assumes app will be
queried from within the cluster; an ingress might be required if using a central
Prometheus instance instead.

Service account is also disabled by default, but best practices recommend
assigning one.

[plex_exporter]: https://github.com/othalla/plex_exporter.git
[x-plex-token]: https://support.plex.tv/articles/204059436-finding-an-authentication-token-x-plex-token/
