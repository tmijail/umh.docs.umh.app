---
title: "Backup and Restore the United Manufacturing Hub"
content_type: task
description: |
        This page describes how to backup and restore the entire United Manufacturing Hub.
weight: 10
---

<!-- overview -->

This page describes how to back up the following:
- All Node-RED flows
- All Grafana dashboards
- The Helm values used for installing the {{< resource type="helm" name="release" >}} release
- All the contents of the United Manufacturing Hub database

It does **not** back up:
- Additional databases other than the United Manufacturing Hub default database
- TimescaleDB continuous aggregates: Follow the [official documentation](https://docs.timescale.com/self-hosted/latest/migration/schema-then-data/#recreate-continuous-aggregates) to learn how.
- TimescaleDB policies: Follow the [official documentation](https://docs.timescale.com/self-hosted/latest/migration/schema-then-data/#recreate-policies) to learn how.
- Everything else not included in the previous list

{{% notice warning %}}
This procedure only works on Windows.
{{% /notice %}}

## {{% heading "prerequisites" %}}

Download the [backup scripts](https://github.com/united-manufacturing-hub/backup/archive/refs/heads/main.zip) and extract the content in a folder of your choice.

For this task, you need to have [PostgreSQL](https://www.postgresql.org/download/)
installed on your machine.

You also need to have enough space on your machine to store the backup. To check
the size of the database, ssh into the system and follow the steps below:

{{< include "open-database-shell" >}}

Connect to the `umh_v2` or `factoryinsight` database:

```bash
\c <database-name>
```

 Run the following command to get the size of the database:

```sql
SELECT pg_size_pretty(pg_database_size('<database-name>'));
```

<!-- steps -->
## Backup

### Generate Grafana API Key

Create a Grafana API Token for an admin user by following these steps:
1. Open the Grafana UI in your browser and log in with an admin user.
2. Click on the **Settings** icon in the left sidebar and select **API Keys**.
3. Give the API key a name and change its role to **Admin**.
4. Optionally set an expiration date.
5. Click **Add**.
6. Copy the generated API key and save it for later.

### Stop workloads

To prevent data inconsistencies, you need to temporarily stop the MQTT and Kafka Brokers.

Access the instance's shell and execute the following commands:

<!-- tested in e2e #1343 -->
```bash
sudo $(which kubectl) scale statefulset {{< resource type="statefulset" name="kafkabroker" >}} --replicas=0 -n united-manufacturing-hub --kubeconfig /etc/rancher/k3s/k3s.yaml
sudo $(which kubectl) scale statefulset {{< resource type="statefulset" name="mqttbroker" >}} --replicas=0 -n united-manufacturing-hub --kubeconfig /etc/rancher/k3s/k3s.yaml
```

### Backup using the script

The backup script is located inside the folder you downloaded earlier.

1. Open a terminal and navigate inside the folder.

   ```powershell
   cd <FOLDER_PATH>
   ```

2. Run the script:

   ```powershell
   .\backup.ps1 -IP <IP_OF_THE_SERVER> -GrafanaToken <GRAFANA_API_KEY> -KubeconfigPath <PATH_TO_KUBECONFIG>
   ```

   You can find a list of all available parameters down below.

   {{% notice info %}}
   If `OutputPath` is not set, the backup will be stored in the current folder.
   {{% /notice %}}

This script might take a while to finish, depending on the size of your database
and your connection speed.

{{% notice warning %}}
If the connection is interrupted, there is currently no option to resume the process, therefore you will need to start again.
{{% /notice %}}

Here is a list of all available parameters:

{{< table caption="Available parameters" >}}
| Parameter             | Description                                                 | Required | Default value    |
| --------------------- | ----------------------------------------------------------- | -------- | ---------------- |
| `GrafanaToken`        | Grafana API key                                             | Yes      |                  |
| `IP`                  | IP of the cluster to backup                                 | Yes      |                  |
| `KubeconfigPath`      | Path to the kubeconfig file                                 | Yes      |                  |
| `DatabaseDatabase`    | Name of the databse to backup                               | No       | factoryinsight   |
| `DatabasePassword`    | Password of the database user                               | No       | changeme         |
| `DatabasePort`        | Port of the database                                        | No       | 5432             |
| `DatabaseUser`        | Database user                                               | No       | factoryinsight   |
| `DaysPerJob`          | Number of days worth of data to backup in each parallel job | No       | 31               |
| `EnableGpgEncryption` | Set to `true` if you want to encrypt the backup             | No       | `false`          |
| `EnableGpgSigning`    | Set to `true` if you want to sign the backup                | No       | `false`          |
| `GpgEncryptionKeyId`  | ID of the GPG key used for encryption                       | No       |                  |
| `GpgSigningKeyId`     | ID of the GPG key used for signing                          | No       |                  |
| `GrafanaPort`         | External port of the Grafana service                        | No       | 8080             |
| `OutputPath`          | Path to the folder where the backup will be stored          | No       | _Current folder_ |
| `ParallelJobs`        | Number of parallel job backups to run                       | No       | 4                |
| `SkipDiskSpaceCheck`  | Skip checking available disk space                          | No       | `false`          |
| `SkipGpgQuestions`    | Set to `true` if you want to sign or encrypt the backup     | No       | `false`          |
{{< /table >}}

## Restore

Each component of the United Manufacturing Hub can be restored separately, in
order to allow for more flexibility and to reduce the damage in case of a
failure.

### Cluster configuration
To restore the Kubernetes cluster, execute the `.\restore-helm.ps1` script with
the following parameters:

```powershell
.\restore-helm.ps1 -KubeconfigPath <PATH_TO_KUBECONFIG> -BackupPath <PATH_TO_BACKUP_FOLDER>
```

Verify that the cluster is up and running by opening {{< resource type="lens" name="name" >}}
and checking if the workloads are running.

### Grafana dashboards
To restore the Grafana dashboards, you first need to create a Grafana API Key
for an admin user in the new cluster by following these steps:

1. Open the Grafana UI in your browser and log in with an admin user.
2. Click on the **Settings** icon in the left sidebar and select **API Keys**.
3. Give the API key a name and change its role to **Admin**.
4. Optionally set an expiration date.
5. Click **Add**.
6. Copy the generated API key and save it for later.

Then, on your local machine, execute the `.\restore-grafana.ps1` script
with the following parameters:

   ````powershell
   .\restore-grafana.ps1 -FullUrl http://<IP_OF_THE_SERVER>:8080 -Token <GRAFANA_API_KEY> -BackupPath <PATH_TO_BACKUP_FOLDER>
   ````


### Restore Node-RED flows

To restore the Node-RED flows, execute the `.\restore-nodered.ps1` script with
the following parameters:

   ````powershell
   .\restore-nodered.ps1 -KubeconfigPath <PATH_TO_KUBECONFIG> -BackupPath <PATH_TO_BACKUP_FOLDER>
   ````

### Restore the database

To restore the database, execute the `.\restore-timescale.ps1` script with the
following parameters:

   ````powershell
   .\restore-timescale.ps1 -Ip <IP_OF_THE_SERVER> -BackupPath <PATH_TO_BACKUP_FOLDER> -PatroniSuperUserPassword <DATABASE_PASSWORD>
   ````

<!-- Optional section; add links to information related to this topic. -->
## {{% heading "whatsnext" %}}

- Take a look at the [UMH-Backup repository](https://github.com/united-manufacturing-hub/backup)
- Learn how to manually [backup and restore the database](/docs/production-guide/backup_recovery/backup-restore-database/)
- Read how to [import and export Node-RED flows](/docs/production-guide/backup_recovery/import-export-node-red-flows/) via the UI