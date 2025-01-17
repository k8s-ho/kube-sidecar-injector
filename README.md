# kube-sidecar-injector

This repo is used for [a tutorial at Medium](https://medium.com/ibm-cloud/diving-into-kubernetes-mutatingadmissionwebhook-6ef3c5695f74) to create a Kubernetes [MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) that injects a nginx sidecar container into pod prior to persistence of the object.

## Prerequisites

- [git](https://git-scm.com/downloads)
- [go](https://golang.org/dl/) version v1.17+
- [docker](https://docs.docker.com/install/) version 19.03+
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) version v1.19+
- Access to a Kubernetes v1.19+ cluster with the `admissionregistration.k8s.io/v1` API enabled. Verify that by the following command:

```
kubectl api-versions | grep admissionregistration.k8s.io
```
The result should be:
```
admissionregistration.k8s.io/v1
admissionregistration.k8s.io/v1beta1
```

> Note: In addition, the `MutatingAdmissionWebhook` and `ValidatingAdmissionWebhook` admission controllers should be added and listed in the correct order in the admission-control flag of kube-apiserver.

## Preparation before build and deployment(k8sho add)
install docker & gcc & golang1.17 & make + setup
```bash
git clone https://github.com/morvencao/kube-sidecar-injector  
apt update  
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin  
systemctl start docker  

wget https://golang.org/dl/go1.17.linux-amd64.tar.gz  
sudo tar -xvf go1.17.linux-amd64.tar.gz  
sudo mv go /usr/local  
export PATH=$PATH:/usr/local/go/bin  
go version  
apt install make -y  
apt install gcc -y  

docker login
```
## Build and Deploy

1. Build and push docker image:

```bash
make docker-build docker-push IMAGE=k8s/sidecar-injector-test:latest
```

2. Deploy the kube-sidecar-injector to kubernetes cluster:

```bash
make deploy IMAGE=k8s/sidecar-injector-test:latest
```

3. Verify the kube-sidecar-injector is up and running:

```bash
# kubectl -n sidecar-injector get pod
# kubectl -n sidecar-injector get pod
NAME                                READY   STATUS    RESTARTS   AGE
sidecar-injector-7c8bc5f4c9-28c84   1/1     Running   0          30s
```

## How to use

1. Create a new namespace `test-ns` and label it with `sidecar-injector=enabled`:

```
# kubectl create ns test-ns
# kubectl label namespace test-ns sidecar-injection=enabled
# kubectl get namespace -L sidecar-injection
NAME                 STATUS   AGE   SIDECAR-INJECTION
default              Active   26m
test-ns              Active   13s   enabled
kube-public          Active   26m
kube-system          Active   26m
sidecar-injector     Active   17m
```

2. Deploy an app in Kubernetes cluster, take `alpine` app as an example

```bash
kubectl -n test-ns run alpine \
    --image=alpine \
    --restart=Never \
    --command -- sleep infinity
```

3. Verify sidecar container is injected:

```
# kubectl -n test-ns get pod
NAME                     READY     STATUS        RESTARTS   AGE
alpine                   2/2       Running       0          10s
# kubectl -n test-ns get pod alpine -o jsonpath="{.spec.containers[*].name}"
alpine sidecar-nginx
```

## Troubleshooting

Sometimes you may find that pod is injected with sidecar container as expected, check the following items:

1. The sidecar-injector pod is in running state and no error logs.
2. The namespace in which application pod is deployed has the correct labels(`sidecar-injector=enabled`) as configured in `mutatingwebhookconfiguration`.
3. Check if the application pod has annotation `sidecar-injector-webhook.morven.me/inject:"yes"`.
