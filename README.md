# Twingate Kubernetes Vector Example
An example demonstrating how to install [Vector Operator](https://github.com/kaasops/vector-operator/tree/main) to kubernetes to monitor Twingate connector pod logs.

# Install Vector Operator
Add a chart helm repository with follow commands:
```bash
helm repo add vector-operator https://kaasops.github.io/vector-operator/helm
helm repo update
```

## Installing the chart
Export default values of `vector-operator` chart to file values.yaml:
```bash
helm show values vector-operator/vector-operator > values.yaml
```
Change the values according to the need of the environment in values.yaml file. 

Please keep the sections `args`, `.vector` and `clustervectorpipeline` default for this example.

Test the installation with command:
```bash
helm install vector-operator vector-operator/vector-operator -f values.yaml -n {VECTOR-NAMESPACE} --debug --dry-run
```

Install chart with command:

```bash
helm install vector-operator vector-operator/vector-operator -f values.yaml -n {VECTOR-NAMESPACE}
```

## Deploy Vector CR
This would deploy the Vector agent on every Kubernetes node, to deploy on specific nodes see [Assign Pod Node](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
```bash
cat <<EOF | kubectl -n {VECTOR-NAMESPACE} apply -f -
apiVersion: observability.kaasops.io/v1alpha1
kind: Vector
metadata:
  name: vector-agent
spec:
  agent:
    tolerations:
    - effect: NoSchedule
      key: node-role.kubernetes.io/master
      operator: Exists
    - effect: NoSchedule
      key: node-role.kubernetes.io/control-plane
      operator: Exists
EOF
```

## Deploy Vector Pipeline
Deploy the Vector Pipeline to the Twingate connector namespace. 

The configuration below parses the Twingate Analytics logs and output to console.

To send logs to specific SIEM see [Vector Sinks](https://vector.dev/docs/reference/configuration/sinks/)

```bash
cat <<EOF | kubectl -n {Connector Namespace} apply -f -
apiVersion: observability.kaasops.io/v1alpha1
kind: VectorPipeline
metadata:
  name: twingate
spec:
  sources:
    twingate_connector:
      type: "kubernetes_logs"
      extra_label_selector: "app.kubernetes.io/name=connector"
  transforms:
    tg_analytics_filter:
      type: "filter"
      inputs: 
        - twingate_connector
      condition: starts_with!(.message, "ANALYTICS")
    tg_analytics_transform:
      type: "remap"
      inputs: 
        -  tg_analytics_filter
      source: |
        .message = parse_json!(parse_grok!(.message, "ANALYTICS%{SPACE}%{GREEDYDATA:json_event}").json_event)
      drop_on_abort: true
  sinks:
    sink-console:
      type: "console"
      encoding:
        codec: "json"
      inputs:
        - tg_analytics_transform
EOF	
```

## Enable Twingate Analytics logs
Enable Twingate Analytics logs by reinstalling Twingate connector with additional flag `--set connector.logAnalytics=v2`, for instance:
```bash
helm upgrade --install twingate-passionate-wallaby twingate/connector -n {Connector NAMESPACE} --set connector.network="acme" --set connector.accessToken="xxx" --set connector.refreshToken="xxx" --set connector.logAnalytics=v2
```

## Testing
Access a Twingate resources and check the vector agent logs
```bash
kubectl -n {Vector Namespace} logs -l app.kubernetes.io/name=vector
```

Sample Output:
```json
{"file":"/var/log/pods/default_twingate-passionate-wallaby-connector-65b6f7757c-nfhx4_5d927296-a205-4e9e-85c8-cf6a186e7e1c/connector/0.log","kubernetes":{"container_id":"containerd://532d2697f23bfa24db63469192a3f75552dbb7bfd8763f51b1fac475716f532a","container_image":"twingate/connector:1","container_name":"connector","namespace_labels":{"kubernetes.io/metadata.name":"default"},"node_labels":{"beta.kubernetes.io/arch":"amd64","beta.kubernetes.io/instance-type":"e2-small","beta.kubernetes.io/os":"linux","cloud.google.com/gke-boot-disk":"pd-balanced","cloud.google.com/gke-container-runtime":"containerd","cloud.google.com/gke-cpu-scaling-level":"2","cloud.google.com/gke-logging-variant":"DEFAULT","cloud.google.com/gke-max-pods-per-node":"110","cloud.google.com/gke-nodepool":"default-pool","cloud.google.com/gke-os-distribution":"cos","cloud.google.com/gke-provisioning":"standard","cloud.google.com/gke-stack-type":"IPV4","cloud.google.com/machine-family":"e2","failure-domain.beta.kubernetes.io/region":"europe-west2","failure-domain.beta.kubernetes.io/zone":"europe-west2-b","kubernetes.io/arch":"amd64","kubernetes.io/hostname":"gke-no-autopilot-test-default-pool-a021ca6b-q4z2","kubernetes.io/os":"linux","node.kubernetes.io/instance-type":"e2-small","topology.gke.io/zone":"europe-west2-b","topology.kubernetes.io/region":"europe-west2","topology.kubernetes.io/zone":"europe-west2-b"},"pod_ip":"xxx.xxx.xxx.xxx","pod_ips":["xxx.xxx.xxx.xxx"],"pod_labels":{"app.kubernetes.io/instance":"twingate-passionate-wallaby","app.kubernetes.io/name":"connector","pod-template-hash":"65b6f7757c"},"pod_name":"twingate-passionate-wallaby-connector-65b6f7757c-nfhx4","pod_namespace":"default","pod_node_name":"gke-no-autopilot-test-default-pool-a021ca6b-q4z2","pod_owner":"ReplicaSet/twingate-passionate-wallaby-connector-65b6f7757c","pod_uid":"5d927296-a205-4e9e-85c8-cf6a186e7e1c"},"message":{"connection":{"cbct_freshness":183,"client_ip":"xxx.xxx.xxx.xxx","duration":30124,"id":"2139eb38-eb7143eb-6-600000f-65044805-8a752","protocol":"udp","resource_ip":"xxx.xxx.xxx.xxx","resource_port":443,"rx":8310,"tx":5513},"connector":{"id":"152169","name":"passionate-wallaby"},"device":{"id":"200830"},"event_type":"closed_connection","relays":[{"ip":"xxx.xxx.xxx.xxx","name":"relaybalancer+https://relays.twingate.com","port":30010},{"ip":"xxx.xxx.xxx.xxx","name":"relaybalancer+https://relays.twingate.com","port":30009},{"ip":"34.147.237.117","name":"relaybalancer+https://relays.twingate.com","port":30018},{"ip":"xxx.xxx.xxx.xxx","name":"relaybalancer+https://relays.twingate.com","port":30005}],"remote_network":{"id":"64894","name":"vectortest"},"resource":{"address":"whatsmyip.com","applied_rule":"whatsmyip.com","id":"2326772"},"timestamp":1694779427692,"user":{"email":"xxx@xxx.com","id":"113312"}},"source_type":"kubernetes_logs","stream":"stderr","timestamp":"2023-09-15T12:03:47.692712516Z","timestamp_end":"2023-09-15T12:03:47.692712516Z"}
```

# Limitations
- [Vector Operator](https://github.com/kaasops/vector-operator/tree/main) currently does not support GKE Autopilot.
