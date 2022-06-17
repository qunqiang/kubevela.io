---
title: Cloud Resources Orchestration on Terraform and Crossplane
---

Cloud-oriented development is now becoming the norm, there is an urgent need to integrate cloud resources from different
sources and types. Whether it is the most basic object storage, cloud database, or load balancing, it is all faced with
the challenges of hybrid cloud, multi-cloud and other complex environments. KubeVela is perfect to satisfy the needs.

KubeVela efficiently and securely integrates different types of cloud resources through resource binding capabilities in
cloud resource Components and Traits. At present, you can directly use the default components of those cloud resources below.
At the same time, more new cloud resources will gradually become the default option under the support of the community in the future.
You can use cloud resources of various manufacturers in a standardized and unified way.

This tutorial will talk about how to provision Cloud Resources by Terraform and Crossplane.

## Terraform

### Prerequisites

Please ask platform engineers to enable cloud resources Terraform addon and authenticate the target cloud provider per the [instruction](../../../reference/addons/terraform).

Let's take Alibaba Cloud as an example.

### Familiar with cloud resources specification

All supported Terraform cloud resources can be seen in the [list](./cloud-resources-list). You can also filter them by
command by `vela components --label type=terraform`.

You can use any way to check the specification of a cloud resource.

- by command `vela show $RESOURCE_DEFINITION`

```yaml
$ vela show alibaba-oss
  ### Properties
  +----------------------------+-------------------------------------------------------------------------+-----------------------------------------------------------+----------+---------+
  |            NAME            |                               DESCRIPTION                               |                           TYPE                            | REQUIRED | DEFAULT |
  +----------------------------+-------------------------------------------------------------------------+-----------------------------------------------------------+----------+---------+
  | acl                        | OSS bucket ACL, supported 'private', 'public-read', 'public-read-write' | string                                                    | false    |         |
  | bucket                     | OSS bucket name                                                         | string                                                    | false    |         |
  | writeConnectionSecretToRef | The secret which the cloud resource connection will be written to       | [writeConnectionSecretToRef](#writeConnectionSecretToRef) | false    |         |
  +----------------------------+-------------------------------------------------------------------------+-----------------------------------------------------------+----------+---------+


  #### writeConnectionSecretToRef
  +-----------+-----------------------------------------------------------------------------+--------+----------+---------+
  |   NAME    |                                 DESCRIPTION                                 |  TYPE  | REQUIRED | DEFAULT |
  +-----------+-----------------------------------------------------------------------------+--------+----------+---------+
  | name      | The secret name which the cloud resource connection will be written to      | string | true     |         |
  | namespace | The secret namespace which the cloud resource connection will be written to | string | false    |         |
  +-----------+-----------------------------------------------------------------------------+--------+----------+---------+
```

You can also add flag `--web` to view the usage by a local browser.

- by official site http://kubevela.net/docs/end-user/components/cloud-services/cloud-resources-list

For example, you can check the specification for Alibaba OSS at http://kubevela.net/docs/end-user/components/cloud-services/terraform/alibaba-oss. 

### Provision cloud resources

Use the following Application to provision an OSS bucket:

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: provision-cloud-resource-sample
spec:
  components:
    - name: sample-oss
      type: alibaba-oss
      properties:
        bucket: vela-website-0911
        acl: private
        writeConnectionSecretToRef:
          name: oss-conn
```

The above `alibaba-oss` component will create an OSS bucket named `vela-website-0911`, with `private` acl, with connection
information stored in a secreted named `oss-conn`.

Apply the above application, then check the status:

```shell
$ vela ls
APP                            	COMPONENT 	TYPE       	TRAITS	PHASE  	HEALTHY	STATUS                                       	CREATED-TIME
provision-cloud-resource-sample	sample-oss	alibaba-oss	      	running	healthy	Cloud resources are deployed and ready to use	2021-09-11 12:55:57 +0800 CST
```

After the phase becomes `running` and `healthy`, you can then check the OSS bucket in Alibaba Cloud console or by [ossutil](https://partners-intl.aliyun.com/help/doc-detail/50452.htm)
command.

```shell
$ ossutil ls oss://
CreationTime                                 Region    StorageClass    BucketName
2021-09-11 12:56:17 +0800 CST        oss-cn-beijing        Standard    oss://vela-website-0911
```

![](../../../resources/crossplane-visit-application-v3.jpg)

### More

For more usages of cloud resources, like how to provision and consume cloud resources, please refer to [Scenarios of Cloud Resources Management](./cloud-resource-scenarios).

## Crossplane

Let's take AWS as an example.

### Enable addon `crossplane-aws`

```shell
$ vela addon enable crossplane-aws
```

### Authenticate AWS Provider for Crossplane

Apply the application below. You can get AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY per https://aws.amazon.com/blogs/security/wheres-my-secret-access-key/.

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: aws
  namespace: vela-system
spec:
  components:
    - name: aws
      type: crossplane-aws
      properties:
        name: aws
        AWS_ACCESS_KEY_ID: xxx
        AWS_SECRET_ACCESS_KEY: yyy

```

### Provision cloud resources

Let's provision a S3 bucket.

Apply the application below.

```yaml
apiVersion: core.oam.dev/v1beta1
kind: Application
metadata:
  name: s3-poc
spec:
  components:
    - name: dev
      type: crossplane-aws-s3
      properties:
        name: kubevela-test-0714
        acl: private
        locationConstraint: us-east-1
```

After the application got `running`, you can check the bucket by AWS [cli](https://aws.amazon.com/cli/?nc1=h_ls) or console.

```shell
$ vela ls
APP   	COMPONENT	TYPE  	             TRAITS	PHASE  	HEALTHY	STATUS	CREATED-TIME
s3-poc	dev      	crossplane-aws-s3	      	running	healthy	      	2022-06-16 15:37:15 +0800 CST

$ aws s3 ls
2022-06-16 15:37:17 kubevela-test-0714
```
