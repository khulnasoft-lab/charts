# IBM UrbanCode Deploy - Helm Chart

## Introduction

[IBM UrbanCode Deploy](https://www.ibm.com/cloud/urbancode/deploy) is a tool for automating application deployments through your environments. It is designed to facilitate rapid feedback and continuous delivery in agile development while providing the audit trails, versioning and approvals needed in production.

## Chart Details

* This chart deploys a single server instance of IBM UrbanCode Deploy that may be scaled to multiple instances.
* The Persistent Volume access modes ReadWriteOnce (RWO) and ReadWriteMany (RWX) are both supported for use with IBM UrbanCode Deploy server.  However, ReadWriteMany is required to successfully scale to more than one replica/instance of the server.
* Includes two statefulSet workload objects, one for server instances and one for distributed front end instances, and corresponding services for them.
* Support has been validated on OpenShift clusters running onPrem, in IBM Satellite, and IBM ROKS.

## Prerequisites

1. Kubernetes 1.19.0+; kubectl and oc CLI; Helm 3;
  * Install and setup oc/kubectl CLI depending on your architecture.
    * [ppc64le](https://mirror.openshift.com/pub/openshift-v4/ppc64le/clients/ocp/stable/openshift-client-linux.tar.gz)
    * [s390x](https://mirror.openshift.com/pub/openshift-v4/s390x/clients/ocp/stable/openshift-client-linux.tar.gz)
    * [x86_64](https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz)
  * [Install and setup the Helm 3 CLI](https://helm.sh/docs/intro/install/).

2. Image and Helm Chart - The UCD server image and helm chart can be accessed via the Entitled Registry and public Helm repository.
  * The public Helm chart repository can be accessed at https://github.com/IBM/charts/tree/master/repo/ibm-helm and directions for accessing the UrbanCode Deploy server chart will be discussed later in this README.
  * Get a key to the entitled registry
    * Log in to [MyIBM Container Software Library](https://myibm.ibm.com/products-services/containerlibrary) with the IBMid and password that are associated with the entitled software.
    * In the Entitlement keys section, select Copy key to copy the entitlement key to the clipboard.
    * An imagePullSecret must be created to be able to authenticate and pull images from the Entitled Registry.  If the secret is named ibm-entitlement-key it will be used as the default pull secret, no value needs to be specified in the image.secret field.  Once this secret has been created you will specify the secret name as the value for the image.secret parameter in the values.yaml you provide to 'helm install ...'  Note: Secrets are namespace scoped, so they must be created in every namespace you plan to install UCD into.  Following is an example command to create an imagePullSecret named 'ibm-entitlement-key'.

```
oc create secret docker-registry ibm-entitlement-key --docker-username=cp --docker-password=<EntitlementKey> --docker-server=cp.icr.io
```

3. Database - UrbanCode Deploy requires a database.  The database may be running in your cluster or on hardware that resides outside of your cluster.  This database  must be configured as described in [Installing the server database](https://www.ibm.com/support/knowledgecenter/SS4GSP_7.1.1/com.ibm.udeploy.install.doc/topics/DBinstall.html) before installing the containerized UrbanCode Deploy server.  The values used to connect to the database are required when installing the UrbanCode Deploy server.  The Apache Derby database type is not supported when running the UrbanCode Deploy server in a Kubernetes cluster.

4. Secret - A Kubernetes Secret object must be created to store the initial UrbanCode Deploy server administrator password, the password used to access the database mentioned above, and the password for all keystores used by the UrbanCode Deploy server.  The name of the secret you create must be specified in the property 'secret.name' in your values.yaml.

* Through the oc/kubectl CLI, create a Secret object in the target namespace.

```bash
oc create secret generic ucd-secrets \
  --from-literal=initpassword=admin \
  --from-literal=dbpassword=MyDbpassword \
  --from-literal=keystorepassword=MyKeystorePassword

```

5. JDBC drivers - A PersistentVolume (PV) that contains the JDBC driver(s) required to connect to the database configured above must be created.  You must either:

* Create Persistence Storage Volume - Create a PV, copy the JDBC driver(s) to the PV, and create a PersistentVolumeClaim (PVC) that is bound to the PV. For more information on Persistent Volumes and Persistent Volume Claims, see the [Kubernetes documentation](https://kubernetes.io/docs/concepts/storage/persistent-volumes). Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-ext-lib
  labels:
    volume: ucd-ext-lib-vol
spec:
  capacity:
    storage: 100Mi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/ucd-ext-lib
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-ext-lib-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 100Mi
  selector:
    matchLabels:
      volume: ucd-ext-lib-vol
```
* Dynamic Volume Provisioning - If your cluster supports [dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/), you may use it to create the PV and PVC. However, the JDBC drivers will still need to be copied to the PV. To copy the JDBC driver(s) to your PV during the chart installation process, first write a bash script that copies the JDBC driver(s) from a location accessible from your cluster to `${UCD_HOME}/ext_lib/`. Next, store the script, named `script.sh`, in a yaml file describing a [ConfigMap](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/).  Finally, create the ConfigMap in your cluster by running a command such as `oc create configmap <map-name> <data-source>`.  Below is an example ConfigMap yaml file that copies a MySQL .jar file from a web server using wget.

```
kind: ConfigMap
apiVersion: v1
metadata:
  name: user-script
data:
  script.sh: |
    #!/bin/bash
    echo "Running script.sh..."
    if [ ! -f ${UCD_HOME}/ext_lib/mysql-jdbc.jar ] ; then
      echo "Copying file(s)..."    
      wget -L -O mysql-jdbc.jar http://webserver-example/mysql-jdbc.jar
      mv mysql-jdbc.jar ${UCD_HOME}/ext_lib/
      echo "Done copying."
    else
      echo "File ${UCD_HOME}/ext_lib/mysql-jdbc.jar already exists."
    fi
```
  * Note the script must be named `script.sh`.

6. A PersistentVolume that will hold the appdata directory for the UrbanCode Deploy server is required.  If your cluster supports dynamic volume provisioning you will not need to manually create a PersistentVolume (PV) or PersistentVolumeClaim (PVC) before installing this chart.  If your cluster does not support dynamic volume provisioing, you will need to either ensure a PV is available or you will need to create one before installing this chart.  You can optionally create the PVC to bind it to a specific PV, or you can let the chart create a PVC and bind to any available PV that meets the required size and storage class.  Sample YAML to create the PV and PVC are provided below.

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ucd-appdata-vol
  labels:
    volume: ucd-appdata-vol
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 192.168.1.17
    path: /volume1/k8/ucd-appdata
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: ucd-appdata-volc
spec:
  storageClassName: ""
  accessModes:
    - "ReadWriteOnce"
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      volume: ucd-appdata-vol
```
  * The following storage options have been tested with IBM UrbanCode Deploy

    * IBM Block Storage supports the ReadWriteOnce access mode.  ReadWriteMany is not supported.

    * IBM File Storage supports ReadWriteMany which is required for Distributed Front End(DFE).

  * IBM UrbanCode Deploy requires non-root access to persistent storage. When using IBM File Storage you need to either use one of the IBM provided “gid” file storage classes (ie. ibmc-file-gold-gid) with default group ID 65531 or create your own customized storage class to specify a different group ID. See the information at https://cloud.ibm.com/docs/containers?topic=containers-cs_storage_nonroot for more details.  Once you know the correct group ID, set the persistence.fsGroup property in the values.yaml to that group ID.

7.  If a route or ingress is used to access the WSS port of the UrbanCode Deploy server from an UrbanCode Deploy agent, then port 443 should be specified along with the configured URL to access the proper service port defined for the UrbanCode Deploy Server.

### PodSecurityPolicy Requirements

If you are running on OpenShift, skip this section and continue to the [SecurityContextConstraints Requirements](#securitycontextconstraints-requirements) section below.

This chart requires a PodSecurityPolicy to be bound to the target namespace prior to installation. Choose either a predefined PodSecurityPolicy or have your cluster administrator create a custom PodSecurityPolicy for you.

The predefined PodSecurityPolicy named [`ibm-restricted-psp`](https://ibm.biz/cpkspec-psp) has been verified for this chart, if your target namespace is bound to this PodSecurityPolicy you can proceed to install the chart.

  * Custom PodSecurityPolicy definition:
    ```
    apiVersion: extensions/v1beta1
    kind: PodSecurityPolicy
    metadata:
      annotations:
        kubernetes.io/description: "This policy is based on the most restrictive policy,
        requiring pods to run with a non-root UID, and preventing pods from accessing the host."
        seccomp.security.alpha.kubernetes.io/allowedProfileNames: docker/default
        seccomp.security.alpha.kubernetes.io/defaultProfileName: docker/default
      name: ibm-ucd-prod-psp
    spec:
      allowPrivilegeEscalation: false
      forbiddenSysctls:
      - '*'
      fsGroup:
        ranges:
        - max: 65535
          min: 1
        rule: MustRunAs
      hostNetwork: false
      hostPID: false
      hostIPC: false
      requiredDropCapabilities:
      - ALL
      runAsUser:
        rule: MustRunAsNonRoot
      seLinux:
        rule: RunAsAny
      supplementalGroups:
        ranges:
        - max: 65535
          min: 1
        rule: MustRunAs
      volumes:
      - configMap
      - emptyDir
      - projected
      - secret
      - downwardAPI
      - persistentVolumeClaim
    ```

### SecurityContextConstraints Requirements

This chart requires a `SecurityContextConstraints` to be bound to the target namespace prior to installation.  The default `SecurityContextConstraints` named restricted has been verified for this chart, if your target namespace is bound to this `SecurityContextConstraints` resource you can proceed to install the chart.

  * Custom SecurityContextConstraints definition:

  ```yaml
  apiVersion: security.openshift.io/v1
  kind: SecurityContextConstraints
  allowHostDirVolumePlugin: false
  allowHostIPC: false
  allowHostNetwork: false
  allowHostPID: false
  allowHostPorts: false
  allowPrivilegeEscalation: true
  allowPrivilegedContainer: false
  allowedCapabilities: null
  defaultAddCapabilities: null
  fsGroup:
    type: MustRunAs
  metadata:
    annotations:
      kubernetes.io/description: restricted denies access to all host features and requires
        pods to be run with a UID, and SELinux context that are allocated to the namespace.  This
        is the most restrictive SCC and it is used by default for authenticated users.
    name: ucd-restricted
  priority: null
  readOnlyRootFilesystem: false
  requiredDropCapabilities:
  - KILL
  - MKNOD
  - SETUID
  - SETGID
  runAsUser:
    type: MustRunAsRange
  seLinuxContext:
    type: MustRunAs
  supplementalGroups:
    type: RunAsAny
  users: []
  volumes:
  - configMap
  - downwardAPI
  - emptyDir
  - persistentVolumeClaim
  - projected
  - secret
  ```

### Licensing Requirements

The UCD server image will attempt to upload UCD license metrics(agent high-water mark) to the license service. For the upload to be successful, this chart needs IBM Licensing operator (a component of IBM Common Services) to be installed in the Openshift cluster. Please follow these [instructions](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/landing_installer.html) to install IBM Common services.

Once the common services are installed, the IBM Licensing service can be made accessible to the UCD server by creating an OperandRequest resource to copy the license service secret(ibm-licensing-upload-token) and configmap(ibm-licensing-upload-config) to the namespace/project the UCD server will be installed in. Click [here](https://www.ibm.com/support/knowledgecenter/SSHKN6/installer/3.x.x/bind_info.html#license-bind) for more information.  It is only required to create an OperandRequest resource with the ibm-licensing-operator information. Following is an example yaml file contents that would create an OperandRequest which was tested with IBM Common services 3.23.0.  Add the following yaml to a file named operandrequest.yaml and then run `oc apply -f ./operandrequest.yaml` in the namespace/project where the UCD server will be installed.

```yaml
apiVersion: operator.ibm.com/v1alpha1
kind: OperandRequest
metadata:
  name: ibm-licensing-request
spec:
  requests:
  - operands:
      - name: ibm-licensing-operator
        bindings:
          public-api-upload:
            secret: ibm-licensing-upload-token
            configmap: ibm-licensing-upload-config
    registry: common-service
    registryNamespace: ibm-common-services
    description: "Requesting the Licensing Service"
```
To retrieve license usage data, please follow these [instructions](https://www.ibm.com/docs/en/cloud-paks/cp-integration/2021.1?topic=service-retrieving-license-usage-data).

## Resources Required

* 4GB of RAM, plus 4MB of RAM for each agent
* 2 CPU cores, plus 2 cores for each 500 agents

## Client Data Storage Locations

All client data is stored in either the user specified database or the appdata persistent volume.  UrbanCode Deploy does not do any active encryption of these data locations.  These locations should be included in whatever backup plans the user chooses to implement.

## Installing the Chart

Add the IBM helm chart repository to the local client.
```bash
$ helm repo add ibm-helm https://raw.githubusercontent.com/IBM/charts/master/repo/ibm-helm/
```

Get a copy of the values.yaml file from the helm chart so you can update it with values used by the install.
```bash
$ helm inspect values ibm-helm/ibm-ucd-prod > myvalues.yaml
```

Edit the file myvalues.yaml to specify the parameter values to use when installing the UCD server instance.  The [configuration](#Configuration) section lists the parameter values that can be set.

To install the chart into namespace 'ucdtest' with the release name `my-ucd-release` and use the values from myvalues.yaml:

```bash
$ helm install my-ucd-release ibm-helm/ibm-ucd-prod --namespace ucdtest --values myvalues.yaml
```

> **Tip**: List all releases using `helm list`.

## Verifying the Chart

See the instructions (from NOTES.txt within chart) after the helm installation completes for chart verification. The instruction can also be viewed by running the command: helm status my-ucd-release --tls.

## Upgrading the Chart

Check [here](https://community.ibm.com/community/user/wasdevops/blogs/laurel-dickson-bull1/2022/07/08/container-upgrade) for information about ugrading the chart.

## Uninstalling the Chart

To uninstall/delete the `my-ucd-release` release:

```bash
$ helm delete my-ucd-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Disaster Recovery

Backup product data and essential Kubernetes resources so that you can recover your UCD server instance after a disaster.

### Backup Kubernetes Resources

Backup the Kubernetes resoures required to redeploy the UCD server after a disaster.  Follow these steps to save the configuration of essential Kubernetes resources.

1. Save Helm values
   Run the following command to save a local copy of the Helm values file
```bash
helm get values <Helm-release-name> --namespace <ucd_namespace> --all >savedHelmValues.yaml
```
2. Save secret containing UCD server product passwords
   Find the value for the Values.secret.name property in the saved Helm values file above.  This is the name of the secret we want to save a local copy of.  Run the following command, replacing **ucdsecrets_name** with the value from the values.secret.name property.
```bash
oc get secret <ucdsecrets_name> -n <ucd_namespace> -o yaml > <ucdsecrets_name>.yaml
```
3. Save image pull secret
   Find the value for the Values.image.secret property in the saved Helm values file above.  This is the name of the secret used to pull images from the IBM Entitled Registry.  Run the following command, replacing **ibm-entitlement-key** with the value from the Values.image.secret property.
```bash
oc get secret <ibm-entitlement-key> -n <ucd_namespace> -o yaml > <ibm-entitlement-key>.yaml
```
4. Save ext-lib configmap

   If a configmap was used to load the JDBC driver file into the ext-lib Persistent Volume, you will need to save a local copy of it.
   Find the value for the Values.extLibVolume.configMapName property in the saved Helm values file above.
   Run the following command, replacing **configMapName** with the value from the Values.extLibVolume.configMapName property.
```bash
oc get secret <configMapName> -n <ucd_namespace> -o yaml > <configMapName>.yaml
```

### Backup Product Data

Backup the database and appdata directory used by the UCD server.  To ensure the most accurate saving of data, no deployments should be active.  Follow these steps to take a backup of the server.

1. Scale the statefulset resource to 0 to shutdown the UCD server.
2. Create a full backup of the database.  For instructions on backing up the database, see the documentation from your database vendor.
3. Backup the appdata Persistent Volume.
4. Backup the ext-lib Persistent Volume.
5. Scale the statefulset resource to 1 to restart the UCD server.

### Recover from a disaster

If you have successfully backed up the resources and data as described in [Backup Kubernetes Resources](#backup-kubernetes-resources) and [Backup Product Data](#backup-product-data) you can recreate an instance of UCD server using that data.  Follow these steps to recreate your UCD server instance.

1. Create a new project/namespace to hold the Kubernetes resources associated with the UCD server instance.
2. Create the Kubernetes secret that contains the UCD server product passwords by running the following command.
```bash
oc apply -n <ucd_namespace> -f <ucdsecrets_name>.yaml
```
3. Create the image pull secret needed to access images in the IBM Entitled Registry by running the following command.
```bash
oc apply -n <ucd_namespace> -f <ibm-entitlement-key>.yaml
```
4. If your original UCD server instance used a configMap resource to load the JDBC driver file into the ext-lib Persistent Volume, then recreate that configMap resource by running the following command.
```bash
oc apply -n <ucd_namespace> -f <configMapName>.yaml
```
Create the ext-lib Persistent Volume that the JDBC driver file will be loaded into.  Also create a Persistent Volume Claim that references the ext-lib Persistent Volume.  If you are not using a configMap to load the JDBC driver file into the ext-lib Persistent Volume, you will need to manually copy the JDBC driver file into the Persistent Volume.
5. Create the appdata Persistent Volume and associated Persistent Volume Claim and load the saved appdata directory contents into the Persistent Volume.
6. Follow the directions from your database vendor to create a new database from the backup/clone.
7. Create a values.yaml file that contains the properties and values from your savedHelmValues.yaml file.  Be sure that the Values.extLibVolume.existingClaimName and Values.appDataVolume.existingClaimName fields are set to the Persistent Volume Claims for the new ext-lib and appdata Persistent Volumes.  Also be sure that the database fields Values.database.* refer to the new database instance created in the step above.
8. Create the new UCD server instance by running the following command.
```bash
helm install my-recovered-release ibm-helm/ibm-ucd-prod --namespace <ucd_namespace> --values myRecoveredValues.yaml
```

## Configuration

### Parameters

The Helm chart has the following values.

##### Common Parameters

| Qualifier | Parameter  | Definition | Allowed Value |
|---|---|---|---|
| version |  | UrbanCode Deploy product version |  |
| replicas | server | Number of UCD server replicas | Non-zero number of replicas.  Defaults to 1 |
|          | dfe | Number of DFE replicas | Number of Distributed Front End replicas.  Defaults to 0 |
| image | pullPolicy | Image Pull Policy | Always, Never, or IfNotPresent. Defaults to Always |
|       | secret |  An image pull secret used to authenticate with the image registry | Empty (default) if no authentication is required to access the image registry. |
| service | type | Specify type of service | Valid options are ClusterIP, NodePort and LoadBalancer (for clusters that support LoadBalancer). Default is ClusterIP |
| database | type | The type of database UCD will connect to | Valid values are db2, mysql, oracle, and sqlserver |
|          | name | The name of the database to use |  |
|          | hostname | The hostname/IP of the database server | |
|          | port | The database port to connect to | |
|          | username | The user to access the database with | |
|          | jdbcConnUrl | The JDBC Connection URL used to connect to the database used by the UCD server. This value is normally constructed using the database type and other database field values, but must be specified here when using Oracle RAC/ORAAS or SQL Server with Integrated Security. | |
| secureConnections  | required | Specify whether UCD server connections are required to be secure | Default value is "true" |
| secret | name | Kubernetes secret which defines required UCD passwords. | You may leave this blank to use default name of HelmReleaseName-secrets where HelmReleaseName is the name of your Helm Release, otherwise specify the secret name here. |
| license | accept | Set to true to indicate you have read and agree to license agreements : http://www-03.ibm.com/software/sla/sladb.nsf/searchlis/?searchview&searchorder=4&searchmax=0&query=(urbancode+deploy) | false |
|  | serverURL | Information required to connect to the UCD license server. | Empty (default) to begin a 60-day evaluation license period.|
| persistence | enabled | Determines if persistent storage will be used to hold the UCD server appdata directory contents. This should always be true to preserve server data on container restarts. | Default value "true" |
|             | useDynamicProvisioning | Set to "true" if the cluster supports dynamic storage provisoning | Default value "false" |
|             | fsGroup | The group ID to use to access persistent volumes | Default value "1001" |
| extLibVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the extlib directory is created by the chart. | Default value is "ext-lib" |
|              | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|              | size | Size of the volume used to hold the JDBC driver .jar files |  |
|              | existingClaimName | Persistent volume claim name for the volume that contains the JDBC driver file(s) used to connect to the UCD database. |  |
|              | configMapName | Name of an existing ConfigMap which contains a script named script.sh. This script is run before UrbanCode Deploy server installation and is useful for copying database driver .jars to the ext-lib persistent volume. |  |
|              | accessMode | Persistent storage access mode for the ext-lib persistent volume. | ReadWriteOnce |
| appDataVolume | name | The base name used when the Persistent Volume and/or Persistent Volume Claim for the UCD server appdata directory is created by the chart. | Default value is "appdata" |
|               | existingClaimName | The name of an existing Persistent Volume Claim that references the Persistent Volume that will be used to hold the UCD server appdata directory. |  |
|               | storageClassName | The name of the storage class to use when persistence.useDynamicProvisioning is set to "true". |  |
|               | size | Size of the volume to hold the UCD server appdata directory |  |
|              | accessMode | Persistent storage access mode for the appdata persistent volume. | ReadWriteOnce |
| ingress | host | Host name used to access the UCD server UI. Leave blank on OpenShift to create default route. |  |
|               | dfehost | Host name used to access the UCD server distributed front end (DFE) UI. Leave blank on OpenShift to create default route. |  |
|               | wsshost | Host name used to access the UCD server WSS port. Leave blank on OpenShift to create default route. |  |
|               | jmshost | Host name used to access the UCD server JMS port. Leave blank on OpenShift to create default route. |  |
| resources | constraints.enabled | Specifies whether the resource constraints specified in this helm chart are enabled.   | true (default) or false  |
|           | limits.cpu  | Describes the maximum amount of CPU allowed | Default is 4000m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu)  |
|           | limits.memory | Describes the maximum amount of memory allowed | Default is 8Gi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|           | requests.cpu  | Describes the minimum amount of CPU required - if not specified will default to limit (if specified) or otherwise implementation-defined value. | Default is 200m. See Kubernetes - [meaning of CPU](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-cpu) |
|           | requests.memory | Describes the minimum amount of memory required If not specified, the memory amount will default to the limit (if specified) or the implementation-defined value | Default is 600Mi. See Kubernetes - [meaning of Memory](https://kubernetes.io/docs/concepts/configuration/manage-compute-resources-container/#meaning-of-memory) |
|      readinessProbe     | initialDelaySeconds | Number of seconds after the container has started before the readiness probe is initiated | Default is 30 |
|           | periodSeconds | How often (in seconds) to perform the readiness probe | Default is 30 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. In the case of the readiness probe, the Pod will be marked Unready. | Default is 10 |
|      livenessProbe     | initialDelaySeconds | Number of seconds after the container has started before the liveness probe is initiated | Default is 300 |
|           | periodSeconds | How often (in seconds) to perform the liveness probe | Default is 300 |
|           | failureThreshold | When a Pod starts and the probe fails, Kubernetes will try this number times before giving up. Giving up in the case of the liveness probe means restarting the Pod. | Default is 3 |

## Storage
See the Prerequisites section of this page for storage information.

## Limitations

The Apache Derby database type is not supported when running the UrbanCode Deploy server in a Kubernetes cluster. This is because the containerized version is running in UCD HA mode, which does not support Derby.
