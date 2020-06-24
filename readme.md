# Cassandra Update and Restore for Axway API-Manager v7.7.LT (rolling updates)

Prepared by: RKI, Axway DE

State: DRAFT

Initial version: 09.04.2020 / last update: 24.04.2020

## Purpose

Several Axway customers were requesting a description for a save upgrade procedure for switching onto the latest API-Managment version 7.7. The following is description on all steps we did in our test for this procedure on proper Cassandra keyspace updates and restores suitable for an installation of Axway API-Manager v7.6.4 to 7.7.LT (rolling updates).

The Axway open documentation is providing a "happy path" upgrade procedure. In contrast to this aim here is to have a procedure to ***clone*** an API-Manager installation along with its configuration database. Cloning of an environment can be particularly useful for doing save upgrades of an API-Gateway topology to conserve the current system to have a quick "way back" in case something might go wrong during the upgrade. So, going back to the initial state before starting the upgrade now doesn't include a restore but instead requires just (re)start of the old system. Out approach can minimize any downtime because of administrative tasks.

Such a way-back might be needed if the version upgrade includes changes on the API-Manager configuration database structure.
Another reason for cloning could be to create a similar installation at a geographically different location, e.g. two installations serving different parts of the world, or to create test systems that uses real data for testing in case of special errors but runs on distinct systems. Sometimes even the migration is used to *move* to other (virtual) systems.

## Considerations

Since API-Management v7.7.LT (rolling updates) releases there is a change in the Axway online documentation provided at [Axway Open Documentation](https://docs.axway.com/bundle/axway-open-docs/page/docs/index.html). Mainly the documentation structure was improved for better readability and task driven instructions.

Now it includes a more handy description on how Apache Cassandra databases need to be operated for Axway API-Manager. This is different from Apache Cassandra installations used as for *big-table* like massive datastores. In conjunction with Axway API-Manager a Cassandra database cluster is used as an **application configuration datastore**. It supports scale out of the count of application instances of API-Gateway as runtime for API-Manager. For an API-Manager cluster, its Cassandra database - more precise the used keyspace - is also used as communication layer between all API-Manager instances to replicate application state information like API usage details to apply quota limits.
The API-Manager datastore holds:
1. static configuration data
2. application configuration data (seldom changed)
3. usage counters (high update frequency)

*The cloning procedure here is considering 1) and 2) but is ignoring 3) for the upgrade process!*

For Axway API-Manager there is hard dependency between applied server configuration and message processing rules - the FED package containing server parameters and policies - and its data stored within the used Cassandra keyspace. Reason for this are the "encrypted fields" that API-Gateway is offering via its key-property-store (KPS) abstraction layer. Those encrypted fields are used as secure storage for password hashes and similar data elements.
**Field (column) encryption is not a Cassandra feature** but instead an application level function provided by the KPS layer of API-Gateway. The encryption key used for encryption within the Cassandra keyspace of an API-Manager instance is the [API Gateway configuration encryption passphrase](https://docs.axway.com/bundle/APIGateway_762_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/general_passphrase.htm).

If an *API Gateway encryption passphrase* was used to secure the API-Gateway configurations, this passphrase together with the FED package and a matching database backup (from timeframe tha passphrase was used) are needed for restoring an API-Gateway installation (e.g. disaster recovery from backups).

```
 API-MANAGER -> API-Gateway runtime -> KPS layer API-Gateway -> KPS for Apache Cassandra -> Cassandra server
|------ FED includes policies and KPS table mappings -------|                              |-- API-M data --|
|------------------------------------------- sensitive data encryption key ---------------------------------|
```

Details on *Group Passwords* and their relation to encrypted KPS fields can be found here within documentation [API-Gateway v7.6.2 KPS](https://docs.axway.com/bundle/APIGateway_762_AdministratorGuide_allOS_en_HTML5/page/Content/AdminGuideTopics/general_passphrase.htm).

### What is KPS?

The KPS layer of API-Gateway is a feature that is used to abstract API-Gateway from several data stores with different capabilities. For some of the currently supported storages systems it is necessary to provide detailed information on the data structure (e.g. plain files were supported in the past). Otherwise, on application level it would not be unknown how to access the needed data or serialize data on a data store.

API-Gateway is using the KPS structure information deployed to it via FED packages to manage the underling storage system (e.g. creating, updating database tables). KPS storage config is separated into *KPS collections*. For Cassandra storage one collection is keeping several *tables*. Each collection is referring to its underlying storage system. The API-Manager component of API-Gateway has a hard dependency on Cassandra as KPS storage for its scale-out capability.
Each KPS within a Cassandra keyspace must be configured either manually or by running Axway provided installer scripts, which add the needed KPS definitions.

For more datails on KPS please consult [KPS FAQ](https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_policydev/apigw_kps/kps_faq/index.html).

The Axway scripts are part of an API-Gateway + API-Manager installation. The files are located at:
```
<install-dir>/apigateway/posix/bin
```

## Cloning an API-Manager

Open Docs for API-Manager includes a chapter on how to administrate the Apache Cassandra database which is delivered along with an Axway API-Manager installation. Documentation reference: [Administer Apache Cassandra](https://docs.axway.com/bundle/axway-open-docs/page/docs/cass_admin/index.html). Especially the chapter [Apache Cassandra backup and restore](https://docs.axway.com/bundle/axway-open-docs/page/docs/cass_admin/cassandra_bur/index.html) is important to ensure correct backups for disaster recovery.

For cloning an API-Manager installation we don't want to rely on Cassandra backup and restore mechanics as this is binding us to the Cassandra topology and version used as source environment. In case of a Cassandra multi-node cluster it seems to be more complex than the API-Gateway provided KPS backup and restore tooling, which is part of each API-Gateway installation.

Axway API-Manager is using Cassandra keyspace differently than big-data applications typically are using a Cassandra database. API-Manager has focus on consistent and available data. Therefore API-Manager typically uses keyspaces with replication factore of 3 with each node storing 100% of all data. In contrast a big-data application will hold 3 copies of each data element spread accross a larger cluster.
So, the amount of data managed by Cassandra for API-Manager is very small. Usually its size is up to 1GB, perhaps slightly more. Cassandra is used here for its replication and consistency mechanisms to support scaling out an API-Manager cluster accross more servers for higher load. For details see [Capacity planning and performance](https://docs.axway.com/bundle/axway-open-docs/page/docs/apimanager_capacityguide/index.html).

Because the amount of data is not very big we can do an backup/restore in a "classic backup approach" by simply exporting the whole data-set at once on one of the API-Gateway nodes of the same API-Gateway configuration-group (all API-Gateway instances hosting API-Manager). Axway is providing a tool for this application layer data management tasks: **kpsadmin** - [Manage KPS using the kpsadmin tool](https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_policydev/apigw_kps/how_to_use_kpsadmin_command/index.html)

### API-Gateway Configuration Backup

As stated above, it is important to know the *API Gateway encryption passphrase* if it was used!

First step should be saving the currently running FED package from the API-Gateway group that hosts API-Manager. Following Admin-Node-Manager (ANM) API call can be used to list the current API-Gateway topology.

The administrative API of Admin-Node-Manager is documented as [API Gateway API v1.0](http://apidocs.axway.com/swagger-ui/index.html?productname=apigateway&productversion=7.7.0&filename=api-gateway-swagger.json#/)

```
# retrieving topology description
# the sample only has "instance-1" gateway running within "group-2"
#
curl -k -u "admin:changeme" https://localhost:8090/api/deployment/domain/deployments | python -m json.tool

{
    "result": {
        "group-2": {
            "instance-1": {
                "deploymentTimestamp": 1586366948256,
                "environmentProperties": {
                    "Description": "Original factory configuration (Environment)",
                    "Manifest-Version": "1.0",
                    "Name": "Default factory configuration for API Gateway (Environment)",
                    "Version": "v1 (Environment)",
                    "VersionComment": "Original factory configuration (Environment)"
                },
                "policyProperties": {
                    "ChangedBy": "admin",
                    "Description": "The QuickStart Server configuration policies",
                    "Manifest-Version": "1.0",
                    "Name": "QuickStart Server Configuration",
                    "Version": "v1",
                    "VersionComment": "Imported API Manager configuration"
                },
                "rootProperties": {
                    "Id": "842d18d5-7498-4286-b136-ae512923253b",
                    "Timestamp": "1586366871502"
                },
                "status": "Loaded",
                "user": "admin"
            }
        }
}
```

Next, one can retrieve the currently running configuration (FED archive) from one of the API-Gateway instances of the API-Manager gateway group, either by API call or via Policy Studio in interactive mode.

```
# sample call to query currently running FED package from "instance-1"
#
curl -k -u "admin:changeme" https://localhost:8090/api/deployment/archive/environment/service/instance-1 | python -m json.tool > instance-1-config.txt

less instance-1-config.txt
{
    "result": {
        "data": "<base64-fed-archive>",
        "rootProperties": {
            "Description": "Original factory configuration (Environment)",
            "Id": "cfde737f-0362-4d55-9de3-71f6dc6f8fee",
            "Manifest-Version": "1.0",
            "Name": "Default factory configuration for API Gateway (Environment)",
            "Timestamp": "1586884449546",
            "Version": "v1 (Environment)",
            "VersionComment": "Original factory configuration (Environment)"
        }
    }
}

# requesting and saving currently running FED package
# 1) get deployment archive ID
curl -k -u "admin:changeme" https://localhost:8090/api/router/service/instance-1/api/configuration/archiveId

{ 
    "result": "842d18d5-7498-4286-b136-ae512923253b"
}

# 2) receive deployment archive
curl -k -u "admin:changeme" https://localhost:8090/api/deployment/archive/group-2/842d18d5-7498-4286-b136-ae512923253b | python -c 'import sys, json; print json.load(sys.stdin)["result"]["data"]' | base64 -d > instance-1-config.fed
```

### Exporting KPS Collections and Tables from Running FED

In order to clone we need the KPS definitions of the configuration group that is currenly running. These definitions are needed for restoring data into a new Cassandra keyspace in a later step.

1) open Policy Studio
2) *New Project*, name (e.g. Original-FED)
3) chose Project starting point *"From .fed file"*
4) navigate to *Environment Configuration -> Key Property Stores*
5) export KPS configuration: right click and chose *Export all Key Property Stores*; save with name, e.g. KPS-definitions.xml
6) close current policy package project (File -> Close Project)

**Hint:** *The export now containes only KPS definitions without any policies or server setings. Those details describe the table structure within Cassandra and for the KPS access layer. The data definitions are needed for the export/import to access the Cassandra data store (keyspace). During deployment of an API-Gateway FED package any changes on KPS settings are replicated onto the configured data store (chosen keyspace). Hence, the API-Gateway will create or update the data definitions on the Cassandra database cluster accordingly!*

### Creating new target Cassandra keyspace

For cloning we now need a new target keyspace at the target Cassandra database cluster. As this will be a new keyspace on an existing Cassandra cluster it should be created similar to the original one.

Creation can be done via Cassandra management tool (cqlsh) or via an API-Gateway FED deployment. For a FED deployment the first API-Gateway that received configuration for a Cassandra keyspace will use its confiured user to create the needed keyspace if it is not already available. If the target keyspace already exists the API-Gateways of the deployment group will not change the existing keyspace configuration!

In order to have full control on this operation Cassandra tooling seems to be the better option. Please make sure the configuration matches the setup of your Cassandra cluster configuration. For more details please consult [Configure a highly available Cassandra cluster](https://docs.axway.com/bundle/axway-open-docs/page/docs/cass_admin/cassandra_config/index.html).

**Hint:** For the special runtime platform *Amazone AWS* third party developers have compiled more detailed descriptions on how to adopt Apache Cassandra on AWS resources for optimum performance and recilience.
+ Cassandra on AWS EC2: [Best Practices for Running Apache Cassandra on Amazon EC2](https://aws.amazon.com/blogs/big-data/best-practices-for-running-apache-cassandra-on-amazon-ec2/)
+ Cassandra K8s managed on AWS: [Deploy a highly-available Cassandra cluster in AWS using Kubernetes](https://medium.com/merapar/deploy-a-high-available-cassandra-cluster-in-aws-using-kubernetes-bd8ba07bfcdd)

*Axway Services DE is currently evaluation both options for recomandation and applicaltion samples customers can adopt. In its current state the referenced descriptions are not yet "production ready" because of security and static initial configurations. Nevertheless, the approach is right for modern system operations, auto scaling and auto healing.*

**Sample for replication option *SimpleStrategy*:**

Assumption: Cassandra cluster of 3 nodes. The underlying infrastructure is responsible for resilience. (e.g. AWS region with 3 availablilty zones - one node per AZ)
```
CREATE KEYSPACE apimanagerv77 WITH replication = {'class': 'SimpleStrategy', 'replication_factor': '3'}  AND durable_writes = true;
```

**Hint:** *According to external informations on the web for [Cassandra keyspace configuration](https://www.guru99.com/cassandra-keyspace.html) - for topologies typically used on AWS for API-Manager the replication option 'Simple Strategy' could be sufficient within same AWS region with 3 availablity zones.*

**Sample for replication option *NetworkTopologyStrategy*:**

Assumption: Cassandra cluster consits of 3 nodes. One node per data center; at least two DC's are always available for resilience. The Cassandra nodes are spread accros DC's and configured appropriately.

```
CREATE KEYSPACE apimgr_v77 WITH replication = {'class': 'NetworkTopologyStrategy', 'DC1': '1', 'DC2': '1', 'DC3': '1'}  AND durable_writes = true
```

Configuration details shold be read as: each data row must be stored 3 times; one data copy must be place on a node belonging to DC1, one copy within DC2 and another one in DC3. The data center names must be configured at the Cassandra node configuration as "DC1" to "DC3" within Cassandra config file *<install dir>/cassandra/conf/cassandra-rackdc.properties*.

**Attention:** *In case your API-Gateway setup uses policies with the filter "Throttling" and the rate limiting Algorithm is configure to "Smooth Rate Limiting" an appropriate Cassandra keyspace must be properly configured! (Policy Studio: Server Settings - Cassandra - Throttling; this keyspace config is only for Throttling; it creates an additonal table "throttling_table" in that keyspace; a dedicated keyspace allows to define different Cassandra distribution settings, tuned for throttling data)*

*See [Configure rate limiting](https://docs.axway.com/bundle/APIGateway_762_PolicyDevGuide_allOS_en_HTML5/page/Content/PolicyDevTopics/general_rate_limiting.htm) within the docs for more details.
**API-Manager** itself is **NOT** relying on this feature. It uses its own quota tracking for rate limiting.*

### Export all data from currently used Cassandra keyspace

Documentation for *kpsadmin* tool is available at [Manage KPS using kpsadmin](https://docs.axway.com/bundle/axway-open-docs/page/docs/apim_policydev/apigw_kps/how_to_use_kpsadmin_command/index.html).

**Create full backup in interactive mode**

The *kpsadmin* tool must be started on one of the hosts API-Gateway is installed on. The installation must be part of the same API gateway domain, as the contact point of the ANM API is read from *<install-dir>/groups/topology.json*. The tool has no parameters to explicetly provide the ANM API endpoint details (host IP + port).

```
<installdir>/apigateway/posix/bin/kpsadmin --username admin --password <secret>
...
    Group Administration:
        25) Clear All
        26) Backup All
        27) Restore All
        28) Re-encrypt All
        29) Group Details

Enter selection: 26
...
Select a Group:
1) CassTestGrp 
2) QuickStart Group 
Enter selection [1]: 2

Select an API Gateway:
1) QuickStart Server 
Enter selection [1]: 1

Select option: 26

This operation makes a backup of the following tables to backup file on the KPS API Gateway.
The backup files are placed in: apigateway/groups/<group-id>/<instance-id>/conf/kps/backup
A unique backup UUID is prepended to each file.
You will need this UUID and the backup files to do a restore.
        API Portal_ApiAppPolicyBindings
        ...
        QuickStart_HeroesCharactersRegistry
Are you sure you wish to continue y/n [n]: y

Backup uuid is: f2c2d44b-d745-45e1-a516-315e0e7781c7
Started backup
Starting backup of table: API Portal_ApiAppPolicyBindings to: f2c2d44b_d745_45e1_a516_315e0e7781c7_api_portal_apiapppolicybindings_json ...
Backup done in 0:00:01 seconds - 200
...
```

The resulting backup files are stored on the server that is hosting the API-Gateway instance which war chosen for the backup operation by *kpsadmin*. File location is *<installdir>/apigateway/groups/group-<n>/instance-<n>/conf/kps/backup/*. All files with the same UUID belong to the backup set. Sizing can be checked using:

```
$ du -h <install-dir>/apigateway/groups/group-2/instance-1/conf/kps/backup/
19M	/opt/Axway/APIM/apigateway/groups/group-2/instance-1/conf/kps/backup/

$ du -h <install-dir>/apigateway/groups/group-2/instance-1/conf/kps/backup/f2c2d44b_d745_45e1_a516_315e0e7781c7_*
```

**Create full backup in scriptable command mode**

```
backupID=$(uuidgen)
echo "BACKUP FILE SET uses UUID=${backupID}"
<install-dir>/apigateway/posix/bin/kpsadmin --username admin --password <secret> --group "QuickStart Group" --name "QuickStart Server" --uuid=${backupID} backup
```

**Hint:** Names of the API-Gateway group and the target API-Gateway can be found on API-Gateway Manager UI Dashboard or *<install-dir>/groups/topology.json*.

### Prepare the new Cassandra keyspace for KPS import

Before importing the backup data set into another Cassandra keyspace we need to provision a new/another API gateway group with at least one API-Gateway. This is needed as *kpsadmin* tool is using this API-Gateway as "worker" to import the backup data set.

#### Creating new API-Gateway Group

3 options for "classic":
- API-Gateway Manager Web-UI
- Policy Studio
- managedomain

#### Import KPS definitions into new API-Gateway configuration

...tbd...

### Import KPS backup data set into Cassandra keyspace

- copy data over
- run importer
...tbd...
- optional check tables

### Switch from original Environment to Cloned environment

...tbd...


## Upgrade using cloned API-Manager environment

(follow the upgrade procedure layed out in docs)
...tbd...
(switch back to source environemnt in case of any error)

## Conclusion

For cloning and correct backup for a disaster recovery it is important to understand the KPS layer of API-Gateway and the application level feature of API-Manager with its tight coupling to Apache Cassandra. API-Manager Backups must consist of:

1) API-Gateway configuration and policy deployment packaged (FED)
2) API-Gateway group passphrase
3) Cassandra keyspace backup

Following the procedure outlined above we are able to completely clone an environment. This allows for a faster, more simple switch between major versions. In case of breaking changes are applied at the database structure on Cassandra keyspace it still allowes to quickly switch back to the original setup in case anything should go wrong during such version upgrade. This switch just requires shutdown of the faulty installation and restart of the original environment.


## Axway API-Management Client Tools

At some customer environments installing the Axway API-Management on a Windows workstation can be tricky because of centrally managed workstation environments with special rules set and centralized software distribution in place.

Right now the Axway API-Management client tools *Policy Studio* and *Configuration Studio* are only provided as Installer packages for the Windows platform. If those cannot be run on a developer workstation one can instruct the installer to run as normal user. For this installer package this is possible as it just needs ti unpack the Eclipse based client applications from the installer archive but no further Windows system modifications like accessing the registry are required.

1) Download the installer from the Axway support portal
2) Open a command line.
3) Set the environment variable "__COMPAT_LAYER" to "RunAsInvoker"<br/>
```> set __COMPAT_LAYER=RunAsInvoker```
4) Start the installer executable and install the packages.

More details on this approach and the security implications can be found here [What does '__COMPAT_LAYER' actually do?](https://stackoverflow.com/questions/37878185/what-does-compat-layer-actually-do#:~:text=Setting%20__COMPAT_LAYER%20to%20RunAsInvoker,as%20whatever%20user%20called%20it.). - *last visited: 24th June 2020*


## Other tools for Cassandra Backup / Restore handling

Medusa is an Apache Cassandra backup system developed by TLP (now DataStax).
Available for free from: https://github.com/thelastpickle/cassandra-medusa
Nice feature is the integration with cloud storage from AWS or Google as backup location.

...tbd...

RKI: Worth further testing!

Apache Cassandra at IaaS cloud providers?
-----------------------------------------
AWS: Best Practices for Running Apache Cassandra on Amazon EC2
https://aws.amazon.com/blogs/big-data/best-practices-for-running-apache-cassandra-on-amazon-ec2/

This approach allows an very easy approach for upgrading Apache Cassandra software without service interruption. But requires setup of Cassandra for allowing it (second network interface). Applying rolling updates becomes more easy this way.

*RKI: I'm not yet happy with the Apache Cassandra within a container articles I have seen so far.*
* A good stating point here us a TLP blog post: Docker Meet Cassandra. Cassandra Meet Docker. -> https://thelastpickle.com/blog/2018/01/23/docker-meet-cassandra.html
* Presentation on SlideShare: Running Apache Cassandra on Docker -> https://www.slideshare.net/JimHatcher/running-apache-cassandra-on-docker

