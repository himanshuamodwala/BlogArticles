# Trino on Azure Kubernetes Services with Private Endpoints
**Author**: Himanshu Amodwala (hiamodwala@microsoft.com)

[[_TOC_]]

This tutorial provides steps for using the Azure portal to setup Trino on Azure Kubernetes Services using Private Endpoints.

### Problem Context

Trino is a distributed SQL query engine designed to query large data sets distributed over one or more heterogeneous data sources. Recently, a few customers approached us with a dilemma on how to deploy Trino on their Azure environment in a secure and scalable fashion that complements their existing On-Premises setup and since there was no holistic documentation available so far for such a use case this led to the creation of this article.

### High Level Architecture Diagram


![image.png](/.attachments/image-3a3b61d1-d8d2-45b8-a802-2d8829700d57.png)

## Section I: Setting up the Infrastructure 

### Background

This is an optional section to replicate the environment that we have used for our deployment. Below are the Azure CLI commands to create / replicate environment, please skip this section entirely if using existing resources or you may pick and choose missing resources for deployment.

### Assumptions

Following assumptions are made while considering the deployment

- Azure setup consists of Enterprise Landing Zone with Hub & Spoke Model
- All Azure resources must communicate via Private Endpoints
- No Public Inbound Internet connectivity
- Heavily Restricted Outbound Internet connectivity
- Synapse Workspace is in Managed Virtual Network & DEP (Data Exfiltration Protection) Enabled
- Heavily Restricted Inbound RDP and SSH connection capabilities to VMs and other PaaS services


### Prerequisties

The following pre-requisites are required to get started

- Azure CLI installed on your Laptop, we will be running all our commands from Azure CLI on Windows 11 Terminal
- Owner permissions on Azure Subscription to avoid any permission issues
- Connectivity between your Laptop to Azure Resources i.e., via P2S, S2S, ER or Internet (if there are no policy restrictions)


### Resource Deployment (BOM)

The Following Azure Resources will be deployed as a part of this section

- Resource Group
- Virtual Network & Subnets
- Azure Bastion
- Azure Virtual Machine
- Azure Data Lake Storage Account Gen2
- Azure Synapse Analytics
- Azure SQL Database
- Azure Container Registry
- Azure Kubernetes Services
- Azure Private Link Services
- Azure Private Endpoints

_Note: Feel free to use as appropriate resource naming conventions, the below resource names might not be available_

| _Important: Please select your SKU's based on your requirements, the below SKU's are used for demonstration purposes and are not to be considered for Production Deployment or as Best Practice from Microsoft_ |
|--|

#### Resource Group

To Hold all the resources for this exercise, helps better organize and makes cleanup of the newly created resources

`az group create -l centralindia -n TrinoRG`

#### Virtual network

Need to create a Virtual Network and corresponding Subnets to hold all resource addresses

`az network vnet create --name trinoVirtualNetwork --resource-group TrinoRG --address-prefixes 10.0.0.0/8 --location centralindia --subnet-name DefaultSubnet --subnet-prefixes 10.0.0.0/24`

`az network vnet subnet create --name VirtualMachineSubnet --address-prefixes 10.0.1.0/24 --resource-group TrinoRG --vnet-name trinoVirtualNetwork`

`az network vnet subnet create --name PrivateEndpointSubnet --address-prefixes 10.0.2.0/24 --resource-group TrinoRG --vnet-name trinoVirtualNetwork`

`az network vnet subnet create --name AzureBastionSubnet --address-prefixes 10.0.3.0/24 --resource-group TrinoRG --vnet-name trinoVirtualNetwork`

`az network vnet subnet create --name KubernetesPrivateClusterSubnet --address-prefixes 10.1.0.0/16 --resource-group TrinoRG --vnet-name trinoVirtualNetwork`

#### Bastion

Since, the environment has very restricted connectivity i.e. No SSH or RDP, we will use Azure Bastion to securely connect to the VMs from local machines.

`az network public-ip create --resource-group TrinoRG --name AzureBastionPIP --sku Standard --location centralindia`

`az network bastion create --name TrinoBastion --sku Basic --public-ip-address AzureBastionPIP --resource-group TrinoRG --vnet-name trinoVirtualNetwork --location centralindia`

#### Azure Virtual Machine

Most of the resources we will deploy will have public internet connectivity disabled and we will use Azure VMs to orchestrate and deploy to such resources.

`az vm create --name DeveloperVM --resource-group TrinoRG --admin-username devadmin --admin-password Some-Strong-Password-4-Here --authentication-type password --vnet-name trinoVirtualNetwork --subnet VirtualMachineSubnet --image UbuntuLTS --size Standard_B4ms --public-ip-address '""' --nsg-rule NONE`

Once the Azure VM is deployed, please follow the below mentioned manual steps to setup rest of the Azure Resources.

Manual Steps:

- Log into VM Server via Bastion ([URL](https://learn.microsoft.com/en-us/azure/bastion/bastion-connect-vm-ssh-linux))
- Install Azure CLI on VM ([URL](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli-linux?pivots=apt))
- Connect Azure CLI to your Tenant (`az login`)
- Connect Azure CLI to your Subscription (`az account set --subscription "YourSubName"`)
- Install Kubectl ([URL](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux))
- Install Helm ([URL](https://get.helm.sh/helm-v3.10.0-linux-amd64.tar.gz))
- Install Docker ([URL](https://docs.docker.com/engine/install/ubuntu/)), 
- Post Install Actions on Docker (`sudo usermod -aG docker $USER`), please note disconnect and reconnect to your bastion session once the post install action is completed for changes to get reflected

| _Important: All further steps would be carried out in the Azure CLI of the Ubuntu VM_ |
|--|

#### Storage Account - ADLS Gen2

As a recommended good practice for Synapse deployment, we will deploy 2 storage accounts viz. One for Synapse to store the Spark Logs & Spark Warehouse data for Managed Tables and other where all the data resides in datalake (bronze or silver or golden). We are creating storage accounts with no public internet connectivity, just the Subnet where the VM resides, and the Private Endpoints are valid connectivity options.

DataLake Storage Account (Golden Layer)

`az storage account create --name trinodatalake --resource-group TrinoRG --location centralindia --access-tier Hot --kind StorageV2 --allow-blob-public-access false --enable-hierarchical-namespace true --sku Standard_RAGRS --https-only true --min-tls-version TLS1_2 --require-infrastructure-encryption true --public-network-access Disabled --publish-internet-endpoints false --publish-microsoft-endpoints false --bypass AzureServices --default-action Deny`

`az storage account blob-service-properties update --account-name trinodatalake --resource-group TrinoRG --enable-delete-retention true --delete-retention-days 7 --enable-container-delete-retention true --container-delete-retention-days 7`

`az network vnet subnet update -g TrinoRG --vnet-name trinoVirtualNetwork --name VirtualMachineSubnet --service-endpoints Microsoft.Storage`

`az storage account update --name trinodatalake --public-network-access Enabled`

`az storage account network-rule add -g TrinoRG --account-name trinodatalake --vnet-name trinoVirtualNetwork --subnet VirtualMachineSubnet`

`export trinodatalakesakey=$(az storage account keys list -g TrinoRG -n trinodatalake --query [0].value -o tsv)`

`az storage container create --name datalake --account-name trinodatalake --account-key $trinodatalakesakey --public-access off`

`az storage account update --name trinodatalake --public-network-access Disabled`

Synapse Primary Storage Account

`az storage account create --name trinosynapseprimary --resource-group TrinoRG --location centralindia --access-tier Hot --kind StorageV2 --allow-blob-public-access false --enable-hierarchical-namespace true --sku Standard_RAGRS --https-only true --min-tls-version TLS1_2 --require-infrastructure-encryption true --public-network-access Disabled --publish-internet-endpoints false --publish-microsoft-endpoints false --bypass AzureServices --default-action Deny`

`az storage account blob-service-properties update --account-name trinosynapseprimary --resource-group TrinoRG --enable-delete-retention true --delete-retention-days 7 --enable-container-delete-retention true --container-delete-retention-days 7`

`az storage account network-rule add -g TrinoRG --account-name trinosynapseprimary --vnet-name trinoVirtualNetwork --subnet VirtualMachineSubnet`

`az storage account update --name trinosynapseprimary --public-network-access Enabled`

`export trinosynapseprimarysakey=$(az storage account keys list -g TrinoRG -n trinosynapseprimary --query [0].value -o tsv)`

`az storage container create --name synapse --account-name trinosynapseprimary --account-key $trinosynapseprimarysakey --public-access off`

`az storage container create --name warehouse --account-name trinosynapseprimary --account-key $trinosynapseprimarysakey --public-access off`

`az storage account update --name trinosynapseprimary --public-network-access Disabled`

#### Synapse Workspace

We will create a Synapse Workspace that resides in a Managed Virtual Network and has Data Exfiltration Protection Enabled to comply with the Security Policies.

`export tenantId=$(az account tenant list --query [0].tenantId -o tsv)`

`az synapse workspace create --name trinosynapse --resource-group TrinoRG --location centralindia --storage-account trinosynapseprimary --file-system synapse --sql-admin-login-user devadmin --sql-admin-login-password Some-Strong-Password-4-Here --enable-managed-virtual-network true --prevent-data-exfiltration true --allowed-tenant-ids $tenantId --no-wait`

Manual Steps: 

Since `az synapse` command is still in preview there are a few missing functionalities below are some manual tasks to be performed 
- Check if your identity has Contributor IAM for workspace if not grant it
- Disable public access to synapse workspace ([URL](https://learn.microsoft.com/en-us/azure/synapse-analytics/security/connectivity-settings#configure-public-network-access-after-you-create-your-workspace)) if you can access via private endpoints.

##### Synapse Spark Pool

With Synapse Workspace created let's create the Spark Pools.

`az synapse spark pool create --name SparkPool --node-count 5 --node-size Medium --resource-group TrinoRG --spark-version 3.2 --workspace-name trinosynapse --enable-auto-pause true --delay 15 --enable-auto-scale true --max-node-count 5 --min-node-count 3 --no-wait`

#### Azure SQL Database

Next, we will create an Azure SQL Database to act as the Hive Metastore DB, we are selecting Azure SQL since it is supported by Hive Standalone Metastore server and developers comfort level. 

_Azure SQL Server_

`az sql server create --name trinoexternalmetastore --resource-group TrinoRG --admin-password Some-Strong-Password-4-Here --admin-user devadmin --enable-public-network false --location centralindia --minimal-tls-version 1.2 --restrict-outbound-network-access true --no-wait`

_Azure SQL Database_

`az sql db create --name HiveMetastore --resource-group TrinoRG --server trinoexternalmetastore --auto-pause-delay 60 --compute-model Serverless --read-scale Disabled --zone-redundant false --family Gen5 --edition GeneralPurpose --min-capacity 1 --capacity 2 --max-size 32GB --backup-storage-redundancy Local --no-wait --yes`

#### Azure Container Registry

Azure Container Registry to hold the images that we will create for Hive Metastore, Trino. Please note that Premium SKU offers Private Endpoint connectivity. 

`az acr create --name trinoregistry --resource-group TrinoRG --sku Premium --location centralindia --allow-trusted-services true --default-action Deny --public-network-enabled false --zone-redundancy Disabled`

#### Azure Kubernetes Services

Next, we will create Azure Kubernetes Services with a Private Cluster.

Pre-requisites Setup:

_Service Principal to own the AKS Resource_

`az ad sp create-for-rbac --name spTrinoAks`

Manual Steps: 

- Copy the "appId" from the output of above command and assign it to the below $appId variable 

- Copy the "password" from the output of the avove command and assign it to the below $password variable since we will not be able to retrieve the password again.

For Example:

`export appId="<appId GUID>"`
`export password="<appId Password>"`

_Networking Configurations_ 

`az network route-table create --name rtTrinoAks --resource-group TrinoRG`

`az network vnet subnet update --name KubernetesPrivateClusterSubnet --resource-group TrinoRG --vnet-name trinoVirtualNetwork --route-table rtTrinoAks`

`export vnetId=$(az network vnet list -g TrinoRG --query [0].id -o tsv)`

`export subnetId=$(az network vnet subnet list -g TrinoRG --vnet-name trinoVirtualNetwork --query [4].id -o tsv)`

`export routeTableId=$(az network route-table list -g TrinoRG --query [0].id -o tsv)`

_Azure RBAC Permissions_

`az role assignment create --assignee $appId --scope $vnetId --role Contributor`

`az role assignment create --assignee $appId --scope $subnetId --role Contributor`

`az role assignment create --assignee $appId --scope $routeTableId --role Contributor`

_Azure Kubernetes Service Private Cluster_

`az aks create --resource-group TrinoRG --name trinoaks --location centralindia --generate-ssh-keys --enable-private-cluster --network-plugin kubenet --disable-public-fqdn --service-cidr 10.10.0.0/16 --dns-service-ip 10.10.0.10 --vnet-subnet-id $subnetId --docker-bridge-address 172.17.0.1/16 --pod-cidr 10.245.0.0/16 --service-principal $appId --client-secret $password --outbound-type userDefinedRouting --attach-acr trinoregistry --enable-cluster-autoscaler --min-count 3 --max-count 10 --node-vm-size Standard_E8_v5`

Please note since Trino is an In-Memory Operation service, it is recommended to use E-Series VMs as Nodes for Node Pools for a better performance.

#### Azure Private Endpoints

The final step is to connect all services created above via Private Endpoint connectivity

_Storage Accounts (ADLS Gen2)_

`az network private-dns zone create --resource-group TrinoRG --name "privatelink.dfs.core.windows.net"`

`az network private-dns link vnet create --resource-group TrinoRG --zone-name "privatelink.dfs.core.windows.net" --name DfsCoreLink --virtual-network trinoVirtualNetwork --registration-enabled false`

`export trinodatalakeid=$(az storage account list --resource-group TrinoRG --query '[0].[id]' -o tsv)`

`az network private-endpoint create --connection-name TrinoDataLakePeConnection --name TrinoDataLakePe --private-connection-resource-id $trinodatalakeid --resource-group TrinoRG --subnet PrivateEndpointSubnet --group-id dfs --vnet-name trinoVirtualNetwork`

`az network private-endpoint dns-zone-group create --resource-group TrinoRG --endpoint-name TrinoDataLakePe --name DfsZoneGroup --private-dns-zone "privatelink.dfs.core.windows.net" --zone-name AdlsDfsZone`

`export trinosynapseprimaryid=$(az storage account list --resource-group TrinoRG --query '[1].[id]' -o tsv)`

`az network private-endpoint create --connection-name TrinoSynapsePrimaryPeConnection --name TrinoSynapsePrimaryPe --private-connection-resource-id $trinosynapseprimaryid --resource-group TrinoRG --subnet PrivateEndpointSubnet --group-id dfs --vnet-name trinoVirtualNetwork`

`az network private-endpoint dns-zone-group create --resource-group TrinoRG --endpoint-name TrinoSynapsePrimaryPe --name DfsZoneGroup --private-dns-zone "privatelink.dfs.core.windows.net" --zone-name AdlsDfsZone`

_Azure SQL Server_

`az network private-dns zone create --resource-group TrinoRG --name "privatelink.database.windows.net"` 

`az network private-dns link vnet create --resource-group TrinoRG --zone-name "privatelink.database.windows.net" --name SqlServerDatabaseLink --virtual-network trinoVirtualNetwork --registration-enabled false`

`export hivemetastoredbid=$(az sql server list -g TrinoRG --query [0].id -o tsv)`

`az network private-endpoint create --connection-name TrinoExternalMetastoreConnection --name TrinoExternalMetastorePe --private-connection-resource-id $hivemetastoredbid --resource-group TrinoRG --subnet PrivateEndpointSubnet --group-id sqlServer --vnet-name trinoVirtualNetwork`

`az network private-endpoint dns-zone-group create --resource-group TrinoRG --endpoint-name TrinoExternalMetastorePe --name SqlServerZoneGroup --private-dns-zone "privatelink.database.windows.net" --zone-name SqlDatabaseZone`

_Azure Container Registry_ 

`az network private-dns zone create --resource-group TrinoRG --name "privatelink.azurecr.io"` 

`az network private-dns link vnet create --resource-group TrinoRG --zone-name "privatelink.azurecr.io" --name AcrLink --virtual-network trinoVirtualNetwork --registration-enabled false`

`export trinoregistryid=$(az acr list -g TrinoRG --query [0].id -o tsv)`

`az network private-endpoint create --connection-name TrinoRegistryConnection --name TrinoRegistryPe --private-connection-resource-id $trinoregistryid --resource-group TrinoRG --subnet PrivateEndpointSubnet --group-id registry --vnet-name trinoVirtualNetwork`

`az network private-endpoint dns-zone-group create --resource-group TrinoRG --endpoint-name TrinoRegistryPe --name AcrZoneGroup --private-dns-zone "privatelink.azurecr.io" --zone-name RegistryZone`

Congratulations, we have successfully deployed the required Infrastructure considering Data & Information Security Policies.

##Section II: Setting up Trino

With the Infrastructure in place, let's get cracking towards setting up Trino on Azure Kubernetes Services. 

Trino Delta Connector has an inherent dependency on Hive Metastore, luckily from version 3 onwards we have a Standalone Metastore server without the need to configure Hive, HDFS and other Hadoop Complications, since this is a Trino Installation we will go ahead with the Standalone Server.

Steps Followed: 
1. Setup Metastore Database Schema on Azure SQL Database
1. Setup Standalone Hive Metastore Server
1. Setup Trino via Helm charts


### Step 1: Setting up Standalone Hive Metastore Database Schema

Login to Ubuntu VM using Bastion and follow the commands below

`export HiveMetastoreVersion="3.1.3"`
`export HadoopVersion="3.3.4"`
`export MsSQLDriverVersion="11.2.0"`

`sudo apt-get install openjdk-11-jdk -y`

`wget https://repo1.maven.org/maven2/org/apache/hive/hive-standalone-metastore/${HiveMetastoreVersion}/hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz`

`tar -xvf hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz`

`rm -f hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz`

`mv apache-hive-metastore-${HiveMetastoreVersion}-bin metastore`

`wget --no-check-certificate https://dlcdn.apache.org/hadoop/common/hadoop-${HadoopVersion}/hadoop-${HadoopVersion}.tar.gz`

`tar -xvf hadoop-${HadoopVersion}.tar.gz` 

`rm -f hadoop-${HadoopVersion}.tar.gz`

`mv hadoop-${HadoopVersion} hadoop`

`export HADOOP_HOME=/home/$USER/hadoop`

`export JAVA_HOME=/usr/lib/jvm/java-1.11.0-openjdk-amd64`

`wget https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/${MsSQLDriverVersion}.jre8/mssql-jdbc-${MsSQLDriverVersion}.jre8.jar`

`mv mssql-jdbc-${MsSQLDriverVersion}.jre8.jar metastore/lib/`

```
cat <<EOT >> metastore/scripts/metastore/upgrade/mssql/hive-schema-3.1.0.mssql.sql

-- -----------------------------------------------------------------
-- HIVE-19416
-- -----------------------------------------------------------------
ALTER TABLE TBLS ADD WRITE_ID bigint NOT NULL DEFAULT(0);
ALTER TABLE PARTITIONS ADD WRITE_ID bigint NOT NULL DEFAULT(0);

EOT
```

Note: Please ensure the formatting of the sql file post the above command

`mv metastore/conf/metastore-site.xml metastore/conf/metastore-site.xml.bak`

```
cat <<EOT >> metastore/conf/metastore-site.xml
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:sqlserver://trinoexternalmetastore.database.windows.net:1433;database=HiveMetastore;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;</value>
        <description>Azure SQL Database Connection String without UserName and Password</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.microsoft.sqlserver.jdbc.SQLServerDriver</value>
        <description>com.microsoft.sqlserver.jdbc.SQLServerDriver</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>devadmin@trinoexternalmetastore</value>
        <description>Hive/Admin user for Azure SQL</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>Some-Strong-Password-4-Here</value>
        <description>Password for Hive/Admin User</description>
    </property>
</configuration>
EOT
```
Note: Please ensure the formatting of the sql file post the above command

`metastore/bin/schematool -initSchema -dbType mssql` 

The Hive SchemaTool will connect to your Azure SQL Database using the credentials and connection string provided in the metastore-site.xml file and initialize the database schema.

### Step 2: Setting up Standalone Hive Metastore

Since we don't have a helm chart (as of me writing this documentation) for Standalone Hive Metastore server we will be creating a custom docker image to have this hosted on AKS with replica set enabled

Login to Ubuntu VM using Bastion and follow the commands below

`mkdir hms-docker`

`cd hms-docker`

_Artifact: DockerFile_

`vim Dockerfile`


```
FROM ubuntu:22.04

ARG HiveMetastoreVersion="3.1.3" 
ARG HadoopVersion="3.3.4"
ARG MsSQLDriverVersion="11.2.0"

# Install Wget, Java 1.8 and clean cache
RUN apt-get update \
  && apt-get install -y openjdk-8-jdk wget \
  && apt-get clean all

# Setup Hive Metastore Standalone Packages
RUN wget https://repo1.maven.org/maven2/org/apache/hive/hive-standalone-metastore/${HiveMetastoreVersion}/hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz && \
    tar -xvf hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz && \
    rm -f hive-standalone-metastore-${HiveMetastoreVersion}-bin.tar.gz && \
    mv apache-hive-metastore-${HiveMetastoreVersion}-bin /opt/metastore

# Setup Hadoop Packages
RUN wget --no-check-certificate https://dlcdn.apache.org/hadoop/common/hadoop-${HadoopVersion}/hadoop-${HadoopVersion}.tar.gz && \
    tar -xvf hadoop-${HadoopVersion}.tar.gz && \
    rm -f hadoop-${HadoopVersion}.tar.gz && \
    mv hadoop-${HadoopVersion} /opt/hadoop-${HadoopVersion}/

# Copy MSSQL JDBC connector
RUN wget https://repo1.maven.org/maven2/com/microsoft/sqlserver/mssql-jdbc/${MsSQLDriverVersion}.jre8/mssql-jdbc-${MsSQLDriverVersion}.jre8.jar && \
    cp mssql-jdbc-${MsSQLDriverVersion}.jre8.jar /opt/metastore/lib/

# Download Dependencies
RUN wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-azure/3.3.4/hadoop-azure-3.3.4.jar \
&& wget https://repo1.maven.org/maven2/com/microsoft/azure/azure-storage/8.6.6/azure-storage-8.6.6.jar \
&& wget https://repo1.maven.org/maven2/org/apache/hadoop/thirdparty/hadoop-shaded-guava/1.1.1/hadoop-shaded-guava-1.1.1.jar \
&& wget https://repo1.maven.org/maven2/org/codehaus/jackson/jackson-core-asl/1.9.13/jackson-core-asl-1.9.13.jar \
&& wget https://repo1.maven.org/maven2/org/eclipse/jetty/jetty-util-ajax/11.0.11/jetty-util-ajax-11.0.11.jar \
&& wget https://repo1.maven.org/maven2/org/wildfly/openssl/wildfly-openssl/2.2.5.Final/wildfly-openssl-2.2.5.Final.jar \
&& wget https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-azure-datalake/3.3.4/hadoop-azure-datalake-3.3.4.jar \
&& wget https://repo1.maven.org/maven2/com/microsoft/azure/azure-data-lake-store-sdk/2.3.10/azure-data-lake-store-sdk-2.3.10.jar \
&& cp *.jar /opt/metastore/lib \
&& rm -f *.jar

# environment variables requested by Hive metastore
ENV JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
ENV HADOOP_HOME=/opt/hadoop-${HadoopVersion}
ENV HADOOP_CONF_DIR=/etc/hadoop/conf

# replace a library and add missing libraries
RUN rm -f /opt/metastore/lib/guava-19.0.jar \
&& cp ${HADOOP_HOME}/share/hadoop/common/lib/guava-27.0-jre.jar /opt/metastore/lib

WORKDIR /opt/metastore

# Copy Hive metastore configuration file
COPY metastore-site.xml /opt/metastore/conf/
COPY core-site.xml /etc/hadoop/conf

# Expose Metastore Port
EXPOSE 9083

# Start Metastore Services
ENTRYPOINT ["/bin/bash"]
CMD ["/opt/metastore/bin/start-metastore"]
```

_Artifact: metastore-site.xml_

`vim metastore-site.xml`

```
<configuration>
    <property>
        <name>fs.azure.account.key.trinosynapseprimary.dfs.core.windows.net</name>
        <value>trinosynapseprimary-Storage-Account-Key-Goes-Here</value>
    </property>
    <property>
        <name>fs.azure.account.key.trinodatalake.dfs.core.windows.net</name>
        <value>trinodatalake-Storage-Account-Key-Goes-Here</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:sqlserver://trinoexternalmetastore.database.windows.net:1433;database=HiveMetastore;encrypt=true;trustServerCertificate=false;hostNameInCertificate=*.database.windows.net;loginTimeout=30;</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.microsoft.sqlserver.jdbc.SQLServerDriver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>devadmin@trinoexternalmetastore</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>Some-Strong-Password-4-Here</value>
    </property>
    <property>
        <name>hive.metastore.event.db.notification.api.auth</name>
        <value>false</value>
    </property>
    <property>
        <name>metastore.thrift.uris</name>
        <value>thrift://localhost:9083</value>
        <description>Thrift URI for the remote metastore. Used by metastore client to connect to remote metastore.</description>
    </property>
    <property>
        <name>metastore.task.threads.always</name>
        <value>org.apache.hadoop.hive.metastore.events.EventCleanerTask</value>
    </property>
    <property>
        <name>metastore.expression.proxy</name>
        <value>org.apache.hadoop.hive.metastore.DefaultPartitionExpressionProxy</value>
    </property>
    <property>
        <name>metastore.warehouse.dir</name>
        <value>abfss://warehouse@trinosynapseprimary.dfs.core.windows.net/trino</value>
    </property>
    <property>
        <name>hive.cluster.delegation.token.store.class</name>
        <value>org.apache.hadoop.hive.thrift.DBTokenStore</value>
    </property>
</configuration>
```

_Artifact: core-site.xml_

`vim core-site.xml`

```
<configuration>
    <property>
        <name>fs.azure.account.auth.type.trinodatalake.dfs.core.windows.net</name>
        <value>SharedKey</value>
    </property>
    <property>
        <name>fs.azure.account.key.trinodatalake.dfs.core.windows.net</name>
        <value>trinodatalake-Storage-Account-Key-Goes-Here</value>
    </property>
    <property>
        <name>fs.azure.account.auth.type.trinosynapseprimary.dfs.core.windows.net</name>
        <value>SharedKey</value>
    </property>
    <property>
        <name>fs.azure.account.key.trinosynapseprimary.dfs.core.windows.net</name>
        <value>trinosynapseprimary-Storage-Account-Key-Goes-Here</value>
    </property>
</configuration>
```


#### Deploy Hivemetastore Image to Azure Container Registry

Navigate to the directory where the Dockerfile, core-site.xml and metastore-site.xml are located

`docker build --no-cache -t hivemetastore/3.1.3:v1 .`

`az acr login --name trinoregistry`

`docker tag hivemetastore/3.1.3:v1 trinoregistry.azurecr.io/hivemetastore/3.1.3:v1`

`docker push trinoregistry.azurecr.io/hivemetastore/3.1.3:v1`

#### Deploy HiveMetastore Image on Kubernetes

_Artifact: Service Yaml for Standalone Hive Metastore Image_

`vim aks-hm-service.yaml`

```
apiVersion: v1
kind: Service
metadata:
  name: hivemetastore
  namespace: default
spec:
  ports:
    - targetPort: 9083
      name: metastore
      port: 9083
      protocol: TCP
  selector:
    app: hivemetastore
```
`az aks get-credentials --resource-group TrinoRG --name trinoaks`

`kubectl apply -f aks-hm-service.yaml`

_Artifact: Deployment Yaml for Standalone Hive Metastore Image_

`vim aks-hm-deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hivemetastore
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hivemetastore
  template:
    metadata:
      labels:
        app: hivemetastore
    spec:
      nodeSelector:
        kubernetes.io/os: linux
      containers:
        - name: hivemetastore
          image: trinoregistry.azurecr.io/hivemetastore/3.1.3:v1
          ports:
            - containerPort: 9083
          resources:
            requests:
              cpu: '1'
              memory: 3G
            limits:
              cpu: '2'
              memory: 6G
```
`kubectl apply -f aks-hm-deployment.yaml`

### Step 3: Setup Trino via Helm charts


_Deploy Trino Image to Azure Container Registry_

`docker pull trinodb/trino`

`docker tag trinodb/trino trinoregistry.azurecr.io/trinodb/trino:v1`

`docker push trinoregistry.azurecr.io/trinodb/trino:v1`

Deploy Trino Helm Chart to Azure Container Registry

Since outbound internet connectivity is restricted from Azure Network, we cannot download the helm charts from GitHub or any source on the internet. To mitigate this situation, we have created a custom helm chart, please download from the link below. It has several modifications from the open-source helm chart viz. HPA configured with Memory as the metric instead of CPU, using ACR Image rather than Docker Hub Image etc. 

[trino-v2.tar.gz](/.attachments/trino-v2.tar-2e53c290-ccb3-4080-97c3-f645a0709991.gz)

Note: You will have to modify the trino-v2.tar.gz file to 

`export registryId=$(az acr show --name trinoregistry --query id --output tsv)`

`export PASSWORD="32k8Q~x8t6K~XG0b8KEVtNCwt7pXFpoIiFCukdcN"`

`export USER_NAME="a13e60f4-224e-46a1-8f27-840ce108a842"`

`az role assignment create --assignee $USER_NAME --scope $registryId --role AcrPush`

`helm registry login trinoregistry.azurecr.io --username $USER_NAME --password $PASSWORD`

`helm push trino-v2.tgz oci://trinoregistry.azurecr.io/trino-helm`

`helm upgrade --install trino oci://trinoregistry.azurecr.io/trino-helm/trino --set image.repository=trinoregistry.azurecr.io/trinodb/trino --set server.workers=3 --set server.config.query.maxMemory=4GB --set server.config.query.maxMemoryPerNode=2GB --set server.config.memory.heapHeadroomPerNode=1GB --set worker.jvm.maxHeapSize=4G --set coordinator.jvm.maxHeapSize=4G --set server.autoscaling.enabled=true --set server.autoscaling.targetRAMUtilizationAverageValue=3000Mi`

`kubectl get pods -o wide`

`kubectl get service hivemetastore -o wide`

 Please note the ClusterIP of the Metastore Service, it will be needed in the next steps

`kubectl edit configmap trino-catalog`

Make changes to following configurations under delta.properties

```
hive.metastore.uri=thrift://<ClusterIP of HiveMetastore Service>:9083
hive.azure.abfs-storage-account=<DataLake ADLS Account Name where Delta Table Resides>
hive.azure.abfs-access-key=<Access Key for the above mentioned Storage Account>
```

Note: If more than one datalake (ADLS) account please repeat the properties of "hive.azure.abfs-storage-account" and "hive.azure.abfs-access-key".

`kubectl rollout restart deploy trino-coordinator trino-worker`

### Step 4: Testing Trino Deployment

_1. Check Connectivity to Trino CLI & Its Catalogs_

`kubectl get pods -o wide`

Please note the Pod Name for Trino Coordinator, will be needed in the next step to connect to Trino CLI

`kubectl exec -it trino-coordinator-5dbcff8f8f-lbg7n -- /usr/bin/trino --debug` 

Once inside of the Trino CLI, we can quickly check for Catalogs

`SHOW CATALOGS;`

_2. Check Connectivity to Delta Catalog_

`USE delta.default;`

Trino CLI should connect to the Delta Catalog successfully

_3. Create Table in Delta Catalog_

In this example, I am using Bike Sharing dataset [link](https://archive.ics.uci.edu/ml/datasets/bike+sharing+dataset)

```
CREATE TABLE delta.default.bikeShareDelta (
	dummy bigint
)
WITH (
  location = 'abfss://datalake@dtrinodatalake.dfs.core.windows.net/bikeSharingDelta/',
  checkpoint_interval = 5
  );
```
Trino will auto-detect schema from the Delta Table if dummy schema is provided.

_4. Select Query from Delta Table_

`SELECT * FROM delta.default.bikeShareDelta LIMIT 10;`

_5. Update / Delete Query to Delta Table_

`UPDATE delta.default.bikeShareDelta SET atemp = atemp + 1 WHERE cnt > 150;`

_6. Check Delta & Query Time Travel_

Let's list all the versions of the Delta Table using History commands

`DESCRIBE HISTORY delta.default.bikeShareDelta;`

Let's understand the difference between the different version's aka Time Travel

`SELECT atemp FROM delta.default.bikeShareDelta WHERE cnt > 150 VERSION AS OF 2;`
`SELECT atemp FROM delta.default.bikeShareDelta WHERE cnt > 150 VERSION AS OF 3;`

_7. Check for HPA Autoscaling_

This check would be difficult to simulate since it requires heavy load conditions, the memory 
 configurations mentioned during helm install / upgrade etc. the essential passing score would be trino-worker pods scaling up and down per memory consumption of the pods. It typically takes 5 min to scale down.

_8. Simulate Failure of Pods_

Let's delete one of the Trino Worker pods manually to see how Kubernetes Handles the failure

`kubectl delete pod trino-worker-5dbcff8f8f-h0dt9`

Now, lets check the current status of the deployment

`kubectl get pods -o wide`

We will see the deleted pod was quickly replaced by a new pod by Kubernetes.

_9. ReScaling of Trino Cluster_

Let's increase the minimum number of worker pods from 3 to 5 and increase the memory threshold from 3000Mi to 3500Mi. 

`helm upgrade --install trino oci://trinoregistry.azurecr.io/trino-helm/trino --set image.repository=trinoregistry.azurecr.io/trinodb/trino --set server.workers=5 --set server.config.query.maxMemory=4GB --set server.config.query.maxMemoryPerNode=2GB --set server.config.memory.heapHeadroomPerNode=1GB --set worker.jvm.maxHeapSize=4G --set coordinator.jvm.maxHeapSize=4G --set server.autoscaling.enabled=true --set server.autoscaling.targetRAMUtilizationAverageValue=3500Mi`

There are several other parameters that can be tweaked for the Trino Helm Chart, the documentation can be found here [link](https://trinodb.github.io/charts/charts/trino/)


## Section III: Connecting Other Analytics Platforms to Hive External Metastore 

### Building a Holistic Analytics Ecosystem

With Trino successfully setup in the above steps, Next step was to build a Centralized Analytics Framework that can spans across multiple technologies like Azure Synapse Analytics, Azure Databricks, Azure HDInsight, Custom Spark & Hadoop Installations on Azure VMs or Azure Kubernetes Services and even On-Premises Spark & Hadoop deployments.

Please refer to the below Documentation links that mentions connecting the various analytics platform to the Hive Metastore that we created, please note we are using 3.1 version of Standalone Hive Metastore.

- Azure Synapse Analytics [link](https://learn.microsoft.com/en-us/azure/synapse-analytics/spark/apache-spark-external-metastore)
- Azure Databricks [link](https://learn.microsoft.com/en-us/azure/databricks/data/metastores/external-hive-metastore)
- Azure HDInsight [link](https://learn.microsoft.com/en-us/azure/hdinsight/hdinsight-use-external-metadata-stores)
- On-Premises Cloudera [link](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_ig_hive_metastore_configure.html#configure_mysql_db_hive_metastore)

There are of course some pros and cons to the above approach the biggest pro is we have a centralized metastore where all the tables are created and consumed across multiple platforms at the same time this feature is also the biggest con considering the governance issues, simply put who creates which tables, who edits it, who owns it, while these questions don't seem quite grave but in an enterprise setting these quickly become bottlenecks for progress.

In this particular case Trino was operated in Read-Only mode for consumption by various Business Intelligence Tools and Data Analysts using SQL IDEs to fire queries and gain insights on a huge datalake. This approach solves the data governance issues since all the development for Data Lake & Creation of Tables can be done using a single platform like Azure Synapse Analytics or Azure Databricks or relevant services. Of course, it bears mentioning that this approach while it was favorable in this use case, for other use cases the mileage may vary.

## Summary

In this tutorial, we setup Hive Metastore Server in HA Mode in AKS, Hive Metastore DB in Azure SQL, Trino Cluster with HPA on Memory Auto-scale in AKS and we connected Trino Cluster to Azure Synapse Analytics to build a holistic ecosystem. By taking advantage of the Kubernetes ecosystem, we were able to build out the Trino cluster per business requirements and demonstrate elasticity & resiliency by killing a pod.

Azure Kubernetes Services and Trino work well together and can be used for large scale deployments.
