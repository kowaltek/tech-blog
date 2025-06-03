+++
title = 'DVA C02'
date = 2025-06-03T12:16:04+02:00
draft = false
+++

# AWS CLI

Default pagination uses page size of 1000 records.
So when no pagination arguments are specified, the command will repeat requests using that page size.
`--page-size` allows page size customization.
`--max-size` sets a limit on the total result records number.
Usually when we see error `timed out` it means the page size is too big for one request to be completed within set time window.

# AWS RDS

Relational database service. Included in free tier.

Advantages:

 - automated provisioning, OS patching
 - continuous backups and restore to specific timestamp (Point In Time restore)
 - monitoring dashboards
 - read replicas for improved read performance
 - multi AZ setup for DR (Disaster Recovery)
 - maintenance windows for upgrades
 - scaling capability (vertical and horizontal)
 - storage backed by EBS (gp2 or io1)
 - storage grows automatically up to 64TB by 10 GB increments

Disadvantages:

 - can't SSH into instances

## Amazon Aurora

Proprietary technology from AWS. Both Postgres and MySQL compatibility is supported by Aurora. Aurora is cloud optimized and claims 5x performance improvement over MySQL on RDS, and over 3x the performance of Postgres on RDS. Aurora storage automatically grows in increments of 10GB, up to 64TB. Aurora costs more than RDS (20% more) - but is more efficient. Aurora is a great choice if using MySQL or Postgres today. Aurora is not included in free tier.

## RDS deployment types

### Read replicas

Scales the read workload of DB. You can create up to 15 read replicas.

### Multi AZ

Failover in case of AZ failure, AWS handles failover automatically. In the event of a planned database maintenance, instance failure, or AZ failure, Amazon RDS will automatically failover to the up-to-date secondary replica. Failover times are typically 60-120 seconds. You can only have one AZ as failover.

### Multi region (read replicas)

Disaster recovery. You can have read replicas in different regions and one primary DB in one region, used for writing. You can also promote a read replica to be a primary DB. We need to be aware of the replication costs as the data is transferred across network.

## Disaster recovery

Two ways to backup the data: snapshots and automated backups.
In both options, the restored database will be a different one, ie.
the endpoints will change.

### Snapshots

These are manual.
Creates a snapshot of a storage volume attached to the database instance.
There's no retention period for snapshots.
They are persisted even if the cluster is removed.
Pretty useful before big changes to a database.
You can copy a snapshot to another region.
You can also share a snapshot with another AWS account.
To do so, the owner of the snapshot must give the other account explicit create volume permissions.

### Automated backups

Enabled by default.
Create daily snapshots during a defined window and stores database transaction logs.
Transaction logs (WAL) are used to replay transactions.
These also enable a point-in-time recovery within the retention period (1-35 days).
During the recovery, first the snapshot is restored and then the appropriate WAL is replayed to a defined recovery point.
Backup data is stored in S3.
During the initialization of backup, the storage IO might be suspended for a moment.

## Encryption

Uses AES-256 and KMS to encrypt all the data at rest if selected during creation.
Encryption includes all underlying storage, automated backups, snapshots, logs and read replicas.
Encryption can not be enabled on unecrypted database.
The way to create an encrypted version of a database is to create unencrypted snapshot, then created encrypted one based on it, and finally create a new database based on the encrypted snapshot.

## RDS Proxy

It's serverless and scales automatically.
Preserves application connections during database failover.
Deployable over multiple AZs.
Up to 66% faster failover times.


## Connecting from Glue to Aurora Postgres

When connecting from Glue keep in mind, that the JDBC driver used by Glue supports only MD5 password encryption.
This makes it problematic to connect to postgresql v14 and later which by default have scram-sha-256 method enabled.
To solve this, the cluster has to have `password_encryption` parameter changed to `md5` in it's parameter set and the user used for Glue connection has to be created after that change (to make sure the password gets hashed using `md5`).
To keep other users secure, we can still specify `scram-sha-256` mechanism when creating their passwords (when attempting to connect with such user, the cluster will switch to `scram-sha-256`).

# Elasticache

It's an in-memory key-value store.
Used for improving database performance.
Also for storing session data for distributed apps.
Typical use case is in load-heavy scenarios with lots of reads and infrequent updates to data.
It's not the best solution in heavy write load (in which case we should scale the database itself) and in OLAP processing (use database dedicate to that workload - redshift).

Two types of elasticache:
 - memcached,
 - redis.

### Memcached

Scales horizontally, but there's no persistence, multi-AZ or failover.
It's a good choice for basic caching.

### Redis

More sophisticated solution with persistence, multi-AZ deployments and failovers.
Supports sorting and ranking data.
Supports complex data types like lists or hashmaps.



# MemoryDB for Redis

Massively scalable in-memory database, that scales above 100TB.
It's highly available (multi-AZ).
Records transaction log for recovery and durability.
Can be used as a database.
It's ultra fast - single digit millisecond write latency and microsecond read latency.
Suitable for low-latency applications.

# Systems Manager

## Parameter Store

Allows for centrally storing values used for configuring our apps.
The values stored can be encrypted using AWS KMS.
Integrates with many other AWS services.

# EC2

### EBS (Elastic Block Store)

Highly available - they are automatically replicated within one AZ to protect against hardware failures.
Scalable - the type or size can be changed dynamically without any downtime or impact on performance.

EBS types
 - gp2 (general purpose ssd) have good balance of performance and price (boot disks and general apps),
 - gp3 are the latest version of gp and are 20% cheaper than gp2 (boot disks and general apps),
 - io1 (provisioned IOPS ssd) are used for IO intensive apps (OLTP databases, latency sensitive apps),
 - io2 latest generation of provisioned IOPS with higher durability (99.999% - highest available) and more IOPS at the same price as io1 (OLTP databases, latency sensitive apps),
 - io2 Block Experess (SAN - storage area network) with hardware that uses EBS Block Express architecture, it supports 4x the throughput, IOPS and capacity of the io2, great for large and mission critical applications (SAP HANA, Oracle, SQL Server),
 - st1 (throughput optimized hdd) is low cost, good for frequently-accessed, throughput intensive use-cases, like data warehouse, etl, log processing, but it cannot be a boot volume,
 - sc1 (cold hdd) lowest cost option, also cannot be a boot volume.

IOPS measures the number of read and write operations per second.
Important for low latency (usually transactional systems).

Throughput is measured in Mb/s and is important when dealing with non-latency sensitive operations on large amounts of data.
Workloads where throughput matters are usually analytical.

Good summary in docs: [ebs volume types](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html).

The EBS volume can only be attached to an EC2 instance within the same AZ.

If encryption by default is enabled in account, it's impossible to create an unencrypted volume.

We can create only encrypted volume from encrypted snapshot.

### Elastic Load Balancer

Four types:
 - application load balancer (HTTP and HTTPS),
 - network load balancer (TCP - high performance),
 - classic load balancer (HTTP, HTTPS, TCP - legacy option),
 - gateway load balancer (allows to load balance traffic to third party appliences running on AWS, like virtual appliences purchased on AWS Marketplace, virtual firewalls).

Aplication load balancer can route the requests based on HTTP headers.

Network load balancer handles milions of requests per second, but is most expensive.

Classic load balancer supports `x-forwarded-for` headers and sticky sessions.

`x-forwarded-for` allows to pass information about the original client's IP address that made the request.

HTTP status 504 (gateway timeout) usually means when the application fails to respond (wrong configuration, application error, etc.).

### EC2 Image Builder

Allows to create virtual machine images (AMIs) and docker containers.
Automates the process of creation and maintenance of images.
It also allows for testing of the images and distributing them.

Image pipeline defines the configuration and end-to-end process of building images.
Includes image recipe, distribution and testing.

Image recipe can be version controlled.
It consists of base image and build components.

To use the AMI in different region, it has to be copied there.

# S3

Storage size and number of objects are unlimited.

### Buckets

Buckets need to have globally unique names (through all regions and accounts).
They are defined at specific regions though.
There's a naming convention to them:
 - no uppercase,
 - no underscore,
 - 3-63 characters long,
 - not an IP,
 - must start with lowercase letter or number,
 - must not start with prefix `xn--`,
 - must not end with suffix `-s3alias`.

Buckets have global scope, but physically they are created in only one selected region.

### Objects

Objects have a key, which is the full path of the file, eg. `s3://mybucket/myfolder/myfile.txt`.
So the key contains a prefix and object name.
Max object size is 5TB.
For uploads greater than 5GB one has to use `multi-part upload`.
Successful upload response is 200.
We can attach metadata and tags to the object and, if versioning is enabled, also versionID.

An object consists of:

 - key,
 - value,
 - metadata,
 - version id.

### Security

By default all newly created buckets are private and only the owner has access to it.

 1. User-based (IAM policies)
 2. Resource-based:
     - bucket policies (apply only to the whole bucket)
     - bucket ACLs (apply to objects)

ACLs define the accounts or groups which have access and the type of the access allowed.

There's an option to block public access to the S3 bucket, which overrides any other permissions granted.

An option exists to enable S3 bucket access logs.
Logs can be written to another S3 bucket.

### Versioning

Works at bucket level. Important: any file that is not versioned prior to enabling versioning will have version 'null'. Suspending versioning does not delete previously saved versions. We are able to delete specific versions of objects if the `Show versions` toggle is on, otherwise we are putting delete marker on the object. If there's a delete marker on an object, then even if there are some previous versions available, S3 will return 404 when trying to access it.

### Replication

 - CRR (cross region replication)
 - SRR (same region replication)

Only works when versioning is enabled.

### Storage classes

1. Standard (general purpose) - has high availability and durability.
Data spans multiple (>=3) AZs.
Suitable for most workloads.
2. Standard - Infrequent Access (IA) - used for data accessed less frequently, but still requires low latency (backups).
Data spans >=3 AZs.
Minimum storage duration is 30 days.
3. One Zone - Infrequent Access - costs 20% less than standard IA.
Great for long lived, non-critical data.
Stored in only one AZ.
In case the AZ goes down, there's a possibility of data loss.
Minimum storage duration is 30 days.
4. Glacier Instant Retreival - millisecond retrieval time, for data accessed approx. once a quarter.
5. Glacier Flexible Retreival - long term data archiving, retrieval in hours or minutes.
6. Glacier Deep Archive - retrieval time of 12 hours.
Minimum storage duration is 120 days.
7. Intelligent Tiering - costs 0.0025$ per 1000 objects monthly.

All Glacier options are very cheap.
Store data in >=3 AZs.
Each data access incurs a fee.
Used only for archiving data.
Instant and Flexible retrieval have minimum storage duration of 90 days.

We can change object's storage class manually or using S3 Lifecycle configurations.

##### S3 Durability

11 9's of durability of objects across multiple AZs (when storing 10.000.000 objects in S3, there's one lost every 10.000 years).
It's the same for all storage classes.

##### S3 Availability

Between 99.95% and 99.99%.
Varies depending on storage class. Eg. S3 Standard has 4 9's of availability (not available for 53min a year).

### Encryption

There are three types of encryption for S3:

 - In transit (TLS)
 - Server side (enabled by default)
 - Client side (objects are encrypted before being sent to S3)

There are three types of keys we can use for server side encryption:

 - SSE-S3 (key managed by S3 - AES 256)
 - SSE-KMS (key managed by KMS)
 - SSE-C (key managed by the client outside of AWS)

It's possible to enforce server side encryption of all objects using a bucket policy.
We deny all requests made without the `x-amz-server-side-encryption` header.
We can also enforce encryption in transit by denying requests not using `aws:SecureTransport`.

### S3 notifications

We can set notifications to be sent on S3 events. notifications can be sent to lambda, sqs, sns or event bridge.

Prefixes used in notifications shouldn't start with `/` character for first folder. Many characters - including `=` require special handling and have different behaviour when setting from console than when setting using cdk for example.
The way to encode such characters when setting prefix or suffix with cdk is to do it manually.

### AWS Snow family

Snow family provides a way for moving large amounts of data in and out of the cloud by using physical devices.

##### Snowball Edge

Paid per transfer job. Used for moving TBs and PBs of data.

 - Snowball Edge Storage Optimized (80TB HDD capacity)
 - Snowball Edge Compute Optimized (42TB HDD or 28TB NVMe capacity)

##### Snowcone

Small, portable, rugged and secure device. Requires to use your own batteries and cables. Has to be sent offline or the AWS DataSync can be used.

 - Snowcone (8TB HDD storage)
 - Snowcone SSD (14TB SSD storage)

##### Snowmobile

Actual truck. Used to move exabytes. One truck can take 100PB of data. Worth using above 10PB.

##### Edge computing

For edge computing AWS makes available following options:

 - Snowcone and Snowcone SSD (smaller options)
 - Snowball Edge Compute Optimized
 - Snowball Edge Storage Optimized

All of these options can run EC2 instances and AWS Lambda Functions (using AWS IoT Greengrass).

##### AWS OpsHub

Software for managing snow family devices.

##### AWS Storage Gateway

Allows for seamless usage of S3 when using hybrid cloud.

# Cloudformation

Template is created using JSON or YAML.
It has to be uploaded to S3.

Sections of template:

 - AWSTemplateFormatVersion,
 - Description,
 - Metadata,
 - Parameters,
 - Conditions,
 - Mappings,
 - Transform (it is used for referencing additional code stored in S3),
 - Resources (the only mandatory section),
 - Outputs.

# AWS CDK

For development purposes use `cdk deploy --hotswap` - this skips hitting cloud formation API and instead hits APIs for changed resources directly. Saves a lot of time. Even better than that is to use `cdk watch` which uses `--hotswap` flag by default. To disable it pass `--no-hotswap` flag.

For typescript lambdas remember to run `npm run build` to compile TS files.

To further increase compilation speed add `"skipLibCheck": true,` line to `tsconfig.json`. For simple CDK stack with lambda it goes from 21s to 8s.

## Dynamic config

Create 'ssm seeder' - it's a stack which creates all PS values for the user to fill them up.
For example create a nested loop - one level for the environments, second level for the parameter names.

# AWS code services

## AWS CodeDeploy

Works with EC2 and on-premises servers.
It's a hybrid service (cloud and on-premises).
Needs CodeDeploy agent installed on the instances.
Can allow for graduall move from on-premises to cloud.

There are two types of deployments supported: rolling and blue-green.

Rolling update means stopping, updating and restarting app on instances one by one.
(Manually configure load balancer not to send traffic to stopped ones?)
This means reduced capacity for the time of deployment.
Reverting changes means redeploying.

In blue/green deployments the 'green' refers to new instances with new version of the app.
The new revision of the app is being run on new infra, so there's no reduced capacity.
Once all traffic is directed to new version of the app, the 'blue' infra is retired.
Until we don't remove 'blue' version, we can easily roll-back.

#### AppSpec file

For EC2 and on-prem deployments can be written only in yaml.
For lambda json is also supported.

AppSpec file sections:

 - version,
 - os (linux, windows)
 - files (info on files needed for deployment, where to copy them from and to)
 - hooks (scripts that run during deployments)

Typical setup for directory structure:

 - appspec.yml
 - Scripts
 - Config
 - Source

`appspec.yml` has to be placed in the root directory of the Revision.
Revision can relate to the Elastic Beanstalk.

#### Hooks

The lifecycle hooks run in a specific order (Run Order).
Phases of the lifecycle hooks:

 1. De-register instances from load balancer:

  - BeforeBlockTraffic
  - BlockTraffic
  - AfterBlockTraffic

 2. Actual application deployment:

  - ApplicationStop
  - DownloadBundle
  - BeforeInstall
  - Install
  - AfterInstall
  - ApplicationStart
  - ValidateService

 3. Re-register instances with load balancer:

  - BeforeAllowTraffic
  - AllowTraffic
  - AfterAllowTraffic

## AWS CodeCommit

Git-based repository.
Can be integrated with CodePipeline and CodeBuild.
Can be used with Git CLI, Eclipse, Visual Studio, etc.

## AWS CodeBuild

Fully managed build service.
Can be integrated with CodePipeline.
Can be used with GitHub, CodeCommit, BitBucket, etc.

## AWS CodePipeline

Continuous delivery service.
Can be integrated with CodeCommit, CodeBuild, CodeDeploy, etc.
Orchestrates the build, test and deploy phases of the software delivery process.

## AWS CodeArtifact

Fully managed artifact repository.
Can be integrated with CodePipeline, CodeBuild, etc.
Alternative to setting up our own artifact management system (eg. package manager on EC2).
Allows to set up common dependency management tools like npm, pip, Maven, Gradle as managed service.

At top level it consists of domains.
A domain can have an external connection to enable pulling of packages from public repositories.
To set up such external connection, we first need to create an upstream repository within domain.
Then we can set up the external connection for the upstream repository with something like:
`aws codeartifact associate-external-connection --domain <domain_name> --repository <upstream_repository> --external-connection <eg. "public:npmjs">`.

## AWS CodeStar

Provides a unified user interface, enabling us to easily manage software development activities in one place.
Can be integrated with CodeCommit, CodeBuild, CodeDeploy, CodePipeline, etc.

## AWS Cloud9

Cloud-based IDE.

# AWS ECS

Elastic Container Service, used to launch docker containers on AWS.
You are responsible for provisioning EC2 instances.
Has integrations with Application Load Balancer.
AWS takes care of starting and stopping containers.

# Fargate

Doesn't require manual provisioning of EC2 instances.
We just specify CPU and memory requirements.

# ECR

Elastic Container Registry.
Used by ECS and Fargate.

# AWS lambda

Virtual functions running on demand. They are limited by the time of execution and the memory used. Payment model is per request and compute time, with very generous free tier. Easy monitoring through CloudWatch. There is a possibility to run docker images on lambda as Lambda Docker Images (the image has to implement Lambda Runtime API). Max invocation time 15 minutes.

### Invocation types

Lambda functions can be invoked either synchronously (eg. by API Gateway) or asynchronously (eg. S3 events).

### Error handling

By default if lambda function returns an error, it retries 2 times.
First retry is after one minute, second is two minutes after the last attempt.

Another strategy is to create a DLQ or using lambda destinations,
and by creating eather SQS queue, SNS topic, another lambda function.
Successful invocations can also trigger event bridge events.
DLQ supports only SQS and SNS, and also messages stored do not contain any useful data.
We can specify lambda destinations and send successfuly processed invocation records to one destination and failed ones to another.

### Lambda function packaging

For deployment packages above 50MB, uploading from local machine directly is impossible.
We need to upload the package to S3 first (in the region, where we want to invoke the function).

### Performance tuning

Increasing memory implicitly increases CPU capacity.

Three factors that contribute to latency:

 - amount of code during initialization,
 - function package size (libraries and lambda layers),
 - anything that requires network calls to initialize (clients, services).

### Using TS in lambda

AWS Lambdas when using CDK use either esbuild for bundling or do it in docker container. There is an edge case for using decorators - we need to emit some metadata - and there are 2 solutions for it:
 - we can precompile code with tsc and then bundle it:
```
properties: {
    ...
    bundling: {
        preCompilation: true,
        esbuildArgs: {
            '--resolve-extensions': '.js',
        },
    },
}
```
This has a drawback though - when cdk invokes tsc it passes all it's config as CLI args instead of file, which means we can't use any plugins (eg. for path transformations).
 - we can use following config for esbuild:
```
properties: {
    ...
    bundling: {
        esbuildArgs: {
            '--supported:decorators': 'true',
            '--alias:@models': './src/graphql-api/models/',
            '--alias:@services': './src/graphql-api/services/',
        },
    },
}
```

### Lambda storage options

Points `1` and `2` are ephemeral, `3` and `4` persistent.

1. `/tmp`

Provided in the execution environment.
By default limited to 512MB, but configurable up to 10GB.
It's like a cached file system, accessible to multiple invocations of the function,
that share the same execution environment.
Available for the lifetime of the execution environment.

2. Lambda layers

By moving some of the libraries, especially the large ones,
to lambda layer, we can minimize the deployment size, which reduces deployment times.
Up to 50MB zipped and 250MB unzipped.

3. S3

Allows persisting objects.
One important limitation is that we cannot append data to objects,
we need to create new versions of them.

4. EFS

Virtual file system.
It is mounted by the execution environment and then shared by the invocations.
To use EFS, the lambda function must be within the same VPC.

# AWS Batch

Used for running batch jobs. Dynamically launches EC2 instances or spot instances. Batch jobs are defined as docker images and run on ECS.

# Lambda vs Batch

Lambda is time constrained. Lambda provides only a few runtimes, whereas anything can be run on Batch as long as it can be packaged as a docker image. Lambda provides restricted disc space. Batch uses EBS and instance store of the instances provisioned.

## Problems using websockets with lambda and gateway services

It is technically possible to create a websocket api gateway, handle the required routes (connect, disconnect and default) and then send data to all subscribed clients using WS gateways API, but there are a lot of inconveniences related to it.
First one is lack of many of the out-of-the-box authorizers (there's only Iam and lambda available).
Second is the need to create additional store for connection id's (in most examples dynamoDB).
Another one is that adding alias domain name might be a problem (need to confirm that).

As an additional point creating graphql api supporting subscriptions using lambda and websockets gateway seems to be even less feasible.
When using `99designs/gqlgen` we'd have to create a custom `Transport` to be able to route the requests appropriately.
We'd also need to create a lambda adapter.

# Step functions

Provide a variety of state machines that cater do different workflows.

1. Standard workflows

 - Long-running: for durable and auditable tasks that take up to a year (full execution history available up to 90 days after completion)
 - At-most-once model: tasks are never executed more than once, unless specifically configured with retries
 - Non-idempotent: running the same task again can generate different state each time

2. Express workflows

 - Short-lived: up to 5 minutes, great for high volume, event processing type of workloads
 - At-least-once model: ideal if execution can be run multiple times
 - Idempotent: given the same input, the execution yields the same results

There are 2 types of express workflows:

 - synchronous - begins workflow, wait until it completes and sends the result,
 - asynchronous - begins workflow, confirms the workflow started, the result can be found in CloudWatch logs.

# X-Ray

It needs SDK for application instrumentation and an agent running on the system.
For ECS, the agent has to be installed in it's own container.
We can use annotations to add key-value data to traces.

# DynamoDB

Fully managed, highly available with replication across 3 AZs.
Scale up and down with no downtime, distributed 'serverless' DB.
Consistent performance.
NoSQL DB.
Standard and infrequent access (IA) table class.
DynamoDB on its own has single digit milisecond latency, but for microsecond latency, use it with DynamoDB Accelerator (DAX).

It supports both document and key-value data models (documents can be JSON, XML or HTML).

DynamoDB has 2 consistency models:

 - eventually consistent (up to 1s delay before most recent writes are visible everywhere),
 - strongly consistent (all writes and updates are reflected instantly),
 - DynamoDB transactions (ACID transactions).

DynamoDB is structured as tables which contain items, which have attributes.

There are 2 types of primary keys (they have to uniquely identify items):

 - partition key,
 - composite key (partition key + sort key).

### Access control

Using `dynamodb:LeadingKeys` in a statement allows access only to items
where the partition key matches the User_ID.

### Indices

##### Local secondary index

Can only be created at table creation time.
It also can't be modified later.
Consists of the table's partition key, but has a different sort key.

##### Global secondary index

Has completely different primary key (different partition key and sort key).
Can be created and modified any time.

### Queries vs scans

Queries retrieve items based on the value of partition key and can be refined by specifying additional conditions based on sort key.
Results are sorted by the sort key.
To reverse the sort, set `ScanIndexForward` to `false` (it only applies to queries, despite the name).
By default both queries and scans return all attributes of matching items,
but this can be changed by passing `ProjectionExpression`,
which allows to define the attributes to be returned.

Scan examins all the items in the table.
We can filter the result set by any of the attributes.
Scans are sequential by default - data is returned in 1MB chunks and partitions are scanned one after another.
Scans should be avoided whenever possible.

To make scans or queries have less impact on the overall performance,
we can set smaller page size for the returned data.
We can also isolate scan operations to specific tables
and isolate them from the mission critical ones.

### API calls

`CreateTable`
`UpdateTable`
`ListTables`
`DescribeTable`
`DeleteTable`
`PutItem`
`GetItem`
`UpdateItem`
`DeleteItem`
`Scan`
`Query`

### Provisioned throghput

Measured in capacity units.
We specify both read and write capacity units.
For each write capacity unit we get one write of 1kB of data every second.
For each read capacity unit we get one read of 4kB of strongly consistent data every second,
or 2 reads of 4kB of eventually consistent data every second.
Reads cannot return multiple items and reads of items bigger then 4 kB always consume whole capacity units.

There's also an option for on-demand capacity.
Suitable for unpredictable traffic and pay-per-use model.

### DynamoDB Accelerator (DAX)

Fully managed in-memory cache for DynamoDB.
Delivers up to 10x read performance improvement.
It's a write-through cache.
If on read we encounter cache miss, DAX performs an eventually consistent read from DynamoDB.
Not suitable for strongly consistent reads.

### DynamoDB TTL

Used to remove stale data from the table.
We need to define the attribute used for TTL.
That attribute has to contain UNIX timestamp values (in seconds).

Once the TTL expires, the item is marked for deletion and removed within next 48 hours.

It's good for session data, logs, etc.

### DynamoDB Stream

It's a time ordered sequence of item level modifications.
Logs are stored encrypted for 24h.
Accessed via dedicated endpoint.
By default only the primary key is recorded.
Used for auditing, archiving, triggering actions or replicating data.

### DynamoDB Global Tables

Makes DynamoDB table accessible across multiple regions. There is a 2-way replication mechanism involved.

# Elastic Beanstalk

Provides a developer centric view of deploying and managing applications on AWS.
It reduces management complexity without restricting choice or control.
You simply upload your application, and Elastic Beanstalk automatically handles the details of capacity provisioning, load balancing, scaling, and application health monitoring (logs pushed to CloudWatch).

It's a PaaS offering.

Does not support Jetty for JBoss applications (whatever that is).

Beanstalk is free but you pay for the underlying resources.

There are different models of deployment:

 - all at once,
 - rolling (deploys new version in batches),
 - rolling with additional batch,
 - immutable (deploys new infra, then retires old),
 - traffic splitting (like immutable, but like canary deployment).

## Architecture models

 - Single instance deployment: good for dev
 - LB + ASG: good for production
 - ASG only: good for non-web apps in production (workers etc.)

## Configuration

(describing only amazon linux 2 here)

We can configure the environment with `Procfile`, `Buildfile` and platform hooks.

`Buildfile` is meant for short running tasks, like shell commands.
It should be placed in the root of the project.
Consists of key-value pairs in a format: `<process_name>: <command>`.

`Procfile` is for long-running processes.
Eg. running an app.
Same format as `Buildfile`.

Platform hooks are executables or scripts that are run at specified stage by elastic beanstalk.
The hooks are stored in dedicated directories of the project:
 - `.platform/hooks/prebuild`
 - `.platform/hooks/predeploy`
 - `.platform/hooks/postdeploy`

 ### Windows Web Application Migration Assistant

# AWS Athena

Serverless, interactive query service to analyze data in Amazon S3 using SQL.
Uses Presto, supports standard SQL, JDBC/ODBC drivers, and REST API.
Supports CSV, JSON, ORC, Avro, Parquet, and other formats.
Can be used with AWS Glue to create a metadata catalog.
Costs 5$ per TB of data scanned.
Use compressed or columnar data to reduce the cost.

Use cases:

 - querying log files stored in S3 (access logs, cloud trail etc.)
 - analysis of AWS cost and usage reports
 - click stream data analysis
 - business analyses based on data stored in S3

# CloudFront

A content delivery network.
The data centers actually serving the end users are called 'edge locations'.
There are more than 200 edge locations.
CloudFront origin is the source for the resources, that the distribution will serve.
The origin can be a server, S3 bucket etc.
CloudFront distribution consists of origin and settings to serve it's contents.
Default TTL is 1 day.
Manual invalidations are separately billed.

Origin access identity is a special CloudFront user that can access the resources from origin and serve them to users.
OAI allows us to control access to the origin's resources.

Allowed HTTP methods specify which of them are going to be supported by the distribution.

### S3 transfer acceleration

After the object is uploaded to edge location, the object is being transferred to S3 using the fast AWS network connections.

### SSL Certificates

When using AWS CertificateManager, the certificates that are to be used with CloudFront need to be created within `us-east-1` region.

# AWS API Gateway

We can import our API using a OpenAPI definition file.

For legacy SOAP APIs we can configure a SOAP web service passthrough.

We can use stages and stage variables for creating dynamic deployments.

Api gateway can transform requests and responses by modifying the headers, query parameters or path.

Api gateway also provides a way to cache the responses for specified TTL in seconds (default is 300).

We can also setup throttling for incoming requests.
The default limits for region are 10k requests per second and 5k parallel requests per second.
After the limits are reached we'll start to get 429 responses.

# KMS

Key management service.
CMK - customer master key is used for encryption of 'data keys'.
It implements 'envelope encryption'.
There are AWS managed CMKs (eg. 'aws/rds').
CMK encrypts only 4KB - that's why we need data keys.

`aws kms encrypt`
`aws kms decrypt`
`aws kms re-encrypt`
`aws kms enable-key-rotation` - enables rotation of the key (by default every year)
`aws kms generate-data-key`

When rotation is enabled, the old key material is persisted,
to enable decryption of data encrypted with key's previous versions.

## ACM

AWS Certificate Manager is used to manage SSL/TLS certificates.
For a cerificate to be used by CloudFront, it has to be created in `us-east-1` region.

# SQS

It's a message queue service.
Messages need to be pulled from the queue by consuming services (poll based system).
Messages can contain up to 256KB.
Maximum retention period for messages is 14 days.
Allows decoupling of infrastructure.

Two types of queues:
 - standard,
 - FIFO.

### Standard queue

Standard queues provide `best-effort` ordering.
At least once delivery guarantee, so duplicates might be introduced.
It's the default queue type.

### FIFO queue

FIFO queues quarantee the ordering.
Exactly once delivery guarantee.
SQS FIFO delivery queues support deduplication based on a message's content.
When enabled, the system identifies and eliminates duplicate messages to prevent them from being processed multiple times.

### Visibility timeout

Default timeout is 30s, max is 12 hours.
After the reader receives the message, it is marked as invisible.
Once the Visibility timeout expires without reader's ACK,
the message becomes visible again for other readers.
To change the visibility timeout we can use the `ChangeMessageVisibility` API call.

### Short vs long polling

When using short polling, SQS always responds immediately,
even if there are no messages in the queue.
That might mean many empty responses,
which we'll be billed for anyway.

Long polling prevents that by polling periodically and returning the response
only when a message appeares or polling timeout expires.
It's the preferred strategy for most use cases.

### SQS delay queues

Postpone delivery of new messages for set delay (0-900 minutes).

### Managing large messages

If our messages are bigger than the SQS limit we can store them in S3.
For Java there's a dedicated `extended library` - limit of 2GB for messages stored in S3.

# SNS

Uses a pub-sub model.
Publishers send messages to a topic.
Subscribers listen for messages on relevant topics.
Messages are delivered using push mechanism.
SNS uses durable storage across multiple AZs.
Supports fan-out mechanism.

# SES

Used for sending and receiving emails.
It can trigger a lambda or SNS notification.

# Kinesis

Used for streaming data.

There are four core services:

 - Kinesis Data Streams,
 - Kinesis Video Streams,
 - Kinesis Data Firehose,
 - Kinesis Data Analytics.

### Kinesis Streams

Stream data and allow building apps that process data in real time.
There are data streams and video streams.
By default retains data for 24 hours (max 365 days).
Kinesis streams are made of shards.
Each shard provides a fixed unit of capacity (5 reads per second, max 2MB per second, 1000 writes per second, max 1MB per second).

### Kinesis Data Firehose

Captures, transforms and loads data streams into AWS stores.
Enables near real-time analytics with BI-tools.
There are no shards.
All capacity is automated.
There's no data retention.
Optionally we can process the data with lambda.
Firehose can dump data to S3, to Redshift via S3 and to OpenSearch.

### Kinesis Data Analytics

Analyze, query and transform stream data in real time using SQL.
Works by querying data coming in from other types of streams,
and then storing the query results in S3, Redshift or OpenSearch.

# IAM

## Policies

 - AWS managed,
 - customer managed,
 - inline.

The inline policies are deleted when the entity it's attached to is removed.

## Cross-account access

To give permissions to entities in another account,
we need to set up a role with trust relationship with it,
and attach policies necessary for access to the resources we want to share.
The consuming account needs to provide a permission to the entity trying to access the resource a persmission
to assume the role created within the account that's sharing the resource.

## Cognito

An identity broker.
Can provide access to AWS resources by providing temporary AWS credentials in exchange for an authentication code from IdP.

Can synchronize user data across multiple devices the user is signed-in on.
Uses SNS under the hood.

### User Pools

User directories used to manage sign-in and sign-up functionality for web and mobile apps.
Users can sign in directly using user pool or use IdP.

### Identity Pools

Enable providing of AWS credentials.

### Web Identity Federation

Allows access after successful authentication with web-base identity provider like Google or Facebook.
The access token? from IdP can be traded for temporary AWS credentials.
The temporary credentials map to a role in IAM.

# AWS CloudWatch

CloudWatch can monitor many AWS services.
CloudWatch Agent can enable custom metrics.

By default all EC2 send key performance and health metrics to CloudWatch.
Metrics are stored indefinitely by default.
EC2 does not send OS level metrics on it's own,
but we can achieve that by installing CloudWatch Agent.

We can monitor applications running on EC2 by using existing
system and application log files.

## Metrics

A metric is a variable we want to monitor.
It's a time-ordered sequence of values.
Each value has it's timestamp.
Metrics are defined by a name, a namespace and zero or more dimensions.

A namespace is simply a grouping of metrics,
eg. `AWS/EC2` namespace contains all metrics related to EC2.
When we're sending custom metrics data, we need to specify a namespace for it to go to.

Dimensions are key-value pairs, which can be used for filtering the data.
An example would be `InstanceId` dimension, which enables filtering specific instance data.
One difference from namespaces is that CloudWatch can aggregate data across different dimensions.
So we can for example see CPU load across all our EC2 instances.

### Metrics frequency

By default the interval is set to 5 minutes.
Detailed monitoring allows 1 minute interval at additional cost.
For custom metrics the default interval is 1 minute
and it can be configured to lower it even to 1 second (high resolution metrics).

### CloudWatch Alarms

We can create an alarm that watches the specified metrics.
We set tresholds, which when reached, cause a specific action (eg. send notification).

### CludWatch API

1. PutMetricData

```bash
aws cloudwatch put-metric-data \
--metric-name <name> \
--namespace <name> \
--value <value> \
--timestamp <ISO8601>
```

PutMetricAlarm

```bash
aws cloudwatch put-metric-alarm \
--alarm-name <name> \
--alarm-description <description> \
--metric-name <name> \
--namespace <name> \
--statistic <statistic_name> \
--period <number_of_seconds> \
--threshold <value> \
--comparison-operator <comparison_operator> \
--evaluation-periods <number_of_ev_periods>
```


# AWS EventBridge

EventBridge can route events, based on rules, to the correct targets.
Target can be an action to be taken or an AWS resource.

EventBridge rules can also run on a schedule.

EventBridge shares the same underlying engine with CloudWatch Events,
but is the preferred way for managing events and provides more features.

The same rules will be shown in CloudWatch Events and EventBridge.

