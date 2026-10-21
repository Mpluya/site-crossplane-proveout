# site-crossplane-proveout

# Site XRD POC (Crossplane v2 on OrbStack)

Proof of concept for a `Site` custom resource. Applying a small `Site` spec produces the Kubernetes resources a site needs. So far, the cache is implemented. The app server, database, and file server come next.

## Layout

Adjust paths to match your repo.

```
platform/
  functions.yaml        # Crossplane composition functions (pinned)
  site-xrd.yaml         # Site CompositeResourceDefinition
  site-composition.yaml # Site Composition (go-templating + auto-ready)
  site-rbac.yaml        # ClusterRole so Crossplane can create composed kinds
sites/
  demo.yaml             # sample Site
```

## 1. Cluster and Crossplane

OrbStack ships a built-in single-node Kubernetes cluster. Enable it in OrbStack (Settings -> Kubernetes), then:

```bash
kubectl config use-context orbstack
kubectl get nodes

helm repo add crossplane-stable https://charts.crossplane.io/stable
helm install crossplane crossplane-stable/crossplane -n crossplane-system --create-namespace
```

Use Crossplane v2 so XRs can be namespaced and compose native Kubernetes objects directly, with no claims and no cloud provider needed.

## 2. Look up function versions

Functions are OCI packages. List tags to find the current version and confirm the path is valid:

```bash
crane ls xpkg.crossplane.io/crossplane-contrib/function-go-templating
crane ls xpkg.crossplane.io/crossplane-contrib/function-auto-ready

# or
skopeo list-tags docker://xpkg.crossplane.io/crossplane-contrib/function-go-templating
```

If the lookup fails, try `xpkg.upbound.io/crossplane-contrib/...`. Other sources: marketplace.upbound.io and the GitHub releases page for each function in the `crossplane-contrib` org. Pin explicit tags, never `latest`.

## 3. Install the functions

`platform/functions.yaml`:

```yaml
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata: { name: function-go-templating }
spec: { package: xpkg.crossplane.io/crossplane-contrib/function-go-templating:v0.12.6 }
---
apiVersion: pkg.crossplane.io/v1
kind: Function
metadata: { name: function-auto-ready }
spec: { package: xpkg.crossplane.io/crossplane-contrib/function-auto-ready:v0.7.2 }
```

```bash
kubectl apply -f platform/functions.yaml
kubectl get functions -w
```

Wait until both show `INSTALLED=True` and `HEALTHY=True` before applying the Composition. If one is stuck, `kubectl describe function <name>` shows whether it's an image pull problem or a runtime issue.

## 4. RBAC for composed resources

Crossplane needs permission for every kind a Composition creates. A ClusterRole with the aggregation label is picked up automatically, so there is no binding to write.

`platform/site-rbac.yaml`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: crossplane-site-composed-resources
  labels:
    rbac.crossplane.io/aggregate-to-crossplane: "true"
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

```bash
kubectl apply -f platform/site-rbac.yaml

# verify (both should print "yes")
kubectl auth can-i create deployments \
  --as=system:serviceaccount:crossplane-system:crossplane -n sites
kubectl auth can-i create services \
  --as=system:serviceaccount:crossplane-system:crossplane -n sites
```

If `can-i` prints `no`, check the label spelling and the service account name (`kubectl get sa -n crossplane-system`). Add a rule here for every new kind the Composition creates (statefulsets, configmaps, secrets, PVCs, ingresses, operator CRs).

## 5. Apply the XRD and Composition

```bash
kubectl apply -f platform/site-xrd.yaml
kubectl apply -f platform/site-composition.yaml

kubectl get xrd           # ESTABLISHED should be True
kubectl get composition
kubectl api-resources | grep -i site
```

`compositeTypeRef` in the Composition must exactly match the XRD's group, version, and kind.

## 6. Create a Site

```bash
kubectl create namespace sites
```

`sites/demo.yaml`:

```yaml
apiVersion: platform.example.org/v1alpha1
kind: Site
metadata:
  name: demo
  namespace: sites
spec:
  cache:
    enabled: true
    memory: 256Mi
```

```bash
kubectl apply -f sites/demo.yaml

kubectl get site -n sites
crossplane beta trace site/demo -n sites
kubectl get deploy,svc -n sites
```

Expected result: the Site shows `SYNCED=True` and `READY=True`, with `demo-cache` as a Deployment and a Service. The `-` in the SYNCED/READY columns for the child objects is expected, since plain Kubernetes objects don't carry Crossplane conditions.

## 7. Verify the cache

```bash
kubectl run -n sites valkey-test --rm --restart=Never --image=valkey/valkey:8 -- valkey-cli -h demo-cache ping
```

Expected output: `PONG`. With `-it` instead of `--restart=Never` you get a harmless "couldn't attach to pod" warning, because the command exits before `kubectl` can attach.

## 8. Verify declarative teardown

Set `cache.enabled: false` in `sites/demo.yaml`, then:

```bash
kubectl apply -f sites/demo.yaml
kubectl get deploy,svc -n sites     # Deployment and Service should be gone
```

Set it back to `true`, reapply, and confirm they return.

## Debugging

```bash
kubectl describe site demo -n sites
kubectl logs -n crossplane-system -l pkg.crossplane.io/function=function-go-templating
kubectl describe pod -n sites
crossplane render ...   # test compositions locally without a cluster
```

| Symptom | Likely cause |
| --- | --- |
| `forbidden` error naming a kind | Missing rule in `site-rbac.yaml` |
| Nothing created, no events | `compositeTypeRef` doesn't match the XRD |
| Template error | Missing `composition-resource-name` annotation, or template syntax (check the function's README for your pinned version) |
| Pod pending or crashing | Normal workload debugging; check the image supports arm64 on Apple Silicon |

## Notes

- **Cache:** a plain Deployment plus Service running `valkey/valkey:8`. No operator is needed for an ephemeral cache. Add one only if you need HA, persistence, or sharding, in which case the Composition renders the operator's CR instead and the Site spec stays the same.
- **OrbStack:** local images are usable without a registry. Services are reachable from the Mac, with `*.k8s.orb.local` domains. It is a single node, so storage and scheduling behavior won't match GKE; re-test the DB and file server parts on a real cluster.
- **Scale (about 5000 sites):** Crossplane's own etcd footprint is small. The cost is mostly the objects each Site composes, plus reconcile load on the API server and function pods. Raise the poll interval, scale the function pods, keep per-site object count low, and consider several cells of 500-1000 sites each. Load-test with kwok and watch:

  ```bash
  kubectl get --raw /metrics | grep -E 'apiserver_storage_objects|etcd_db_total_size'
  ```

## Next steps

1. Add `spec.app` (image, replicas, port) and an app Deployment and Service, passing the cache address in as `CACHE_HOST`.
2. Add the database (CloudNativePG) and the file server (RWX PVC or object storage).
3. GitOps: platform layer first (sync wave 0 or Flux `dependsOn`), then `sites/`. Run `crossplane render` and `crossplane beta validate` on PRs.
4. Package the XRD and Compositions as a Crossplane Configuration (OCI) so platform versions can be promoted across clusters.

