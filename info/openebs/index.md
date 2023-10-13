
# OpenEBS with cStor on OpenShift 4.12

OpenEBS helps Developers and Platform SREs easily deploy Kubernetes Stateful Workloads that require fast and highly reliable container attached storage. 

OpenEBS turns any storage available on the Kubernetes worker nodes into local or distributed Kubernetes Persistent Volumes.

## Installation
I've try to use the Operator(3.0.0) provided in the Marketplace but is seems to be broken, so I choose to use the Helm method

### Help setup

```console
# helm repo add openebs https://openebs.github.io/charts
# helm repo update

# helm install openebs --namespace openebs openebs/openebs --create-namespace --set cstor.enabled=true
```

### Fix permissions 
```console
# oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs-cstor-csi-node-sa
# oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs-cstor-operator
# oc adm policy add-scc-to-user privileged system:serviceaccount:openebs:openebs
```

Note: you may need to add the service account directly to the priviled SCC user list, 
to do that edit the scc/privileged object and add the service account to the list of users like that :
```console
# oc edit scc/privileged 
```

> ...
> users:
> ...
> - system:serviceaccount:openebs:openebs
> - system:serviceaccount:openebs:openebs-cstor-operator
> - system:serviceaccount:openebs:openebs-cstor-csi-node-sa
> ...

Optional: add a new label to trigger rescheduling 
```console
# oc label ds/openebs-cstor-csi-node PERM=FIXED
# oc label ds/openebs-ndm PERM=FIXED
```

Start iscsi daemon on Infra nodes and workers:
```console
# cat <<EOF> 99-infra-worker-custom-enable-iscsid.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
    machineconfiguration.openshift.io/role: infra
  name: 99-infra-worker-custom-enable-iscsid
spec:
  config:
    ignition:
      version: 3.2.0
    systemd:
      units:
      - enabled: true
        name: iscsid.service
EOF
```

Apply the definition on the cluster and wait for Machine Configuration to be deployed
```console
# oc create -f 99-infra-worker-custom-enable-iscsid.yaml
```

Watch configuration being applied
```console
# watch oc get mcp
```

### Confirm installation status
```console
# helm ls -n openebs
# oc get pods -n openebs
# oc get sc
```

List detected block device
```console
# oc get blockdevice -n openebs  --show-labels
```

I've created the following script to provide a YaML definition easily 
```console
cat <<EOF>> pool_yaml_generator.sh
#!/bin/bash
##################
# S.CAPS OCt2023 #
##################

echo -e "apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:"

for bd in $( kubectl get bd -n openebs | grep -Ev '^NAME' | awk '{ print $1 ";" $2 }' ); do
  BLOCKDEV=$( echo $bd | cut -d ';' -f 1 )
  WORKER=$( echo $bd | cut -d ';' -f 2 )
  echo "   - nodeSelector:"
  echo "       kubernetes.io/hostname: \"$WORKER\""
  echo "     dataRaidGroups:"
  echo "       - blockDevices:"
  echo "         - blockDeviceName: \"$BLOCKDEV\""
  echo "     poolConfig:"
  echo "       dataRaidGroupType: \"stripe\""
done

# EOF
```

Then start it
```console
# sh pool_yaml_generator.sh > cstor-disk-pool.yaml
```

Review the content
```console
# cat cstor-disk-pool.yaml
```

and apply it if the content is good

```console
# oc create -f cstor-disk-pool.yaml
```

Checkout the results :
```console
# oc get cspc -n openebs
```

Create a new storage Class
```console
# cat <<EOF> cstor-csi-storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: cstor-csi-sc
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
  cas-type: cstor
  # cstorPoolCluster should have the name of the CSPC
  cstorPoolCluster: cstor-disk-pool
  # replicaCount should be <= no. of CSPI created in the selected CSPC
  replicaCount: "3"
EOF

# oc create -f cstor-csi-storageclass.yaml
```

Optionnal: make this storage class the default one
```console
# oc annotate sc/cstor-csi-sc storageclass.kubernetes.io/is-default-class="true"
```

Enjoy !
