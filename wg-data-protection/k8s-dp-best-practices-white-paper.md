# Kubernetes Data Protection: Best Practices
Kubernetes applications are commonly including state today.  With state comes the need to protect data.  When we talk about data protection in this context, we are talking about keeping data safe against problems such as hardware failure, corruption, accidental deletion, ransomware and other issues that can prevent your application from functioning correctly.

Data protection can take multiple approaches, the most common are backup/restore and replication.  For either approach, the application must be ready on some basic level to be protected.  In this white paper we outline common scenarios for backup/restore and replication, how this affects applications and how the application can be designed to get best results in the event that disaster strikes.

<!-- toc -->

- [Backup/Restore](#backuprestore)
  * [Overview](#overview)
  * [Backup](#backup)
    + [Consistency](#consistency)
      - [Crash consistent](#crash-consistent)
      - [Fully consistent](#fully-consistent)
        * [Application quiescing](#application-quiescing)
        * [Application level backup](#application-level-backup)
      - [Inconsistent](#inconsistent)
    + [Backup types](#backup-types)
      - [Volume backup](#volume-backup)
      - [Block backup](#block-backup)
      - [File backup](#file-backup)
    + [Backup extensions](#backup-extensions)
        * [Changed Block Tracking (CBT)](#changed-block-tracking-cbt)
        * [Volume Group Snapshotting](#volume-group-snapshotting)
    + [Application specific backup](#application-specific-backup)
      - [Triggering mechanisms](#triggering-mechanisms)
      - [Application internal snapshots](#application-internal-snapshots)
      - [Export to disk](#export-to-disk)
      - [Export to object storage](#export-to-object-storage)
    + [Kubernetes resource backup](#kubernetes-resource-backup)
      - [External storage backup](#external-storage-backup)
    + [Quiescing](#quiescing)
    + [Completeness](#completeness)
      - [Persistent Volumes (PVs)](#persistent-volumes-pvs)
      - [Kubernetes resources](#kubernetes-resources)
        * [Custom Resource Definitions (CRDs) and versioning](#custom-resource-definitions-crds-and-versioning)
        * [Network ingresses and addresses](#network-ingresses-and-addresses)
      - [External resources](#external-resources)
      - [Backup windows and Application Consistency](#backup-windows-and-application-consistency)
      - [SLA (can we guarantee SLAs during backups?)](#sla-can-we-guarantee-slas-during-backups)
    + [Restore](#restore)
      - [Recovering from a crash](#recovering-from-a-crash)
      - [Restoring Kubernetes resources or Persistent Volumes independently](#restoring-kubernetes-resources-or-persistent-volumes-independently)
          + [Single file restore](#single-file-restore)
          + [Single resource restore](#single-resource-restore)
      - [Recovering from inconsistency](#recovering-from-inconsistency)
      - [Test restores](#test-restores)
      - [Dynamic vs Static provisioning for Persistent Volumes](#dynamic-vs-static-provisioning-for-persistent-volumes)
        * [Volume topology and affinity](#volume-topology-and-affinity)
        * [Kubernetes volume populator](#kubernetes-volume-populator)
- [Disaster recovery for clusters](#disaster-recovery-for-clusters)
- [Replication](#replication)
    + [Overview](#overview-1)
    + [Hot/cold failover](#hotcold-failover)
    + [Replication of all resources and data](#replication-of-all-resources-and-data)
    + [Volume only replication](#volume-only-replication)
    + [Handling network partitions](#handling-network-partitions)
    + [What gets replicated](#what-gets-replicated)
    + [Fully consistent versus eventually consistent](#fully-consistent-versus-eventually-consistent)
- [Cyber Recovery](#cyber-recovery)
  * [Immutability & Air-Gapping in Cloud-Native Storage](#immutability--air-gapping-in-cloud-native-storage)
  * [Isolated Recovery Environments ("Clean Rooms")](#isolated-recovery-environments-clean-rooms)
  * [Application & Workload Hardening for Cyber Recovery](#application--workload-hardening-for-cyber-recovery)
  * [Verification and Cyber Recovery Drills](#verification-and-cyber-recovery-drills)
- [How to define the boundaries of an application for data protection](#how-to-define-the-boundaries-of-an-application-for-data-protection)
- [Scenarios to expect](#scenarios-to-expect)
  * [Restore to same cluster](#restore-to-same-cluster)
    + [Handling PVC Name Collisions When Restoring to the Same Cluster](#handling-pvc-name-collisions-when-restoring-to-the-same-cluster)
    + [Custom Resource Definitions](#custom-resource-definitions)
      - [Best Practices:](#best-practices)
  * [Restore to a different cluster](#restore-to-a-different-cluster)
  * [Restore for testing](#restore-for-testing)
  * [Restore for migration](#restore-for-migration)
  * [Restore for disaster recovery](#restore-for-disaster-recovery)
- [Best practices for applications](#best-practices-for-applications)
  * [Identify what needs to be protected](#identify-what-needs-to-be-protected)
  * [Identify the Value, Cost, and Impact of Data Loss or downtime](#identify-the-value-cost-and-impact-of-data-loss-or-downtime)
  * [Appropriate Data Protection](#appropriate-data-protection)
  * [Understand what modifications your application might need for your data protection strategy](#understand-what-modifications-your-application-might-need-for-your-data-protection-strategy)
    + [Restore from crash consistent backup](#restore-from-crash-consistent-backup)
  * [Test your backup/replication plans](#test-your-backupreplication-plans)
  * [Continue to evolve your data protection strategy](#continue-to-evolve-your-data-protection-strategy)
  * [Best practices for Operators](#best-practices-for-operators)
  * [Best Practices for Backup/Restore applications](#best-practices-for-backuprestore-applications)
  * [Handling failed backups \- cleanup of resources](#handling-failed-backups---cleanup-of-resources)
- [Support you can get from backup/restore applications](#support-you-can-get-from-backuprestore-applications)
  * [Hooks](#hooks)
  * [Best practices when moving existing applications into Kubernetes](#best-practices-when-moving-existing-applications-into-kubernetes)
    + [Virtual machines](#virtual-machines)
  * [Key considerations when migrating from a legacy environment to Kubernetes](#key-considerations-when-migrating-from-a-legacy-environment-to-kubernetes)
    + [Persistent Volume Snapshots as replacement for VM DIsk Snapshots](#persistent-volume-snapshots-as-replacement-for-vm-disk-snapshots)

<!-- tocstop -->

<!-- /toc -->

# Backup/Restore

## Overview

Backup/Restore has a long history in information science.  At its most basic, a copy of an application's state is made and then, at some point in the future, the state is restored and the application should continue as if it were returned to that point in the past.

In older systems, this was done by backing up the application's persistent data, saved to disk, and its configuration information.  This might have been done by backing up the disks with a utility such as *tar*, or taking a snapshot of disks or entire virtual machines.  More sophisticated applications, such as databases, may have their own backup/restore mechanisms built in to be faster or be more compact or to handle the backup without requiring any downtime.

Kubernetes applications are similar, with data being stored in persistent volumes (PVs), the API server and also in external services such as databases-as-a-service or object storage.

Even in a system that is fully replicated and available, backup/restore may be important as another line of defense against ransomware and other forms of data corruption.

## Backup

Backup of a system makes a copy of the data needed to restore the system to a working state after some kind of data loss.  When considering how to backup an application we need to consider how quickly the backup will happen and how we can ensure that the system will be recoverable from the backup.  It's not uncommon for backups to be taken and then discovered that they are not actually recoverable.  It's important when planning the backup of a system to understand all of the pieces that need to be backed up and what level of consistency they require.

### Consistency

Applications will require some level of consistency in their data in order to operate.  For example, if a stored object has an owner, the record for the owner needs to exist.  At lower levels, applications expect that if they write record A then record B, if they see record B, record A must exist.

There are multiple types of consistency and it is important to understand what kind of consistency is necessary for your recovery plan and how you will achieve that type of consistency.

#### Crash consistent

We require applications to be able to recover from a "crash" of the system \- this may occur because of a software or hardware problem or power failure.  We take advantage of this in backup/restore by making an image of the application that is "crash consistent" and then on restore, using the application's recovery mechanism to finish restoring the system to a working state.  An example of a recovery mechanism is the *fsck* utility used to repair file systems.  Databases have internal mechanisms to recover their data structures.  It's important to understand what the effects of a crash consistent recovery might be \- *fsck*, for example, is usually able to recover the file system to the point where it can be used, but does not make guarantees that all files will be recovered.  Many file systems provide options such as logging that improve the possibility of recovering all data after a crash/recovery from a crash consistent backup.

Crash consistency expects that all records that have been written to persistent storage are present and that write order consistency has been maintained.

Crash consistent backups are usually achieved by taking snapshots of the application's disks and then backing up the snapshot.  Taking a snapshot typically pauses I/O for a short period of time, often only a few milliseconds, while the snapshot is being created, then I/O is restarted and the application can continue to operate.  Backup of the snapshotted data will take longer but if the storage system is configured correctly, the backup will have no effect on the application.

Maintaining crash consistency is more difficult in a large distributed application than for a single machine.  Taking snapshots of multiple disks independently may result in individually crash consistent disks but write order consistency across the disks may be lost.

#### Fully consistent

A fully consistent backup is one where all of the application's data has been backed up and no repair operations are needed.  There are several ways to achieve this:

##### Application quiescing

To achieve a fully consistent backup the application must quiesced or shutdown, preventing it from modifying any data and ensuring all data structures are written to persistent storage. The time required to quiesce an application can vary but the worst case is that the application requires a complete shutdown.  In this case, the application will remain unavailable until the backup is finished and the application is restarted. A storage snapshot may be taken while the application is quiesced \- in this case, the snapshot will be a fully consistent snapshot of the data and the application can be resumed while the data is being backed up from the snapshot.

##### Application level backup

Applications may have their own backup mechanisms which export the application's data in a consistent manner.  Many databases are able to perform an internal snapshot and export a consistent view of the data.  Recovery will be application specific as well.

#### Inconsistent

In general, inconsistent backups should be avoided because the likelihood of being able to recover the system from an inconsistent backup is low.  The exceptions to this are for data that is important to be preserved but will not actually affect application operation if it is not preserved consistently \- for example, log files.

### Backup types

#### Volume backup

Data storage in the Kubernetes cluster is primarily stored in Persistent Volumes.  Backing up those volumes is the basic way of protecting data for applications in Kubernetes unless an application specific method is used.  There are two methods used for backing up volumes \- block backup and file backup.

#### Block backup

Persistent Volumes are block devices used either as file systems or, less commonly, as raw block devices with application data written directly to the block device (e.g. a database that uses block devices).  Backing up and restoring at the block level is agnostic of the format of the data in the volume.  It also performs well because the backup does not involve interpreting the data and can be done without excessive seeking (less important with solid state disks).  However, the entire volume must be backed up which makes subsequent backups less efficient since data that has already been backed up is being backed up again.  If the storage system implements Changed Block Tracking (CBT), the efficiency on subsequent backups can be much better.

#### File backup

Backing up and restoring data at the file level is another way to protect the contents of a Persistent Volume Claim (PVC).  In some instances, for example PVCs that are network volumes, this may be the only way to access the data in the PVC.  The backup application copies the data on a file-by-file basis.  It can create incremental backups by copying files that have been changed since the most recent backup.  File level backups can be made with or without snapshots. 

### Backup extensions

##### Changed Block Tracking (CBT)

Many storage systems have the ability to track which blocks have changed between two snapshots.  CBT enables a backup application to efficiently make incremental backups since only the changed data needs to be backed up.  On restore, the backup application will combine a full backup with subsequent incremental backups to restore the volume as a whole.

The Data Protection Working Group has standardized how to expose this to backup applications for Kubernetes with KEP-3314 ([https\://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/3314-csi-changed-block-tracking/README.md](https://github.com/kubernetes/enhancements/blob/master/keps/sig-storage/3314-csi-changed-block-tracking/README.md))  and implementations are underway.

##### Volume Group Snapshotting

This is a storage system feature that creates a point-in-time of a logical volume group (LVG) . This technique allows you to snapshot the state of multiple logical volumes within the group while guaranteeing data uniformity and write order consistency.  Volume Group Snapshots for Kubernetes have been standardized in KEP-3476 (https\://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/3476-volume-group-snapshot)

### Application specific backup

This process refers to a tailored strategy that is unique to data structures and requirements of a particular software application. This approach takes into consideration that the generic data backups may be insufficient for specific recovery needs.  An application specific backup may be faster, provide better consistency, allow for recovery into different environments, etc.

Approaches to application specific backup usually involve application specific utilities or commands that cause data to be protected.  This might involve an internal snapshot, or export to a file or an object store.

In addition to triggering the application specific backup/restore flows, it is also important to synchronize the backup with the overall Kubernetes backup and to manage and protect the backup artifacts.  If the application's data is being stored in Kubernetes PVs, those PVs should be excluded from the overall backup since the application specific backup would already be protecting the data..

#### Triggering mechanisms

Application specific backups are usually triggered either through scripts or by creating Custom Resources that cause an Operator for the application to perform the operation.  Either of these can be done by a backup hook (see the Hooks section) 

#### Application internal snapshots

Some applications have the ability to create snapshots internally.  It's important to understand whether these snapshots are *durable* or not.  Many cloud services, for example, will internally export a snapshot to object storage with management via the service API.  These snapshots could be considered durable because the data in the snapshot would survive the loss of the service instance.  On the other hand, a snapshot that is kept solely within the application and will be lost if the application instance is destroyed should be considered non-durable and needs some other form of backup.

#### Export to disk

Some applications will export a backup to a file/directory.  A common approach with Kubernetes backup is to attach a PV to the application's pod as the backup directory, trigger the application to export into that PV, and then use the backup application to backup the data in the PV.

#### Export to object storage

Some applications will export a backup to an object store bucket.  Ideally the object identifiers will be embedded into the backup and be used automatically in the restore process.

### Kubernetes resource backup

Kubernetes applications are composed of more than just containers and pods. These applications rely on a complex set of resources such as Deployments, Stateful Sets, Config Maps, Secrets, Custom Resources Definitions (CRDs) and Persistent Volume (PVs). Backing up these resources ensures that the application’s state configuration, and runtime configuration can be fully restored after migration, failure or disaster.  
Backing up Kubernetes resources introduces challenges that do not exist in traditional VM based data protection because Kubernetes workloads consist of multiple independent API  objects rather than monolithic machine images.These challenges should be understood and addressed to ensure that back up and restore operations preserve application functionality across failure scenarios.

Kubernetes resources depend on other resources through direct and indirect relationships. For example, a Stateful Set references a Persistence Volume Claim. A Deployment may reference a Config Map or Secret. A Pod may reference a Service Account that in turn may reference a Secret containing credentials.

Understanding Kubernetes Resources

* Core Resources: Pods, Deployments, Services,Replica Sets, Stateful Sets  
* Configuration Resources : Config Maps, Secrets \-hold environmental variables, configuration files, or sensitive credentials.  
* Storage Resources: Persistent Volume Claims (PVCs) and associated Persistent Volumes (PVs) contain the actual application data.  
* Custom resources: CRDs and Operator-managed resources extend kubernetes with domain-specific objects.

Backup Strategies 

1. Namespace-level Backup   
* Back up all resources in a namespace to capture the complete application context.

2. Label-based Backup   
* Use Labels to selectively back up only resources belonging to a specific application.

3.  Resource Type Segmentation   
* Some team may choose to back up stateful resources (PVCs, CRDs) separate from stateless  resources (Deployment, Services)


4. Challenges and Considerations

Kubernetes resource backup is a critical part of disaster recovery that involves capturing the state of your Kubernetes resources so that you may confidently restore them in event of data loss.

#### External storage backup

Kubernetes workloads often operate on data stored outside the cluster boundary. External data includes databases, object storage services, Saas platforms and external file systems. While Kubernetes-native backup tools protect cluster resources using PVC’s, Deployments, and Custom Resource Definitions (CRDs), they do not automatically safeguard external hosted data sources. 

Data protection strategies must incorporate explicit provisions for these external dependencies.

1. Identify and Classify External Data Dependencies  
     
   Organizations should inventory all  external data sources accessed by Kubernetes workloads before defining backup mechanisms. These commonly include:   
     
* Managed Databases   
* Object Storage   
* Third party Saas platforms  
* Shared filesystems 

             
            Each dependency should be classified based on:

* Recovery Time Objective (RTO)  
* Recovery Point Objective (RPO)  
* Regulatory or compliance requirements  
    
  (see the section "Identify the Value, Cost, and Impact of Data Loss or downtime" for more information)

2. Align Backup Strategy with Service Ownership  
   External systems are often managed by separate teams or cloud providers. Responsibility for backup configuration must be clearly defined within the organization’s framework.   
     
     
   Key Considerations:  
     
* Is backup responsibility owned  by the operation team or delegated to the cloud provider?  
* Are backups enabled by default or should they be explicitly configured?


  

Assumptions that the cloud provider handles backups often results in gaps. Managed services often provide backup capabilities but not enforced protection  policies.

3. Native Service Backups and Snapshots  
     
   Utilize the native backup capabilities of the external system whenever possible:  
     
* Automate database snapshots  
* Point-in-time Recovery (PITR)  
* Object Versioning and life cycle policies   
* Cross-region replication

As a reminder, backup verification procedures should be established. Backup alone does not guarantee recoverability. 

4. Cross-Region and Cross Account Protection

    Effective protection against regional outages, account compromise or ransomware scenarios requires that backups be stored outside the administrative boundary of the primary data. 

   

   

   Recommended Best Practices:

   

* Cross-account backup storage   
* Cross-region replication  
* Write once storage configurations  
* Encryption with managed key rotations policies

5. Restore Testing and Application Consistency  
     
   Restoring external data should be tested in coordinations with Kubernetes cluster restoration procedures.    
     
* Application-level consistency  
* Schema compatibility during restore   
* Secret and credential alignment  
* Endpoint reconfiguration after failover


  


6. Security and Access Control 

   Backup systems represent high value targets. Access to backup storage systems must follow least privilege principles. Controls should include:

   

* Role-based access control  
* Multi-factor authentication   
* Audit logging   
* Encryption at rest and in transit

### Quiescing

Quiescing is the process of temporarily bringing an application or system into a uniform state before snapshot or backup is taken. During normal operation, applications continuously update and modify data and write data to a disk. If a snapshot is taken during these operations, the backup may contain inconsistent or partially written data.  

To ensure data consistency, the application pauses write operations, flushes cached data to persistent storage and commits any pending transactions. This allows the snapshot to capture a consistent view of the application that is free from corruption or incomplete transactions before normal operations continue. 

### Completeness

Completeness refers to the assurance that all required data has been successfully captured during the backup or snapshot process. This includes all data, every file, storage blocks, metadata and application state necessary to restore the workload to a fully operational and consistent state. 

Achieving completeness involves more than simply copying data. It requires verifying that essential data, metadata and application state are captured without omission.  It also requires verifying that the backup process  completes successfully without interruption and that the captured data accurately reflects the applications state at the time of the backup.

In Kubernetes environment, completeness may also extend to include backuping up not only the contents of a Persistent Volume but also associated Kubernetes resources such as Persistent Volume Claims, Secrets, Config Maps, Stateful Sets, and other objects needed to fully restore the application. 

#### Persistent Volumes (PVs)

Volume modes \- block vs file

Kubernetes supports two volume modes that determine how storage is presented to an application. Filesystem mode and Block mode.

Filesystem Mode (VolumeMode: Filesystem)

The Filesystem mode is the default and most commonly used mode. Kubernetes formats the storage device with a filesystem and mounts it into the Pod as a directory.

Applications interact with the volume using familiar file operations such as reading, writing, creating and deleting files. This mode is appropriate for most workloads including web servers, application servers,  logging systems and many databases.

Advantages:

* Easy to use  
* Compatible with most applications  
* Supports directories, permissions and file metadata  
* Ideal for general-purpose workloads

Common use cases:  
  
* Web applications  
* Shared application data  
* Configurations Storage  
    
  


Block Mode (VolumeMode: Block)

Block Mode presents the raw storage device directly to the application without creating or mounting a filesystem. The application communicates directly with the underlying block device and is responsible for managing how data is organized.

This mode is often used by applications that require maximum performance or need complete control over storage management. Enterprise databases, virtualization platforms and software-defined storage systems commonly use block devices because they implement their own caching buffering  and data management.

Advantages:

* Lower overhead   
* Higher I/O performance   
* Greater control over storage layout

Common uses cases:

* High-Performance databases  
* Virtual machine disk images  
* Applications that manage that manage their own file systems  
* Database clusters

PVs are typically attached in file mode.  This mounts the volume as a file system in the worker node and Pods have the file system mapped into their directory hierarchy.

Block mode PVs are attached as raw devices to the Pods.  These are used with applications such as databases that are built to read/write directly to raw devices.  Block mode PVs must be backed up in block mode.

Backup applications performing block mode backups on file volumes will snapshot and then create a PV clone of the snapshot that is accessed in block mode.

Shared volumes issues  
File sharing  
Multi-attached block volumes

#### Kubernetes resources

Resources in Kubernetes are persistent objects such as Pods, Replica Sets, Deployments, Namespaces, Secrets and Config Maps used to represent the expected state of the cluster or store configuration information.

##### Custom Resource Definitions (CRDs) and versioning

Custom Resource Definitions extend the Kubernetes api by allowing Operators  to define custom resource types that act similarly to native kubernetes resources.  CRDS are commonly used by Kubernetes Operators to represent application specific state, configuration and operational requirements.

Since CRDs define the schema and versions of customers resources, CRD versioning is an important consideration  for backup , restore, replication and disaster recovery workflows. A backup system must preserve not only the custom resource instances but also sufficient information about the CRD schema and API version to correctly interpret and restore those resources.

CRD Versioning 

CRDS define the structure, validation rules, and API versions of custom resources. When a CRD is changed or upgraded, the custom resources that depend on it may also be affected.

This is important for back up and restore because a custom resource may have been created using one API version, while the destination cluster may use a newer version. The restore process must therefore ensure that the destination cluster has a compatible CRD and schema before restoring the associated resources.

In practice, backup  and recovery should account for both the CRD definition and the custom resources that use it . The CRD should  generally be restored and established before its dependent custom resources are restored. When CRD version schemas have changed between the source and destination clusters, compatibility and required version conversion should also be verified.

Implications for Backup and Restore

CRD versioning introduces several considerations for data protection:

* CRD definition must be protected;  
  Backing up only custom resource instances may be insufficient. The Corresponding CRD contains the schema and API information and is required to understand those resources.   
* Resources and CRD ordering matters  
  During restoration, the CRD generally needs to exist before its custom resources can be recreated successfully.   
* API versions may change between backup and restore  
* A resource backed up using one API version may need to be restored into a cluster where a newer API version is preferred or stored.  
* Conversion must be considered  
  If multiple CRD versions are supported, Kubernetes conversion mechanism may transform resources between versions. Backup and restore solutions should account for whether conversion occurs during the restore process or whether the original is preserved.  
* Schema evolution can affect compatibility   
  Changes to a CRD schema can introduce or remove fields, modify validation rules, or change the meaning of existing fields. A restore operation  should therefore consider compatibility between the source and destination CRD versions.  
  


Back up Strategy 

A robust backup strategy should treat a CRD and its associated custom resources as related objects rather than independent resources.

The backup should capture the CRD configuration with the custom resources that depend on it. If multiple versions are supported, the backup design should also record the relevant API version and storage version information.

Restore Considerations

Before restoring custom resources, the recovery process should verify:

* The required CRD exists.  
* The required API version is supported.  
* The CRD schema is compatible with the backed-up resources.  
* The destination Operator version is compatible with restored CRD.  
* Required conversion configuration is available.  
* Dependencies required by the custom resources are restored.  
* The CRD is established and available through the Kubernetes API before dependent resources are restored.  
  


##### Network ingresses and addresses

#### External resources

Shared resources that might be changed by other applications/users (e.g. RBAC) between backup and restore

#### Backup windows and Application Consistency
Backup windows define the schedule periods and duration of backup operations. Backup scheduling should account for application activity, data consistency requirements as well as recovery point objectives.  For stateful workloads, backup operations may require coordination with application Operators  or quiescing mechanisms to maintain data consistency during backup.

**1\.**  Application Consistency 

Stateful applications, such as databases, may continue to process transactions and write data while a backup is in progress. Capturing data during active writes can result in inconsistent back up if related data is captured at different points in time.

Backup procedures should account for application activity and , where necessary , coordinate backup operations with application Operators or quiescing mechanisms to ensure that application data is captured in a consistent state. 

2\. Backup Scheduling and Workload Impact 

Backup windows should consider application workload patterns, resource utilization, and the potential impact of backup operations on cluster performance.

Scheduling backups during periods of reduced activity may help minimize resource contention, including CPU, memory, storage, I/), and network bandwidth.

3\. Coordination with Kubernetes Operators 

Kubernetes Operators may provide application-specific  mechanisms for preparing workloads for backup and restoring them to a functional state. 

These mechanisms may include:

* Temporarily pausing application writes or transactions.  
* Executing pre-backup and post-backup hooks.  
* Coordinates database checkpoints or flushing pending writes.  
* Ensuring applications return to normal operation after backup completion.

The availability and behavior of these mechanisms depend on the application and its Operator. 

4\. Backup Scheduling and Recovery Objectives

Backup Scheduling frequency influences the recovery point achievable following a failure.  
Backup solutions should account for the time required to complete backup operations and ensure that the backup schedule aligns with the applications (Recovery Point Objective).

Long running backups, missed backup windows or failed backup operations may increase the potential amount of data lost during a recovery.

#### SLA (can we guarantee SLAs during backups?)

Understanding what Kubernetes features you should use  
Support in environments (e.g. CSI snapshots may not be available everywhere)  
Multi-architecture clusters (e.g. ARM & Intel nodes)

### Restore

#### Recovering from a crash

Cloning from snapshot versus reverting to a previous snapshot

When we look closely at cloning from a snapshot versus reverting to a previous snapshot we can observe different purposes and different outcomes. Reverting to a previous snapshot entails replacing its current state with a state that was captured in the selected snapshot. It reverts  the system back to the point in time the snapshot was taken.

Cloning from a snapshot leaves the original system unchanged. It is a complete and separate entity that can be used for testing and development. Unlike reverting to a previous snapshot the original system is unaffected. Use cases for  cloning include debugging, performance testing and disaster recovery.

Kubernetes does not provide APIs for reverting to a snapshot.  

#### Restoring Kubernetes resources or Persistent Volumes independently

Restoring individual pieces of an application can be desirable when the corrupted or lost data is known and can be restored into the overall state of the application.

###### Single file restore
Recovering a single file is sometimes useful to correct a single piece of data corruption without replacing the entire volume with its previous contents.

###### Single resource restore

Recovering individual resources is sometimes useful to correct a single piece of data corruption or loss without replacing all resources.

#### Recovering from inconsistency

Recovery from an  inconsistent backup may require the use of an earlier known-consistent back up or application-specific recovery mechanisms. For stateful workloads, recovery procedures should validate the consistency of related data before the application is returned to service. Where supported, transaction logs, application recovery mechanism, or coordinated snapshots may be used to restore the application to a consistent state. 

#### Test restores

Test restore verify replication strategies and backups are performing as desired by restoring data from backups or replicated sources. This is a vital component of disaster recovery.

#### Dynamic vs Static provisioning for Persistent Volumes

Persistent Volumes may be provisioned in two different fashions, dynamically and statically.  A dynamically provisioned PV is created by Kubernetes (typically via the CSI driver) to match the Spec of a Persistent Volume Claim.  A statically provisioned PV refers to a volume that already exists in the storage system.  A statically provisioned PV is created with the information needed by the storage driver to reference the volume within the storage system.

For backup, statically provisioned PVs do not pose any special problems.  On restore, however, statically provisioned PVs can be problematic.  If the storage system has been reinitialized, the static volumes will not exist and would need to be recreated before the system could be restored.  If a test restore is being made and the application is still present, the storage system volumes referred to by the statically provisioned PVs being restored would be in use.

In general, dynamically provisioned volumes are easier to handle while restoring the system since the restore process will create new, empty volumes for the restore and will not try to reattach to the original volumes. 

##### Volume topology and affinity

Volume topology, in Kubernetes, refers to the zones or nodes that a volume can be attached to.  This is useful in multi-zone clusters where volumes need to be allocated in the same zone as the compute nodes which will use them.  This is done by using the "WaitForFirstConsumer" VolumeBindingMode in a StorageClass with dynamic provisioning.  When the PV is created, the actual volume will not be allocated until a Pod using it is scheduled. The volume will then be created in the same zone as where the Pod will run.

This does not present any challenges during backup, however, during restore, the data protection application needs to create and restore the data into volumes before the application starts.  This creates a chicken-and-egg problem because Kubernetes cannot determine which  availability zone the pod will run in until it schedules the pod. At the same time, the volume must be provisioned in the same availability zone before the pod can use it. The Kubernetes Volume Populator provides a solution for this. 

##### Kubernetes volume populator

A Kubernetes Volume Populator populates (loads) a volume with data after it is allocated but before the Pod it is attached to is started.  

# Disaster recovery for clusters

A true disaster may destroy the entire cluster hardware and other infrastructure.  Have a disaster recovery plan that will enable you to get a cluster ready for your applications to be restored into.  This plan should include:

* Recovery site  
* Hardware required  
* Network settings

Many modern systems work in a cloud environment with Infrastructure as Code used to provision environments.  The IaC definitions should be kept in safe storage to be available in the event of a disaster.

# Replication

### Overview

All data protection ultimately relies on some form of replication.  In the case of backup, that replication takes place periodically and the replicated copy is static.  With online replication, data is copied to another location or locations.

### Hot/cold failover

Hot failover strategies are used in high-availability systems, such as cloud services and databases, to ensure immediate continuity in the event of failure. Since data is continuously replicated, users experience minimal or no disruption.  
A cold failover is a backup strategy used in non-critical systems that  remains completely offline until the primary system fails. When a failure occurs, the backup system must be manually started and configured, leading to significant downtime before services are restored.  
 A warm failover is a backup strategy where a standby system is partially active and periodically synchronized with the primary system. Unlike a hot failover which is live and ready to take over instantly, a warm failover requires some time to activate, leading to brief downtime before services are restored. 

### Replication of all resources and data

### Volume only replication

Volume only replication focuses solely on reproducing data within persistent volumes without duplicating the entire cluster’s configuration. 

### Handling network partitions

Handling network partitions requires special attention that will balance availability, consistency, and partition tolerance. There are several strategies available to achieve these end goals. First let's look at what happens when a network partition occurs.   
\<this section needs to be expanded\>

### What gets replicated

On Kubernetes, it is expected that a persistent volume can be replicated, as well as volume groups, for example this might be accomplished using a new resource type that when created would cause a CSI driver to configure replication on a storage array.

Storage arrays that support replication typically support configuration of replication with various custom parameters.  A Kubernetes solution for replication would need to support the notion of replication parameters which would get passed into the CSI context when replication is requested.

### Fully consistent versus eventually consistent

Replication can either be asynchronous or synchronous for a given volume(s).  The ability to support these replication modes is dependent upon the storage array’s capabilities.

# Cyber Recovery

While traditional disaster recovery focuses on business continuity during site outages or natural disasters, Cyber Recovery addresses intentional, adversarial threats such as ransomware attacks, insider threats, and supply-chain compromises. In a Kubernetes environment, cyber recovery requires specialized architectural patterns beyond standard backup and restore.

## Immutability & Air-Gapping in Cloud-Native Storage

Standard volume snapshots and online backups can be deleted or encrypted if an attacker compromises the Kubernetes control plane or backup credentials.

* WORM (Write-Once-Read-Many) & Object Lock: Backups and CSI volume snapshots intended for disaster recovery must be offloaded to immutable object storage configured with Object Lock (Compliance mode is required for complete immutability). Even if an attacker gains "cluster-admin" access and/or high level access to the object store, they cannot overwrite or delete locked objects until the retention period expires.  
* Software-Defined Air-Gapping: Physical air-gapping is impractical for ephemeral clusters because the amount of effort needed to set up the hardware is high for a cluster that will not be used very long. Implement logical/software-defined air gaps by isolating backup storage into dedicated management accounts/clusters with strict, uni-directional network boundaries and zero-trust identity checks (OIDC/short-lived credentials rather than static API tokens).

## Isolated Recovery Environments ("Clean Rooms")

Restoring a compromised workload into the same production cluster or network risks re-activating malware, triggering Command & Control beaconing, or enabling lateral movement across the cluster.

* Fenced "Clean Room" Namespaces/Clusters: Recovery should default to an isolated "Clean Room", a dedicated namespace or cluster pre-configured with strict network isolation.  
* Network Quarantine via NetworkPolicies: Before restoring workloads or pods, apply default-deny ingress and egress policies (e.g., standard K8s NetworkPolicies or CNI-level policies such as Antrea/Cilium cluster network policies). This cuts off egress traffic to prevent malicious beaconing while forensic analysis and data integrity verification take place.  
* Safe Inspection & Forensics: Once in the clean room, security teams can run vulnerability scans, inspect database tables, and verify data consistency before opening egress/ingress paths to production.

## Application & Workload Hardening for Cyber Recovery

To ensure an application can survive and recover from a ransomware attack:

* Decouple Identity from Static Tokens: Backup and restore controllers should leverage dynamic service account token volume projection or cloud OIDC federation (Workload Identity) rather than long-lived secrets that can be exfiltrated.  
* Clean Manifest Reconstruction: Store declarative Kubernetes manifests (Deployments, Stateful Sets, Services) in version-controlled Git repositories (GitOps). During a ransomware recovery, restore the \*data\* from immutable storage but recreate the *compute and workload specifications* clean from trusted Git commits to avoid restoring compromised or tampered resource definitions.  
* In-Memory and Storage Quiescing: Malware often corrupts database indexes or volatile memory. Utilizing pre-backup hooks to flush memory buffers and place applications into a clean transaction state ensures the snapshot stored in the immutable target is consistent and can be used to recover when data is corrupted by malware.

## Verification and Cyber Recovery Drills

A backup is only as good as its proven ability to restore safely under attack conditions.

* Automated Clean Room Restoration Drills: Periodically trigger automated restore pipelines into an isolated clean room namespace.  
* Data Integrity and Malware Scanning: Run automated integrity checks on restored Persistent Volumes (e.g., checksum verification or database consistency tools) while fenced off from external network connectivity.

# How to define the boundaries of an application for data protection 

\<Needs to be expanded\>

# Scenarios to expect

## Restore to same cluster

When restoring to the same cluster there is the possibility of PVC with the same name and name space may already exist. This may result in a collision because Kubernetes will not let two PVC’s with the same name coexist. 

Another challenge of same cluster restore is with the Bound state. Usually a PVC is already bound to its PV. If you restore the same PVC spec Kubernetes will not rebind it because the “old” binding is still valid.    

### Handling PVC Name Collisions When Restoring to the Same Cluster

When restoring an application to the same cluster, a common issue is that a Persistent Volume Claim(PVC) may already exist, leading to name collisions. Kubernetes does not allow two PVCs with the same name in the same namespace, so a restore will fail if not handled properly. 

1. Detect Existing PVCs:  
   Before restoring, check if PVCs already exist in the target namespace.  
     
2. Options to Resolve Collisions:  
   Delete existing PVCs if data is obsolete or already backed up.   
     
   Restore with new PVC Names:  
   Use PVC renaming or prefixing during restore to avoid collisions for standalone PVCs.  If the PVC is defined by a Stateful Set it may be necessary to change the Stateful Set naming to avoid PVC collision.

   Update the manifest to map original PVCs to new names

   Use Namespaced Backup Restore Features:

   Some backup tools provide options such as:

*    namespace mappings to restore to a new namespace  
*   volume restore with rename policies to automatically avoid collisions.


### Custom Resource Definitions

#### Best Practices:

Dry-run restores: Always simulate a restore to detect conflicts without affecting live resources.

Avoid overwriting production PVCs unless intentionally replacing old data

Versioned PVC names: For environments with frequent restores, adopt naming conventions that include timestamps or backup IDs

## Restore to a different cluster

## Restore for testing

Strategies for same cluster restore.  
A critical aspect of data recovery strategy in Kubernetes is testing for the recoverability of workloads.  However restoring “in place” on the same cluster can introduce substantial risk such as overwriting live workloads

## Restore for migration

## Restore for disaster recovery

# Best practices for applications

\<This section needs to be expanded.  This should discuss what the best practices for applications are that make data protection most effective. For example, being able to recover from a crash (so that crash consistent backups can be used) or storing all data in persistent volumes.\>

Outline for this section:  
~~Identify what needs to be protected~~
~~Identify the value/cost of data/data loss/downtime~~  
~~Pick a data protection strategy~~  
Understand what modifications your application might need for your data protection strategy  
Implement your data protection strategy
~~Test your data protection strategy~~  
Continue to monitor your application as it evolves and see if anything needs to be adjusted  
Periodically test your data protection strategy

## Identify what needs to be protected
    
The first step in developing data protection strategy is identifying the application data, data dependencies, and configuration that are critical for recovery. Since not all data has the same protection requirements, applications should classify data based on its importance, recovery needs and sensitivity.

Considerations include:

* Application data: Identify databases, objects storage,files and other persistent data generated or consumed by applications.  
* Persistent storage requirements: Ensure all critical application data is stored on persistent volumes (PVs) rather than ephemeral storage. Data stored inside containers or temporary filesystems may be lost when pods are restarted, rescheduled, or recreated.  
* Application configuration: Protect Kubernetes resources required to rebuild the application environment, including Deployments, Stateful Sets, Config Maps,Secrets, Services, Persistent Volume Claims and Custom Resources.  
* Data dependencies: Identify relationships between application components. For example, an application may require both a database and associated object storage to be restored together to maintain consistency.  
* Data consistency requirements: Applications should support crash-consistent recovery. This means that if a failure occurs during operation, restored data should represent a state that the application can safely recover from without corruption 

Applications should be designed to tolerate failures by supporting restart and recovery workflows. For example, databases should use transaction mechanisms,  write-ahead logging or similar techniques to ensure data consistency after an unexpected shutdown.

## Identify the Value, Cost, and Impact of Data Loss or downtime 

Understanding the business impact of data loss helps determine the appropriate level of protection required. Organizations should evaluate the cost of losing data, application downtime and the effort required to restore services.  
Important factors include:

* Recovery Point Objective (RPO): Determines how much data loss is acceptable. Applications requiring minimal data loss may require frequent snapshots, replications or continuous backup mechanisms.  
* Recovery Time Objective (RTO) Determines how quickly an application must be restored. Critical applications may require automated recovery processes and highly available storage solutions.  
* Business impact: Consider revenue loss, operational disruption, regulatory requirements  and reputational damage.   
* Data criticality: Product databases, customer records, and transaction systems typically require stronger protection  than temporary application data or development environments.  
* Compliance requirements: Certain workloads may require retention policies, encryption,audit trails or geography distributed copies.

## Appropriate Data Protection

Once data requirements and business impact are understood, organizations can select a protection strategy that aligns with their applications needs. A comprehensive strategy typically combines backup, recovery, replication and application design practices.

## Understand what modifications your application might need for your data protection strategy 

Ideally, your data protection strategy will not require any changes in how your application works. You should understand how your data protection strategy will interact with your application to see if any changes are required.

### Restore from crash consistent backup

Restoring from a crash consistent backup is the equivalent of a crash and reboot of your application.  If your application cannot recover from a crash, you will need to change it so that it can.

Best practices include:  
Use persistent volumes for Stateful Data  
Applications requiring durable storage should store data on Persistent Volumes. Persistent volume claims allow applications to remain independent of the underlying storage implementation while enabling backup, migration and recovery workflows

Enable crash consistent Backups  
Backups should capture data in a state that allows applications to recover after unexpected failures. Crash consistent backups preserve the state of storage at a specific point in time.  
To improve consistency:

* Use storage snapshots that coordinate with the application state.  
* Quiesce applications when required  
* Validatee store data can successfully restart the application.


Protect Application and Kubernetes Meta Data  
Backing up only application data is not enough. Recovery may also require Kubernetes resources that define how the application runs.

Protection strategies should include:

* Persistent VolumeClaims  
* Secrets and Config Maps  
* Stateful Sets and Deployment definitions  
* Custom Resource Definitions (CRDs)

Persistent data:

* All stateful data must live in Persistent Volumes (PV’s). Applications that write to ephemeral container storage will lose that data on pod restart making any backup strategy ineffective for that data.    
* Databases, filestores,queues and caches with durable modes should each have a dedicated PVC so they can be snapshotted or backed up independently. 

Cluster state:  
   

* Etcd is the authoritative record for all Kubernetes objects.  Losing etcd without a backup of the resources requires redeploying all of the resources.  GitOps practices may enable easy redeploy of the resources but this should be tested to ensure that all resources are held in GitOps.

## Test your backup/replication plans

It is very important that you test your backups or replication setup by making sure that after restoring or replicating that your application is fully functional and that no data has been lost.  It is unfortunately quite easy to create backups that do not work when restored.

Your recovery process needs to be tested periodically to ensure that changes in the application or cluster have not created problems in the backup/restore flow.

Replication also requires periodic testing.  In the case of a hot/hot replication, making sure that traffic is routed to all of the clusters is necessary to make sure that all of the clusters are fully functional.  It's also important to ensure that your replicated clusters are capable of handling the load that will be shifted to them if their replica(s) fails.

## Continue to evolve your data protection strategy

Your application and environment will continue to evolve.  Your data protection strategy needs to evolve with them.  In addition to technical changes, be aware of business level changes.  An application that was not mission critical may become mission critical or vice versa.  Changes in the importance of applications will change what the appropriate data protection strategy for your application is.

## Best practices for Operators

Operators are responsible for managing the lifecycle of Kubernetes applications, including upgrading scaling and configuration. During backup, restore or replication operations, the reconciliation behavior may unintentionally interfere with data protection workflows. To ensure that backup, restore and replication processes work reliably, Operators should implement features and guidance that allow these processes to interact safely with the applications.   

1. Quiescing Support  
   Operators should provide a way to temporarily pause or quiesce the managed application. This prevents the Operator from reconciling changes or modifying resources while a backup or restore is in process.   
     
   Quiescing is also important during a response to a cyber attack.  Changes while forensic tools are being used could interfere with understanding an attack.  Containment efforts could be undermined by Operator changes as well.  
     
   Quiescing can include suspending automatic updates, scaling actions or background tasks that could interfere with a consistent backup.  
     
   A quiesced state may include:  
     
* Suspending scaling operations  
* Disabling automated upgrades or configurations changes   
* Preventing scheduled maintenance tasks  
* Temporarily suspending application write activity when necessary to achieve a consistent back up state.

Once the backup or restore operation completes, the Operator should safely resume normal reconciliation without manual intervention.

2. Application Aware Backup Hooks  
   Operators should expose hooks or APIs that allow backup applications to perform applications-specific actions, such as committing in-flight transactions, flushing caches, or triggering database dumps. This ensures that backups capture data in a consistent and recoverable state.  
     
     
   Examples include:  
     
* Pausing write operations  
* Flushing in-memory cache  
* Committing or completing in-flight transactions  
* Triggering database snapshots or checkpoints  
* Exporting application metadata for recovery  
    
    
    
3. Restore Coordination

During operations. Operators should recognize that  restored Kubernetes resources may temporarily differ from the desired stare maintained by the Operator. Rather than immediately reconciling or replacing restored resources, Operators should support a coordinated restore workflow. 

This may include;

* Validating all dependent resources have been restored  
* Reinitializing internal state from resources  
* Delaying reconciliation until the restore is complete  
* Performing post-restore validation before resuming normal management

Proper restore coordination helps prevent the Operator from unintentionally overwriting restored data or recreating resources before recovery has finished.

4.  Replication Awareness

Applications that support disaster recovery often rely on asynchronous or synchronous replication between clusters. Operators should allow replication status and health to be monitored and coordinated with backup and failover workflows.

## Best Practices for Backup/Restore applications

## Handling failed backups \- cleanup of resources

* Make sure temporary resources (VolumeSnapshots, PVCs, Pods) are cleaned up when failure occurs

# Support you can get from backup/restore applications

## Hooks

Hooks are particularly important in Kubernetes workloads, where crash consistent backups may not provide sufficient data integrity. A crash-consistent snapshot captures data exactly as it is at an instant in time. Although this approach may be adequate for some workloads, applications such as databases, messages queues and distributed systems, often maintain data in memory, maintain pending transactions, and process write operations asynchronously. Without proper coordination these applications may require lengthy recovery procedures or, in some cases, experience data corruption or transaction loss.

Backup hooks help place an application into an application consistent state before the data is snapshotted or otherwise backed up and then return the application to operation if necessary. This process ensures that in-flight transactions are completed, cached data is flushed from memory to persistent storage, and write operations are temporarily paused. By coordinating these actions with the backup process, hooks significantly improve the reliability and recoverability of back ups.

Since hooks execute automatically and can directly impact production workloads they should be carefully designed, tested,  and properly documented before deployment. Misconfigured hooks can introduce downtime, fail silently or leave applications in a degraded state.

Regular testing of both backup and restore procedures is recommended to ensure that the hook behavior remains consistent as applications and the Kubernetes environment continues to evolve.  

Common pre-back up hook tasks include: 

* Flushing application caches and memory buffers to disk  
* Rolling back or committing  outstanding database transactions  
* Temporarily pausing  write operations or quiescing file system activity to ensure data consistency   
* Placing the application into a maintenance or back-up safe mode  
* Coordinating distributed services so all components reach a consistent state before a snapshot is taken


When the backup is complete, post backup hooks restore normal application behavior.

Post-backup hook tasks may include:

* Resuming write operations and unfreezing file systems.  
* Restarting paused services or background workers.  
* Re-enabling replication or synchronization processes.  
* Perform health checks to verify the application is operating normally.  
* Recording audit information that the backup completed successfully.

## Best practices when moving existing applications into Kubernetes

### Virtual machines

\<This section needs to be expanded.  Some legacy applications are being moved into Kubernetes as virtual machines, which need to be handled differently than container based applications, especially with multi-volume consistency\>

Key considerations when migrating from a legacy environment to Kubernetes. 

VM snapshots → CSI Volume Snapshots at the storage layer  
Legacy applications: VM snapshots captured the entire machine state at the hypervisor level.

Kubernetes equivalent:

* Volume snapshot resources via CSI drivers storage-layer snapshots of Persistent Volumes  
* Snapshots are storage native (Utilizing underlying storage systems such as Amazon EBS, Persistent Disk and Ceph) to improve reliability and performance.  
* Create snapshots using Volume snapshot custom resources  
* Restore by creating new PVCs from VolumeSnapshot sources

## Key considerations when migrating from a legacy environment to Kubernetes

Migrating from a VM centric environment into Kubernetes requires a shift in how data protection, back up and recovery are implemented. The key is understanding what each legacy practice and finding the Kubernetes native equivalent at the appropriate layer. A common point of failure is attempting  to replicate VM-era practices directly instead of implementing equivalent controls using kubernetes-native mechanisms.

Replacing Traditional VM Snapshots and backups in Kubernetes environment. Traditional VM snapshots and backup strategies are designed for monolithic infrastructure where applications, operating systems and data are tightly coupled in a single image. Kubernetes introduces a fundamental model based on containerized work loads, declarative configuration and decoupled state.  As a result, VM-centric protection mechanisms are neither sufficient nor optimal for Kubernetes.

In Kubernetes the functional equivalents of VM snapshots and backups are achieved through a combination of persistent data protection,declarative configuration management and control plan preservation. 

### Persistent Volume Snapshots as replacement for VM DIsk Snapshots

VM snapshots capture the entire disk state of a machine, including the operating system and application binaries. In Kubernetes node operating systems are considered disposable infrastructure and application binaries are stored in container images.

Persistent Volume (PV) snapshots implemented through the Container Storage Interface (CSI) snapshot APIs, provide a direct replacement for VM disk snapshots by capturing point-in-time copies of application storage volumes without including the underlying node.  VM snapshots typically pause the VM while snapshotting the disks, ensuring that all of the individual disk snapshots are consistent with each other.  If an application relies on strong consistency between individual disks, using CSI Volume Group Snapshots can provide that.

Application-Centric Backup as a Replacement for VM Backup images

Traditional VM backups preserve full machine images to enable system restoration.   
Kubernetes environments instead require protection of the following components:

* Kubernetes resource manifest (Deployments, Stateful Sets, Services, etc.)  
* Configuration data (Config Maps and Secrets)  
* Persist Volumes and associated storage metadata  
* Application-level state, such as message queues  
* Namespace and cluster metadata

This approach shifted protection from infrastructure to application state. Recovery involves redeploying workloads from declarative definitions and restoring persistent data,rather than restoring entire machines.

Rethinking VM Snapshots in Kubernetes:

In legacy environments, VM snapshots are often used as catch-all protection mechanism:

* Point-in-time recovery  
* Crash Consistent backups   
* Fast roll back after failed changes

In Kubernetes,  there is no equivalent to a VM snapshot, because:

* Applications are distributed across pods  
* Compute is ephemeral by design  
* Storage is decoupled from workloads

Instead snapshot and back up responsibilities are distributed across layers.

Kubernetes replacement for VM snapshot and back up.

1. Persistent Volume (PV) snapshots  
   What is replaces:  
   VM disk snapshots  
     
   How it works in Kubernetes:  
     
* Kubernetes supports CSI Snapshot at the storage level.  
* Snapshots are taken per Persistent Volume Claim (PVC), not per workload  
* Snapshot behavior depends on underlying storage provider

      
Operational Considerations:

* Multi-volume applications may require coordinated snapshots  
* Snapshots should be viewed as infrastructure-level recovery tools and not as comprehensive backups.