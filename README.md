# CockroachDB Distributed TPC-C on Kubernetes



Based on: https://www.cockroachlabs.com/docs/stable/orchestrate-cockroachdb-with-kubernetes-multi-cluster.html



## Reference



### GCP US Regions & Zones to choose from

us-east1	b, c, d		Moncks Corner, South Carolina, USA
us-east4	a, b, c		Ashburn, Northern Virginia, USA
us-west1	a, b, c		The Dalles, Oregon, USA
us-west2	a, b, c		Los Angeles, California, USA
us-west3	a, b, c		Salt Lake City, Utah, USA
us-west4	a, b, c		Las Vegas, Nevada, USA

### GCP Quotas

If you need to increase GCP quotas, go to this URL: https://console.cloud.google.com/iam-admin/quotas?usage=USED&project=<your GCPproject id>



## Setup Steps



#### Create GCP firewall rules to allow communication on port 26257 on all private address ranges

```bash
gcloud compute firewall-rules create allow-cockroach-internal \
  --allow=tcp:26257 --source-ranges=10.0.0.0/8,172.16.0.0/12,192.168.0.0/16
```



#### Assumptions and Pre-requisites

- gcloud is installed locally and configured for authentication and for the correct project
- kubectl is installed locally



#### Create GCP K8S clusters
```bash
MACHINETYPE=e2-standard-16
GCEREGION1=us-west1
GCEREGION2=us-west2
GCEREGION3=us-west3
ACCOUNT=`gcloud info | grep Account | awk '{print $2}' | cut -d "[" -f 2 | cut -d "]" -f 1`
GCLOUDPROJECT=`gcloud config get-value project`
CLUSTER1=gke_${GCLOUDPROJECT}_${GCEREGION1}_cockroachdb1
CLUSTER2=gke_${GCLOUDPROJECT}_${GCEREGION2}_cockroachdb2
CLUSTER3=gke_${GCLOUDPROJECT}_${GCEREGION3}_cockroachdb3


gcloud container clusters create cockroachdb1 \
  --region=$GCEREGION1 --machine-type=$MACHINETYPE --num-nodes=1 \
  --cluster-ipv4-cidr=10.1.0.0/16 --node-locations=$GCEREGION1-a,$GCEREGION1-b,$GCEREGION1-c
gcloud container clusters create cockroachdb2 \
  --region=$GCEREGION2 --machine-type=$MACHINETYPE --num-nodes=1 \
  --cluster-ipv4-cidr=10.2.0.0/16 --node-locations=$GCEREGION2-a,$GCEREGION2-b,$GCEREGION2-c 
gcloud container clusters create cockroachdb3 \
  --region=$GCEREGION3 --machine-type=$MACHINETYPE --num-nodes=1 \
  --cluster-ipv4-cidr=10.3.0.0/16 --node-locations=$GCEREGION3-a,$GCEREGION3-b,$GCEREGION3-c
```

Notes:

- I tried to use c2-standard-16, but they weren't available in all zones; so I used e2-standard-16 instead.

- If you don't specify a machine type parameter in the `gcloud config get-value project`, it will default to e2-medium

- I'm specifying explicit ranges here for the pod networks.  I don't know if GCP is smart enough to create pod networks that don't overlap between clusters, but I like being explicit about it here because then I know the ranges won't overlap and I can be a little more certain about what's actually happening.

- When specifying `--num-nodes=1`, there will be 1 node created in each of the node-locations.  So, the above commands actually create 9 nodes.



 #### Validate settings

```bash
kubectl config get-contexts
```

You should see all 3 GCP k8s clusters listed.



 #### Create role bindings for the k8s clusters

```bash
kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$ACCOUNT --context=$CLUSTER1
kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$ACCOUNT --context=$CLUSTER2
kubectl create clusterrolebinding $USER-cluster-admin-binding --clusterrole=cluster-admin --user=$ACCOUNT --context=$CLUSTER3
```



 #### Get CockroachDB Kubernetes scripts

```bash
mkdir multiregion
cd $_
curl -OOOOOOOOO https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/{README.md,client-secure.yaml,cluster-init-secure.yaml,cockroachdb-statefulset-secure.yaml,dns-lb.yaml,example-app-secure.yaml,external-name-svc.yaml,setup.py,teardown.py}
```



#### Edit setup.py and teardown.py

Update setup.py with the correct `contexts` and `regions` variables
```python
contexts = {
    'us-west1': 'gke_cockroachlabs-hatcher-284815_us-west1_cockroachdb1',
    'us-west2': 'gke_cockroachlabs-hatcher-284815_us-west2_cockroachdb2',
    'us-west3': 'gke_cockroachlabs-hatcher-284815_us-west3_cockroachdb3',
}
```

```python
regions = {
}
```
Update tearddown.py with the correct `contexts` variable (same as above).

```python
contexts = {
    'us-west1': 'gke_cockroachlabs-hatcher-284815_us-west1_cockroachdb1',
    'us-west2': 'gke_cockroachlabs-hatcher-284815_us-west2_cockroachdb2',
    'us-west3': 'gke_cockroachlabs-hatcher-284815_us-west3_cockroachdb3',
}
```



#### Edit cockroachdb-statefulset-secure.yaml 

To get the best performance, we want to make sure that we're:

- using SSD disks
- using larger drives which will give us better IOPS and throughput
- Requesting enough CPU and memory that our k8s pods get distributed onto different k8s nodes to avoid noisy neighbor issues

Find this section (towards the bottom) and make the noted edits:
```yaml
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes:
        - "ReadWriteOnce"
      #add this next line
      storageClassName: storage-class-ssd
      resources:
        requests:
          #change this next line from 100Gi to 1024Gi
          storage: 1024Gi
```
Find this section and make the noted edits:
```yaml
      containers:
      - name: cockroachdb
        image: cockroachdb/cockroach:v20.1.4
        imagePullPolicy: IfNotPresent
        # add this next "resources" section
        resources:
          requests:
            cpu: "15"
            memory: "60G"
        ports:
```



#### Create a manifest file for our SSD drives and create these objects in Kubernetes


```bash
cat << EOF > storage-class-ssd.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: storage-class-ssd
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
EOF
```
```bash
kubectl create -f storage-class-ssd.yaml --context $CLUSTER1
kubectl create -f storage-class-ssd.yaml --context $CLUSTER2
kubectl create -f storage-class-ssd.yaml --context $CLUSTER3
```



#### Run the setup.py script

```bash
python2.7 setup.py
```



#### Verify that the pods are created

```bash
kubectl get pods --selector app=cockroachdb --all-namespaces --context $CLUSTER1
kubectl get pods --selector app=cockroachdb --all-namespaces --context $CLUSTER2
kubectl get pods --selector app=cockroachdb --all-namespaces --context $CLUSTER3
```
You should see `3/3` for the CockroachDB pods 



#### Create additional k8s nodes in GCP to host the TPC-C workload pods

```bash
gcloud container node-pools create workloadnodes --cluster=cockroachdb1 --disk-type=pd-ssd --machine-type=$MACHINETYPE --num-nodes=1 --zone=$GCEREGION1 --node-locations=$GCEREGION1-a
gcloud container node-pools create workloadnodes --cluster=cockroachdb2 --disk-type=pd-ssd --machine-type=$MACHINETYPE --num-nodes=1 --zone=$GCEREGION2 --node-locations=$GCEREGION2-a
gcloud container node-pools create workloadnodes --cluster=cockroachdb3 --disk-type=pd-ssd --machine-type=$MACHINETYPE --num-nodes=1 --zone=$GCEREGION3 --node-locations=$GCEREGION3-a
```



#### Create a pod in each k8s cluster which will be used to run our TPC-C workload

```bash
kubectl create -f client-secure.yaml --context $CLUSTER1
kubectl create -f client-secure.yaml --context $CLUSTER2
kubectl create -f client-secure.yaml --context $CLUSTER3
```



#### If you want to get a sql shell...

```bash
kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
kubectl exec -it cockroachdb-client-secure --context $CLUSTER2 --namespace default -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
kubectl exec -it cockroachdb-client-secure --context $CLUSTER3 --namespace default -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```



#### Create a CockroachDB user which we can use to access the Admin UI

```bash
kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default \
  -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public \
  --execute="CREATE USER roach WITH PASSWORD 'whateverpasswordyouwant'; GRANT admin TO roach;"
```



#### Port-foward 8080 to the Admin UI on our first k8s cluster

```bash
kubectl port-forward cockroachdb-0 8080 --context $CLUSTER1 --namespace $GCEREGION1
```



#### Set enterprise license

```bash
kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default \
  -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public \
  --execute="SET CLUSTER SETTING cluster.organization = 'MultiRegionDemo'; SET CLUSTER SETTING enterprise.license = 'crl-0-?????????????';"
```

Note: you need the license in place to do the partitioning stuff in the next step.



#### Import the correct data for TPC-C

```bash
time kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- \
  ./cockroach workload fixtures import tpcc \
  --warehouses 1000 \
  --partition-affinity=0 --partitions=3 --partition-strategy=leases --zones=us-west1,us-west2,us-west3 \
  'postgresql://root@cockroachdb-public:26257?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt'
```
Note: I'm using `cockroach workload fixtures import tpcc` instead of `cockroach workload init tpcc`.  By doing so, the data is imported, rathen than being inserted.  It runs in about 30 minutes, as opposed to 90 minutes.
Note: You can run this on one workload, not all three.



#### Export the schema, zones, and ranges so that we can validate the setup

```bash
kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- \
  ./cockroach dump tpcc --dump-mode=schema --certs-dir=/cockroach-certs --host=cockroachdb-public > schema.txt

kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- \
  ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public \
  --execute="SHOW ALL ZONE CONFIGURATIONS;" > zoneconfigs.txt

kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- \
  ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public \
  --execute="SELECT COUNT(1), table_name, lease_holder_locality FROM [SHOW RANGES FROM DATABASE tpcc] GROUP BY 2,3 ORDER BY 3,2;" > ranges.txt
```



#### Run the TPC-C workload on all three workload nodes

```bash
kubectl exec -it cockroachdb-client-secure --context $CLUSTER1 --namespace default -- \
  ./cockroach workload run tpcc \
  --duration=60m --warehouses 1000 --ramp=180s \
  --partition-affinity=0 --partitions=3 --partition-strategy=leases \
  'postgresql://root@cockroachdb-public:26257?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt'

kubectl exec -it cockroachdb-client-secure --context $CLUSTER2 --namespace default -- \
  ./cockroach workload run tpcc \
  --duration=60m --warehouses 1000 --ramp=180s \
  --partition-affinity=1 --partitions=3 --partition-strategy=leases \
  'postgresql://root@cockroachdb-public:26257?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt'

kubectl exec -it cockroachdb-client-secure --context $CLUSTER3 --namespace default -- \
  ./cockroach workload run tpcc \
  --duration=60m --warehouses 1000 --ramp=180s \
  --partition-affinity=2 --partitions=3 --partition-strategy=leases \
  'postgresql://root@cockroachdb-public:26257?sslmode=verify-full&sslcert=/cockroach-certs/client.root.crt&sslkey=/cockroach-certs/client.root.key&sslrootcert=/cockroach-certs/ca.crt'
```



#### Look at results
You can see the summary of the three workload apps after they've finished.  And you can look at metrics from the Admin UI.

Your output should look like:
```
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
3600.0s        0         575026          159.7     98.2    109.1    184.5    251.7   1208.0 
Audit check 9.2.1.7: PASS
Audit check 9.2.2.5.1: PASS
Audit check 9.2.2.5.2: PASS
Audit check 9.2.2.5.3: PASS
Audit check 9.2.2.5.4: PASS
Audit check 9.2.2.5.5: PASS
Audit check 9.2.2.5.6: PASS
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
3600.0s     4160.5  32.4%    128.1    117.4    159.4    226.5    268.4   1040.2

```
```
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
3600.0s        0         575130          159.8     72.3     71.3    142.6    201.3   2281.7 
Audit check 9.2.1.7: PASS
Audit check 9.2.2.5.1: PASS
Audit check 9.2.2.5.2: PASS
Audit check 9.2.2.5.3: PASS
Audit check 9.2.2.5.4: PASS
Audit check 9.2.2.5.5: PASS
Audit check 9.2.2.5.6: PASS
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
3600.0s     4162.2  32.4%     94.1     88.1    130.0    176.2    218.1   2281.7

```
```
_elapsed___errors_____ops(total)___ops/sec(cum)__avg(ms)__p50(ms)__p95(ms)__p99(ms)_pMax(ms)__result
3600.0s        0         577825          160.5     70.8     58.7    176.2    234.9   1040.2 
Audit check 9.2.1.7: PASS
Audit check 9.2.2.5.1: PASS
Audit check 9.2.2.5.2: PASS
Audit check 9.2.2.5.3: PASS
Audit check 9.2.2.5.4: PASS
Audit check 9.2.2.5.5: PASS
Audit check 9.2.2.5.6: PASS
_elapsed_______tpmC____efc__avg(ms)__p50(ms)__p90(ms)__p95(ms)__p99(ms)_pMax(ms)
3600.0s     4178.4  32.5%     90.1     88.1    130.0    184.5    243.3   1040.2

```



## Teardown Steps



#### Run the teardown.py script
```bash
python2.7 teardown.py
```



#### Delete the storage class we created earlier

```bash
kubectl delete storageclass storage-class-ssd --cluster $CLUSTER1 
kubectl delete storageclass storage-class-ssd --cluster $CLUSTER2
kubectl delete storageclass storage-class-ssd --cluster $CLUSTER3 
```



#### Destroy the GCP k8s clusters

```bash
gcloud container clusters delete cockroachdb1 --region=$GCEREGION1 --quiet
gcloud container clusters delete cockroachdb2 --region=$GCEREGION2 --quiet
gcloud container clusters delete cockroachdb3 --region=$GCEREGION3 --quiet
```