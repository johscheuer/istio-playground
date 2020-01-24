# Transparent Multicluster

This guide is oriented on: https://istio.io/docs/setup/install/multicluster/gateways/

## Prerequisites

For the multi cluster demo we need at least two Kubernetes clusters:

Create cluster-a:

```bash
gcloud container clusters create cluster-a \
  --cluster-version latest \
  --num-nodes 4 \
  --machine-type=n1-standard-2 \
  --zone=europe-north1-b

gcloud container clusters get-credentials --zone=europe-north1-b cluster-a

kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)

# Store the context for a later use
export CTX_CLUSTER_A=$(kubectl config current-context)

# Quick check of the Kubernetes cluster
kubectl --context=${CTX_CLUSTER_A} get nodes -L failure-domain.beta.kubernetes.io/region
```

and create cluster-b:

```bash
gcloud container clusters create cluster-b \
  --cluster-version latest \
  --num-nodes 4 \
  --machine-type n1-standard-2 \
  --zone=europe-west1-b

gcloud container clusters get-credentials --zone=europe-west1-b cluster-b

kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)

# Store the context for a later use
export CTX_CLUSTER_B=$(kubectl config current-context)

# Quick check of the Kubernetes cluster
kubectl --context=${CTX_CLUSTER_B} get nodes -L failure-domain.beta.kubernetes.io/region
```

## Install Istio

At first we need the according binaries and helm charts:

```bash
export ISTIO_VERSION="1.4.3"
curl -Lo istio.tar.gz "https://github.com/istio/istio/releases/download/${ISTIO_VERSION}/istio-${ISTIO_VERSION}-osx.tar.gz"
tar xfz istio.tar.gz
rm istio.tar.gz
```

In order to be able to use `istioctl` move the binary to your `$PATH` or set the according variable:

```bash
export PATH="$PATH:$(pwd)/istio-${ISTIO_VERSION}/bin"
```

Now we can verify if our cluster meets all requirements:

```bash
istioctl --context=${CTX_CLUSTER_A} verify-install
istioctl --context=${CTX_CLUSTER_B} verify-install
```

In the next step we can create the `istio-system` namespace:

```bash
kubectl --context=${CTX_CLUSTER_A} create namespace istio-system
kubectl --context=${CTX_CLUSTER_B} create namespace istio-system
```

Now we can install the Istio CRD's:

```bash
kubectl --context=${CTX_CLUSTER_A} -n istio-system apply -f <(helm template istio-init istio-${ISTIO_VERSION}/install/kubernetes/helm/istio-init -n istio-system -f ./configs/mutlicluster.yml)
kubectl --context=${CTX_CLUSTER_B} -n istio-system apply -f <(helm template istio-init istio-${ISTIO_VERSION}/install/kubernetes/helm/istio-init -n istio-system -f ./configs/mutlicluster.yml)
```

Check that the installation is done:

```bash
kubectl --context=${CTX_CLUSTER_A}  -n istio-system get job
NAME                COMPLETIONS   DURATION   AGE
istio-init-crd-10   1/1           14s        2m13s
istio-init-crd-11   1/1           12s        2m13s
istio-init-crd-12   1/1           12s        2m12s
```

Before we install Istio we need to create the certificates (in a production setup you would use sub-CA's):

```bash
kubectl --context=${CTX_CLUSTER_A} create secret generic cacerts -n istio-system \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/ca-cert.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/ca-key.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/root-cert.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/cert-chain.pem

kubectl --context=${CTX_CLUSTER_B} create secret generic cacerts -n istio-system \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/ca-cert.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/ca-key.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/root-cert.pem \
    --from-file=istio-${ISTIO_VERSION}/samples/certs/cert-chain.pem
```

Now we ca install Istio:

```bash
helm template istio istio-${ISTIO_VERSION}/install/kubernetes/helm/istio --namespace istio-system -f ./configs/mutlicluster.yml > istio.yaml
kubectl --context=${CTX_CLUSTER_A} -n istio-system apply -f istio.yaml
kubectl --context=${CTX_CLUSTER_B} -n istio-system apply -f istio.yaml
```

And finally check that everything is running:

```bash
$ istioctl --context=${CTX_CLUSTER_A} verify-install -f istio.yaml
...
Checked 0 crds
Checked 7 Istio Deployments
Istio is installed successfully

$ istioctl --context=${CTX_CLUSTER_B} verify-install -f istio.yaml
...
Checked 0 crds
Checked 7 Istio Deployments
Istio is installed successfully
```

## Adjustments for transparent failover

In order to make the transparent failover work we need to adjust the multicluster ingress gateway to also have `"*.svc.cluster.local"` in the hosts spec:

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f manifests/transparent_multicluster/gateway.yml
kubectl --context=${CTX_CLUSTER_B} apply -f manifests/transparent_multicluster/gateway.yml
```

## Demo

Since everything is now setup we will deploy a [small application](https://github.com/istio/istio/tree/master/samples/helloworld) and see how the failover works.
In the first step we need to enable Istio for the default namespace:

```bash
kubectl --context=${CTX_CLUSTER_A} label namespace default istio-injection=enabled
kubectl --context=${CTX_CLUSTER_B} label namespace default istio-injection=enabled
```

Now we can deploy a simple echo application:

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f  <(sed "s/REPLACEME/${CTX_CLUSTER_A}/g" manifests/transparent_multicluster/service.yml)
kubectl --context=${CTX_CLUSTER_B} apply -f  <(sed "s/REPLACEME/${CTX_CLUSTER_B}/g" manifests/transparent_multicluster/service.yml)
```

And as a client service we deploy a simple [curl image](https://hub.docker.com/r/curlimages/curl):

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f  manifests/transparent_multicluster/sleep.yml
kubectl --context=${CTX_CLUSTER_B} apply -f  manifests/transparent_multicluster/sleep.yml
```

Let's try to curl our application:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
```

So the calls inside of our mesh are working. Now we need to configure Istio to also make calls to the remote cluster. For this we will create a [ServiceEntry](https://istio.io/docs/reference/config/networking/service-entry):

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
  - address: $(kubectl ${CTX_CLUSTER_B} -n istio-system get svc istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
    locality: europe-west1/europe-west1-b
    ports:
      http: 15443
EOF
```

Let's see what happens now if we curl the service:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
```

Now we see that we make calls to the internal endpoint and the remote endpoint (currently this is just round robin).
With `istioctl` we can take a look at the endpoints of a specific cluster:

```bash
$ istioctl --context=${CTX_CLUSTER_A} pc ep $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}').default --cluster='outbound|80||call-me-maybe.default.svc.cluster.local'
ENDPOINT               STATUS      OUTLIER CHECK     CLUSTER
10.48.3.10:5000        HEALTHY     OK                outbound|80||call-me-maybe.default.svc.cluster.local
35.205.57.99:15443     HEALTHY     OK                outbound|80||call-me-maybe.default.svc.cluster.local
```

The endpoint `10.48.3.10:5000` is the internal endpoint and `35.205.57.99:15443` is the external endpoint which we created with the `ServiceEntry`.
This is quite nice because our calling service can just call `call-me-maybe.default.svc.cluster.local` and has the benefit of multicluster.
In order to achieve only remote calls for failed services we need to activate/use the [locality load balancing](https://istio.io/docs/ops/configuration/traffic-management/locality-load-balancing).
We can activate the locality load balancing with the following [Destination Rule](https://istio.io/docs/reference/config/networking/destination-rule/):

```bash
kubectl --context=${CTX_CLUSTER_A} apply -f manifests/transparent_multicluster/dest-rule.yml
```

If we call the service again we can see all calls are to the internal endpoint:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
Hello version: gke_pg-tm-cloud_europe-north1-b_cluster-a, instance: call-me-maybe-7c76668bf5-xfn29
```

Let's remove the internal endpoint and see what will happen:

```bash
kubectl --context=${CTX_CLUSTER_A} scale deploy call-me-maybe --replicas=0
```

Since there is no endpoint available all calls are now routed to the remote cluster:

```bash
$ kubectl --context=${CTX_CLUSTER_A} exec -ti $(kubectl --context=${CTX_CLUSTER_A} get po -l app=sleep-svc -o jsonpath='{.items[0].metadata.name}') -c sleep -- /bin/sh -c 'for i in 0 1 2 3 4; do curl --connect-timeout 1 call-me-maybe/hello; sleep 0.5; done'
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
Hello version: gke_pg-tm-cloud_europe-west1-b_cluster-b, instance: call-me-maybe-947cdd654-49lc9
```

You can also add multiple remote clusters by adding multiple endpoints in the `ServiceEntry`.

## Cleanup

```bash
kubectl --context=${CTX_CLUSTER_A} delete ns istio-system
kubectl --context=${CTX_CLUSTER_B} delete ns istio-system
# Remove the GKE clusters
gcloud container clusters delete cluster-a --zone=europe-north1-b
gcloud container clusters delete cluster-b --zone=europe-west1-b
```
