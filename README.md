# OpenShift EFS Storage Class with stunnel

This project provides a complete implementation for TLS secured EFS storage
provisioner that can be used with OpenShift. While the build and installation
process is written specifically for OpenShift it could be easily adapted to work
with any Kubernetes deployment on AWS.

## Design Overview

OpenShift EFS stunnel storage provisioner consists of two components, a central
manager and a daemonset which runs stunnel processes on nodes in the cluster.

The manager process detects available EFS file-systems, prepares mountpoints for
volumes, and provisions persistent volumes to satisfy persistent volume claims
which match the storage class.

The stunnel daemonset is responsible for generating an stunnel configuration on
each node and running an stunnel process which listens on the host network
loopback so that NFS persistent volumes can reference the loopback interface
for mounting EFS filesystems securely.

### The `efs-stunnel` ConfigMap

Communication between the manager and stunnel pods is orchestrated through the
`efs-stunnel` configmap. This configmap has a data content item named
`efs-stunnel.yaml` which stores the mapping of EFS filesystems to mount targets
and configured stunnel ports in the key `efs_stunnel_targets`.

```
efs_stunnel_targets:
  fs-34a3p3:
    stunnel_port: 20490
    mount_target_ip_by_subnet:
      subnet-ao4u3pai: 10.0.0.4
      subnet-b4oup3be: 10.0.1.4
```

### EFS Filesystem Root Persistent Volumes and Claims

For each EFS filesystem available for the manage there will be a
PersistentVolume (PV) for the root of the EFS filesystem root. These
PVs will have `spec.nfs.path` set to "/" and `spec.storageClassName` set to
"efs-stunnel-system". For each of these there will be a bound
PersistentVolumeClaim (PVC).


The root PVs are used to create subdirectories within the EFS filesystems for
for satisfying PVCs created for the `efs-stunnel` StorageClass. Note that the
"efs-stunnel-system" storage class name used for the root PVs and PVCs does not
reference a real StorageClass.


## Installation

As an administratior, create namespace openshift-efs-stunnel:

```
oc adm new-project openshift-efs-stunnel
```

Check if rhel7 imagestream is defined in the openshift namespace:

```
oc get imagestream rhel7 -n openshift
```

Create the rhel7 imagestream if needed:

```
oc create -f rhel7-imagestream.yaml
```

Process build template to create image streams and build configs for
efs-manager and efs-stunnel components:

```
oc process -f build-template.yaml | oc create -n openshift-efs-stunnel -f -
```

Build both components:

```
oc start-build --follow efs-manager -n openshift-efs-stunnel --from-dir=.
oc start-build --follow efs-stunnel -n openshift-efs-stunnel --from-dir=.
```

Process installation template to install:

```
oc process -f install-template.yaml | oc create -f -
```

## Usage

When persistent volumes are created the provisioner components will
automatically create persistent volumes to match the claim. Claims must
include a `storageClassName` to match a configured storage class with the
provisioner "gnuthought.com/efs-stunnel". Additionally claims must at
least specify a selector with "matchLabels" set including a "volume\_name"
label. The volume name must consist of only lowercase "a-z" or the numbers
"0-9" or underscore.

For example:

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: efs-example
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
  selector:
    matchLabels:
      volume_name: example
  storageClassName: efs-stunnel
```

This will cause an NFS persistent volume to be created that points to a
managed stunnel port to map to the path `/{namespace}/{volume_name}` within
the the assigned EFS file system.

Additional supported parameters include:

`file_system_id` - File system id to request from the provisioner.

`reclaim_policy` - Reclaim policy override. Can be set to "Delete" or "Retain".

## Configuration

The installation template creates a storage class with the following definition:

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: efs-stunnel
provisioner: gnuthought.com/efs-stunnel
parameters:
  default_file_system_id: auto
reclaimPolicy: Delete
```

With this configuration the efs-manager process will automatically discover
available EFS filesystem ids by querying the EFS API. If multiple EFS
filesystems are available then one will be arbitrarily selected.

Supported parameters are:

`default_file_system_id` - Set this to the EFS filesystem id for the storage
class to provision persistent volumes.

`file_system_ids` - Restrict available file systems that can be requested to
this list. Must be given as a comma separated list.

The efs-manager deployment supports the following environment variables:

`BASE_STUNNEL_PORT` - Base local host port for creating stunnel local
configuration. Defaults to 20490.

`EFS_POLLING_INTERVAL` - How often in seconds the manager should poll the EFS
API to discover file systems. Defaults to 300 (5 minutes).

`LOGGING_LEVEL` - Logging level, defaults to "INFO".

`PROVISION_BACKOFF_INTERVAL` - Minimum time to wait between processing and
retrying after provisioning a persistent volume fails.


`STORAGE_PROVISIONER_NAME` - Provisioner name used on the storage class and in
annotations. Defaults to "gnuthought.com/efs-stunnel".

`WORKER_IMAGE` - Docker image used to launch worker containers for running
shell commands to create and clean up mount point directories. Defaults to
"rhel7:latest".

The efs-stunnel daemonset supports the following environment variables:

`LOGGING_LEVEL` - Logging level, defaults to "INFO".

`STUNNEL_CONF_PATH` - Location where the dynamically created stunnel
configuration will be created within the image. Defaults to "/tmp/stunnel.conf".

`STUNNEL_PATH` - Location of the stunnel binary within the image. By default it
searches for "/usr/local/bin/stunnel" and "/usr/bin/stunnel".

## Troubleshooting

Troubleshooting checklist:

* `efs-manager` pod logs.

* `efs-stunnel` pod logs for node where the issue was encountered.

* Events related to pod which has issue mounting EFS. This can be seen with
`oc describe pod ...`

* From the node encountering the issue, check stunnel is listening on expected
port with `sudo ss -tnlp | grep stunnel`

* Test manual NFS mount over EFS stunnel port using a command like
`mount -o port=20491 127.0.0.1:/ /mnt/efs-stunnel-test`

* Check logs of systemd using `journalctl -u atomic-openshift-node` and
`journalctl -u docker`

* On the OpenShift masters, check output of
  * `master-logs api api`
  * `master-logs controllers controllers`
