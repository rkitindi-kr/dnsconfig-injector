# Mutating Admission Webhook for dnsconfig pod injection

# Problem fixed 

In original version, pod’s annotations are getting clobbered during admission. 🙂
When a mutating webhook also patches annotations, the final object the API server stores is the result of all mutations applied to your original manifest. If a webhook replaces the entire metadata.annotations map (instead of adding just its own keys), your annotation from the manifest can disappear.

# What was the fix?

updateAnnotation function in webhook.go file was commented out:

/*
func updateAnnotation(target map[string]string, added map[string]string) (patch []patchOperation) {
        for key, value := range added {
                if target == nil || target[key] == "" {
                        target = map[string]string{}
                        patch = append(patch, patchOperation {
                                Op:   "add",
                                Path: "/metadata/annotations",
                                Value: map[string]string{
                                        key: value,
                                },
                        })
                } else {
                        patch = append(patch, patchOperation {
                                Op:    "replace",
                                Path:  "/metadata/annotations/" + key,
                                Value: value,
                        })
                }
        }
        return patch
}

*/

It was replaced by this function which enables webhook to add annotation without clobbing existing annotations:

func updateAnnotation(existing map[string]string, added map[string]string) (patch []patchOperation) {
    // If annotations are nil on the object, create the map **once**.
    if existing == nil {
        patch = append(patch, patchOperation{
            Op:    "add",
            Path:  "/metadata/annotations",
            Value: map[string]string{},
        })
        existing = map[string]string{} // local shadow so logic below treats as present
    }

    for k, v := range added {
        esc := escapeJSONPointerSegment(k)
        //if cur, ok := existing[k]; !ok {
        if cur, ok := existing[k]; !ok || cur == "" {
            // Key absent -> add that single key
            patch = append(patch, patchOperation{
                Op:    "add",
                Path:  "/metadata/annotations/" + esc,
                Value: v,
            })
        } else if cur != v {
            // Key present but different -> replace just that key
            patch = append(patch, patchOperation{
                Op:    "replace",
                Path:  "/metadata/annotations/" + esc,
                Value: v,
            })
        }
        // If cur == v, no-op: avoid noisy patches.
    }

    return patch
}


## Deploy

1. Create a signed cert/key pair and store it in a Kubernetes `secret` that will be consumed by dnsconfig deployment

```
./scripts/create-signed-cert.sh
```

2. Create the `MutatingWebhookConfiguration` by set `caBundle` with correct value from Kubernetes cluster

```
./scripts/patch-ca-bundle.sh
```

3. Deploy resources

```
kubectl apply -R -f k8s/

# #k8s/configmap.yaml  #custom DNSConfig
# apiVersion: v1
# kind: ConfigMap
# metadata:
#   name: dnsconfig-injector-webhook-configmap
#   namespace: kube-system
# data:
#   dnsconfig.yaml: |
#     nameservers:
#       - 1.2.3.4
#     searches:
#       - my.dns.search.suffix
#     options:
#       - name: ndots
#         value: "2"
#       - name: edns0
# dnsconfig-injector (master *)
```

## Verify

1. The dsnconfig inject webhook should be running
```
# kubectl -n kube-system get pods
NAME                                                  READY     STATUS    RESTARTS   AGE
dnsconfig-injector-webhook-deployment-bbb689d69-882dd   1/1       Running   0          5m
```

2. Label the default namespace with `dnsconfig-injector=enabled`
```
# kubectl label namespace default dnsconfig-injector=enabled
# kubectl get namespace --show-labels
NAME          STATUS    AGE       LABELS
default       Active    18h       dnsconfig-injector=enabled
kube-public   Active    18h
kube-system   Active    18h
```

3. Deploy an app in Kubernetes cluster, take `sleep` app as an example
```
# cat <<EOF | kubectl create -f -
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: sleep
    spec:
      containers:
      - name: sleep
        image: tutum/curl
        command: ["/bin/sleep","infinity"]
EOF
```

4. Verify that the dnsconfig has been injected
```
# kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
sleep-7c97fff775-fbb8w   1/1     Running   0          36s

# kubectl exec -it sleep-7c97fff775-fbb8w -- cat /etc/resolv.conf
nameserver 10.100.200.10
nameserver 1.2.3.4
search default.svc.cluster.local svc.cluster.local cluster.local my.dns.search.suffix
options ndots:2 edns0
```
