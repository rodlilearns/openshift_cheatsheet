# OpenShift Cheat Sheet

---
## Verify Health of a Cluster

Display a column with status of each node:  
`$ oc get nodes`

Display the current (actual) CPU and memory usage of each node:  
`$ oc adm top nodes`

Display resources available and used from the scheduler p.o.v.:  
`$ oc describe node <node_name>`

Get cluster version:  
`$ oc get clusterversion`

Obtain more detailed information about the cluster status (version, channel, unique identifier, available images for updating cluster, history):  
`$ oc describe clusterversion`

Review cluster operators:  
`$ oc get clusteroperators`

Display logs of CRI-O container engine:  
`$ oc adm node-logs -u crio <node_name>`

Display logs of the Kubelet:  
`$ oc adm node-logs -u kubelet <node_name>`

Display all journal logs of a node:  
`$ oc adm node-logs <node_name>`

Run local commands from the node (not recommended):  
`$ oc debug node/<node_name>`  
`sh-4.4# chroot /host`  
`sh-4.4# systemctl is-active kubelet`

Get low-level information about all local containers running on the node:  
`# oc debug node/<node_name>`  
`sh-4.4# chroot /host`  
`sh-4.4# crictl ps`

### Troubleshoot pods that fail to start

List events from current project:  
`$ oc get events`

Get a list of events filtered by pod:  
`$ oc describe pod`

### Troubleshoot running and terminated pods

Get logs from pod:  
`oc logs <pod_name>`

Get logs from container:  
`oc logs <pod_name> -c <container_name>`

Note: Interpreting application logs requires specific knowledge of the application. Hopefully the application provides clear messages that can help find the problem.  

`--loglevel` starts from 6 and goes up to 10. For example:  
`$ oc get pods --loglevel 6`

Get authentication token to authenticate OpenShift API requests:  
`$ oc whoami -t`
---

---
## Storage

View available storage classes:  
`$ oc get storageclass`

Add a volume to an app:  
`$ oc set volumes deployment/<app_name> --add --name <pv_storage_name> --type pvc --claim-class <storage_class_name> --claim-mode <rwo> --claim-size <15Gi> --mount-path </var/lib/app_name> --claim-name <pv_claim_name>`

Example PersistentVolumeClaim API object:  

```
---
apiVersion: v1
kind: PersistentVolumeClaim 1
metadata:
  name: example-pv-claim 2
  labels:
    app: example-application
spec:
  accessModes:
    - ReadWriteOnce 3
  resources:
    requests:
      storage: 15Gi
```

Storage access modes:  
| Access Mode | CLI Abbreviation | Description |
| ----------- | ---------------- | ----------- |
| ReadWriteMany | RWX | Kubernetes can mount the volume as read-write on many nodes. |
| ReadOnlyMany | ROX | Kubernetes can mount the volume as read-only on many nodes. |
| ReadWriteOnce | RWO | Kubernetes can mount the volume as read-write on only a single node |

Add PVC to application:  

```
spec:
   volumes:
     - name: example-pv-storage
       persistentVolumeClaim:
         claimName: example-pv-claim
   containers:
   - name: example-application
     image: registry.redhat.io/rhel8/example-app
     ports:
     - containerPort: 1234
     volumeMounts:
       - mountPath: "/var/lib/example-app"
         name: example-pv-storage
```

Delete a persistent volume claim:  
`$ oc delete pvc/<pvc_storage_name>