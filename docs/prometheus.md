---
id: prometheus
title: Using OpenEBS as TSDB for Prometheus
sidebar_label: Prometheus
---
------

<img src="/docs/assets/o-prometheus.png" alt="OpenEBS and Prometheus" style="width:400px;">

## Introduction

Prometheus is the mostly widely used application for scraping cloud native application metrics. Prometheus and OpenEBS togethr provide a complete open source stack for monitoring. In this solution, OpenEBS is used as Prometheus TSDB, where all the metrics are permanently stored on local Kubernetes cluster. When using OpenEBS as TSDB, following are the advantages:

- All the data is stored locally and managed natively to Kubernetes

- No need of externally managed Prometheus storage

- Start with small storage and expand the size of TSDB as needed on the fly

- Prometheus metrics are highly available. When a node fails or rebooted, Prometheus pod is rescheduled onto on of the two other nodes where cStor volume replica is available. The metrics data is  rebuilt when the node becomes available

-  Take backup of the Prometheus metrics periodically and back them up to S3 or any object storage so that restoration of the same metrics is possible to the same or any other Kubernetes cluster

  

## Deployment model 

<img src="/docs/assets/sol-prometheus.png" style="width:800px;">

As shown above, OpenEBS volumes need to be configured with three replicas for high availability. This configuration work fine when the nodes (hence the cStor pool) is deployed across Kubernetes zones.



## Configuration workflow



1. **Install OpenEBS :** If OpenEBS is not installed on the Kubernetes already, start by <a href="/docs/next/installation.html" target="_blank">installing</a> OpenEBS on all or some of the cluster nodes. If OpenEBS is already installed, go to step 2.

2. **Connect to MayaOnline (Optional)** : Connecting  the Kubernetes cluster to <a href="app.mayaonline.io" target="_blank">MayaOnline</a> provides good visibility of storage resources. MayaOnline has various **support options for enterprise customers**.

3. **Configure cStor Pool** : After OpenEBS installation,  cStor pool has to be configured. As prometheus TSDB needs high availability of data, OpenEBS cStor volume has to be configured with three replicas. During cStor Pool creation, make sure that the maxPools parameter is set to >=3. If cStor Pool is already configured as required go to Step 4 to create Prometheus StorageClass. 

4. **Create Storage Class** : 

   StorageClass is the interface through which most of the OpenEBS storage policies  are defined. See [Prometheus Storage Class](/docs/next/prometheus.html#prometheus-storageclass) section below. 

5. **Configure PVC** : Prometheus needs only one volume to store the data. See [PVC example spec](/docs/next/prometheus.html#pvc-spec-for-prometheus) below.

6. **Launch and test prometheus**:

   Run `kubectl apply -f <prometheus.yaml>` to see Prometheus running. For more information on configuring more services to be monitored, see Prometheus documentation.

7. Persistent volume for Grafana:

   Grafana needs a much smaller persistent storage for storing metadata. Typically the storage class used for Prometheus is reused for Grafana as well. Just construct a new PVC with smaller storage size.  

   

## Sample Prometheus deployment at openebs.ci

A <a href="https://openebs.ci/prometheus-cstor" target="_blank">sample Prometheus server</a> at <a href="https://openebs.ci" target="_blank">https://openebs.ci</a>

Sample yaml specs for running Prometheus using cStor are <a href="https://github.com/openebs/e2e-infrastructure/tree/54fe55c5da8b46503e207fe0bc08f9624b31e24c/production/prometheus-cstor" target="_blank">here</a>. 



## Post deployment Operations

- Monitor OpenEBS Volume size 

It is not seamless to increase the cStor volume size (refer to the roadmap item). Hence, it is recommended that sufficient size is allocated during the initial configuration. However, an alert can be setup for volume size threshold using MayaOnline.

- Monitor cStor Pool size

As in most cases, cStor pool may not be dedicated to just Prometheus alone. It is recommended to watch the pool capacity and add more disks to the pool before it hits 80% threshold. 

 

## Best Practices

### Maintain volume replica quorum always

### Maintain cStor pool used capacity below 80%



## Troubleshooting guidelines 

### Read-Only volume

### Snapshots were failing





## Configuration details

### Sample cStor Pool spec

```
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk
spec:
  name: cstor-disk
  type: disk
  maxPools: 3
  poolSpec:
    poolType: striped
  # NOTE - Appropriate disks need to be fetched using `kubectl get disks`
  disks:
    diskList:
       - disk-3326ca3a735ba9cbe5a7753d12d5a55a
       - disk-d80800ca121b85b136d6a50a1079c921
       - disk-49e1ca76d1901a01ff6f366b8c53e44a
```





### Prometheus StorageClass 

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-cstor-disk
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-disk"
      - name: ReplicaCount
        value: "3"
     #  value: |-
     #      memory: 1Gi
     #      cpu: 200m
     #- name: AuxResourceLimits
     #  value: |-
     #      memory: 0.5Gi
     #      cpu: 50m
#(Optional) Below 3 lines schedules the target pods deployed on the labeled nodes
     #- name: TargetNodeSelector
     #  value: |-
     #      node: appnode
provisioner: openebs.io/provisioner-iscsi
```



### PVC spec for Prometheus

```
#PersistentVolumeClaim for prometheus
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cstor-prometheus-storage-volume-claim
  namespace: openebs
spec:
  storageClassName: OpenEBS-cStor-Prometheus 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500G  
```



### PVC spec for Grafana

```
#PersistentVolumeClaim for grafana
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: grafana-cstor-claim
  namespace: grafana-cstor
spec:
  storageClassName: OpenEBS-cStor-Prometheus 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50G   
     
```

See the <a href="https://github.com/openebs/e2e-infrastructure/blob/54fe55c5da8b46503e207fe0bc08f9624b31e24c/production/grafana-cstor/grafana-cstor-deployment.yaml" target="_blank"> sample spec files</a> for Grafana using cStor here. 



### Complete YAML file for prometheus+OpenEBS

Following is the yaml file that has Prometheus and OpenEBS configuration

```
# The following file is intended for deployments that are not already
# configured with prometheus. This is a minified version of the config
# from the files present under ./openebs-monitoring/
# 
# Prometheus tunables
apiVersion: v1
kind: ConfigMap
metadata:
  name: openebs-prometheus-tunables
  namespace: openebs
data:
  storage-retention: 24h
---
# Define the openebs prometheus jobs
kind: ConfigMap
metadata:
  name: openebs-prometheus-config
  namespace: openebs
apiVersion: v1
data:
  prometheus.yml: |-
    global:
      external_labels:
        slave: slave1
      scrape_interval: 5s
      evaluation_interval: 5s
    scrape_configs:
    #- job_name: 'prometheus'
      #static_configs:
      # Added this to allow for federation collection
      # Replace localhost with the nodeIP
      #  - targets: ['localhost:32514']
    - job_name: 'prometheus'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_name]
        regex: openebs-prometheus-server
        action: keep
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
    - job_name: 'openebs-volumes'
      scheme: http
      kubernetes_sd_configs:
      - role: pod
      relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_monitoring]
        regex: volume_exporter_prometheus
        action: keep
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod_name
      # Below entry ending with vsm is deprecated and is maintained for
      # backward compatibility purpose.
      - source_labels: [__meta_kubernetes_pod_label_vsm]
        action: replace
        target_label: openebs_pv
      # Below entry is the correct entry. Though the above and below entries
      # are having same target_label as openebs_pv, only one of them will be
      # valid for any release.
      - source_labels: [__meta_kubernetes_pod_label_openebs_io_persistent_volume]
        action: replace
        target_label: openebs_pv
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: drop
        regex: '(.*)9501'
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: drop
        regex: '(.*)3260'
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        action: drop
        regex: '(.*)80'
---
# prometheus-deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: openebs-prometheus
  namespace: openebs
  labels:
    name: prometheus
    type: workload  
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: openebs-prometheus-server
        type: workload 
    spec:
      serviceAccountName: openebs-maya-operator
      securityContext:
        fsGroup: 1000
      containers:
        - name: prometheus
          image: prom/prometheus:v2.3.0
          args:
            - "--config.file=/etc/prometheus/conf/prometheus.yml"
            # Metrics are stored in an emptyDir volume which
            # exists as long as the Pod is running on that Node.
            # The data in an emptyDir volume is safe across container crashes.
            - "--storage.tsdb.path=/prometheus"
            # How long to retain samples in the local storage.
            - "--storage.tsdb.retention=$(STORAGE_RETENTION)"
          ports:
            - containerPort: 9090
          env:
            # environment vars are stored in prometheus-env configmap. 
            - name: STORAGE_RETENTION
              valueFrom:
                configMapKeyRef:
                  name: openebs-prometheus-tunables
                  key: storage-retention
          resources:
            requests:
              # A memory request of 250M means it will try to ensure minimum
              # 250MB RAM .
              memory: "128M"
              # A cpu request of 128m means it will try to ensure minimum
              # .125 CPU; where 1 CPU means :
              # 1 *Hyperthread* on a bare-metal Intel processor with Hyperthreading
              cpu: "128m"
            limits:
              memory: "700M"
              cpu: "500m"
          
          volumeMounts:
            # metrics collected by prometheus will be stored at the given mountpath.
            - name: cstor-prometheus-storage-volume
              mountPath: /prometheus                        
            # prometheus config file stored in the given mountpath
            - name: prometheus-server-volume
              mountPath: /etc/prometheus/conf
      volumes:
        # All the time series stored in this volume in form of .db file.
        - name: cstor-prometheus-storage-volume
          persistentVolumeClaim:
            claimName: cstor-prometheus-storage-volume-claim 
        # Prometheus Config file will be stored in this volume 
        - name: prometheus-server-volume
          configMap:
            name: openebs-prometheus-config
        
---
#PersistentVolumeClaim for prometheus
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cstor-prometheus-storage-volume-claim
  namespace: openebs
spec:
  storageClassName: openebs-cstor-disk 
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G             
---
# prometheus-service
apiVersion: v1
kind: Service
metadata:
  name: openebs-prometheus-service
  namespace: openebs
spec:
  selector: 
    name: openebs-prometheus-server
  type: NodePort
  ports:
    - port: 80 # this Service's port (cluster-internal IP clusterIP)
      targetPort: 9090 # pods expose this port
      nodePort: 32514
      # Note that this Service will be visible as both NodeIP:nodePort and clusterIp:Port
```





<!-- Hotjar Tracking Code for https://docs.openebs.io -->

<script>
   (function(h,o,t,j,a,r){
       h.hj=h.hj||function(){(h.hj.q=h.hj.q||[]).push(arguments)};
       h._hjSettings={hjid:785693,hjsv:6};
       a=o.getElementsByTagName('head')[0];
       r=o.createElement('script');r.async=1;
       r.src=t+h._hjSettings.hjid+j+h._hjSettings.hjsv;
       a.appendChild(r);
   })(window,document,'https://static.hotjar.com/c/hotjar-','.js?sv=');
</script>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-92076314-12"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-92076314-12');
</script>