# Cockroachdb Bring Your Own Certs

This guide supplements [crdb-single-region-k8s-with-non-default-namespace](https://github.com/jhatcher9999/crdb-single-region-k8s-with-non-default-namespace/) to demonstrate using non-Kubernetes CA with a 3-node Tanzu Kubernetest cluster.


## Pre-requisites

To create custom certs, one can use cockroach cli or openssl. The following instructions uses cockroach cli which can be downloaded [here](https://www.cockroachlabs.com/docs/stable/install-cockroachdb-windows.html).

Also download tkgi and kubectl required versions for your Tanzu Kubernetes environment.

## Your Own Cert Creation
After deploying Tanzu Kubernetes cluster, retrieve config data. When prompting, provide password
```cmd
tkgi get-kubeconfig [your-tkgi-cluster-name] -a [your-tanzu-k8s-dns-or-ip] -k -u [user-id]
```
If you're using multiple clustesr, set default context for kubectl command
```cmd
kubectl config use-context tkgi-cluster-1
```
Create directory to store certs and keys
```cmd
mkdir certs my-safe-directory
```
Create CA, certs and keys
```cmd
cockroach cert create-ca --certs-dir=certs --ca-key=my-safe-directory/ca.key
```
Create cert for client root
```cmd
cockroach cert create-client root --certs-dir=certs --ca-key=my-safe-directory/ca.key
```
Create cert for nodes. Note the wildcard *.your-cockroachdb-ns.default.svc.cluster.local which corresponds to a namespace created by kubectl cmd (see below)
```cmd
cockroach cert create-node --certs-dir=certs --ca-key=my-safe-directory/ca.key localhost 127.0.0.1 cockroachdb-public cockroachdb-public.default cockroachdb-public.default.svc.cluster.local .cockroachdb *.cockroachdb.default *.your-cockroachdb-ns.default.svc.cluster.local
```

## Cert Uploading
Substitute your own namespace as required.
```cmd
kubectl  create namespace your-cockroachdb-ns
kubectl  get ns
```
Uploading your certs for joininig nodes
```cmd
kubectl create secret generic cockroachdb.client.root --from-file=certs -n your-cockroachdb-ns
```
```cmd
kubectl create secret generic cockroachdb.node --from-file=certs -n your-cockroachdb-ns
```
## Deploy statefulset cockroachdb nodes
Apply manifest to deploy statefulset
```cmd
kubectl  create -f cockroachdb-statefulset-secure-non-default-ns.yaml
```
Check pods for running status
```cmd
kubectl get pods -n non-default-namespac
```
To run SQL client first launch a pod that runs indefinitely with the cockroach binary inside it
```cmd
kubectl create -f client.yaml
```
Use built-in SQL client
```cmd
kubectl exec -it cockroachdb-client-secure -n cockroachdb -- ./cockroach sql --url="postgres://root@cockroachdb-public:26257/?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt"
```
Create, add data or launch Admin UI using [crdb-single-region-k8s-with-non-default-namespace](https://github.com/jhatcher9999/crdb-single-region-k8s-with-non-default-namespace/) instructions.