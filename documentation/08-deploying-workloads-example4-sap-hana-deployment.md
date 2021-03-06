# Spoke example 4: SAP HANA deployment

The toolkit's final example workload workload is a [SAP HANA application](https://docs.microsoft.com/azure/virtual-machines/workloads/sap/get-started). In support of this application the following workload resources will be deployed to the workload:

- Virtualized iSCSI arrays
- NFS servers to provide access to the iSCSI arrays
- SAP Hana servers
- SAP ASCS servers for enqueue capabilities
- A NetWeaver instance.

## Prerequisite: create and configure your parameters file

As discussed in the [parameter files](03-parameters-files.md#parameters-files) topic, the VDC Automation Toolkit provides a default test version of the top-level deployment parameter file. You will need to create a new version of this file before running your deployment. 

To do this, navigate to the toolkit's [archetypes/sap-hana](../archetypes/sap-hana) folder, then make a copy of the *archetype.test.json*, and name this copy *archetype.json*. Then proceed to edit archetype.json providing the subscription, organization, networking, and other configuration information that you want to use for your deployment. Make sure you use values for the hub and on-premises parameters consistent with those components of your VDC deployment.

If your copy of the toolkit is associated with a git repository, the [.gitignore](../.gitignore) file provided by the default VDC Automation Toolkit is set to prevent this archetype.json file from being pushed to your code repository.

## Deploy workload infrastructure

All workload environments require a common set of operations, key vault, and virtual network resources before they can connect to the hub network and host workloads. The following steps will  deploy these required resources. 

### Step 1: Deploy workload operations and monitoring resources

**Required role: SysOps**

This step provisions the operations and monitoring resources for the workload
environment.

Start the "ops" deployment by running the following command in the terminal or
command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "la"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "la"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "la"*

This deployment creates the *{organization name}-{deployment name}-la-rg*
resource group that hosts the resources in the following table.

| **Resource**                             | **Type**      | **Description**                              |
|------------------------------------------|---------------|----------------------------------------------|
| {organization name}-{deployment name}-oms | Log Analytics | Log Analytics instance for monitoring the hub network. |

### Step 2: Deploy workload Key Vault

**Required role: SecOps**

This step deploys the workload "kv" resource, which deploys a Key Vault for the
workload environment and generates the encryption keys that are used for resources
deployed by the workspace DevOps teams.

In addition to the workload Key Vault, this deployment generates a password for the local-admin-user name defined in the workload parameters file. This password is stored as a secret in the vault. To modify the default values for these passwords edit the [Key Vault deployment parameters file](../modules/workload-kv/1.0/azureDeploy.parameters.json) and update the secrets-object parameter.

Start the "kv" deployment by running the following command in the terminal or
command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-kv"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-kv"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-kv"*

This deployment creates the *{organization name}-{deployment name}-kv-rg*
resource group that hosts the resources listed in the following table.

| **Resource**                                           | **Type**        | **Description**                                                        |
|--------------------------------------------------------|-----------------|------------------------------------------------------------------------|
| {organization name}-{deployment name}-kv               | Key Vault       | Key Vault instance for the workload. One certificate deployed by default. |
| {organization name}{deployment name (dashes removed)}kvdiag{random characters} | Storage account | Location of Key Vault audit logs.                                      |

### Step 3: Deploy workload virtual network

**Required role: NetOps**

This step involves two resource deployments in the following order:

-   The "nsg" deployment module creates  the network security groups (NSGs) and
    Azure security groups (ASGs) that secure the workload virtual network. By
    default, the example workload net deployment creates a set of NSGs and ASGs
    compatible with an *n*-tier application, consisting of web, business, and
    data tiers.

-   The "net" deployment module creates  the workload virtual network, along with
    setting up the default subnet and User Defined Routes (UDRs) used to route
    traffic to the hub network. This deployment also creates the VNet peering
    that connects the hub and workload networks.

Start the "nsg" deployment by running the following command in the terminal or
command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nsg"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nsg"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nsg"*

Then start the "net" deployment by running the following command in the terminal
or command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-net"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-net"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "workload-net"*

These deployments create the *{organization name}-{deployment name}-net-rg*
resource group that hosts the resources detailed in the following table.

| **Resource**                                                 | **Type**               | **Description**                                                      |
|--------------------------------------------------------------|------------------------|----------------------------------------------------------------------|
| {organization name}-{deployment name}-vnet                    | Virtual network        | The primary workload virtual network, with a single default subnet.     |
| {organization name}-{deployment name}-{defaultsubnetname}-nsg | Network security group | Network security group attached to the default subnet.               |
| {organization name}-{deployment name}-udr                     | Route table            | User Defined Routes for routing traffic to and from the hub network. |
| business-asg                                                  | Azure security group   | ASG for business-tier assets.                                        |
| web-asg                                                       | Azure security group   | ASG for web-tier assets.                                             |
| data-asg                                                      | Azure security group   | ASG for data-tier assets.                                            |
| {deployment name (dashes removed)}diag{random characters}     | Storage account        | Storage location for virtual network diagnostic data.                                      |

## Deploy workload resources 

Once the workload operations, Key Vault, and virtual network resources are provisioned, your team can begin deploying actual workload resources. This will create an iSCSI virtual storage array, an NFS storage server, an SAP HANA server, an ASCS messaging server, and an SAP NetWeaver application server. 

A local user account will be created for all of these machines. The user name is defined in the local-admin-user parameter of the main deployment parameters file. The password for this user is generated and stored in the workload key vault as part of the "kv" deployment.

### Deploy iSCSI resources

The first deployment module used for this workload will deploy a single iSCSI virtual storage array and related resources.  

Start the "iscsi" deployment by running the following command in the terminal or
command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "iscsi"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "iscsi"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "iscsi"*

This deployment creates the *{organization name}-{deployment
name}-iscsi-rg* resource group that hosts the resources listed in
the following table.

| **Resource**                                                  | **Type**                             | **Description**                               |
|---------------------------------------------------------------|--------------------------------------|-----------------------------------------------|
| {deployment name (dashes removed)}diag{random characters}     | Storage account   | Storage account used for diagnostic logs related to the iSCSI VMs. |
| {organization name}-{deployment name}-iscsi-vm1                               | Virtual machine   | Virtual iSCSI VM.                                                       |
| {organization name}-{deployment name}-iscsi-vm1-nic                           | Network interface | Virtual network interface for iSCSI VM.                         |
| {organization name}{deployment name (spaces removed)}iscsivm1osdsk{random characters} | Disk              | Virtual OS disk for iSCSI VM.                                   |


### Deploy NFS resources

The next deployment module creates a pair of NFS servers with an associated load balancer providing access to the iSCSI virtual array.

Start the "nfs" deployment by running the following command in the terminal or
command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nfs"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nfs"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "nfs"*

This deployment creates the *{organization name}-{deployment
name}-nfs-rg* resource group that hosts the resources listed in
the following table.

| **Resource**                                                  | **Type**                             | **Description**                               |
|---------------------------------------------------------------|--------------------------------------|-----------------------------------------------|
| {organization name}-{deployment name}-sap-iscsi-lb                                | Load balancer  | Load balancer used for NFS servers.                                         |
| {organization name}-{deployment name}-sap-iscsi-vm1                               | Virtual machine   | Primary NFS server.                                                       |
| {organization name}-{deployment name}-sap-iscsi-vm1-nic                           | Network interface | Virtual network interface for primary NFS server.                         |
| {organization name}{deployment name (spaces removed)}sapiscsivm1osdsk{random characters} | Disk              | Virtual OS disk for primary NFS server.                                   |
| {organization name}{deployment name  (spaces removed)}sapiscsidiag{random characters}  | Storage account   | Storage account used to store diagnostic logs related to the NFS servers. |
| {organization name}-{deployment name}-sap-iscsi-vm2                               | Virtual machine   | Secondary NFS server.                                                     |
| {organization name}-{deployment name}-sap-iscsi-vm2-nic                           | Network interface | Virtual network interface for secondary NFS server.                       |
| {organization name}{deployment name (spaces removed)}sapiscsivm2osdsk{random characters} | Disk              | Virtual OS disk for secondary NFS server.                                 |

### Deploy SAP HANA resources

After successfully deploying NFS servers, use the "hana" deployment module to create a pair of SAP HANA servers with accompanying data, backup, and log drives.

Start the "hana" deployment by running the following command in the terminal or command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "hana"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "hana"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "hana"*

This deployment creates the *{organization name}-{deployment
name}-hana-rg* resource group that hosts the resources listed in
the following table.

| **Resource**                                                  | **Type**                             | **Description**                               |
|---------------------------------------------------------------|--------------------------------------|-----------------------------------------------|
| hanavm1backup{random characters} | Disk              | Virtual disk for primary SAP HANA server backups.                                   |
| hanavm1data1{random characters} | Disk              | First virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm1data2{random characters} | Disk              | Second virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm1data3{random characters} | Disk              | Third virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm1log1{random characters} | Disk              | First virtual disk attached to primary SAP HANA server for log volumes.                                   |
| hanavm1log2{random characters} | Disk              | Second virtual disk attached to primary SAP HANA server for log volumes.                                   |
| hanavm1sap{random characters} | Disk              | Virtual disk used for the primary SAP HANA server's system volume.                                   |
| hanavm1shared{random characters} | Disk              | Virtual disk used for the primary SAP HANA server's shared volume.                                   |
| hanavm2backup{random characters} | Disk              | Virtual disk for primary SAP HANA server backups.                                   |
| hanavm2data1{random characters} | Disk              | First virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm2data2{random characters} | Disk              | Second virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm2data3{random characters} | Disk              | Third virtual disk attached to primary SAP HANA server for data volumes.                                   |
| hanavm2log1{random characters} | Disk              | First virtual disk attached to primary SAP HANA server for log volumes.                                   |
| hanavm2log2{random characters} | Disk              | Second virtual disk attached to primary SAP HANA server for log volumes.                                   |
| hanavm2sap{random characters} | Disk              | Virtual disk used for the primary SAP HANA server's system volume.                                   |
| hanavm2shared{random characters} | Disk              | Virtual disk used for the primary SAP HANA server's shared volume.                                   |
| {organization name}-{deployment name}-hana-as                                | Availability set  | Availability set associated with SAP HANA VMs.                                         |
| {organization name}-{deployment name}-hana-lb                                | Load balancer  | Load balancer used to distribute traffic between SAP HANA servers.                                         |
| {organization name}-{deployment name}-hana-vm1                               | Virtual machine   | Primary SAP HANA server VM.                                                       |
| {organization name}-{deployment name}-hana-vm1-nic                           | Network interface | Virtual network interface for primary SAP HANA server.                         |
| {organization name}{deployment name (spaces removed)}hanavm1osdsk{random characters} | Disk              | Virtual OS disk for primary SAP HANA server.                                   |
| {organization name}{deployment name  (spaces removed)}hanadiag{random characters}  | Storage account   | Storage account used to store diagnostic logs related to the SAP HANA servers. |
| {organization name}-{deployment name}-hana-vm2                               | Virtual machine   | Secondary SAP HANA server VM.                                                     |
| {organization name}-{deployment name}-hana-vm2-nic                           | Network interface | Virtual network interface for secondary SAP HANA server.                       |
| {organization name}{deployment name (spaces removed)}hanavm2osdsk{random characters} | Disk              | Virtual OS disk for secondary SAP HANA server.                                 |


### Deploy ASCS resources

The "ascs" deployment module creates a pair of SAP ASCS servers to provide enqueue capabilities for your SAP HANA deployment.

Start the "ascs" deployment by running the following command in the terminal or command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "ascs"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "ascs"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "ascs"*

This deployment creates the *{organization name}-{deployment
name}-ascs-rg* resource group that hosts the resources listed in
the following table.

| **Resource**                                                  | **Type**                             | **Description**                               |
|---------------------------------------------------------------|--------------------------------------|-----------------------------------------------|
| {organization name}{deployment name  (spaces removed)}diag{random characters}  | Storage account   | Storage account used to store diagnostic logs related to the ASCS servers. |
| {organization name}-{deployment name}-sap-as                                | Availability set  | Availability set associated with ASCS VMs.                                         |
| {organization name}-{deployment name}-sap-ascs-lb                                | Load balancer  | Load balancer used to distribute traffic between ASCS servers.                                         |
| {organization name}-{deployment name}-sap-ascs-vm1                               | Virtual machine   | Primary ASCS server VM.                                                       |
| {organization name}-{deployment name}-sap-ascs-vm1_disk2_{random characters} | Disk              | Virtual data disk for primary ASCS server.                                   |
| {organization name}-{deployment name}-sap-ascs-vm1_disk3_{random characters} | Disk              | Virtual data disk for primary ASCS server.                                   |
| {organization name}-{deployment name}-sap-ascs-vm1-nic                           | Network interface | Virtual network interface for primary ASCS server.                         |
| {organization name}-{deployment name}-sap-ascs-vm1-pip | Public IP address | Public IP address used by the providing external access to the primary ASCS server [*see note].                               |
| {organization name}{deployment name (spaces removed)}sapascsvm1osdsk{random characters} | Disk              | Virtual OS disk for primary ASCS server.                                   |
| {organization name}-{deployment name}-sap-ascs-vm2                               | Virtual machine   | Secondary ASCS server VM.                                                       |
| {organization name}-{deployment name}-sap-ascs-vm2_disk2_{random characters} | Disk              | Virtual data disk for secondary ASCS server.                                   |
| {organization name}-{deployment name}-sap-ascs-vm2_disk3_{random characters} | Disk              | Virtual data disk for secondary ASCS server.                                   |
| {organization name}-{deployment name}-sap-ascs-vm2-nic                           | Network interface | Virtual network interface for secondary ASCS server.                         |
| {organization name}-{deployment name}-sap-ascs-vm2-pip | Public IP address | Public IP address used by the providing external access to the secondary ASCS server [*see note].                               |
| {organization name}{deployment name (spaces removed)}sapascsvm2osdsk{random characters} | Disk              | Virtual OS disk for secondary ASCS server.                                   |

\**Note*: Although this deployment creates Public IP Addresses, these will not be accessible to the public internet unless the SecOps teams modifies the workload NSG to allow it.


### Deploy NetWeaver resources

The final deployment for this workload creates an SAP NetWeaver instance used in conjunction with the previously deployed SAP HANA servers.

Start the "netweaver" deployment by running the following command in the terminal or command-line interface:

[Linux/OSX]

>   *python3 vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "netweaver"*

[Windows]

>   *py vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "netweaver"*

[Docker]

>   *python vdc.py create workload -path "archetypes/sap-hana/archetype.json" -m "netweaver"*

This deployment creates the *{organization name}-{deployment
name}-netweaver-rg* resource group that hosts the resources listed in
the following table.

| **Resource**                                                  | **Type**                             | **Description**                               |
|---------------------------------------------------------------|--------------------------------------|-----------------------------------------------|
| nwdiag{random characters}  | Storage account   | Storage account used to store diagnostic logs related to the NetWeaver VM. |
| {organization name}-{deployment name}-sap-as                                | Availability set  | Availability set associated with NetWeaver VM.                                         |
| {organization name}-{deployment name}-sap-nw-vm1                               | Virtual machine   | NetWeaver virtual machine.                                                       |
| {organization name}-{deployment name}-sap-nw-vm1-nic                           | Network interface | Virtual network interface for NetWeaver VM.                         |
| {organization name}{deployment name (spaces removed)}sapnwvm1osdsk{random characters} | Disk              | Virtual OS disk for NetWeaver VM.                                   |

