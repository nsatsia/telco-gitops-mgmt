# Mirroring OCP

## Reference
https://docs.openshift.com/container-platform/4.8/installing/installing-mirroring-installation-images.html

# Get pull-secret from here: 
https://console.redhat.com/openshift/install/pull-secret

**place in ~/pull-secret.json**

## Generate the base64-encoded user name and password for Quay and add to pull-secrets

```bash

host_fqdn=lab-registry.ocpsmc.lab.diktio.net; echo $host_fqdn

b64auth=$(echo -n 'nsatsia:Redhat01' | base64 -w0); echo $b64auth
AUTHSTRING="{\"$host_fqdn\": {\"auth\": \"$b64auth\",\"email\": \"nsatsia@redhat.com\"}}"; echo $AUTHSTRING

jq -c ".auths += $AUTHSTRING" < ~/pull-secret.json > ~/pull-secret-update.json
```

# Mirroring the OpenShift Container Platform image repository

## Prepare
```bash
mkdir -p ~/.docker
cp ~/pull-secret-update.json ~/.docker/config.json

export OCP_REL_MAJ=4
export OCP_REL_MIN=8
export OCP_REL_MNT=17

export RELEASE_IMAGE=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}/release.txt | grep 'Pull From: quay.io' | awk -F ' ' '{print $3}'); echo $RELEASE_IMAGE

export UPSTREAM_REPO=${RELEASE_IMAGE}
export LOCAL_REGISTRY='lab-registry.ocpsmc.lab.diktio.net'
export LOCAL_REPOSITORY='nsatsia/ocp'${OCP_REL_MAJ}${OCP_REL_MIN}${OCP_REL_MNT}
export LOCAL_SECRET_JSON=~/pull-secret-update.json

```

## Option1: Directly to Mirror (Quay)
```bash
oc adm release mirror \
  -a $LOCAL_SECRET_JSON \
  --from=$UPSTREAM_REPO \
  --to-release-image=$LOCAL_REGISTRY/$LOCAL_REPOSITORY:${OCP_RELEASE} \
  --to=$LOCAL_REGISTRY/$LOCAL_REPOSITORY

for image in \
  "assisted-installer-agent:latest" \
  "assisted-installer-controller:latest" \
  "assisted-installer:latest" \
  "assisted-service:latest"  \
  "assisted-service:latest" \
  "ocp-metal-ui:latest" \
  "postgresql-12-centos7" \
  "s3server" 
  do oc image mirror quay.io/ocpmetal/$image lab-registry.ocpmsmc.lab.diktio.net/nsatsia/$image
done

oc image mirror quay.io/openshift/origin-cli:latest lab-registry.ocpsmc.lab.diktio.net/nsatsia/origin-cli:latest

oc -a $LOCAL_SECRET_JSON image mirror quay.io/openshift-release-dev/ocp-release:${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}-x86_64 lab-registry.ocpsmc.lab.diktio.net/nsatsia/ocp${OCP_REL_MAJ}${OCP_REL_MIN}${OCP_REL_MNT} --insecure=true

export CVO_DIGEST=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}/release.txt | grep 'cluster-version-operator' | awk '{print $2}'); echo $CVO_DIGEST
oc -a $LOCAL_SECRET_JSON image mirror $CVO_DIGEST lab-registry.ocpsmc.lab.diktio.net/nsatsia/ocp${OCP_REL_MAJ}${OCP_REL_MIN}${OCP_REL_MNT} --insecure=true

export AIC_DIGEST=$(skopeo inspect docker://quay.io/ocpmetal/assisted-installer-controller:latest | egrep Digest | awk -F '"' '{print $4}'); echo $AIC_DIGEST

```
**Set AIC_DIGEST to "CONTROLLER_IMAGE" in "assisted-service-config" ConfigMap**

## Option2: Mirror to local disk first
Change to the desired base directory.
Example:
```bash
cd /mnt/2tRaid1/mirror
```

```bash
oc adm release mirror \
  -a $LOCAL_SECRET_JSON \
  --from=$UPSTREAM_REPO \
  --to-dir=ocp/${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}
```

Example completion output:
```bash
.
.
.
info: Mirroring completed in 17m49.75s (8.616MB/s)

Success
Update image:  openshift/release:4.8.15-x86_64

To upload local images to a registry, run:

    oc image mirror --from-dir=ocp/4.8.15 'file://openshift/release:4.8.15-x86_64*' REGISTRY/REPOSITORY

Configmap signature file ocp/4.8.15/config/signature-sha256-92b684258b9f80da.yaml created
[root@hypervisor1 mirror]# 
```

```bash
for image in \
  "assisted-installer-agent:latest" \
  "assisted-installer-controller:latest" \
  "assisted-installer:latest" \
  "assisted-service:latest"  \
  "assisted-service:latest" \
  "ocp-metal-ui:latest" \
  "postgresql-12-centos7" \
  "s3server" 
  do oc image mirror quay.io/ocpmetal/$image file://ocp/ocpmetal/$image
done

oc image mirror quay.io/openshift/origin-cli:latest file://ocp/openshift/origin-cli:latest

oc -a ~/pull-secret-update.json image mirror quay.io/openshift-release-dev/ocp-release:${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}-x86_64 file://ocp/openshift-release-dev/ocp-release:${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}-x86_64 --insecure=true

export CVO_DIGEST=$(curl -s https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}/release.txt | grep 'cluster-version-operator' | awk '{print $2}'); echo $CVO_DIGEST
oc -a ~/pull-secret-update.json image mirror $CVO_DIGEST file://ocp/openshift-release-dev/ocp-v4.0-art-dev --insecure=true

export AIC_DIGEST=$(skopeo inspect docker://quay.io/ocpmetal/assisted-installer-controller:latest | egrep Digest | awk -F '"' '{print $4}'); echo $AIC_DIGEST

```
**Set AIC_DIGEST to "CONTROLLER_IMAGE" in "assisted-service-config" ConfigMap**

## Option2(cont.): From local disk to Mirror (Quay)
```bash
oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=/mnt/2tRaid1/mirror/ocp/${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT} "file://openshift/release:${OCP_REL_MAJ}.${OCP_REL_MIN}.${OCP_REL_MNT}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} 

for image in \
  "assisted-installer-agent:latest" \
  "assisted-installer-controller:latest" \
  "assisted-installer:latest" \
  "assisted-service:latest"  \
  "assisted-service:latest" \
  "ocp-metal-ui:latest" \
  "postgresql-12-centos7" \
  "s3server" 
  do oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=/mnt/2tRaid1/mirror file://ocp/ocpmetal/$image ${LOCAL_REGISTRY}/nsatsia/$image
done

oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=/mnt/2tRaid1/mirror file://ocp/openshift/origin-cli:latest ${LOCAL_REGISTRY}/nsatsia/origin-cli:latest

TAG=$(ls -t1 v2/ocp/openshift-release-dev/ocp-v4.0-art-dev/manifests/| sed -n '1p'); echo $TAG
oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=/mnt/2tRaid1/mirror file://ocp/openshift-release-dev/ocp-v4.0-art-dev@$TAG ${LOCAL_REGISTRY}/nsatsia/ocp${OCP_REL_MAJ}${OCP_REL_MIN}${OCP_REL_MNT}
