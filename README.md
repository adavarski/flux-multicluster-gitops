## Multi-cluster mesh routing /w GitOps
This demo will build you 3 clusters that will all
share their routing information with each other and
forward DNS for cross-cluster Services.

The clusters are created using `kind`, and
`cluster0` is used as a [Flux](https://fluxcd.io) management cluster.
Access to apply to the remaining clusters is done by mocking ClusterAPI kubeconfigs.

Discovery of other clusters' Nodes is accomplished through
a fun bash controller that queries a multicast Serf cluster.
This works well on a single docker network or any network that supports multicast.
You can also configure Serf to bootstrap from some fixed IP's.

A neat thing about this strategy is that it's declarative!
Fork this repo and try it out :)

## Requirements:
   - git
   - hub (optional)
   - flux
   - docker
   - kind
   - kubectl

## Demo

```shell
kind/setup.sh
kind/load.sh

# bootstrap Calico for Flux
kubectl apply --context kind-cluster0 -k ./config/cluster0/kube-system
kubectl apply --context kind-cluster1 -k ./config/cluster0/kube-system
kubectl apply --context kind-cluster2 -k ./config/cluster0/kube-system

GITHUB_USER=adavarski
# set your own user here to match your fork

export GITHUB_TOKEN="<personal access token with repo and SSH key rights>"

kubectl config use-context kind-cluster0
flux bootstrap github \
  --owner "${GITHUB_USER}" \
  --personal \
  --repository "flux-multicluster-gitops" \
  --path "./config/cluster0"

kubectl config use-context kind-cluster1
flux bootstrap github \
  --owner "${GITHUB_USER}" \
  --personal \
  --repository "flux-multicluster-gitops" \
  --path "./config/cluster1"

kubectl config use-context kind-cluster2
flux bootstrap github \
  --owner "${GITHUB_USER}" \
  --personal \
  --repository "flux-multicluster-gitops" \
  --path "./config/cluster2"
```
alternatively, if you want to not use github & flux, apply the `kube-system` and `default` kustomizations to the proper clusters:
```shell
for cl in cluster{0..2}; do
  kubectl apply --context "kind-${cl}" -k "./config/${cl}/"{default,kube-system}
done
```

## Looking around
- Get the `Kustomization` resources the cluster0 flux-system uses to apply to the other clusters
- Use the `kubectl config use-context kind-{cluster0|cluster1|cluster2}` to switch between `kind-cluster0|1|2` on demand
- Check that the serf and calico dameonsets and deploys become ready
- Check out the Corefile ConfigMap extensions in kube-system
- Examine the `BGPPeer` resources that the serf-query controller created from the serf member list
- Exec into the debug pods for each cluster and run `host podinfo.default.svc.cluster1.lan`
- Try curling the service from and to different clusters!


## Clean Up
```shell
kind/cleanup.sh
```

____


## More demos!
Flux's GPG signature verification and remote-cluster management over Cluster API: https://github.com/stealthybox/capi-flux-demo
