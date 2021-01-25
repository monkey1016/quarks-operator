## Quarks Operator
* Some code changes to `container_factory.go`
* Build quarks-operator image
  * ```
  export PROJECT=quarks-operator
  ./bin/build-image
  podman tag cfcontainerization/quarks-operator:v7.1.3-dirty-0.g0ce050b9 chunks1016/quarks-operator:v7.1.3-dirty-0.g0ce050b9
  podman login docker.io
  podman push chunks1016/quarks-operator:v7.1.3-dirty-0.g0ce050b9
  ```
* Deploy
  * `helm install cf-operator /home/kjanania/source/clients/citi/quarks-operator/helm/quarks-7.1.3-dirty-0.g0ce050b9.tgz --namespace cf-operator --set "global.singleNamespace.name=kubecf" --set "image.org=chunks1016" --set "global.image.pullPolicy=Always" --debug`
* Added `cluster-admin` to cf-operator-quarks, cf-operator-quarks-secret, cf-operator-quarks-job, and
  cf-operator-quarks-statefulset
  * Need to figure out what finalizers need to be added to permissions. Likely quarks-secret

### KubeCF
* Add privileged SCC for kubecf
* Change `runAsUser` to `0` for app-dns container
* Change `runAsUser` to `0` for database container
* Update rootfs jobs to add SYS_CHROOT capability
* Build kubecf using `make kubecf-build`
* Deploy `helm install kubecf --namespace kubecf --set "system_domain=kubecf.apps.aws-test-cluster.pocs.down-time.io" --set "kube.flavor=openshift" ./output/kubecf-2.7.12-dirty.tgz`

## Quarks Job
* Update job definition to allow priviliged and runAsUser 0
* rebuild chart and docker image


## Things to fix
* `helm delete kubecf` does not get rid of the secrets as expected, this leads to encryption key errors on startup
* Enable wildcard routes: https://docs.openshift.com/container-platform/4.6/networking/ingress-operator.html#using-wildcard-routes_configuring-ingress
* Deploy a route:
```yaml
kind: Route
apiVersion: route.openshift.io/v1
metadata:
  name: router
  namespace: kubecf
  labels:
    quarks.cloudfoundry.org/az-index: '0'
    quarks.cloudfoundry.org/deployment-name: kubecf
    quarks.cloudfoundry.org/instance-group-name: router
    quarks.cloudfoundry.org/pod-ordinal: '0'
spec:
  host: api.kubecf.apps.aws-test-cluster.pocs.down-time.io
  to:
    kind: Service
    name: router-0
    weight: 100
  port:
    targetPort: router
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: None
  wildcardPolicy: Subdomain
```


## Steps to Install
1. `git clone -b feature/openshift https://github.com/monkey1016/kubecf.git`
2. `git clone -b feature/openshift https://github.com/monkey1016/quarks-operator.git`
3. Create the `cf-operator` namespace: `oc new-project cf-operator`
4. Add cluster-admin role to quarks users:
```
oc adm policy add-cluster-role-to-user cluster-admin system:cf-operator:cf-operator-quarks-secret
oc adm policy add-cluster-role-to-user cluster-admin system:cf-operator:cf-operator-quarks-job
oc adm policy add-cluster-role-to-user cluster-admin system:cf-operator:cf-operator-quarks-statefulset
```
5. Build the quarks-operator helm chart:
```bash
cd quarks-operator
export PROJECT=quarks-operator
make build-helm
```
6. Deploy the quarks-operator helm chart:
```bash
cd quarks-operator
helm install cf-operator helm/quarks-7.1.3-dirty-0.g0ce050b9.tgz --namespace cf-operator --set "global.singleNamespace.name=kubecf" --set "image.org=chunks1016" --set "global.image.pullPolicy=Always" --debug
```
7. Build the kubecf chart:
```bash
cd kubecf
make kubecf-build
```
8. Deploy the kubecf chart:
```bash
cd kubecf
helm install kubecf --namespace kubecf --set "system_domain=kubecf.apps.aws-test-cluster.pocs.down-time.io" --set "kube.flavor=openshift" ./output/kubecf-2.7.12-dirty.tgz
```
9. Enable wildcard routes (see above)
10. Deploy a route for kubecf (see above)
