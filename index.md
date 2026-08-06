## ViReR.NeT 

### Articles

Here is some usefull articles below :
- [OpenEBS storage installation on OpenShift](info/openebs/)
- [Providing Ceph storage using Rook on OpenShift](info/ceph/rook/)
- [Apache Guacamole with podman](info/guacamole/)
- [Floating IP with Keepalived](info/keepalived/)
- [UEFI compatible iPXE ISO](info/ipxe-uefi/)

### Kubernetes / OpenShift ressources:

You may find [here a git repo](https://github.com/virer/ocp-k8s-tkn-basic-tools){:target="_blank"} for a container bundling OpenShift cli,kubectl and tekton cli.
You may use it with that kind of script:
```bash
    #!/bin/bash
    export PATH=$PATH:/usr/local/bin 
    
    # Here you extract the token exposed by your serviceAccount
    TOKEN=$( cat /run/secrets/kubernetes.io/serviceaccount/token ) 
    
    # Then you login on the OpenShift cluster
    oc login --token=$TOKEN https://kubernetes.default.svc 
    
    # Then change namespace/project and make some cleanup inside :
    oc project $NAMESPACE
    for pod in $(/usr/local/bin/tkn pipelinerun list | awk '/[2-9] days ago.*(Cancelled|Succeeded|Failed)/ { print $1 }'); do
      tkn pipelinerun delete ${pod} --force
    done
```
### Project

Current project :
- [Virium](https://github.com/virer/viriumd){:target="_blank"} A storage solution for Kubernetes based on iSCSI and LVM(Logical Volume Manager).
- [Konsumo](https://konsumo.virer.net/){:target="_blank"} A home energy consumption reporting charts
- [BBQ Timer](https://virer.github.io/bbq-timer/){:target="_blank"} A barbecue time manager to never eat burned food again!
- [My GitHub](https://github.com/virer/){:target="_blank"} repositories

### Homelab notes

Some usefull notes, scripts, tips and tricks used in my homelabs or in real-life:
- [Homelab-public](https://github.com/virer/homelab-public){:target="_blank"} homelab-public repository




