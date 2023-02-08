# RHACM Active-Passive Hub for Disaster Recovery

## __Introduction__

---
How to configure an active-passive hub cluster configuration, where the initial hub cluster backs up data and one, or more passive hub clusters are on stand-by. When the active cluster becomes unavailable the passive hub become active restoring passive data and active data to control the managed clusters.

- OCP v4.6 or above
- RHACM v2.6
- OADP v1.0
- S3 Object Storage

The original files contained in *backup* and *restore* folders can be found [here](https://github.com/stolostron/cluster-backup-operator/tree/release-2.6/config/samples).

---

## __How to do__

1) Once the `Advanced Cluster Management for Kubernetes` operator is installed in OCP, it is necessary to create a `multiclusterhub` instance.

2) When the state of `multiclusterhub` instance is `phase:available`, it is possible to enable the backup configuation modifying the `YAML` definition:

```
- enabled: true
  name: cluster-backup
```

3) The ACM operator will reconcilie and start the instalation of the OADP operator in the namespace `open-cluster-management-backup`. The OADP operator is installed in v1.0, wich is linked to Velero v1.7, check the [Velero Version Relationship](https://github.com/openshift/oadp-operator).

### __NOTE:__ Do not cange the OADP operator channel to v1.1, the Velero pod is not able to connect to S3 storage. See this [Knowledge Base Article](https://access.redhat.com/solutions/6984040).

- The activation of the `cluster-backup` will also provide a policy, you can find the policy in the Governance section under the ACM console.

4) Create and configure [S3 Object Storage](https://docs.openshift.com/container-platform/4.11/backup_and_restore/application_backup_and_restore/installing/installing-oadp-aws.html) where you want to save all the buckups.

5) Create the secret in the same namespace that ACM has installed the OADP operator:

```
$ oc create secret generic cloud-credentials -n open-cluster-management-backup --from-file cloud=credentials-velero
```

6) Now, create a [Data Protection Application](https://github.com/jtovarro/active-pasive-hub-cluster/blob/main/oadp-operator/data-protection-application.yaml) instance in the OADP operator. Change the `region`, `bucket` and `prefix` labels accordingly with your environment.

7) Configure your first [buckup](https://github.com/jtovarro/active-pasive-hub-cluster/blob/main/backup/backup-schedule.yaml) in the active hub.

8) In the passive hub cluster we will need to install the same operators with the same configurations as in our active hub. As well, it is necessary to install the ACM operator with the `cluster-buckup` label enabled, and create the `Data Protection Applicantion` instance linked to the same `S3 Object Storage` where the buckups from the current active hub are pointing. 

 - There are two kind of data to restore: 
   - Passive data: secrets, ConfigMaps, apps, policies and all the managed cluster custom resources.
   - Activation data: results in managed clusters being actively managed by the cluster when it is restored on a new hub cluster.

### __NOTE__: make sure the active hub is power-off in case you want to restore the activation data in the passive hub, if not the active hub will try to add the managed cluster back again.

9) In the context of a failure in the active hub, we have the chance to recover our data in the passive hub, with the passive hub going to an activate status. We can also make restores only with passive data and apply the restore of the activation data as last step. 

- Apply this [yaml](https://github.com/jtovarro/active-pasive-hub-cluster/blob/main/restore/restore-passive-sync.yaml) if you want to set synchronized restores for passive data:  

```
$ oc apply -f https://raw.githubusercontent.com/jtovarro/active-pasive-hub-cluster/main/restore/restore-passive-sync.yaml
```

- Apply this [yaml](https://github.com/jtovarro/active-pasive-hub-cluster/blob/main/restore/restore.yaml) if you want to restore the activation data as well as the passive data:

```
$ oc apply -f https://raw.githubusercontent.com/jtovarro/active-pasive-hub-cluster/main/restore/restore.yaml
```

10) If the `old hub` becomes available again we can delete the `backupschedule` object and the `managedcluster` objects so this hub now is available as passive hub.

---
### __Summary__

This repo provides steps to set up a DR scenario with ACM.

---
### Related Links
[1] [Backup and Restore documentation](https://access.redhat.com/documentation/en-us/red_hat_advanced_cluster_management_for_kubernetes/2.6/html-single/backup_and_restore/index)

[2] [Backup and Restore blog](https://cloud.redhat.com/blog/backup-and-restore-hub-clusters-with-red-hat-advanced-cluster-management-for-kubernetes)

[3] [Cluster Backup Operator](https://github.com/stolostron/cluster-backup-operator/tree/release-2.6/config/samples)

[4] [Install OADP operator using OperatorHub](https://github.com/openshift/oadp-operator/blob/master/docs/install_olm.md#create-the-dataprotectionapplication-custom-resource)

