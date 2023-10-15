
Loki is a log aggregation system in LGTM stack, which is responsible to collect, store, and query logs. Loki is very easy to start with and highly compatible with any log format. This article discussed an example Loki Kubernetes deployment solution, which can be used in production environment.

### Architecture

Loki relies on other applications for log collection (agent) and query (client).

![[LokiStack.excalidraw]]


On the log collection side, Loki does not pull log from apps, but relies on agents to push log via Rest API. In this example, promtail is used as the agent as [the official recommended]([Send log data to Loki | Grafana Loki documentation](https://grafana.com/docs/loki/latest/send-data/)) when deployed with K8s.

On the client side, this example uses Grafana to provide the user interface, allowing user to make query and retrieve logs from Loki server.

Around the Loki server, optional components can be deployed to make the solution more reliable. In this example, a nginx server served as reverse proxy, which enables load balancing when Loki server replicated. More importantly,  [Loki server does not have authentication feature](https://grafana.com/docs/loki/latest/operations/authentication/). If authentication is required for agent or clients, authentication can be enabled on nginx. 

It is usually need to save log on a more reliable storage service rather than on the local driver of the Loki servers. In this example, S3 is used. 


### Deployment

1. Prerequisites: helm, kubectl 

2. Install Loki server via helm chart 
```shell
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm search repo loki

kubectl create namespace loki
kubectl config set-context --current --namespace loki


echo -e '
loki:
  commonConfig:
    replication_factor: 2
  auth_enabled: false
  storage:
    bucketNames:
      chunks: bucketname-chunks
      ruler: bucketname-ruler
      admin: bucketname-admin
    type: "s3"
    s3:
      region: ap-southeast-2
      endpoint: "http://s3.ap-southeast-2.amazonaws.com"
singleBinary:
  replicas: 3
test:
  enabled: true
monitoring:
  selfMonitoring:
    enabled: true
  lokiCanary:
    enabled: true
gateway:
  enabled: true
minio:
  enabled: false
'> /tmp/values.yml

helm upgrade --install --namespace loki logging grafana/loki --values /tmp/values.yml 
```

Some explainations:
- Logs are stored on S3 with the given region and bucket. If need to save to Azure Blob Storage, please refer to [`loki.storage` in this document](https://grafana.com/docs/loki/latest/setup/install/helm/reference/) for related parameters.
- If need store to a local server rather than cloud storage, the parameter value `minio.enabled` can be set to `true` to host a local object storage. In this case, remove all values under `loki.storage`.
- The `loki.auth_enabled` flag is to control [Multi-tenancy](https://grafana.com/docs/loki/latest/operations/multi-tenancy/), rather than [Authentication](https://grafana.com/docs/loki/latest/operations/authentication/). Please be careful not to be confused by the parameter value name.

3. **Install Promotail in APP's name space**

```shell

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

echo -e '
config:
  clients:
    - url: http://loki-gateway.loki.svc.cluster.local/loki/api/v1/push
    # -- set log pushing target url to loki gateway
'>/tmp/promtail-values.yaml

helm upgrade --install promtail grafana/promtail --values /tmp/promtail-values.yaml --namespace app_namespace

```
Note that, the values need to be set to loki's gateway.

4. **Install and access Grafana**
```shell
helm upgrade --install --namespace=loki loki-grafana grafana/grafana
kubectl get secret --namespace loki loki-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
kubectl port-forward --namespace loki service/loki-grafana 3000:80
```

The above commands will install Grafana to loki namespace, print admin password and forward grafana's service to local 3000 port. After that, open web browser and visit `http://localhost:3000`. Username is `admin` and password was printed in second step.

### Configure Grafana and query logs

After logging into Grafana's home page, create a connection via the `Add new connection` link from the top left hamburg menu.

![[Pasted image 20231015115339.png|300]]

Search for "loki" in data source list.

![[Pasted image 20231015115018.png]]


On the Loki connection edit page, type in the url of Loki's gateway `http://loki-gateway` or `http://loki-gateway.loki.svc.cluster.local`. Then click `Save & test` button. Then, create a grafana dashboard or simply explore logs via the two buttons on top right.

![[Pasted image 20231015134814.png]]


In this example, the explore page was opened and a simple query was build with the help of builder as shown below. For more information about writing your own query, please refer to [the document](https://grafana.com/docs/loki/latest/query/).

![[Pasted image 20231015134130.png]]


### Summary

In this article, a simple yet reliable loki stack is deployed via Kubernetes. It consists of Loki server, cloud storage layer, agent and client. The helm chart provided by Loki has lots of configuration to customize the deployments and its components. For more values, please refer to its [document](https://grafana.com/docs/loki/latest/setup/install/helm/reference/) and [github repo](https://github.com/grafana/loki/blob/8e9635f41a4c32c79d5779b8d719473ea413542e/production/helm/loki/values.yaml)
