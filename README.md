# Install-CP4I-on-OpenShift
Installation of IBM CloudPak for Integration 19.4.1 on RH OpenShift 4.2

Install common services
Prihlasit na load_balancer
export KUBECONFIG=~/cp4i/auth/kubeconfig
oc login --username=kubeadmin --password=7zNfS-ux6zp-zJCLD-I6Skb

sudo apt  install docker.io
tar xvf ibm-cp-int-2019.4.x-offline.tar.gz
tar xvf installer_files/cluster/images/common-services-armonk-x86_64.tar.gz -O | sudo docker load
oc config view --flatten >kubeconfig

installer_files/cluster/config.yaml	- doplnit nodes, heslo, storage

---
cluster_nodes:
  master:
    - worker1.cp4i.tec.cz.ibm.com
    - worker2.cp4i.tec.cz.ibm.com
  proxy:
    - worker1.cp4i.tec.cz.ibm.com
    - worker2.cp4i.tec.cz.ibm.com
  management:
    - worker1.cp4i.tec.cz.ibm.com
    - worker2.cp4i.tec.cz.ibm.com

default_admin_password: Passw0rd
password_rules:
#   - '^([a-zA-Z0-9\-]{32,})$'
- '(.*)'

storage_class: thin
---

sudo docker run -t --net=host -e LICENSE=accept -v $(pwd):/installer/cluster:z -v /var/run:/var/run:z -v /etc/docker:/etc/docker:z --security-opt label:disable ibmcom/icp-inception-amd64:3.2.2 addon

sudo docker run -t --net=host -e LICENSE=accept -v $(pwd):/installer/cluster:z -v /var/run:/var/run:z -v /etc/docker:/etc/docker:z --security-opt label:disable ibmcom/icp-inception-amd64:3.2.2 uninstall-addon

ICP / CloudPak / Catalog
	https://icp-console.apps.cp4i.tec.cz.ibm.com:443
	admin / Passw0rd
CP4I
	https://ibm-icp4i-prod-integration.apps.cp4i.tec.cz.ibm.com
OpenShift console
	https://console-openshift-console.apps.cp4i.tec.cz.ibm.com


Install ACE

Asi netreba, lebo instalacia common services to urobila. Rozpor medzi dokumentaciou a wizardom (ibm-anyuid-hostpath-scc vs ibm-anyuid-scc):
oc get rolebindings -n ace
oc create rolebinding ibm-anyuid-hostpath-scc --clusterrole ibm-anyuid-hostpath-scc --group=system:serviceaccounts:ace -n ace
oc get -n ace roles ibm-anyuid-scc-ace -o yaml
oc create role ibm-anyuid-hostpath-scc --verb=use --resource=securitycontextconstraints --resource-name=ibm-anyuid-hostpath-scc -n ace
oc describe -n ace role ibm-anyuid-hostpath-scc

imagePullSecret
oc get secret -n ace
"deployer-dockercfg-bwfv5"

StorageClass, persistent volume
oc get pv
oc get storageclasses

Share nfs volume z nasho servra pre RWX
cat /etc/exports
cd /mnt/nfs
mkdir ace
cd ace
mkdir ace01
chmod -R 777 .

Persistent Volume
v OpenShifte - Storage - Persistent volumes - create
apiVersion: v1
kind: PersistentVolume
metadata:
  name: acepv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: nfs
  nfs:
    path: /mnt/nfs/ace
    server: 192.168.28.17

Persistent Volume Claim
v openshifte vytvori novy PV claim, prejst do YAML zobrazenia, skopirovat yaml. Zeditovat. Vytvorit z CLI - neda sa dat iny storage class.

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: acepvc01
  namespace: ace
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  storageClassName: nfs
  volumeMode: Filesystem

oc create -n ace -f pvc.yaml

V GUI
create ACE instance:
zrusit Use dynamic provisioning

Registry
oc get services -A | grep registry					=> image-registry
oc get pods -n openshift-image-registry				=> image-registry-667d6d6d56-lfwdl
oc exec image-registry-667d6d6d56-lfwdl -i -t  bash -n openshift-image-registry
	registry_path = /registry/docker/registry/v2/repositories/
	path je odtialto dalej (ls /registry/docker/registry/v2/repositories/)
Registry = name.namespace.svc.cluster.local:Port/path/image:version
image-registry.openshift-image-registry.svc.cluster.local:5000/ace/ibm-ace-content-server-prod:11.0.0.6.1
image-registry.openshift-image-registry.svc.cluster.local:5000/mq/ibm-mqadvanced-server-integration

login to ACE admin/Passw0rd

Create integration server
Add Server, Add BAR file
