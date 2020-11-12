_This repository was part of HCA DCP/1 and is not maintained anymore. DCP/2 development of this component continues under `infra/helm-charts/mongo/charts/ingestbackup` in the repository at https://github.com/ebi-ait/ingest-kube-deployment._

# Ingest Backup

## Introduction
The backup strategies devised in here is to ensure availability, and prevent significant loss of data after they have been ingested through HCA infrastructure. The utility is meant to complement a more robust way to persist data on Ingest's deployment environment.

While the strategies were created with HCA infrastructure specifically in mind, the tools may be reused to accommodate any system that uses Mongo DB deployed as part of of a cluster of Docker containers.

## Assumptions
Ingest infrastructure is deployed as a system of multiple self contained microservices through Docker with the use of Kubernetes. The backup system is deployed as a Docker container that has direct access to a predefined Kubernetes service, named `mongo-service` from which it gets the data it backs up to an S3 bucket defined through the `S3_BUCKET` environment variable.

## Usage

### Backing Up
By default, the backup system is set to run every second hour of the day between 0000H to 2300H, or, in other terms, on hours of the day divisible by 2 (`hour % 2 == 0`). This can be configured by updating the cron schedule in the `backup.yml` file to match another preferred schedule. Alternatively, for intances of ingest backup already deployed through Kubernetes, the schedule can be patched by updating the `spec.schedule` property of the cron job:

```
kubectl patch cronjob <job_name> -p '{  "spec": { "schedule": "0 0-23/4 * * *"  }  }'
```

The patch above will reset the schedule to be every 4 hours instead of every 2 hours.

#### Security Credentials
The backup system uses Amazon's AWS CLI tools to copy data to AWS. As the backup data will be dumped into a remote S3 bucket, the process running the backups need to be configured to have access to the bucket in question. Security credentials should be set through the environment variables to get the backup system working correctly. AWS provides documentation on [how to setup security credentials for the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).

#### Environment Variables
Several environment variables are defined as part of the configuration of the running container.

* `AWS_ACCESS_KEY_ID` - the access key ID generated by AWS for the IAM user running the backup utility
* `AWS_SECRET_ACCESS_KEY` - the secret access key generated by AWS for the IAM user running the backup utility
* `S3_BUCKET` - the name of the AWS S3 to which the backup data will be moved
* `BACKUP_DIR` - a subdirectory in the S3 bucket to which the data will be copied. This is useful for when there are multiple environments sharing the same S3 bucket for backup (e.g. dev, integration, production)

The first 2 environment variables above (access keys) are directly exposed variables used by AWS client itself.

#### Backup Data
The backup system takes the output of Mongo's `mongodump` utility and puts them into a compressed directory (tarball), which are moved to the specified S3 bucket. They are preserved in format that Mongo utilities should be able to process and understand.

### Restoring Data
The compressed directories of database backups available in the S3 bucket can be decompressed using the `tar` utility as follows:

    tar -xzvf 2018-04-04T11_37.tar.gz

This will create a directory structure, `data/db/dump/2018-04-04T11_37`, which contains the output of the `mongodump`. The `tar` utility provides more options documented in its manual (`man tar`).

To restore the backup data, the `mongorestore` utility is used:

    mongorestore 2018-04-04T11_37

The [official documentation for `mongorestore`](https://docs.mongodb.com/manual/reference/program/mongorestore/) tool lists more options for customising the restoration process.

#### Verifying Restoration

(Note: a step by step guide of the verification process for Ingest has been documented [here](https://docs.google.com/document/d/1y2pgzoK2Xt7ZCGVt_big7Lfto5nSKvNNSc70yU38t0Y/edit?usp=sharing).)

To check if the backups contain correct information, the following general strategy may be adopted:

1) Connect to source Mongo DB instance, through a shell for example. As the Mongo instance is on a Kubernetes cluster, the shell can be invoked through the `exec` utility of `kubectl`:

        kubectl -n <namespace> exec -it <mongo-pod> -- /bin/bash

2) Create a new Mongo DB instance to which the backup data will be restored. This process can easily be done by running a new Mongo container through Docker. It is advised that when running a new Mongo instance for testing the backup data is first decompressed (using tools like `tar` described above), and the resulting directory is used as a host volume:

        docker run -d --name mongo-test -v $PWD/data/db:/data/db mongo

3) Connect to the new Mongo instance through the Docker exec utility:

        docker exec -it mongo-test /bin/bash

4) While connected to the new container hosting the new Mongo DB instance, the backup can be restored through `mongorestore` tool:

        mongorestore /data/db/dump/2018-04-04T11_37

5) To verify that the restored data is consistent with the source, connect to the new Mongo DB instance (perhaps using `mongo` client) and verify that the data match with the source. As a simple test, `show collections` should display the same collections in both the source DB and the new one. Each collection in the source should contain the same number of documents as its counterpart on the new DB. This can be checked using the `count` method of each collection:

        db.<collection_name>.count()
