# How I installed kubernetes and argo on macbook air m1

## Rancher Desktop

Installed Rancher Desktop for mac [version 1.2.1](https://github.com/rancher-sandbox/rancher-desktop/releases/download/v1.2.1/Rancher.Desktop-1.2.1.aarch64.dmg) from [rancherdesktop.io](https://rancherdesktop.io/)

Inside Rancher Desktop GUI, selected latest stable kubernetes version (1.23.5) from a dropdown which got downloaded & installed automatically in the background (nice:)

Changed ownership of dir `/usr/local/bin` recursively to my local mac user which is handy for Rancher Desktop's symbolic links that need to be created there.

A clean one-node cluster `rancher-desktop` is now installed.

A multi-node cluster can also be installed following [this guide](https://docs.rancherdesktop.io/how-to-guides/create-multi-node-cluster)

## Argo Workflows simplest all-in-one install

```bash
# Install latest Argo Workflows from master branch
# incl. Minio for Artifacts, Postgres DB with auto archiving finished workflows set up.

kubectl create namespace argo
kubectl apply -n argo -f https://raw.githubusercontent.com/argoproj/argo-workflows/master/manifests/quick-start-postgres.yaml

# Create service account in namespace `argo`. 
kubectl -n argo create sa eduard
# Created a clusterrolebinding to be able to do anything in Argo GUI as eduard.
kubectl create clusterrolebinding eduard --clusterrole=cluster-admin --serviceaccount=argo:eduard

SECRET=$(kubectl -n argo get sa eduard -o=jsonpath='{.secrets[0].name}')
ARGO_TOKEN="Bearer $(kubectl -n argo get secret $SECRET -o=jsonpath='{.data.token}' | base64 --decode)"

# Print out the TOKEN
echo "\nCopy paste the following to the Argo GUI Login field:\n"$(printf "%0.s-" {1..53}) "\n${ARGO_TOKEN}\n"$(printf "%0.s-" {1..53})

# IN DIFFERENT TERMINAL: Set up port forwarding to make Argo GUI available at https://localhost:2746/workflows
kubectl -n argo port-forward deployment/argo-server 2746:2746

# IN YET ANOTHER TERMINAL: Set up port forwarding for minio to make its GUI available at http://localhost:9001
kubectl -n argo port-forward deployment/minio 9001:9001
```

## S3 storage artifacts examples

```bash
# Submit the workflow and tail print logs until finished
argo submit --log s3-example-wf.yaml

<img width="884" alt="workflow-producing-and-consuming-artifact-on-s3-storage" src="https://user-images.githubusercontent.com/752688/170572591-58cb8628-cad2-464b-ab18-1ec3a620d079.png">

# Also worth exploring advanced artifacts example...
kubectl -n argo apply -f https://raw.githubusercontent.com/argoproj/argo-workflows/master/examples/artifacts-workflowtemplate.yaml

# Submit it after uploading the workflow template to argo server
argo submit --watch --from wftmpl/artifacts
```

## Minio via helm (alternative way to quick-start manifests)

NOTE: MinIO is already included in the quick-start manifests.

The following steps only necessary if Argo Workflows were not installed via quick-start manifests.

Follow Argo Workflows docs to install Minio [here](https://argoproj.github.io/argo-workflows/configure-artifact-repository/#configuring-minio) to have Artifacts configured.

```bash
brew install helm # mac, helm 3.x
helm repo add minio https://helm.min.io/ # official minio Helm charts
helm repo update
helm install argo-artifacts minio/minio --set service.type=LoadBalancer --set fullnameOverride=argo-artifacts
```

Then follow the output of the `helm install` command.

```bash
brew install minio/stable/mc
mc --help

ACCESS_KEY=$(kubectl get secret argo-artifacts --namespace argo -o jsonpath="{.data.accesskey}" | base64 --decode)
SECRET_KEY=$(kubectl get secret argo-artifacts --namespace argo -o jsonpath="{.data.secretkey}" | base64 --decode)

mc alias set argo-artifacts http://<EXTERNAL_IP_OF_MINIO_SVc>:9000 "$ACCESS_KEY" "$SECRET_KEY" --api s3v4

mc ls argo-artifacts
```
