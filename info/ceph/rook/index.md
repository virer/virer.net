# Ceph
Ceph is an open-source, distributed storage system that can be deployed on many platforms.

Ceph can even deployed on Raspberry-pi clusters :)

Vendors :
- Red Hat provides Ceph via OpenShift Data Foundation.
- SuSE provides Ceph via Longhorn (Mostly on Rancher).
- Rook is an alternive one.


## Rook

Pre-req:
Local storage will be need for 2 elements of the ceph clusters, "mon" and "osd".

Each of those will run on 3 nodes.

For this example I use infra0,1 and worker0 for the storage part.

I've create pvc as it is easier to me to identify where is the location of the storage.

### Rook Install
Installation of the Rook operator on OpenShift

```console
ssh nodes*
sudo su -
pvcreate /dev/sdb
vgcreate virerdatavg /dev/sdb
lvcreate -n rookdata -L 8G virerdatavg
lvcreate -n rookmon -L 8G virerdatavg
```

/!\ Format ONLY the mon ones (not data/osd !!!)
```
mkfs.ext4 /dev/virerdatavg/rookmon
```

Download defintion from the remote repository
```console
git clone --single-branch --branch v1.12.3 https://github.com/rook/rook.git
cd rook/deploy/examples
```

Create the project and apply defintions
```console
oc new-project rook-ceph

oc create -f crds.yaml
oc create -f common.yaml
oc create -f operator-openshift.yaml
```

Here we need to adapt the definition to match our cluster hostnames and lvm partition
```console
cp cluster-on-local-pvc.yaml cluster-on-local-pvc-virer.yaml
sed -i 's/local-storage/local-ceph-storage/g' cluster-on-local-pvc-virer.yaml

sed -i 's/host0/infra0.ocp-lab.virer.net/g' cluster-on-local-pvc-virer.yaml
sed -i 's/host1/infra1.ocp-lab.virer.net/g' cluster-on-local-pvc-virer.yaml
sed -i 's/host2/worker0.ocp-lab.virer.net/g' cluster-on-local-pvc-virer.yaml

sed -i 's,/dev/sdb,/dev/virerdatavg/rookmon,g' cluster-on-local-pvc-virer.yaml
sed -i 's,/dev/sdc,/dev/virerdatavg/rookdata,g' cluster-on-local-pvc-virer.yaml

oc create -f cluster-on-local-pvc-virer.yaml
```

Then create some object
```console
oc create -f object-openshift.yaml
```

Create the storage pool and storage class that will be default
```console

cp csi/rbd/storageclass.yaml pool-and-storageclass-virer.yaml
sed -i 's/replicapool/virerreplicapool/g' pool-and-storageclass-virer.yaml
sed -i 's/reclaimPolicy: .*$/reclaimPolicy: Retain/g' pool-and-storageclass-virer.yaml
oc create -f pool-and-storageclass-virer.yaml
```
And create a route to access the dashboard, and get the "admin" password
```console
 oc create route passthrough --service=rook-ceph-mgr-dashboard --hostname rook-ceph-mgr-dashboard-rook-ceph.apps.ocp-lab.virer.net
 oc get secret rook-ceph-dashboard-password -o jsonpath='{.data.password}' | base64 -d
```


Enable snapshot option
```console
oc create -f ./csi/rbd/snapshotclass.yaml
```
