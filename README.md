# Fluent Bit - Azure Log Analytics

> Setup Fluent Bit and Azure Log Analytics

This is an end-to-end walkthrough of setting up Fluent Bit for log forwarding from a Kubernetes cluster to Azure Log Analytics

The sample application generates JSON logs. Normal logs are written to stdout. Error logs are written to stderr.

```json

{"date":"2020-12-28T21:19:06.1347849Z","statusCode":200,"path":"/log/app","duration":78,"value":"HWIkixicjA"}
{"date":"2020-12-28T21:19:06.1846021Z","statusCode":400,"path":"/log/app","duration":9,"message":"Invalid paramater: cMwyFA"}
{"date":"2020-12-28T21:19:06.1444807Z","statusCode":500,"path":"/log/app","duration":266,"message":"Server error 9750"}

```

## Prerequisites

- Knowledge of `Azure`, `Kubernetes`, `Fluent Bit` and `kubectl`
- Bash shell (tested on GitHub Codespaces, Mac, Ubuntu, WSL2)
- Azure CLI ([download](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest))
- Kubernetes cluster
- kubectl with access to the Kubernetes cluster

## Clone this repo

```bash

git clone https://github.com/Microsoft/fluent-bit-azure-log-analytics
cd fluent-bit-azure-log-analytics/fluentbit

```

## Create Kubernetes Dependencies

> For safety, we use the `log-test` namespace
>
> Most deployments use the `logging` namespace by convention

```bash

# create namespace, service account, cluster role, cluster role binding
kubectl apply -f account.yaml

```

## Create Azure Log Analytics Workspace

### Select Azure Subscription

```bash

# login to Azure (if necessary)
az login

# get list of subscriptions
az account list -o table

# select subscription (if necesary)
az account set -s YourSubscriptionName

```

### Create Log Analytics Workspace

```bash

# add az cli extension
#   this extension is in preview
az extension add --name log-analytics

# set environment variables (edit if desired)
export LogAppLoc=westus2
export LogAppRG=LogAppRG

# this value must be unique
export LogAppName=LogAppLogs

# create resource group
az group create -n $LogAppRG -l $LogAppLoc

# create Log Analytics Workspace
# check for workspace name is not unique error
#    export LogAppName with a different name and try again
az monitor log-analytics workspace create -g $LogAppRG -n $LogAppName -l $LogAppLoc

# delete secrets (if exist)
#    you can safely ignore a not found error
kubectl delete secret fluentbit-secrets -n log-test

# add Log Analytics secrets
kubectl create secret generic fluentbit-secrets -n log-test \
  --from-literal=WorkspaceId=$(az monitor log-analytics workspace show -g $LogAppRG -n $LogAppName --query customerId -o tsv) \
  --from-literal=SharedKey=$(az monitor log-analytics workspace get-shared-keys -g $LogAppRG -n $LogAppName --query primarySharedKey -o tsv)

# verify the secrets are set properly (base 64 encoded)
kubectl get secret fluentbit-secrets -n log-test -o jsonpath='{.data}'

```

## Deploy to Kubernetes

### Update Config (if required)

> The config works with `containerd` or `cri-o` runtimes
>
> config.yaml must be changed to work with `dockerd` or `dockershim`
>
> More details [here](https://github.com/microsoft/fluentbit-containerd-cri-o-json-log)

Check the Kubernetes runtime

```bash

kubectl describe node | grep "Container Runtime Version:"

```

If the result shows `dockerd` or `dockershim` edit config.yaml

- Replace
  - config.yaml
    - input-kubernetes.conf
      - `Parser  cri`
- with
  - `Parser docker`

```bash

# apply the fluentbit config
kubectl apply -f config.yaml

# start fluentbit daemonset
kubectl apply -f fluentbit-daemonset.yaml

# check daemonset until fluent-bit is running
kubectl get daemonset -n log-test

# check fluent-bit logs
kubectl logs -l k8s-app=fluent-bit-logging -n log-test

# run log app - this will generate 5 log entries
kubectl apply -f logapp.yaml

# check logapp pod
# logapp will run and then exit with Completed state
kubectl get pods

# check logs
kubectl logs logapp

# check fluent-bit logs
kubectl logs -l k8s-app=fluent-bit-logging -n log-test

# looking for a line like:
#   [2021/02/02 21:54:19] [ info] [output:azure:azure.0]

# check Log Analytics for your data on the Azure portal
# this can take 10-15 minutes initially

# generate more logs
kubectl delete -f logapp.yaml
kubectl apply -f logapp.yaml

# check Log Analytics for your data
# this should only take a few seconds

# delete logapp
kubectl delete -f logapp.yaml

# check daemonset
kubectl get daemonset -n log-test

# Result - fluent-bit daemonset is still running

```

### Cleaning up

```bash

kubectl delete secret fluentbit-secrets -n log-test
kubectl delete -f fluentbit-daemonset.yaml
kubectl delete -f config.yaml
kubectl delete -f account.yaml

# verify everything is deleted
#    Error from server (NotFound)
kubectl get ns log-test

```

## Contributing

This project welcomes contributions and suggestions.  Most contributions require you to agree to a Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us the rights to use your contribution. For details, visit <https://cla.opensource.microsoft.com>.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).

For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft trademarks or logos is subject to and must follow [Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).

Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
