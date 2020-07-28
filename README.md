# MySQL Operator

## Release

#### v1.0.8

* Features
  * Operator upgrade to support MySQL version 8
  * Added Table of contents in Readme for ease of traversing through different topics
  
* Bug Fixes
  * NA

* Release Notes
  * Refer to the steps from [Update Cluster](#update) to enable Internal Load Balancer in your environment
  * **Operator Job Image: `mysql-operator:1.0.8` Helm Chart: `mysql-operator-1.1.4.tgz`**

* IMPORTANT - [Helm Chart Upgrade](#helm) IS ONLY APPLICABLE IF YOU ARE ALREADY USING `mysql-operator-1.1.3.tgz` - PLEASE IGNORE IF IT DOES NOT APPLY
  * Helm chart has been upgraded from `mysql-operator-1.1.3.tgz` to `mysql-operator-1.1.4.tgz` - refer to [Helm Chart Upgrade](#helm)
  for the steps to follow when a helm chart is upgraded.

## Demo

Please view the below recording for a demonstration of the steps documented in this README.

Watch the recordings: [Deploying the cluster](https://drive.google.com/open?id=1Oin8sRTgA04KTkf6-sWD2yXc4SlJux1s) [Backup and Restore](https://drive.google.com/open?id=1rveRpEXkrHW2eHHv7f1YV-DkEXeahdoJ) [Scheduled Backups](https://drive.google.com/open?id=1P96JqAwrOvVNN66H2Iday2gCOgbl9OEJ)

## Overview

This README provides instructions to deploy and maintain a MySQL cluster to a GKE Kubernetes namespace using the [Percona](https://github.com/percona/percona-xtradb-cluster-operator) MySQL (XtraDB) Cluster Operator.
 
A prerequesite before using this is that a Kubernetes Cluster Administrator have previously pushed the Percona XtraDB MySQL images to Artifactory, added the Percona XtraDB MySQL Custom Resource Definitions (CRDs) and run the `push` SaaS CD Pipeline to create the `mysql-operator` namespace to facilitate using the Percona operator. These actions will have been performed by the SaaS Ops team for any of the managed GKE environments.

Please read the [Percona documentation](https://www.percona.com/doc/kubernetes-operator-for-pxc/index.html) to gain an understanding of the features of the Operator.

Table of contents
=================

* [Prerequsites](#pre)
* [Available Commands for MySQL Operator](#available-commands)
  * [deploy-cluster](#deploy)
  * [backup-cluster](#backup)
  * [restore-cluster](#restore)
  * [delete-cluster](#delete)
  * [update-cluster](#update)
  * [list-backup](#list)
  * [delete-backup](#delete-backup)
* [Connecting to MySQL Database](#how-to-connect)
* [Scaling the Cluster](#how-to-scale)
* [Internal Load Balancer](#ilb)

## <a name="pre"></a>Prerequsites

In the [deploy-info.yml](./deploy-info.yml), the following base project, operator and team needs to be defined:

```
base_project_name: "mysql-operator"
base_image_pull_service_account: "all"
operator: "true"
operator_service_account: "mysql-operator"
team:
  name: "mysql-operator"
  token: "77fc334c-7ff4-8f6d-f317-680f676e4675"
```

The token value for the environments created and managed by the SaaS Ops team. 

## Install and manage the Operator via the SaaS Ops CD pipeline

In your own gitops repo, create a new branch and copy the [/operator](./operator) folder and its yaml contents, [Jenkinsfile](./Jenkinsfile), [deploy-info.yml](./deploy-info.yml), [helm-command.yml](./helm-command.yml) and [values.yaml](./values.yaml) to it from this branch. Ensure your copied `Jenksinfile` is triggering the expected SaaS Ops CD Pipeline code (dev for dockcpdev repos and master for dockcp repos) and edit your copied `deploy-info.yml` to ensure it is deploying to the expected namespace (defined by the `project_name` setting in [deploy-info.yml](./deploy-info.yml) and GKE cluster (defined by the `kubernetes.env.name` setting in [deploy-info.yml](./deploy-info.yml)).

Performing a commit to the branch triggers the SaaS Ops CD Pipeline and will run one of the commands as documented below.

### <a name="helm"></a>Helm Chart Upgrade
* THIS SECTION IS ONLY APPLICABLE IF YOU HAVE PREVIOUSLY DEPLOYED A MYSQL CLUSTER USING OLDER VERSION OF HELM CHART- PLEASE IGNORE IF YOU ARE USING THE OPERATOR FOR THE FIRST TIME
* When a new helm chart is relesed, please use the following options in [helm-command.yaml](./helm-command.yaml) in addition to other instructions, edit the values for `command`  and `commandOptions` as shown below.
```
......
command: upgrade
commandOptions: --install
releaseName: mysql-operator
```

## <a name="available-commands"></a>Commands available for the MySQL Operator

### Presteps before you run the Saas Ops CD Pipeline for the first time

* In [values.yaml](./values.yaml), edit the `image.repository` value to use your GCP project ID and env name (as defined in your [deploy-info.yml](./deploy-info.yml).
* Likewise, edit the [/operator/operator.yaml](./operator/operator.yaml) and [/operator/cr.yaml](./operator/cr.yaml) to ensure any image references use your GCP project ID and the environment name as defined in your [deploy-info.yml](./deploy-info.yml). Essentially, ensure that anywhere you see reference to `gcr.io` that you update to have the path to the images in your projects container registry - i.e. `gcr.io/<gcp-project-id>/<env-name>/mysql-operator`.
* If you want to backup and restore from a Google Storage bucket, you need to provide a HMAC key to [/operator/backup-s3.yaml](./operator/backup-s3.yaml). See [Google HMAC documentation](https://cloud.google.com/storage/docs/authentication/hmackeys) that can authenticate to the Google Storage bucket you will need to have already created.
* You need to add appropriate users and base64 encoded passwords to [/operator/secrets.yaml](./operator/secrets.yaml).
* You need to provide certificates that will be used to secure communication within the cluster. If you don't already have certificates to use, see the [Percona documentation](https://www.percona.com/doc/kubernetes-operator-for-pxc/TLS.html#generate-certificates-manually) on how to create a Certifcate Authority and generated required certificates. Add these to the [/operator/ssl-secrets.yaml](./operator/ssl-secrets.yaml) and [/operator/ssl-internal-secrets.yaml](./operator/ssl-internal-secrets.yaml) respectively.
* The cluster name value must be identical in [values.yaml](./values.yaml) for `clusterName` and [/operator/cr.yaml](./operator/cr.yaml) for `name:` under `metadata:`

### <a name="deploy"></a>deploy-cluster

You need to define the relevant settings for your cluster to the [/operator/cr.yaml](./operator/cr.yaml). If you have not deployed the PMM Server, ensure that the pmm section in [/operator/cr.yaml](./operator/cr.yaml) has `enabled: false`. The PMM Server can be deployed separate to this operator as a [helm chart](https://www.percona.com/doc/kubernetes-operator-for-pxc/monitoring.html). Example `schedule` settings in the [/operator/cr.yaml](./operator/cr.yaml) file demonstrate how to backup to Google Persistent Disk storage every day and to a Google Storage Bucket every Saturday.

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

If you have deployed without the PMM Server and with the [/operator/cr.yaml](./operator/cr.yaml) file settings showing a cluster name of `cluster1`, pxc and proxysql size of `3`, you should see a deployment similar to the following after a number of minutes. Inspect the container logs to ensure you don't see errors.

If you have used the example settings in [/operator/cr.yaml](./operator/cr.yaml) you should see the following objects:

* 1 stateful set `cluster1-proxysql` running 3 pod instances
* 1 stateful set `cluster1-pxc` running 3 pod instances
* 1 completed job `mysql-operator-1-XXXXXXXXX` that performed this action. For subsequent runs, you will see a job per run
* 1 deployment for `percona-xtradb-cluster-operator` that contains the operator functionality

![screenshot1.png](./images/screenshot1.png)

#### <a name="how-to-connect"></a>Testing the Connection to the MySQL Cluster

Test the connection by running the percona-client and connect its console output to your terminal (running it may require some time to deploy the corresponding Pod):

```
kubectl --namespace mysql-operator-demo run -i --rm --tty percona-client --image=percona:5.7 --restart=Never -- bash -il
```

Change the namespace value to match your namespace as per the `project_name` setting in your [deploy-info.yml](./deploy-info.yml). It may take a minute or so for a command prompt in the running container to be available.

Now run the mysql tool in the percona-client command shell using the `root` password obtained from base64 decoding the value from [/operator/secrets.yaml](./operator/secrets.yaml). If you are just using the example user password values provided in the secrets.yaml for demo purposes, the example password is `root_password`:

```
mysql -h cluster1-proxysql -uroot -proot_password
```

![screenshot2.png](./images/screenshot2.png)

#### <a name="ilb"></a>Internal Load Balancer

if you need connect the mysql cluster outside the kubernetes network, we need create a loadbalancer for the service. Change the values.yaml file to have `loadbalancer: true`, and re-deploy the cluster. And use the `EXTERNAL-IP` in service as the mysql database IP address to connect.

![screenshot3.png](./images/screenshot3.png)

#### <a name="how-to-scale"></a>Scaling the MySQL Cluster

To scale a cluster, do not use `kubectl scale ...` from the command line or via the GKE Kubernetes Console. Instead, set the relevant `size` value in the pxc or proxysql section(s), as appropriate, and retrigger the SaaS Ops CD pipeline using the `deploy-cluster` command in the values.yaml. If there are other settings you want to update, you would edit the [/operator/cr.yaml](./operator/cr.yaml) accordingly and run a `deploy-cluster` command via the SaasOps CD pipeline. This will perform a Kubernetes rolling update of your MySQL deployment.

### <a name="backup"></a>backup-cluster

This performs a backup of a MySQL cluster deployment to either GCP Persistent Disk or a pre-existing Google Storage Bucket (using HMAC credentials mimicking AWS S3).

Presteps to take before you run the SaaS Ops CD Pipeline to perform a backup:

* In [values.yaml](./values.yaml), set the `command` field to `backup-cluster`.
* If backing up to a Google Storage Bucket, you must have the HMAC credentials (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`) in the [/operator/backup-s3.yaml](./operator/backup-s3.yaml) and define an appropriate storages sub-section within the backup section in [/operator/cr.yaml](./operator/cr.yaml). For example:   

```
    backup:
      image: gcr.io/<gcp-projectd-id>/<env-name>/mysql-operator/percona-xtradb-cluster-operator-backup:1.3.0
      serviceAccountName: percona-xtradb-cluster-operator
      storages:
        s3-us-west:
          type: s3
          podSecurityContext:
            runAsUser: 1001
            runAsGroup: 1001
            fsGroup: 1001
          s3:
            bucket: mysql-operator-demo
            credentialsSecret: mysql-cluster-backup-s3
            region: us-west2
            endpointUrl: https://storage.googleapis.com
```

Replace gcp-project-id with the project ID of your Google Project and the name of your environment as defined in your [deploy-info.yml](./deploy-info.yml).

* If backing up to a Google Persistent disk, you must have defined an appropriate storages sub-section within the backup section in [/operator/cr.yaml](./opertor/cr.yaml). For example:

```
    backup:
      image: gcr.io/<gcp-projectd-id>/<env-name>/mysql-operator/percona-xtradb-cluster-operator-backup:1.3.0
      serviceAccountName: percona-xtradb-cluster-operator
      storages:
        fs-pvc:
          type: filesystem
          podSecurityContext:
            runAsUser: 1001
            runAsGroup: 1001
            fsGroup: 1001
          volume:
            persistentVolumeClaim:
              storageClassName: us-east4-a
              accessModes: [ "ReadWriteOnce" ]
              resources:
                requests:
                  storage: 6Gi
```

where the volume definition follows standard Kubernetes terminology.

Replace gcp-project-id with the project ID of your Google Project and the name of your environment as defined in your [deploy-info.yml](./deploy-info.yml).

Ensure the [/operator/backup.yaml](.operator/backup.yaml) refers to the cluster name you want to be backed up and the storage location to backup to.

Trigger the CD pipeline by making a commit to the github branch. 

You will see a backup job (pod) be run and you can review it's logs to see what it has done.

If you want to automate your backups via a cron job, edit the `schedule` section in [/operator/cr.yaml](./operator/cr.yaml). For example, the following will automate a backup of the named cluster in cr.yaml each Saturday to the Google Storage Bucket defined by the s3-us-west settings (see above) and a daily backup to GCP Persistent Disk defined by the fs-pvc settings (see above). The `keep` value determines how many backups will be retained with the oldest being deleted on any new backup if the number of current backups equals the `keep` value:

```
      schedule:
        - name: "sat-night-backup"
          schedule: "0 0 * * 6"
          keep: 3
          storageName: s3-us-west
        - name: "daily-backup"
          schedule: "0 0 * * *"
          keep: 5
          storageName: fs-pvc
```

### <a name="restore"></a>restore-cluster

This performs a restore of a backup to a MySQL cluster deployment from either GCP Persistent Disk or a Google Storage Bucket (using HMAC credentials mimicking AWS S3).

Presteps before you run the SaaS Ops CD Pipeline:

* In [values.yaml](./values.yaml), set the `command` field to `restore-cluster`.
* See the presteps defined above for `backup-cluster` regarding the settings in [/operator/cr.yaml](./operator/cr.yaml).

Ensure the [/operator/restore.yaml](.operator/restore.yaml) refers to the cluster name you want to restore to and the backup name to restore.

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

You will see a restore job be run and you can review it's logs to see what it has done.

### <a name="delete"></a>delete-cluster

This performs a delete of a MySQL cluster deployment. You can decide to delete the data as well as the deployment or just the deployment.

Presteps before you run the CD Pipeline:

* In [values.yaml](./values.yaml), set the `command` field to `delete-cluster`<br/>
* Define the name of the cluster to be deleted in the [/operator/cr.yaml](./operator/cr.yaml). If you want to keep the data and only delete the deployment, remove or comment out the `delete-proxysql-pvc` and `delete-pxc-pvc` finalizer lines:

```
  metadata:
    name: cluster1
    finalizers:
      - delete-pxc-pods-in-order
      - delete-proxysql-pvc
      - delete-pxc-pvc
```

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

If you run `kubetl get pxc` you should not see the cluster name listed.

### <a name="list"></a>list-backups

This returns a list of the previously performed backups. List can be viewed in the container logs for the mysql-operator-cmd-xxx job

if you have kubectl access the following command can be run to retrieve the list of backups

```
kubectl get pxc-backup --namespace <YOUR_NAMESPACE>
```

Presteps before you run the SaaS Ops CD Pipeline:

* In [values.yaml](./values.yaml), set the `command` field to `list-backups`.

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

### <a name="delete-backup"></a>delete-backup

This deletes the named backup

Presteps before you run the SaaS Ops CD Pipeline:

* In [values.yaml](./values.yaml), set the `command` field to `delete-backup` and the `backupName` field to the name of the backup to delete.

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

### <a name="update"></a>update-cluster

This updates an already deployed cluster to a later Percona release

Presteps before you run the SaaS Ops CD Pipeline:

* In [values.yaml](./values.yaml), set the `command` field to `update-cluster`, `perconaOperatorVersion`to the new Percona version (e.g. 1.4.0), `perconaOperatorApiVersion` to the new Percona API version (e.g. v1-4-0) and the `clusterName` field to the name of the cluster to update.
* The images used by the new Percona version must have previously been downloaded from the [Percona Operator location on Docker Hub](https://hub.docker.com/r/percona/percona-xtradb-cluster-operator) by a cluster admin, retagged and pushed to Artifactory. Then the cluster admin must have pushed those images to the local Google Container Registry in the mysql-operator namespace.

Trigger the SaaS Ops CD pipeline by making a commit to the github branch. 

You should see the following:

* The operator pod is terminated and restarted, using the latest operator image
* The pxc, proxysql, backup and pmm pods are terminated and restarted using their latest images. The pxc and proxysql pods are terminated in sequence and thus there should be no loss of service as at least one pxc and proxysql pod should be running during the update.

## <a name="aiops"></a>AIOPS APM section

This section provides instructions on how to deploy an AIOPS APM agent along with the operator to collect the application metrics.

Pre-steps before you run the CD Pipeline:

* In **[values.yaml](./values.yaml)**, set the command field to `deployAiopsAgent: true` (This flag is applied everytime a `deploy-cluster` or `update-cluster` command is executed)
* Define with necessary configuration in **[aiops.yaml](./operators/aiops.yaml)** file and pass the values as in below code capture.
* To delete the monitoring agent when you delete the cluster, ensure the flag is `deployAiopsAgent:` is set to `true` while executing the `delete-cluster` command.

### AIOPS agent deployment
The **[aiops.yaml](./operators/aiops.yaml)** has description as below. When the flag `deployAiopsAgent:` is set to `true`, configurations for AIOPS agent as described in **[aiops.yaml](./operators/aiops.yaml)** is executed:

```YAML
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: <Pod_Name>  (1)
    labels:
      tier: monitoring
      app: caagent_apmia
      version: v1
  spec:
    replicas: 1
    selector:
      matchLabels:
        tier: monitoring
        app: caagent_apmia
        version: v1
    template:
      metadata:
        labels:
          tier: monitoring
          app: caagent_apmia
          version: v1
      spec:
        containers:
          - resources:
            env:
              - name: APMENV_AGENTMANAGER_URL_1
                value: <wss://############:XXX>          (2)
              - name: APMENV_AGENTMANAGER_CREDENTIAL
                value: <############>                    (3)
              - name: APMENV_INTROSCOPE_AGENT_DBMONITOR_MYSQL_PROFILES
                value: Cluster2                       
              - name: APMENV_INTROSCOPE_AGENT_DBMONITOR_MYSQL_PROFILES_CLUSTER1_INSTANCENAME
                value: <DB Instancename, EX: sys>        (4)
              - name: APMENV_INTROSCOPE_AGENT_DBMONITOR_MYSQL_PROFILES_CLUSTER1_HOSTNAME
                value: <SVC Name>-proxysql               (5)
              - name: APMENV_INTROSCOPE_AGENT_DBMONITOR_MYSQL_PROFILES_CLUSTER1_USERNAME
                value: root
              - name: APMENV_INTROSCOPE_AGENT_DBMONITOR_MYSQL_PROFILES_CLUSTER1_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: mysql-cluster-secrets
                    key: root
              - name: APMENV_INTROSCOPE_AGENT_EXTENSIONS_BUNDLES_BOOT_LOAD
                value: MySQL
              - name: APMENV_INTROSCOPE_AGENT_AGENTNAME
                value: MySQL-Monitor
              - name: APMENV_INTROSCOPE_AGENT_HOSTNAME
                value: <your-namespace>                  (6)
              - name: APMENV_INTROSCOPE_AGENT_CUSTOMPROCESSNAME
                value: <NAMESPACE>-MySQL-Monitor         (7) 
            name: apmia
            image: caapm/apmia:latest
            imagePullPolicy: Always
            resources:
              limits:
                cpu: 0.7
                memory: 700M
```
----------------------------
1. Name your Kubernetes agent pod
2. the AgentManager URL is the endpoint where the agents report the polling information. The URL needs to be collected from AIOPS team based on your Tenancy/Env.
3. The Credentials shall be shared by AIOPS team based on your Tenancy/Env.
4. DB Instance names. Separate multiple values using commas.
5.  Name of the host on which MySQL is installed that the Agent monitors.
6. The hostname needs to be equivalent to your namespace. This section is to group all your deployments.
7. This field is to scope or collect the agents under a custom_scope. we can use this field to group the agents of a particular application.
----------------------------


[MySQL Database Monitoring official reference](https://techdocs.broadcom.com/content/broadcom/techdocs/us/en/ca-enterprise-software/it-operations-management/dx-apm-saas/SaaS/implementing-agents/infrastructure-agent/mysql-database-monitoring.html)
