# Mirroring an Operator catalog

## References
https://docs.openshift.com/container-platform/4.8/operators/admin/olm-restricted-networks.html#olm-mirror-catalog_olm-restricted-networks

https://github.com/fullstorydev/grpcurl/releases

https://docs.openshift.com/container-platform/4.8/operators/admin/olm-restricted-networks.html#olm-mirror-catalog_olm-restricted-networks

https://docs.openshift.com/container-platform/4.8/cli_reference/opm-cli.html

https://docs.openshift.com/container-platform/4.8/operators/admin/olm-restricted-networks.html#olm-pruning-index-image_olm-restricted-networks

# Mirror to local disk first
Change to the desired base directory.

Example:
```bash
cd /mnt/2tRaid1/mirror
```
# Login to registries
podman login registry.redhat.io
podman login lab-registry.ocpsmc.lab.diktio.net

# Optional, if you need a list off current operators

Get utilities:
```bash
wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.2/grpcurl_1.8.2_linux_x86_64.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/latest-4.8/opm-linux-4.8.13.tar.gz
tar xvfz grpcurl_1.8.2_linux_x86_64.tar.gz
tar xvfz opm-linux-4.8.13.tar.gz
```
## Start a local container (assumes podman installed):
```bash
podman run -p50051:50051 \
    -it registry.redhat.io/redhat/redhat-operator-index:v4.8
```
## Get current list of Red Hat Operators
```bash
./grpcurl -plaintext localhost:50051 api.Registry/ListPackages > packages.out

```

## Decide on Operators from packages.out
**To be used in next step**

## Prune Index down to required operators
If running this step again, delete the existing index in local podman:
```bash
podman rmi lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8
```
Create a new pruned index:
```bash
./opm index prune \
        -f  registry.redhat.io/redhat/redhat-operator-index:v4.8  \
        -p  advanced-cluster-management,ansible-automation-platform-operator,container-security-operator,kubernetes-nmstate-operator,kubevirt-hyperconverged,local-storage-operator,nfd,ocs-operator,openshift-gitops-operator,openshift-pipelines-operator-rh,performance-addon-operator,ptp-operator,quay-operator,rhacs-operator,rhsso-operator,sandboxed-containers-operator,serverless-operator,service-registry-operator,service-telemetry-operator,servicemeshoperator,skupper-operator,smart-gateway-operator,sriov-network-operator,submariner,vertical-pod-autoscaler,web-terminal \
        -t lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8
```
Push the new index to the mirror registry (Quay) so the next step oc command can use it:
```bash
podman push lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8
```
Mirror reduced index to local disk
```bash
oc adm catalog mirror \
    lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8 \
    file://operator-mirror \
    -a ~/pull-secret-update.json \
    --index-filter-by-os='linux/amd64' 
```
Example completion output:
```bash
.
.
.
info: Mirroring completed in 55m59.25s (8.065MB/s)
error mirroring image: one or more errors occurred
wrote mirroring manifests to manifests-redhat-operator-index-1635832087

To upload local images to a registry, run:

        oc adm catalog mirror file://operator-mirror/nsatsia/redhat-operator-index:v4.8 REGISTRY/REPOSITORY
[root@hypervisor1 mirror]# 
```
-------------------------------------------

# Mirror from local disk to the mirror (Quay)
```bash
oc adm catalog mirror \
    file://operator-mirror/nsatsia/redhat-operator-index:v4.8 \
    lab-registry.ocpsmc.lab.diktio.net/nsatsia \
    -a ~/pull-secret-update.json
```

Example completion output:
```bash
.
.
.
info: Mirroring completed in 3h28m17.26s (20.13MB/s)
error mirroring image: one or more errors occurred
no digest mapping available for file://operator-mirror/nsatsia/redhat-operator-index/openshift-service-mesh/proxy-init-rhel7:2.0.4, skip writing to ImageContentSourcePolicy
no digest mapping available for file://operator-mirror/nsatsia/redhat-operator-index:v4.8, skip writing to ImageContentSourcePolicy
wrote mirroring manifests to manifests-nsatsia/redhat-operator-index-1635842800
[root@hypervisor1 mirror]# 
```
**NOTE** the "mirroring manifest" diectrory for the next step.
```bash
export MANIFESTS="manifests-nsatsia/redhat-operator-index-1635842800"
```


# Define mirror to disconnected cluster
Set KUBECONFIG for disconnected cluster
```bash
export KUBECONFIG=~/kubeconfig-ocptest
```

Adjust the imageContentSourcePolicy.yaml and catalogSource.yaml to reflect the upstream registry and not the local file one.

**Adjust the sed search for the username used on the mirror registry (Quay)

```bash
sed -i -e 's/operator-mirror\/nsatsia\/redhat-operator-index/registry.redhat.io/g' $MANIFESTS/imageContentSourcePolicy.yaml

sed -i -e 's/name\:\ nsatsia\/redhat-operator-index/name\:\ redhat-operator-index/g' ma$MANIFESTS/catalogSource.yaml
```
## Push the CRs to the disconnected cluster

```bash
oc create -f manifests-nsatsia/redhat-operator-index-1635842800/imageContentSourcePolicy.yaml

oc create -f manifests-nsatsia/redhat-operator-index-1635842800/catalogSource.yaml
```

**NOTE** the following from the documentation. Normally does not apply for a disconnected cluster as the mirror for the OCP build is set already so the nodes will not reboot.
```bash
####
Applying the ICSP causes all worker nodes in the cluster to restart. You must wait for this reboot process to finish cycling through each of your worker nodes before proceeding.
####
```
# OPTIONAL: Mirror direct to the mirror (Quay) from Internet
```bash
podman login registry.redhat.io
./opm index prune \
  -f registry.redhat.io/redhat/redhat-operator-index:v4.8 \
  -p  advanced-cluster-management,ansible-automation-platform-operator,container-security-operator,kubernetes-nmstate-operator,kubevirt-hyperconverged,local-storage-operator,nfd,ocs-operator,openshift-gitops-operator,openshift-pipelines-operator-rh,performance-addon-operator,ptp-operator,quay-operator,rhacs-operator,rhsso-operator,sandboxed-containers-operator,serverless-operator,service-registry-operator,service-telemetry-operator,servicemeshoperator,skupper-operator,smart-gateway-operator,sriov-network-operator,submariner,vertical-pod-autoscaler,web-terminal  \
  -t lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8

podman push lab-registry.ocpsmc.lab.diktio.net/nsatsia/redhat-operator-index:v4.8

oc adm catalog mirror \
  lab-registry.ocpsmc.lab.diktio.net:443/nsatsia/redhat-operator-index:v4.8 \
  lab-registry.ocpsmc.lab.diktio.net:443/nsatsia \
  -a pull-secret-update.json \
  --index-filter-by-os='linux/amd64' 
```


# Nice "oc" command option: run an instant registry 
```bash
oc image serve --tls-crt='/root/hypervisor1.crt.pem' --tls-key='/root/hypervisor1.key.pem' --dir /mnt/2tRaid1/mirror --listen='192.19.130.11:5000'

```
