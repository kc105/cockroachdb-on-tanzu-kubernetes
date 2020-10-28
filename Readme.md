# Cockroachdb Bring-Your-Own-Certs

This guide supplements [crdb-single-region-k8s-with-non-default-namespace](https://github.com/jhatcher9999/crdb-single-region-k8s-with-non-default-namespace/) to demonstrate using non-Kubernetes CA with a 3-node Tanzu Kubernetest cluster.


## Pre-requisites

To create custom certs, one can use cockroach cli or openssl. The following instructions uses cockroach cli which can be downloaded [here](https://www.cockroachlabs.com/docs/stable/install-cockroachdb-windows.html).

Also download tkgi and kubectl required versions for your Tanzu Kubernetes environment.

## Your Own Cert Creation
After deploying Tanzu Kubernetes cluster, retrieve config data (when prompted, provide password to complete this step)
```cmd
tkgi get-kubeconfig [your-tkgi-cluster-name] -a [your-tanzu-k8s-dns-or-ip] -k -u [user-id]
```
If using multiple clusters, set default context for kubectl command
```cmd
kubectl config use-context [your-tkgi-cluster-name]
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
cockroach cert create-node --certs-dir=certs --ca-key=my-safe-directory/ca.key localhost 127.0.0.1 cockroachdb-public cockroachdb-public.default cockroachdb-public.default.svc.cluster.local .cockroachdb *.cockroachdb.default *.non-default-namespace.default.svc.cluster.local
```

## Cert Uploading
Substitute [your-namespace] with your own namepsace
```cmd
kubectl  create namespace [your-namespace]
kubectl  get ns
```
Uploading your certs for joining nodes
```cmd
kubectl create secret generic cockroachdb.client.root --from-file=certs -n your-namespace
```
```cmd
kubectl create secret generic cockroachdb.node --from-file=certs -n your-namespace
```
## Deploy statefulset cockroachdb nodes
Refer remaining steps to [crdb-single-region-k8s-with-non-default-namespace](https://github.com/jhatcher9999/crdb-single-region-k8s-with-non-default-namespace/) ...
Follow step #4 by creating individual manifest files or use this manifest to deploy statefulset. Be sure to change namespace in metadata.
```cmd
kubectl  create -f cockroachdb-statefulset-secure-non-default-ns.yaml
```
Skip steps #5, #6 & #7. Instead run following command to join cockroachdb nodes into cluster.
```cmd
kubectl exec -it cockroachdb-0 -n cockroachdb -- /cockroach/cockroach init --certs-dir=/cockroach/cockroach-certs
```
Check all pods for running status
```cmd
kubectl get pods -n non-default-namespac
```
Observe for the following, otherwise use kubectl logs command to see where the error is
```output
NAME                        READY   STATUS    RESTARTS   AGE
cockroachdb-0               1/1     Running   0          18h
cockroachdb-1               1/1     Running   0          41h
cockroachdb-2               1/1     Running   0          41h
cockroachdb-client-secure   1/1     Running   0          40h
```
To run SQL client first launch a pod that runs indefinitely with the cockroach binary inside it
```cmd
kubectl create -f client.yaml
```
Use built-in SQL client
```cmd
kubectl exec -it cockroachdb-client-secure -n cockroachdb -- ./cockroach sql --url="postgres://root@cockroachdb-public:26257/?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt"
```
Follow [crdb-single-region-k8s-with-non-default-namespace](https://github.com/jhatcher9999/crdb-single-region-k8s-with-non-default-namespace/) to create, add data or launch Admin UI.