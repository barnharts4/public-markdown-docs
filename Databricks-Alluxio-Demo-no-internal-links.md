# Databricks/Alluxio Integration Demo

## Overview
Expedia has integrated Alluxio support into our datalakes and into our query tools to allow efficient cached cross-region 
querying of data stored in AWS S3 buckets.  For this demo, we will be showing how our Analytics datalake and platform
can access data in the Hotels.com datalake in a different region. We do this transparently to the query engine, in this 
case Databricks, with the following methodology:

1. Databricks is in AWS region `us-east-1`. Hotels.com datalake is in `us-west-2`.
2. Create an Alluxio cluster in AWS region `us-east-1`.  This cluster is configured with a list of S3 buckets in `us-west-2` that it should mount.
3. Install the Alluxio client JAR into the classpath of all Databricks clusters so that the URI scheme `alluxio://` is supported by the query engine.
4. Set the Hive Metastore URI for Databricks to our Hive federation process [Waggledance](https://github.com/ExpediaGroup/waggle-dance).  Waggledance is an Hive/Thrift API-compatible proxy that can federate multiple Hive metastores into one logical metastore.
5. Waggledance is configured with the same list of S3 buckets. When Waggledance get an HMS response where a table or partition path contains one of those buckets, it will replace the location with a corresponding `alluxio://` URI pointing to our Alluxio cluster.
6. When Databricks makes a call to the Hive metastore, any tables that reference those buckets will be processed with the returned `alluxio://` URI, and the Alluxio client jar will be used to read that data from the Alluxio cluster.

## Technology Involved
#### Note - all ExpediaGroup internal source code urls have been removed

* [Apiary](https://github.com/ExpediaGroup/apiary): Expedia Groups's open source collection of Terraform, Docker, and Java to create a federated cloud data lake with Ranger authorization.
* [Waggledance](https://github.com/ExpediaGroup/waggle-dance): Open-source Hive Metastore Federation Proxy
  * Responsible for presenting a federated view of various Expedia datalakes into one logical datalake, allowing cross-datalake queries, joins, etc.
  * This is the piece of code doing the actual S3 to Alluxio path translations on Hive tables and partitions
* [Apiary Hive Metastore Filter](https://github.com/ExpediaGroup/apiary-extensions/tree/main/hive-hooks): Open-source Hive metastore filter that can do arbitrary path replacements.
   * We install this into the container running Waggledance.
   * It is configured to replace `s3://` paths of our target bucket list with the corresponding `alluxio://` URI.
* [Apiary Waggledance Docker](https://github.com/ExpediaGroup/apiary-waggledance-docker): Open-source Docker image to configure Waggledance to run in Expedia's `Apiary` datalake.
    * Hive filter is configured [here](https://github.com/ExpediaGroup/apiary-waggledance-docker/blob/master/Dockerfile#L23).
* [Apiary Federation](https://github.com/ExpediaGroup/apiary-federation): Open-source Terraform module to configure Waggledance via our Docker image.
   * Alluxio replacements are configured [here](https://github.com/ExpediaGroup/apiary-federation/blob/master/hive-site.tf).
* [eg-tf-mod-alluxio](about:blank): Inner-source Terraform module to deploy an Alluxio cluster
  * List of S3 buckets for path replacement set in Terraform and mounted using AWS State Manager (Ansible playbook):
```yaml
  - name: script to mount s3 buckets
    copy:
      dest: /tmp/mount-s3-buckets.sh
      mode: 0755
      owner: hdfs
      group: hdfs
      content: |
        #!/bin/bash
        IFS=","
        s3_mounts_ro="${s3_mounts_ro}"
        for s3_bucket in $$s3_mounts_ro
        do
          /opt/alluxio/bin/alluxio fs mount --shared --readonly /$$s3_bucket s3://$$s3_bucket
        done
  - name: mount s3 buckets
    command: /tmp/mount-s3-buckets.sh
    become: yes
    become_user: hdfs
    ignore_errors: yes
```
* [egdp-tf-app-egap-apiary Federation component](about:blank): Inner-source Terraform application used to configure Waggledance, Alluxio, and the federation layer for our Analytics Platform Data Lake.
  * List of cross-region S3 buckets to access via Alluxio specified as a Terraform variable.
  * Alluxio created/configured by using the `eg-tf-mod-alluxio` Terraform module and configuring it with the above list of buckets.
  * Waggledance instance created by using the `apiary-federation` Terraform module, which also configures the PathConversion Hive filter in Waggledance to use the same list of buckets.
* [egdp-tf-app-egap-databricks](about:blank): Inner-source Terraform application to deploy Databricks to the EG Analytics Platform.
  * Alluxio client jar configured in each Databricks cluster using code similar to:
```shell
#!/bin/bash
# alluxio client
aws s3 cp "${TFVAR_ALLUXIO_CLIENT_PACKAGE}" "/databricks/jars/alluxio-${TFVAR_ALLUXIO_VERSION}-client.jar"

```
  