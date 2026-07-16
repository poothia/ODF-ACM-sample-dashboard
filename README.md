# ODF-ACM-sample-dashboard
## Steps to configure the ACM Observability on Hub cluster
1. Create an S3 Bucket with NooBaa
```bash
cat << EOF > noobaa-object-storage.yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: observability-acm-s3
  namespace: open-cluster-management
spec:
  generateBucketName: observability-acm-s3-bucket
  storageClassName: openshift-storage.noobaa.io
EOF
oc create -f noobaa-object-storage.yaml
```

2. Create the Observability Namespace
```bash
oc new-project open-cluster-management-observability
```

3. Create the Pull Secret for MCO
```bash
DOCKER_CONFIG_JSON=$(oc get secret pull-secret -n openshift-config --template='{{index .data ".dockerconfigjson" | base64decode}}')
echo $DOCKER_CONFIG_JSON
```

4. Grab the Bucket Details and Credentials
```bash
NS="open-cluster-management"
OBC="observability-acm-s3"
BUCKET_HOST=$(oc get route s3 -n openshift-storage -o jsonpath='{.spec.host}')
echo $BUCKET_HOST

BUCKET_NAME=`oc describe ob obc-${NS}-${OBC} -n $NS | grep 'Bucket Name' | head -n 1 | cut -d: -f2 | tr -d " "`
echo $BUCKET_NAME

AWS_ACCESS_KEY_ID=`oc get secret $OBC -n $NS -o yaml|grep -m1 AWS_ACCESS_KEY_ID|cut -d: -f2|tr -d " "| base64 -d`
echo $AWS_ACCESS_KEY_ID

AWS_SECRET_ACCESS_KEY=`oc get secret $OBC -n $NS -o yaml|grep -m1 AWS_SECRET_ACCESS_KEY|cut -d: -f2|tr -d " "| base64 -d`
echo $AWS_SECRET_ACCESS_KEY
```

5. Create the Thanos Object Storage Secret
```bash
cat <<EOF > thanos-object-storage.yaml
apiVersion: v1
kind: Secret
metadata:
  name: thanos-object-storage
  namespace: open-cluster-management-observability
type: Opaque
stringData:
  thanos.yaml: |
    type: s3
    config:
      bucket: $BUCKET_NAME
      endpoint: $BUCKET_HOST
      insecure: true
      access_key: $AWS_ACCESS_KEY_ID
      secret_key: $AWS_SECRET_ACCESS_KEY
EOF

oc apply -f thanos-object-storage.yaml
```

6. Deploy the MultiClusterObservability CR
```bash
cat <<EOF > mco.yaml
apiVersion: observability.open-cluster-management.io/v1beta2
kind: MultiClusterObservability
metadata:
  name: observability
spec:
  observabilityAddonSpec: {}
  storageConfig:
    metricObjectStorage:
      name: thanos-object-storage
      key: thanos.yaml
EOF

oc apply -f mco.yaml
```

7. Enable developer instance to make custom dashboards following [standard documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/observability/index#setting-up-grafana-developer-instance).
8. Switch to admin of developer instance of grafana  [standard documentation](https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/observability/index#setting-up-grafana-developer-instance).
9. Import the ODF performance dashboard from the odf-grafana repository [dashboard](/dashboard/dashboard.json).

## Steps to configure the Spoke cluster to ship the metrics
1. Check if prometheus is scraping the metrics and store it in metrics.json
```bash
export HOST=$(oc -n openshift-monitoring get route thanos-querier -o jsonpath='{.status.ingress[].host}') 
export TOKEN=$(oc whoami -t)
curl -k -G -XGET "https://$HOST/api/v1/label/__name__/values" \
  -H "Authorization: Bearer $TOKEN" | jq > metrics.json
```

2. Make a configmap from the metrics in openshift-storage namespace
```bash
cat << EOF > observability-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: observability-metrics-custom-allowlist
  namespace: openshift-storage 
data:
  metrics_list.yaml: |
    names:
$(jq -r '.data[] | select(test("ceph|noob|fusion|odf|ocs"; "i")) | "      - \(.)"' metrics.json)
EOF
oc apply -f observability-configmap.yaml
```
3. Make a configmap in openshift-monitoring namespace
```bash
cat << EOF > openshift-monitoring-observability-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: observability-metrics-custom-allowlist
  namespace: openshift-monitoring
data:
  metrics_list.yaml: |
    names:
      - kube_persistentvolumeclaim_info
      - container_cpu_usage_seconds_total
      - node_disk_writes_completed_total
      - kube_storageclass_info
      - node_network_receive_bytes_total
      - node_network_info
      - node_network_transmit_bytes_total
      - node_network_speed_bytes
      - node_network_mtu_bytes
      - node_network_up
      - node_disk_read_bytes_total
      - node_disk_reads_completed_total
      - node_disk_writes_completed_total
      - node_disk_written_bytes_total
      - node_disk_read_time_seconds_total
      - node_disk_write_time_seconds_total
EOF
oc apply -f openshift-monitoring-observability-configmap.yaml
```

## References
[1] <https://docs.redhat.com/en/documentation/red_hat_advanced_cluster_management_for_kubernetes/2.16/html-single/observability/index#setting-up-grafana-developer-instance>  
[2] <https://github.com/redhat-performance/odf-grafana>
