In this video we move ahead with Kubernetes concepts 

## Kubernetes Architecture 
First we will discuss Kubernetes Architecture and try to understand what happens under the hood when you run `kubectl run nginx --image=nginx`

## Create CSR
openssl genrsa -out shahebaz.key 2048
openssl req -new -key shahebaz.key -out shahebaz.csr -subj "/CN=shahebaz/O=group1"

## Sign CSE with Kubernetes CA
cat shahebaz.csr | base64 | tr -d '\n'

```
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: saiyam
spec:
  request: BASE64_CSR
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
```
kubectl apply -f csr.yaml
kubectl certificate approve shahebaz

kubectl get csr shahebaz -o jsonpath='{.status.certificate}' | base64 --decode > shahebaz.crt

## Role and role binding
```
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: shahebaz
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
### setup kubeconfig
kubectl config set-credentials shahebaz --client-certificate=shahebaz.crt --client-key=shahebaz.key
kubectl config get-contexts
kubectl config set-context shahebaz-context --cluster=kubernetes --namespace=default --user=shahebaz
kubectl config use-context shahebaz-context


### Merging multiple KubeConfig files
export KUBECONFIG=/path/to/first/config:/path/to/second/config:/path/to/third/config



========================================

Create a file deploy.json
``` 
kubectl create deployment nginx --image=nginx --dry-run=client -o json > deploy.json
kubectl run nginx --image=nginx --dry-run=client -o json

```

SA creation
```
kubectl create serviceaccount sam --namespace default
kubectl create clusterrolebinding sam-clusteradmin-binding --clusterrole=cluster-admin --serviceaccount=default:sam
kubectl create token sam
TOKEN=outputfromabove
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
List deployments
curl -X GET $APISERVER/apis/apps/v1/namespaces/default/deployments -H "Authorization: Bearer $TOKEN" -k
Create Deployment
curl -X POST $APISERVER/apis/apps/v1/namespaces/default/deployments \
  -H "Authorization: Bearer $TOKEN" \
  -H 'Content-Type: application/json' \
  -d @deploy.json \
  -k

List pods 
curl -X GET $APISERVER/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -k  
```
