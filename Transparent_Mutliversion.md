# Transparent Multicluster with multi version routing

This guide assumes that you already have a service mesh setup with [transparent failover](./Transparent_Multicluster.md).

Before we start ensure that we have a clean state and no interferring resources:

```bash
kubectl --context=${CTX_CLUSTER_A} delete deploy,dr,svc,se --all
kubectl --context=${CTX_CLUSTER_B} delete deploy,dr,svc,se --all
```

## Demo

Like in the transparent failover we will use a simple application:

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f  <(sed "s/REPLACEME/${CTX_CLUSTER_A}/g" manifests/transparent_multiversion/service.yml)
kubectl --context=${CTX_CLUSTER_B} apply -f  <(sed "s/REPLACEME/${CTX_CLUSTER_B}/g" manifests/transparent_multiversion/service.yml)
```

And as a client service we deploy a simple [curl image](https://hub.docker.com/r/curlimages/curl):

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f  manifests/transparent_multiversion/sleep.yml
kubectl --context=${CTX_CLUSTER_B} apply -f  manifests/transparent_multiversion/sleep.yml
```

Let's try to curl our application:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
```

Since we didn't defined any rule how to route the traffic the calls are load balanced (round robin) to the endpoints and the different versions.
So let's define a `DestinationRule` and a `VirtualService`:

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f manifests/transparent_multiversion/dest-rule.yml -f manifests/transparent_multiversion/virtual-service.yml
kubectl --context=${CTX_CLUSTER_B} apply -f manifests/transparent_multiversion/dest-rule.yml -f manifests/transparent_multiversion/virtual-service.yml
```

Let's try the `curl` command again:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
```

and verify that version 2 is also in place:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl -H "end-user: Miss Piggy" --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v2, instance: call-me-maybe-v2-85c896f6b9-vgq7v
```

If we scale down verion 2 we will get some connection timeouts (since we didn't setup the failover yet):

```bash
$ kubectl --context=${CTX_CLUSTER_A} scale deploy call-me-maybe-v2 --replicas=0
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl -H "end-user: Miss Piggy" --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
no healthy upstreamno healthy upstreamno healthy upstreamno healthy upstreamno healthy upstream
```

The `no healthy upstream` indicates that the envoy proxy can't find any healthy endpoint to route thee traffic to.
So let's setup the (transparent) failover, like in the previous scenario we create a [ServiceEntry](https://istio.io/docs/reference/config/networking/service-entry):

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: call-me-maybe
spec:
  hosts:
  - call-me-maybe.default.svc.cluster.local
  ports:
  - name: http
    number: 80
    protocol: http
  resolution: STATIC
  location: MESH_INTERNAL
  endpoints:
  - address: $(kubectl --context=${CTX_CLUSTER_B} -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    locality: europe-west1/europe-west1-b
    labels:
      version: v1
    ports:
      http: 15443
  - address: $(kubectl --context=${CTX_CLUSTER_B} -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    locality: europe-west1/europe-west1-b
    labels:
      version: v2
    ports:
      http: 15443
EOF
```

One difference that you can see is that we specified the same external ingress gateway two times with the different labels.
If we run now the `curl` command from above a second time we will see a result from the external cluster:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl -H "end-user: Miss Piggy" --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b-v2, instance: call-me-maybe-v2-6c78c6f668-t965p
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b-v2, instance: call-me-maybe-v2-6c78c6f668-t965p
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b-v2, instance: call-me-maybe-v2-6c78c6f668-t965p
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b-v2, instance: call-me-maybe-v2-6c78c6f668-t965p
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b-v2, instance: call-me-maybe-v2-6c78c6f668-t965p

$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a-v1, instance: call-me-maybe-v1-bdd6444b7-5qw5b
```

## Visualize

For the Service Mesh visualization we use [Kiali](https://kiali.io).
Before we can access the dashboard we need to create an user and a password:

```bash
kubectl --context=${CTX_CLUSTER_A} -n istio-system create secret generic kiali --from-literal=username=admin --from-literal=passphrase=admin
```

Now we can use the `port-forward` command to access the dashboard:

```bash
kubectl --context=${CTX_CLUSTER_A} -n istio-system port-forward svc/kiali 20001:20001
```

Go to: [kiali](http://localhost:20001/kiali/) and enter use the user from above.

## Cleanup

```bash
kubectl --context=${CTX_CLUSTER_A} delete ns istio-system
kubectl --context=${CTX_CLUSTER_B} delete ns istio-system
# Remove the GKE clusters
gcloud container clusters delete cluster-a --zone=europe-north1-b
gcloud container clusters delete cluster-b --zone=europe-west1-b
```
