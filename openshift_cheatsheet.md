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
`$ oc delete pvc/<pvc_storage_name>`

---

# Configuring ID Providers

## HTPassword

1. Create an HTPasswd authentication file, and add a user:  
`$ htpasswd -c -B -b ~/path/to/file/<htpasswd_file_name> <user> <password>`

2. Add another user to the HTPasswd authentication file:  
`$ htpasswd -b ~/path/to/file/<htpasswd_file_name> <user2> <password2>`

3. Log into OpenShift and create a secret that contains the HTPasswd users file:  
`$ oc login -u kubeadmin -p ${kubeadmin_passwd} https://<url>:6443`  
`$ oc create secret generic localusers --from-file htpasswd=~/path/to/file/<htpasswd_file_name> -n openshift-config`

4. Assign a user the cluster-admin role:  
`oc adm policy add-cluster-role-to-user cluster-admin <user>`

5. Configure the customer resource file and update the cluster:  
`$ oc get oauth cluster -o yaml > ~/path/to/file/oauth.yaml`  
Edit the oauth.yaml file:  
```
spec:
  identityProviders:
  - htpasswd:
      fileData:
        name: localusers
    mappingMethod: claim
    name: myusers
    type: HTPasswd
```
Apply custom resource defined:  
`oc replace -f ~/path/to/file/oauth.yaml`

Verify a user has cluster-admin role:  
`$ oc login -u <user> -p <passwd>`
`$ oc get nodes`

List current users:  
`$ oc get users`

Display list of current identities:  
`$ oc get identity`

Create a new HTPasswd user:  
`$ oc extract secret/localusers -n openshift-config --to ~/path/to/file/ --confirm /path/to/file/htpasswd`
`$ htpasswd -b ~/path/to/file/htpasswd <user3> <passwd3>`
`$ oc set data secret/localusers --from-file htpasswd=~/path/to/file/htpasswd -n openshift-config`

Note: Use `--to -` to send the secret to STDOUT rather than save it to a file.  

Create a new project:  
`$ oc new-project <project_name>`

Delete project:  
`$ oc delete project <project_name>`

Delete user from htpasswd file:  
`$ htpasswd -D ~/path/to/file/htpasswd <user>`

Delete identity resource for user:  
`$ oc delete identity "myusers:<user>`

Delete user resource for user:  
`$ oc delete user <user>`

Remove Identity Provider and clean up all users:  
Remove ID provider from OAuth:  
`$ oc edit oauth`  
`spec: {}`  
Delete localusers secret from openshift-config namespace:  
`$ oc delete secret localuesrs -n openshift-config`  
Delete all user resources:  
`$ oc delete user --all`  
Delete all identity resources:  
`$ oc delete identity --all`  

---

## RBAC

Add a cluster role to a user:  
`$ oc adm policy add-cluster-role-to-user <cluster_role> <user>`

Remove a cluster role from a user:  
`$ oc adm policy remove-cluster-role-from-user <cluster_role> <user>`  

Determine if a user can execute an action on a resource:  
`$ oc adm policy who-can delete <user>`
 
| Default roles | Description |
| ------------- | ----------- |
| admin | Manage all project resources, including granting access to other users to access the project |
| basic-user | Read access to the project |
| cluster-admin | Superuser access to the cluster resources. Perform any action on the cluster, and have full control of all projects. |
| cluster-status | Can get cluster status information |
| edit | Create, change, delete common app resources from the project e.g. services and deployments. Cannot act on management resources e.g. limit ranges and quotas, and can't manage access permissions on the project |
| self-provisioner | Create new projects. This is a cluster role, not a project role |
| view | View project resources, but can't modify project resources |