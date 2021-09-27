# Capsule Development Guide

## Prerequisites

### Tools

Make sure you have these tools installed:

- [Go 1.16+](https://golang.org/dl/)
- [OperatorSDK 1.7.2+](https://github.com/operator-framework/operator-sdk), or [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)
- [KinD](https://github.com/kubernetes-sigs/kind), or [k3d](https://k3d.io/), with kubectl
- [ngrok](https://ngrok.com/) (if you want to run locally with remote Kubernetes)
- [golangci-lint](https://github.com/golangci/golangci-lint)
- OpenSSL

### Kubernetes Cluster

A lightweight Kubernetes within your laptop can be very handy for Kubernetes-native development like Capsule.

#### By `k3d`

```sh
# Install K3d cli by brew in Mac, or your preferred way
$ brew install k3d

# Export your laptop's IP, e.g. retrieving it by: ifconfig
# Do change this IP to yours
$ export LAPTOP_HOST_IP=192.168.10.101

# Spin up a bare minimum cluster
# Refer to here for more options: https://k3d.io/v4.4.8/usage/commands/k3d_cluster_create/
$ k3d cluster create k3s-capsule --servers 1 --agents 1 --no-lb --k3s-server-arg --tls-san=${K8S_API_IP}

# This will create a cluster with 1 server and 1 worker node
$ kubectl get nodes
NAME                       STATUS   ROLES                  AGE     VERSION
k3d-k3s-capsule-server-0   Ready    control-plane,master   2m13s   v1.21.2+k3s1
k3d-k3s-capsule-agent-0    Ready    <none>                 2m3s    v1.21.2+k3s1

# Or 2 Docker containers if you view it from Docker perspective
$ docker ps
CONTAINER ID   IMAGE                      COMMAND                  CREATED          STATUS          PORTS                     NAMES
5c26ad840c62   rancher/k3s:v1.21.2-k3s1   "/bin/k3s agent"         53 seconds ago   Up 45 seconds                             k3d-k3s-capsule-agent-0
753998879b28   rancher/k3s:v1.21.2-k3s1   "/bin/k3s server --t…"   53 seconds ago   Up 51 seconds   0.0.0.0:49708->6443/tcp   k3d-k3s-capsule-server-0
```

#### By `kind`

```sh
# Prepare a kind config file with necessary customization
$ cat > kind.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "0.0.0.0"
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    metadata:
      name: config
    apiServer:
      certSANs:
      - localhost
      - 127.0.0.1
      - kubernetes
      - kubernetes.default.svc
      - kubernetes.default.svc.cluster.local
      - kind
      - 0.0.0.0
      - ${LAPTOP_HOST_IP}
- role: worker
EOF

# Spin up a bare minimum cluster with 1 master 1 worker node
$ kind create cluster --name kind-capsule --config kind.yaml

# This will create a cluster with 1 server and 1 worker node
$ kubectl get nodes
NAME                         STATUS   ROLES                  AGE   VERSION
kind-capsule-control-plane   Ready    control-plane,master   84s   v1.21.1
kind-capsule-worker          Ready    <none>                 56s   v1.21.1

# Or 2 Docker containers if you view it from Docker perspective
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                     NAMES
7b329fd3a838   kindest/node:v1.21.1   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   0.0.0.0:54894->6443/tcp   kind-capsule-control-plane
7d50f1633555   kindest/node:v1.21.1   "/usr/local/bin/entr…"   About a minute ago   Up About a minute                             kind-capsule-worker
```

## Folk & clone the repository

The `folk-clone-contribute-pr` flow is common for contributing to OSS projects like Kubernetes, Capsule.

Let's assume you've folked it into your GitHub namespace, say `myuser`, and then you can clone it with Git protocol.
Do remember to change the `myuser` to yours.

```sh
$ git clone git@github.com:myuser/capsule.git && cd capsule
```

It's a good practice to add the upsteam as the remote too so we can easily fetch and merge the upstream to our folk:

```sh
$ git remote add upstream https://github.com/clastix/capsule.git
$ git remote -vv
origin	git@github.com:myuser/capsule.git (fetch)
origin	git@github.com:myuser/capsule.git (push)
upstream	https://github.com/clastix/capsule.git (fetch)
upstream	https://github.com/clastix/capsule.git (push)
```

## Build & deploy Capsule

```sh
# Download the project dependencies
$ go mod download

# Build the Capsule image
$ make docker-build

# Retrieve the built image version
$ export CAPSULE_IMAGE_VESION=`docker images --format '{{.Tag}}' quay.io/clastix/capsule`

# If k3s, load the image into cluster by
$ k3d image import --cluster k3s-capsule capsule quay.io/clastix/capsule:${CAPSULE_IMAGE_VESION}
# If Kind, load the image into cluster by
$ kind load docker-image --name kind-capsule quay.io/clastix/capsule:${CAPSULE_IMAGE_VESION}

# deploy all the required manifests
# Note: 1) please retry if you saw errors; 2) if you want to clean it up first, run: make remove
$ make deploy

# Make sure the controller is running
$ kubectl get pod -n capsule-system
NAME                                          READY   STATUS    RESTARTS   AGE
capsule-controller-manager-5c6b8445cf-566dc   1/1     Running   0          23s

# Check the logs if needed
$ kubectl -n capsule-system logs --all-containers -l control-plane=controller-manager

# You may have a try to deploy a Tenant too to make sure it works end to end
$ kubectl apply -f - <<EOF
apiVersion: capsule.clastix.io/v1beta1
kind: Tenant
metadata:
  name: oil
spec:
  owners:
  - name: alice
    kind: User
EOF

# There shouldn't be any errors and you should see the newly created tenant
$ kubectl get tenants
NAME   STATE    NAMESPACE QUOTA   NAMESPACE COUNT   NODE SELECTOR   AGE
oil    Active                     0                                 14s
```

## Scaling down the deployed Pod

As of now, a complete env has been set up in `kind` or `k3d` powred cluster, and the `capsule-controller-manager` is running as a deployment serving as:

- The reconcilers for CRDs and;
- A series of webhooks 

During development, we prefer the code is running within our IDE locally, so the `capsule-controller-manager` running in the Kubernetes will competing with us.

To avoid that, we need to scale the existing replicas of `capsule-controller-manager` to 0:

```sh
$ kubectl -n capsule-system scale deployment capsule-controller-manager --replicas=0
deployment.apps/capsule-controller-manager scaled
```

## Preparing TLS certificate for webhooks

Running webhooks requires TLS, so let's prepare the TLS key pair in our development env to handle HTTPS requests.

```sh
# Create this dir to mimic the Pod mount point
mkdir -p /tmp/k8s-webhook-server/serving-certs

# Generate cert/key under /tmp/k8s-webhook-server/
openssl req -newkey rsa:4096 -days 3650 -nodes -x509 \
  -subj "/C=SG/ST=SG/L=SG/O=CAPSULE/CN=CAPSULE" \
  -extensions SAN \
  -config <( cat $( [[ "Darwin" -eq "$(uname -s)" ]]  && echo /System/Library/OpenSSL/openssl.cnf || echo /etc/ssl/openssl.cnf  ) \
    <(printf "[ SAN ]\nsubjectAltName = \"IP:${LAPTOP_HOST_IP}\"")) \
  -keyout /tmp/k8s-webhook-server/serving-certs/tls.key \
  -out /tmp/k8s-webhook-server/serving-certs/tls.crt
```

Do note that there is an [known issue in controller-runtime, #900](https://github.com/kubernetes-sigs/controller-runtime/issues/900) when you now try to run the code locally:

```sh
$ go run .
```

As you will get errors:
```log
{"level":"error","ts":"2021-09-26T20:42:23.165+0800","logger":"setup","msg":"problem running manager","error":"[open /var/folders/d2/wlg1hjdj29z9vxc2lhndp8m80000gn/T/k8s-webhook-server/serving-certs/tls.crt: no such file or directory, failed waiting for all runnables to end within grace period of 30s: context deadline exceeded]","errorCauses":[{"error":"open /var/folders/d2/wlg1hjdj29z9vxc2lhndp8m80000gn/T/k8s-webhook-server/serving-certs/tls.crt: no such file or directory"},
...
exit status 1
```

And the workaround is to explicitly set `TMPDIR=/tmp/`:

```sh
$ TMPDIR=/tmp/ go run .
```

## Patching the Webhooks

And we need to _delegate_ the controllers' and wehbooks' services to the code running in our IDE.

So we need to patch the `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` objects with the right pointers.

```sh
# Export your laptop's IP with the 9443 port exposed by controllers/webhooks' services
$ export WEBHOOK_URL="https://${LAPTOP_HOST_IP}:9443"

# Trust the cert we generated for webhook TLS
$ export CA_BUNDLE=`openssl base64 -in /tmp/k8s-webhook-server/serving-certs/tls.crt | tr -d '\n'`

# Patch the MutatingWebhookConfiguration
$ kubectl patch MutatingWebhookConfiguration capsule-mutating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/0/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/mutate-v1-namespace-owner-reference\",'caBundle':\"${CA_BUNDLE}\"}}]"

# We can verify it by command
$ kubectl get MutatingWebhookConfiguration capsule-mutating-webhook-configuration -o yaml

# Patch the ValidatingWebhookConfiguration
# Note: there is a list of validating webhook endpoints, not just one
$ kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/0/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/cordoning\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/1/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/ingresses\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/2/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/namespaces\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/3/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/networkpolicies\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/4/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/pods\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/5/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/persistentvolumeclaims\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/6/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/services\",'caBundle':\"${CA_BUNDLE}\"}}]" && \
  kubectl patch ValidatingWebhookConfiguration capsule-validating-webhook-configuration \
    --type='json' -p="[{'op': 'replace', 'path': '/webhooks/7/clientConfig', 'value':{'url':\"${WEBHOOK_URL}/tenants\",'caBundle':\"${CA_BUNDLE}\"}}]"

# We can verify it by command
$ kubectl get ValidatingWebhookConfiguration capsule-validating-webhook-configuration -o yaml
```

## Run Capsule

```sh
$ export NAMESPACE=capsule-system
$ make run
```

## Work in your prefered IDE

Now it's time to work through our familiar inner development loop for code contribution.

For example, if you're using [Visual Studio Code](https://code.visualstudio.com), this `launch.json` file can be a good start.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Launch",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceFolder}",
            "args": [
                "--zap-encoder=console",
                "--zap-log-level=debug",
                "--configuration-name=capsule-default"
            ],
            "env": {
                "NAMESPACE": "capsule-system",
                "TMPDIR": "/tmp/"
            }
        }
    ]
}
```