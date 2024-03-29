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
- [Konsumo](https://konsumo.virer.net/){:target="_blank"} A home energy consumption reporting charts
- [DSX-Networks](https://dsx-networks.com/){:target="_blank"} A web hosting company
- [My GitHub](https://github.com/virer/){:target="_blank"} repositories

### Other

Other information here:
- [Check your public IP address here](http://ip.virer.net/){:target="_blank"}
