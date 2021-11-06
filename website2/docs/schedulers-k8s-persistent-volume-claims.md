---
id: schedulers-k8s-persistent-volume-claims
title: Kubernetes Persistent Volume Claims via CLI
sidebar_label: Kubernetes Persistent Volume Claims (CLI)
---
<!--
    Licensed to the Apache Software Foundation (ASF) under one
    or more contributor license agreements.  See the NOTICE file
    distributed with this work for additional information
    regarding copyright ownership.  The ASF licenses this file
    to you under the Apache License, Version 2.0 (the
    "License"); you may not use this file except in compliance
    with the License.  You may obtain a copy of the License at
      http://www.apache.org/licenses/LICENSE-2.0
    Unless required by applicable law or agreed to in writing,
    software distributed under the License is distributed on an
    "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
    KIND, either express or implied.  See the License for the
    specific language governing permissions and limitations
    under the License.
-->

> This document demonstrates how you can utilize dynamic [Persistent Volume Claims](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) in the `Executor` container. You will need to enable Dynamic Provisioning in your Kubernetes cluster to proceed to use this functionality.

<br/>

It is possible to leverage Persistent Volumes with custom Pod Templates. The CLI commands allow you to configure a Dynamic Persistent Volume Claim when you submit your topology. They also permit you to configure a Persistent Volume without a custom Pod Template. The CLI commands override any configurations you may have present in the Pod Template, but Heron's configurations will take precedence over all others.

**Note:** Heron will *not* remove any `Persistent Volume Claim`s it creates when a topology is terminated. It is thus *your responsibility* to remove them when they are no longer required.

<br>

> ***System Administrators:***
>
> * You may wish to disable the ability to configure dynamic Persistent Volume Claims specified on the CLI. To achieve this, you must pass the define option `-D heron.kubernetes.persistent.volume.claims.cli.disabled=true` to the Heron API Server on the command line during boot. This command has been added to the Kubernetes configuration files to deploy the Heron API Server and is set to `false` by default.
> * If you have a custom `Role`/`ClusterRole` for the Heron API Server you will need to ensure the `ServiceAccount` attached to the API server has the correct permissions to access the `Persistent Volume Claim`s:
>
>```yaml
>rules:
>- apiGroups: 
>  - ""
>  resources: 
>  - persistentvolumeclaims
>  verbs: 
>  - create
>```

<br>

## Usage

To configure a Persistent Volume Claim you must use the `--config-property` option with the `heron.kubernetes.volumes.persistentVolumeClaim.` command prefix. Heron will not validate your Persistent Volume Claim configurations, so please validate it to ensure is well-formed.

The command pattern is as follows:
`heron.kubernetes.volumes.persistentVolumeClaim.[VOLUME NAME].[OPTION]=[VALUE]`

The currently supported CLI `options` are:

* `claimName`
* `storageClass`
* `sizeLimit`
* `accessModes`
* `volumeMode`
* `path`
* `subPath`

***Note:*** The `accessModes` must be a comma separated list of values *without* any white space.

<br>

### Example

An example series of commands and the YAML entries they make in their respective configurations are as follows.

Commands:

```bash
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.claimName=nameOfVolumeClaim
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.storageClassName=storageClassNameOfChoice
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.accessModes=comma,separated,list
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.sizeLimit=555Gi
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.volumeMode=volumeModeOfChoice
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.path=path/to/mount
--config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.subPath=sub/path/to/mount
```

Generated `Persistent Volume Claim`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nameOfVolumeClaim
spec:
  volumeName: volumeNameOfChoice
  accessModes:
    - comma
    - separated
    - list
  volumeMode: volumeModeOfChoice
  resources:
    requests:
      storage: 555Gi
  storageClassName: storageClassNameOfChoice
```

Pod Spec entries for `Volume`:

```yaml
volumes:
  - name: volumeNameOfChoice
    persistentVolumeClaim:
      claimName: nameOfVolumeClaim
```

`Executor` container entries for `VolumeMounts`:

```yaml
volumeMounts:
  - mountPath: path/to/mount
    subPath: sub/path/to/mount
    name: volumeNameOfChoice
```

## Submitting

An example of sumbitting a topology using the example CLI commands above:

```bash
heron submit kubernetes \
  --service-url=http://localhost:8001/api/v1/namespaces/default/services/heron-apiserver:9000/proxy \
  ~/.heron/examples/heron-api-examples.jar \
  org.apache.heron.examples.api.AckingTopology acking \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.claimName=nameOfVolumeClaim \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.storageClassName=storageClassNameOfChoice \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.accessModes=comma,separated,list \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.sizeLimit=555Gi \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.volumeMode=volumeModeOfChoice \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.path=path/to/mount \
  --config-property heron.kubernetes.volumes.persistentVolumeClaim.volumeNameOfChoice.subPath=sub/path/to/mount
```

## Configuration Items Created and Entries Made

The configuration items and entries in the tables below will made in their respective areas.

One `Persistent Volume Claim`, a `Volume`, and a `VolumeMount` will be created for each `volume name` which you specify.

| Name | Description | Policy |
|---|---|---|
| `VOLUME NAME` | The `name` of the `Volume`. | Entries made in the `Persistent Volume Claim`s spec, the Pod Spec's `Volumes`, and the `executor` containers `volumeMounts`.
| `path` | The `mountPath` of the `Volume`. | Entries made in the `executor` containers `volumeMounts`.
| `subPath` | The `subPath` of the `Volume`. | Entries made in the `executor` containers `volumeMounts`.
| `claimName` | The identifier name of the `Persistent Volume Claim`. | Entries made in the `Persistent Volume Claim`s metadata and the Pod Spec's `Volume`.
| `storageClassName` | The identifier name used to reference the dynamic `StorageClass`. | Entries made in the `Persistent Volume Claim` and Pod Spec's `Volume`.
| `accessModes` | A comma separated list of access modes. | Entries made in the `Persistent Volume Claim`.
| `sizeLimit` | A resource request for storage space. | Entries made in the `Persistent Volume Claim`.
| `volumeMode` | Either `FileSystem` (default) or `Block` (raw block). [Read more](https://kubernetes.io/docs/concepts/storage/_print/#volume-mode). | Entries made in the `Persistent Volume Claim`.