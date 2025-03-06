# metallb loadbalancer for steadforce k8s clusters

Repository containing helm chart for metallb installation. Never install the content of this repo on our clusters manually. This is all done by argocd.

## Dependencies

This chart pulls in `metallb` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

```
 helm dependency update
```

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.

## IP range for loadbalancer ip's

To avoid problems inside Steadforce network we have to use fixed IP ranges known by our Sys-Team.
The blocks to use are:

### Local Environment

```
Network:   10.172.242.240/28
Broadcast: 10.172.242.255  
HostMin:   10.172.242.241  
HostMax:   10.172.242.254  
Hosts/Net: 14
```

HostMin is reserved for gateway, it can't be used, due to this fact we configure ip's from

```
 10.172.242.242-10.172.242.254
```

for loadbalancers in the kubernetes cluster.

### Dev Environment

```
Network:   10.195.5.240/28
Broadcast: 10.195.5.255  
HostMin:   10.195.5.241  
HostMax:   10.195.5.254  
Hosts/Net: 14
```

HostMin is reserved for gateway, it can't be used, due to this fact we configure ip's from

```
 10.195.5.242-10.195.5.254
```

for loadbalancers in the kubernetes cluster.

### Prod Environment

```
Network:   10.193.5.240/28
Broadcast: 10.193.5.255  
HostMin:   10.193.5.241  
HostMax:   10.193.5.254  
Hosts/Net: 14  
```

HostMin is reserved for gateway, it can't be used, due to this fact we configure ip's from

```
 10.193.5.242-10.193.5.254
```

for loadbalancers in the kubernetes cluster.

## Used IP's

### Local Environment

* Istio-Gateway `10.172.242.242`

### Dev Environment

* Istio-Gateway `10.195.5.242`

### Prod Environment

* Istio-Gateway `10.193.5.242`

## Render resource local

```
 helm template -n metallb --release-name metallb --include-crds --skip-tests \
  -a metallb.io/v1beta1 \
  -f values-local.yaml --output-dir _local .
```
