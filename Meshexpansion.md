# Istio Multi Network Mesh

Inspired by the [Istio official docs](https://istio.io/docs/examples/virtual-machines/multi-network)

## Use a flat CA (no intermediate)

### Kubernetes and Istio Setup

```bash
gcloud container clusters create mesh-cluster \
  --cluster-version latest \
  --num-nodes 4 \
  --machine-type n1-standard-8 \
  --services-ipv4-cidr "172.16.0.0/16"

gcloud container clusters get-credentials mesh-cluster

kubectl create clusterrolebinding cluster-admin-binding \
    --clusterrole=cluster-admin \
    --user=$(gcloud config get-value core/account)
```

Install Istio:

```bash
curl -sLO https://github.com/istio/istio/releases/download/1.4.2/istio-1.4.2-osx.tar.gz
tar xfvz istio-1.4.2-osx.tar.gz
# Create the istio-system namespace
kubectl create namespace istio-system
# Apply the samples certificates
kubectl create secret generic cacerts -n istio-system \
    --from-file=samples/certs/ca-cert.pem \
    --from-file=samples/certs/ca-key.pem \
    --from-file=samples/certs/root-cert.pem \
    --from-file=samples/certs/cert-chain.pem
# Install with the operator
istioctl manifest generate \
    -f install/kubernetes/operator/examples/vm/values-istio-meshexpansion-gateways.yaml \
    --set coreDNS.enabled=true --set tag=1.4.2 --set controlPlaneSecurityEnabled=true > istio.yaml
# Apply manifests
istioctl manifest apply -f istio.yml
# Verify Installation
istioctl verify-install -f istio.yml
```

Now we can prepare a `Service` and a `ServiceAccount` for the VM:

```bash
# The setup will be in this namespace
kubectl create ns expandvm
kubectl -n expandvm create sa expandvm
```

Fetch the certificates:

```bash
export SERVICE_NAMESPACE="expandvm"
mkdir -p expandvm
kubectl -n $SERVICE_NAMESPACE get secret istio.expandvm  \
    -o jsonpath='{.data.root-cert\.pem}' | base64 --decode > expandvm/root-cert.pem
kubectl -n $SERVICE_NAMESPACE get secret istio.expandvm  \
    -o jsonpath='{.data.key\.pem}' | base64 --decode > expandvm/key.pem
kubectl -n $SERVICE_NAMESPACE get secret istio.expandvm  \
      -o jsonpath='{.data.cert-chain\.pem}' | base64 --decode > expandvm/cert-chain.pem
```

Fetch some additional information required to join the mesh:

```bash
kubectl get -n istio-system service istio-ingressgateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}' > expandvm/gwip.txt
cat << EOF> expandvm/cluster.env
ISTIO_CP_AUTH=MUTUAL_TLS
ISTIO_SERVICE=expandvm
ISTIO_SERVICE_CIDR=172.16.0.0/16
ISTIO_INBOUND_PORTS=80
ISTIO_NAMESPACE=expandvm
EOF
```

Finally `register` the vm:

```bash
istioctl register expandvm -s expandvm -n expandvm $(gcloud compute instances describe expand-vm --format=json | jq -r ".networkInterfaces[].accessConfigs[].natIP") 80
2019-12-12T18:48:07.180840Z	info	Registering for service 'expandvm' ip '35.187.36.54', ports list [{80 http}]
2019-12-12T18:48:07.180867Z	info	0 labels ([]) and 1 annotations ([alpha.istio.io/kubernetes-serviceaccounts=expandvm])
2019-12-12T18:48:07.577906Z	info	Before: found endpoints &Endpoints{ObjectMeta:{expandvm  expandvm /api/v1/namespaces/expandvm/endpoints/expandvm e9c7ce72-1d0f-11ea-96e9-42010a840013 9856 0 2019-12-12 19:47:57 +0100 CET <nil> <nil> map[] map[alpha.istio.io/kubernetes-serviceaccounts:expandvm] [] []  []},Subsets:[]EndpointSubset{},}
2019-12-12T18:48:07.577943Z	info	No pre existing exact matching ports list found, created new subset {[{35.187.36.54  <nil> nil}] [] [{http 80 }]}
2019-12-12T18:48:07.619468Z	info	Successfully updated expandvm, now with 1 endpoints
2019-12-12T18:48:07.619944Z	info	Details: &Endpoints{ObjectMeta:{expandvm  expandvm /api/v1/namespaces/expandvm/endpoints/expandvm e9c7ce72-1d0f-11ea-96e9-42010a840013 9893 0 2019-12-12 19:47:57 +0100 CET <nil> <nil> map[] map[alpha.istio.io/kubernetes-serviceaccounts:expandvm] [] []  []},Subsets:[]EndpointSubset{EndpointSubset{Addresses:[]EndpointAddress{EndpointAddress{IP:35.187.36.54,TargetRef:nil,Hostname:,NodeName:nil,},},NotReadyAddresses:[]EndpointAddress{},Ports:[]EndpointPort{EndpointPort{Name:http,Port:80,Protocol:TCP,},},},},}
```

### VM Setup

Now we can create a VM to expand the mesh:

```bash
gcloud compute instances create expand-vm \
    --image-family ubuntu-1804-lts \
    --image-project ubuntu-os-cloud
# Copy the local files to the machine
gcloud compute scp --recurse ./expandvm ubuntu@expand-vm:/tmp
# SSH into the machine
gcloud compute ssh ubuntu@expand-vm
```

Now we can install Istio on the VM:

```bash
curl -L https://storage.googleapis.com/istio-release/releases/1.4.2/deb/istio-sidecar.deb > istio-sidecar.deb
sudo dpkg -i istio-sidecar.deb
```

Set some DNS entries for the istio-system services:

```bash
export GWIP=$(cat /tmp/expandvm/gwip.txt)
sudo tee -a /etc/hosts <<EOF
${GWIP} istio-citadel istio-citadel.istio-system
${GWIP} istio-pilot istio-pilot.istio-system
${GWIP} istio-telemetry istio-telemetry.istio-system
${GWIP} istio-policy istio-policy.istio-system
EOF
```

Copy the certificates:

```bash
sudo mkdir -p /etc/certs
sudo cp /tmp/expandvm/{root-cert.pem,cert-chain.pem,key.pem} /etc/certs
```

and set the `cluster.env`:

```bash
sudo cp /tmp/expandvm/cluster.env /var/lib/istio/envoy
```

Set the ownership for `istio-proxy`:

```bash
sudo chown -R istio-proxy /etc/certs /var/lib/istio/envoy
```

Verify `node-agent`:

```bash
$ sudo /usr/local/bin/istio-node-agent-start.sh
2019-12-12T19:44:00.757011Z	info	parsed scheme: ""
2019-12-12T19:44:00.757345Z	info	scheme "" not registered, fallback to default scheme
2019-12-12T19:44:00.757501Z	info	ccResolverWrapper: sending update to cc: {[{istio-citadel:8060 0  <nil>}] <nil>}
2019-12-12T19:44:00.757630Z	info	ClientConn switching balancer to "pick_first"
2019-12-12T19:44:00.757778Z	info	Starting Node Agent
2019-12-12T19:44:00.757901Z	info	Node Agent starts successfully.
2019-12-12T19:44:00.758067Z	info	pickfirstBalancer: HandleSubConnStateChange: 0xc000112ff0, CONNECTING
2019-12-12T19:44:00.985501Z	info	Sending CSR (retrial #0) ...
2019-12-12T19:44:01.223527Z	info	pickfirstBalancer: HandleSubConnStateChange: 0xc000112ff0, READY
2019-12-12T19:44:01.228440Z	info	CSR is approved successfully. Will renew cert in 1079h59m59.771856842s
```

## Cleanup

```bash
# Ensure we prune everything and don't leak resources
kubectl delete ns --all
# Destroy the cluster
echo "Y" | gcloud container clusters delete mesh-cluster --async
# Destroy the VM
echo "Y" | gcloud compute instances delete expand-vm  --delete-disks=all
```

## Gotchas

The new `istioctl manifest` operator seem to not install the default DestinationRule (for mTLS) which means we need to add these to DestinationRules if we enable mTLS STRICT:

```bash
kubectl apply -n istio-system -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: default
spec:
  host: "*.local"
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
EOF

kubectl apply -n istio-system -f - << EOF
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: meshexpansion-dr-citadel
  namespace: istio-system
  labels:
    istio: citadel
spec:
  host: istio-citadel.istio-system.svc.cluster.local
  trafficPolicy:
    portLevelSettings:
    - port:
        number: 8060
      tls:
        mode: DISABLE
EOF
```
