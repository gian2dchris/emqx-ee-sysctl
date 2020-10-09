# emqx-ee-sysctl

A Helm Chart for emqx-ee with extra configs and sysctls

Also an example of setting Linux kernel parameters, network protocol stack parameters, Erlang  virtual machine parameters, and EMQ X message server parameter using a Helm Chart and Daemon Sets.

Includes extra configuration described by the documentation, such as:

* Linux Network Tuning
* TCP Network Tuning
* Erlang VM tuning and Garbage collection
* Enable SSL and mount certificates
* Load and configure plugins
* Push metrics to Prometheus Gateway
* Expose MQTT services using NodePorts

Setting these up using Helm and K8s is not covered in the documentation, so it may be of help to someone.

Tuning parameters are set for a EMQ X Message Server 4.x version running on an 8-core, 32G memory CentOS server. Adjust settings to match your hardware specs, or things may break.

## Kernel and TCP Tuning and sysctls

When evaluating the MQTT broker we run few scenarios and stress tests using benchmarks. Before Kernel level tuning was in place we noticed a strange behaviour; as connection number was rising the brokers started to allocate memory up to the point they crashed. Garbage collection was in place, but it did not seem to help and metrics from erlang's processes could not justify the resources consumption, so we concluded that it was some sort of TCP connection and/or socket buffering.

To understand this better I recommend checking this brilliant blog post on [How TCP backlog works in Linux](http://veithen.io/2014/01/01/how-tcp-backlog-works-in-linux.html).

Kubernetes Documentation states:

*A number of sysctls are namespaced in today's Linux kernels. This means that they can be set independently for each pod on a node. Being namespaced is a requirement for sysctls to be accessible in a pod context within Kubernetes. [...] Sysctls are grouped into safe and unsafe sysctls. In addition to proper namespacing, a safe sysctl must be properly isolated between pods on the same node. By far, most of the namespaced sysctls are not necessarily considered safe.*

## EMQX configuration: `etc/emqx.conf`

To set `listener.ssl.external` in `emqx.conf` to `"8883"` add the following line in `values.yaml`

```yaml
emqxConfig:
  ...
  EMQX_LISTENER__SSL__EXTERNAL: "8883"
  ...
```

## Configure a EMQX plugin: `etc/plugins/emqx_prometheus`.conf

1. Edit `values.yaml`
2. To enable plugin.

```yaml
emqxConfig:
...
  EMQX_LOADED_PLUGINS: "emqx_recon,emqx_retainer,emqx_management,emqx_dashboard,emqx_prometheus"
```

2. To set `prometheus.push.gateway.server` in `etc/emqx/plugins/emqx_prometheus.conf` add the following line in `values.yaml`

```yaml
emqxConfig:
  ...
  EMQX_PROMETHEUS__PUSH__GATEWAY__SERVER: "http://localhost:9091"
  ...
```

You can enable and configure other plugins this way, too. Keep in mind that community version metrics [plugin](https://docs.emqx.io/enterprise/latest/en/tutorial/prometheus.html) is called `emqx_statsd.conf` and enterprise version calls it `emqx_prometheus.conf`. I did not find this documented anywhere.

```sh
##--------------------------------------------------------------------
## Statsd for EMQ X
##--------------------------------------------------------------------
prometheus.push.gateway.server = http://localhost:9091
prometheus.interval = 5000
#prometheus.collector.1 = emqx_prometheus
```

## Enable SSL

`kubectl apply -f emqx-certificates.yaml` and set `emqxCertSecretName: emqx-certificates` in `values.yaml`\

Note that the secret must already exist, for the pods to mount it. Same goes for the license secret.

## Node Port services

To create the NodePort services set `emqxEnableNodePort` to `enabled` in `values.yaml`

## Persistance

To create PVCs set `persistence.enabled` to `true` in `values.yaml`

## Install

```bash
$ kubectl apply -f emqx-certificates.yaml
$ kubectl apply -f emqx-license.yaml
$ kubectl apply -f emqx-daemonset.yaml
$ helm install emqx-ee ./ \
--set enableNodePort=true \
--set emqxLicneseSecretName=emqx-license \
--set emqxCertSecretName=emqx-certificates
```

## References

[EMQX: Configuration](https://docs.emqx.io/enterprise/latest/en/configuration/configuration.html)

[EMQX: Tuning](https://docs.emqx.io/enterprise/latest/en/tutorial/tune.html)