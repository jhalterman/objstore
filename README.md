<p align="center"><img src="docs/img/Thanos-logo_fullmedium.png" alt="Thanos Logo"></p>

[![Latest Release](https://img.shields.io/github/release/thanos-io/objstore.svg?style=flat-square)](https://github.com/thanos-io/objstore/releases/latest) [![Slack](https://img.shields.io/badge/join%20slack-%23thanos-brightgreen.svg)](https://slack.cncf.io/)

[![Go Report Card](https://goreportcard.com/badge/github.com/thanos-io/objstore)](https://goreportcard.com/report/github.com/thanos-io/objstore) [![Go Code reference](https://img.shields.io/badge/code%20reference-go.dev-darkblue.svg)](https://pkg.go.dev/github.com/thanos-io/objstore?tab=subdirectories) 

[![Tests](https://github.com/thanos-io/objstore/workflows/Test/badge.svg)](https://github.com/thanos-io/objstore/actions?query=workflow%3Atest)

# Thanos Object Storage

Thanos uses object storage as primary storage for metrics and metadata related to them.
This repo contains a standalone Go package that handles object storage operations.

## Documentation 

Check out the [documentation](https://thanos.io/tip/thanos/storage.md/) for more up-to-date version.

### Configuring Access to Object Storage

Thanos supports any object stores that can be implemented against Thanos [objstore.Bucket interface](../pkg/objstore/objstore.go).

All clients can be configured using `--objstore.config-file` to reference to the configuration file or `--objstore.config` to put yaml config directly.

#### How to use our special `config` flags?

**You can either pass YAML file defined below in `--objstore.config-file` or pass the YAML content directly using `--objstore.config`** We recommend the latter as it gives an explicit static view of configuration for each component. It also saves you the fuss of creating and managing additional file.

Don't be afraid of multiline flags!

In Kubernetes it is as easy as (on Thanos sidecar example):

```yaml
      - args:
        - sidecar
        - |
          --objstore.config=type: GCS
          config:
            bucket: <bucket>
        - --prometheus.url=http://localhost:9090
        - |
          --tracing.config=type: STACKDRIVER
          config:
            service_name: ""
            project_id: <project>
            sample_factor: 16
        - --tsdb.path=/prometheus-data
```

#### Supported Clients

Current object storage client implementations:

| Provider                                                                               | Maturity           | Aimed For             | Auto-tested on CI | Maintainers             |
|----------------------------------------------------------------------------------------|--------------------|-----------------------|-------------------|-------------------------|
| [Google Cloud Storage](#gcs)                                                           | Stable             | Production Usage      | yes               | @bwplotka               |
| [AWS/S3](#s3) (and all S3-compatible storages e.g disk-based [Minio](https://min.io/)) | Stable             | Production Usage      | yes               | @bwplotka               |
| [Azure Storage Account](#azure)                                                        | Stable             | Production Usage      | no                | @vglafirov              |
| [OpenStack Swift](#openstack-swift)                                                    | Beta (working PoC) | Production Usage      | yes               | @FUSAKLA                |
| [Tencent COS](#tencent-cos)                                                            | Beta               | Production Usage      | no                | @jojohappy,@hanjm       |
| [AliYun OSS](#aliyun-oss)                                                              | Beta               | Production Usage      | no                | @shaulboozhiao,@wujinhu |
| [Local Filesystem](#filesystem)                                                        | Stable             | Testing and Demo only | yes               | @bwplotka               |

**Missing support to some object storage?** Check out [how to add your client section](#how-to-add-a-new-client-to-thanos)

NOTE: Currently Thanos requires strong consistency (write-read) for object store implementation for singleton Compaction purposes.

##### S3

Thanos uses the [minio client](https://github.com/minio/minio-go) library to upload Prometheus data into AWS S3.

You can configure an S3 bucket as an object store with YAML, either by passing the configuration directly to the `--objstore.config` parameter, or (preferably) by passing the path to a configuration file to the `--objstore.config-file` option.

NOTE: Minio client was mainly for AWS S3, but it can be configured against other S3-compatible object storages e.g Ceph

```yaml mdox-exec="go run scripts/cfggen/main.go --name=s3.Config"
type: S3
config:
  bucket: ""
  endpoint: ""
  region: ""
  aws_sdk_auth: false
  access_key: ""
  insecure: false
  signature_version2: false
  secret_key: ""
  put_user_metadata: {}
  http_config:
    idle_conn_timeout: 1m30s
    response_header_timeout: 2m
    insecure_skip_verify: false
    tls_handshake_timeout: 10s
    expect_continue_timeout: 1s
    max_idle_conns: 100
    max_idle_conns_per_host: 100
    max_conns_per_host: 0
    tls_config:
      ca_file: ""
      cert_file: ""
      key_file: ""
      server_name: ""
      insecure_skip_verify: false
    disable_compression: false
  trace:
    enable: false
  list_objects_version: ""
  bucket_lookup_type: auto
  part_size: 67108864
  sse_config:
    type: ""
    kms_key_id: ""
    kms_encryption_context: {}
    encryption_key: ""
  sts_endpoint: ""
prefix: ""
```

At a minimum, you will need to provide a value for the `bucket`, `endpoint`, `access_key`, and `secret_key` keys. The rest of the keys are optional.

However if you set `aws_sdk_auth: true` Thanos will use the default authentication methods of the AWS SDK for go based on [known environment variables](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-envvars.html) (`AWS_PROFILE`, `AWS_WEB_IDENTITY_TOKEN_FILE` ... etc) and known AWS config files (~/.aws/config). If you turn this on, then the `bucket` and `endpoint` are the required config keys.

The field `prefix` can be used to transparently use prefixes in your S3 bucket. This allows you to separate blocks coming from different sources into paths with different prefixes, making it easier to understand what's going on (i.e. you don't have to use Thanos tooling to know from where which blocks came).

The AWS region to endpoint mapping can be found in this [link](https://docs.aws.amazon.com/general/latest/gr/s3.html).

Make sure you use a correct signature version. Currently AWS requires signature v4, so it needs `signature_version2: false`. If you don't specify it, you will get an `Access Denied` error. On the other hand, several S3 compatible APIs use `signature_version2: true`.

You can configure the timeout settings for the HTTP client by setting the `http_config.idle_conn_timeout` and `http_config.response_header_timeout` keys. As a rule of thumb, if you are seeing errors like `timeout awaiting response headers` in your logs, you may want to increase the value of `http_config.response_header_timeout`.

Please refer to the documentation of [the Transport type](https://golang.org/pkg/net/http/#Transport) in the `net/http` package for detailed information on what each option does.

`part_size` is specified in bytes and refers to the minimum file size used for multipart uploads, as some custom S3 implementations may have different requirements. A value of `0` means to use a default 128 MiB size.

Set `list_objects_version: "v1"` for S3 compatible APIs that don't support ListObjectsV2 (e.g. some versions of Ceph). Default value (`""`) is equivalent to `"v2"`.

`http_config.tls_config` allows configuring TLS connections. Please refer to the document of [tls_config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#tls_config) for detailed information on what each option does.

`bucket_lookup_type` can be `auto`, `virtual-hosted` or `path`. Read more about it [here](https://docs.aws.amazon.com/AmazonS3/latest/userguide/VirtualHosting.html).

For debug and testing purposes you can set

* `insecure: true` to switch to plain insecure HTTP instead of HTTPS

* `http_config.insecure_skip_verify: true` to disable TLS certificate verification (if your S3 based storage is using a self-signed certificate, for example)

* `trace.enable: true` to enable the minio client's verbose logging. Each request and response will be logged into the debug logger, so debug level logging must be enabled for this functionality.

###### S3 Server-Side Encryption

SSE can be configued using the `sse_config`. [SSE-S3](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingServerSideEncryption.html), [SSE-KMS](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingKMSEncryption.html), and [SSE-C](https://docs.aws.amazon.com/AmazonS3/latest/dev/ServerSideEncryptionCustomerKeys.html) are supported.

* If type is set to `SSE-S3` you do not need to configure other options.

* If type is set to `SSE-KMS` you must set `kms_key_id`. The `kms_encryption_context` is optional, as [AWS provides a default encryption context](https://docs.aws.amazon.com/kms/latest/developerguide/services-s3.html#s3-encryption-context).

* If type is set to `SSE-C` you must provide a path to the encryption key using `encryption_key`.

If the SSE Config block is set but the `type` is not one of `SSE-S3`, `SSE-KMS`, or `SSE-C`, an error is raised.

You will also need to apply the following AWS IAM policy for the user to access the KMS key:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "KMSAccess",
            "Effect": "Allow",
            "Action": [
                "kms:GenerateDataKey",
                "kms:Encrypt",
                "kms:Decrypt"
            ],
            "Resource": "arn:aws:kms:<region>:<account>:key/<KMS key id>"
        }
    ]
}
```

###### Credentials

By default Thanos will try to retrieve credentials from the following sources:

1. From config file if BOTH `access_key` and `secret_key` are present.
2. From the standard AWS environment variable - `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
3. From `~/.aws/credentials`
4. IAM credentials retrieved from an instance profile.

NOTE: Getting access key from config file and secret key from other method (and vice versa) is not supported.

###### AWS Policies

Example working AWS IAM policy for user:

* For deployment (policy for Thanos services):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket>/*",
                "arn:aws:s3:::<bucket>"
            ]
        }
    ]
}
```

(No bucket policy)

To test the policy, set env vars for S3 access for *empty, not used* bucket as well as:

```
THANOS_TEST_OBJSTORE_SKIP=GCS,AZURE,SWIFT,COS,ALIYUNOSS
THANOS_ALLOW_EXISTING_BUCKET_USE=true
```

And run: `GOCACHE=off go test -v -run TestObjStore_AcceptanceTest_e2e ./pkg/...`

* For testing (policy to run e2e tests):

We need access to CreateBucket and DeleteBucket and access to all buckets:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Statement",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject",
                "s3:DeleteObject",
                "s3:PutObject",
                "s3:CreateBucket",
                "s3:DeleteBucket"
            ],
            "Resource": [
                "arn:aws:s3:::<bucket>/*",
                "arn:aws:s3:::<bucket>"
            ]
        }
    ]
}
```

With this policy you should be able to run set `THANOS_TEST_OBJSTORE_SKIP=GCS,AZURE,SWIFT,COS,ALIYUNOSS` and unset `S3_BUCKET` and run all tests using `make test`.

Details about AWS policies: https://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html

###### STS Endpoint

If you want to use IAM credential retrieved from an instance profile, Thanos needs to authenticate through AWS STS. For this purposes you can specify your own STS Endpoint.

By default Thanos will use endpoint: https://sts.amazonaws.com and AWS region coresponding endpoints.

##### GCS

To configure Google Cloud Storage bucket as an object store you need to set `bucket` with GCS bucket name and configure Google Application credentials.

For example:

```yaml mdox-exec="go run scripts/cfggen/main.go --name=gcs.Config"
type: GCS
config:
  bucket: ""
  service_account: ""
prefix: ""
```

###### Using GOOGLE_APPLICATION_CREDENTIALS

Application credentials are configured via JSON file and only the bucket needs to be specified, the client looks for:

1. A JSON file whose path is specified by the `GOOGLE_APPLICATION_CREDENTIALS` environment variable.
2. A JSON file in a location known to the gcloud command-line tool. On Windows, this is `%APPDATA%/gcloud/application_default_credentials.json`. On other systems, `$HOME/.config/gcloud/application_default_credentials.json`.
3. On Google App Engine it uses the `appengine.AccessToken` function.
4. On Google Compute Engine and Google App Engine Managed VMs, it fetches credentials from the metadata server. (In this final case any provided scopes are ignored.)

You can read more on how to get application credential json file in [https://cloud.google.com/docs/authentication/production](https://cloud.google.com/docs/authentication/production)

###### Using inline a Service Account

Another possibility is to inline the ServiceAccount into the Thanos configuration and only maintain one file. This feature was added, so that the Prometheus Operator only needs to take care of one secret file.

```yaml
type: GCS
config:
  bucket: "thanos"
  service_account: |-
    {
      "type": "service_account",
      "project_id": "project",
      "private_key_id": "abcdefghijklmnopqrstuvwxyz12345678906666",
      "private_key": "-----BEGIN PRIVATE KEY-----\...\n-----END PRIVATE KEY-----\n",
      "client_email": "project@thanos.iam.gserviceaccount.com",
      "client_id": "123456789012345678901",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token",
      "auth_provider_x509_cert_url": "https://www.googleapis.com/oauth2/v1/certs",
      "client_x509_cert_url": "https://www.googleapis.com/robot/v1/metadata/x509/thanos%40gitpods.iam.gserviceaccount.com"
    }
```

###### GCS Policies

**Note:** GCS Policies should be applied at the project level, not at the bucket level

For deployment:

`Storage Object Creator` and `Storage Object Viewer`

For testing:

`Storage Object Admin` for ability to create and delete temporary buckets.

To test the policy is working as expected, exec into the sidecar container, eg:

```sh
kubectl exec -it -n <namespace> <prometheus with sidecar pod name> -c <sidecar container name> -- /bin/sh
```

Then test that you can at least list objects in the bucket, eg:

```sh
thanos tools bucket ls --objstore.config="${OBJSTORE_CONFIG}"
```

##### Azure

To use Azure Storage as Thanos object store, you need to precreate storage account from Azure portal or using Azure CLI. Follow the instructions from Azure Storage Documentation: [https://docs.microsoft.com/en-us/azure/storage/common/storage-quickstart-create-account](https://docs.microsoft.com/en-us/azure/storage/common/storage-quickstart-create-account?tabs=portal)

To configure Azure Storage account as an object store you need to provide a path to Azure storage config file in flag `--objstore.config-file`.

Config file format is the following:

```yaml mdox-exec="go run scripts/cfggen/main.go --name=azure.Config"
type: AZURE
config:
  storage_account: ""
  storage_account_key: ""
  container: ""
  endpoint: ""
  max_retries: 0
  msi_resource: ""
  user_assigned_id: ""
  pipeline_config:
    max_tries: 0
    try_timeout: 0s
    retry_delay: 0s
    max_retry_delay: 0s
  reader_config:
    max_retry_requests: 0
  http_config:
    idle_conn_timeout: 0s
    response_header_timeout: 0s
    insecure_skip_verify: false
    tls_handshake_timeout: 0s
    expect_continue_timeout: 0s
    max_idle_conns: 0
    max_idle_conns_per_host: 0
    max_conns_per_host: 0
    tls_config:
      ca_file: ""
      cert_file: ""
      key_file: ""
      server_name: ""
      insecure_skip_verify: false
    disable_compression: false
prefix: ""
```

If `msi_resource` is used, authentication is done via system-assigned managed identity. The value for Azure should be `https://<storage-account-name>.blob.core.windows.net`.

If `user_assigned_id` is used, authentication is done via user-assigned managed identity. When using `user_assigned_id` the `msi_resource` defaults to `https://<storage_account>.<endpoint>`

The generic `max_retries` will be used as value for the `pipeline_config`'s `max_tries` and `reader_config`'s `max_retry_requests`. For more control, `max_retries` could be ignored (0) and one could set specific retry values.

##### OpenStack Swift

Thanos uses [ncw/swift](https://github.com/ncw/swift) client to upload Prometheus data into [OpenStack Swift](https://docs.openstack.org/swift/latest/).

Below is an example configuration file for thanos to use OpenStack swift container as an object store. Note that if the `name` of a user, project or tenant is used one must also specify its domain by ID or name. Various examples for OpenStack authentication can be found in the [official documentation](https://developer.openstack.org/api-ref/identity/v3/index.html?expanded=password-authentication-with-scoped-authorization-detail#password-authentication-with-unscoped-authorization).

By default, OpenStack Swift has a limit for maximum file size of 5 GiB. Thanos index files are often larger than that. To resolve this issue, Thanos uses [Static Large Objects (SLO)](https://docs.openstack.org/swift/latest/overview_large_objects.html) which are uploaded as segments. These are by default put into the `segments` directory of the same container. The default limit for using SLO is 1 GiB which is also the maximum size of the segment. If you don't want to use the same container for the segments (best practise is to use `<container_name>_segments` to avoid polluting listing of the container objects) you can use the `large_file_segments_container_name` option to override the default and put the segments to other container. *In rare cases you can switch to [Dynamic Large Objects (DLO)](https://docs.openstack.org/swift/latest/overview_large_objects.html) by setting the `use_dynamic_large_objects` to true, but use it with caution since it even more relies on eventual consistency.*

```yaml mdox-exec="go run scripts/cfggen/main.go --name=swift.Config"
type: SWIFT
config:
  auth_version: 0
  auth_url: ""
  username: ""
  user_domain_name: ""
  user_domain_id: ""
  user_id: ""
  password: ""
  domain_id: ""
  domain_name: ""
  project_id: ""
  project_name: ""
  project_domain_id: ""
  project_domain_name: ""
  region_name: ""
  container_name: ""
  large_object_chunk_size: 1073741824
  large_object_segments_container_name: ""
  retries: 3
  connect_timeout: 10s
  timeout: 5m
  use_dynamic_large_objects: false
prefix: ""
```

##### Tencent COS

To use Tencent COS as storage store, you should apply a Tencent Account to create an object storage bucket at first. Note that detailed from Tencent Cloud Documents: [https://cloud.tencent.com/document/product/436](https://cloud.tencent.com/document/product/436)

To configure Tencent Account to use COS as storage store you need to set these parameters in yaml format stored in a file:

```yaml mdox-exec="go run scripts/cfggen/main.go --name=cos.Config"
type: COS
config:
  bucket: ""
  region: ""
  app_id: ""
  endpoint: ""
  secret_key: ""
  secret_id: ""
  http_config:
    idle_conn_timeout: 1m30s
    response_header_timeout: 2m
    insecure_skip_verify: false
    tls_handshake_timeout: 10s
    expect_continue_timeout: 1s
    max_idle_conns: 100
    max_idle_conns_per_host: 100
    max_conns_per_host: 0
    tls_config:
      ca_file: ""
      cert_file: ""
      key_file: ""
      server_name: ""
      insecure_skip_verify: false
    disable_compression: false
prefix: ""
```

The `secret_key` and `secret_id` field is required. The `http_config` field is optional for optimize HTTP transport settings. There are two ways to configure the required bucket information:
1. Provide the values of `bucket`, `region` and `app_id` keys.
2. Provide the values of `endpoint` key with url format when you want to specify vpc internal endpoint. Please refer to the document of [endpoint](https://intl.cloud.tencent.com/document/product/436/6224) for more detail.

Set the flags `--objstore.config-file` to reference to the configuration file.

##### AliYun OSS

In order to use AliYun OSS object storage, you should first create a bucket with proper Storage Class , ACLs and get the access key on the AliYun cloud. Go to [https://www.alibabacloud.com/product/oss](https://www.alibabacloud.com/product/oss) for more detail.

To use AliYun OSS object storage, please specify following yaml configuration file in `objstore.config*` flag.

```yaml mdox-exec="go run scripts/cfggen/main.go --name=oss.Config"
type: ALIYUNOSS
config:
  endpoint: ""
  bucket: ""
  access_key_id: ""
  access_key_secret: ""
prefix: ""
```

Use --objstore.config-file to reference to this configuration file.

##### Baidu BOS

In order to use Baidu BOS object storage, you should apply for a Baidu Account and create an object storage bucket first. Refer to [Baidu Cloud Documents](https://cloud.baidu.com/doc/BOS/index.html) for more details. To use Baidu BOS object storage, please specify the following yaml configuration file in `--objstore.config*` flag.

```yaml mdox-exec="go run scripts/cfggen/main.go --name=bos.Config"
type: BOS
config:
  bucket: ""
  endpoint: ""
  access_key: ""
  secret_key: ""
prefix: ""
```

##### Filesystem

This storage type is used when user wants to store and access the bucket in the local filesystem. We treat filesystem the same way we would treat object storage, so all optimization for remote bucket applies even though, we might have the files locally.

NOTE: This storage type is experimental and might be inefficient. It is NOT advised to use it as the main storage for metrics in production environment. Particularly there is no planned support for distributed filesystems like NFS. This is mainly useful for testing and demos.

```yaml mdox-exec="go run scripts/cfggen/main.go --name=filesystem.Config"
type: FILESYSTEM
config:
  directory: ""
prefix: ""
```

#### How to add a new client to Thanos?

Following checklist allows adding new Go code client to supported providers:

1. Create new directory under `pkg/objstore/<provider>`
2. Implement [objstore.Bucket interface](objstore.go)
3. Add `NewTestBucket` constructor for testing purposes, that creates and deletes temporary bucket.
4. Use created `NewTestBucket` in [ForeachStore method](objtesting/foreach.go) to ensure we can run tests against new provider. (In PR)
5. RUN the [TestObjStoreAcceptanceTest](objtesting/acceptance_e2e_test.go) against your provider to ensure it fits. Fix any found error until test passes. (In PR)
6. Add client implementation to the factory in [factory](client/factory.go) code. (Using as small amount of flags as possible in every command)

[//]: # (7. Add client struct config to [bucketcfggen]&#40;scripts/cfggen/main.go&#41; to allow config auto generation.)

At that point, anyone can use your provider by spec.

## Contributing

Contributions are very welcome! See our [CONTRIBUTING.md](https://github.com/thanos-io/thanos/blob/main/CONTRIBUTING.md) for more information.

## Community

Thanos is an open source project and we value and welcome new contributors and members of the community. Here are ways to get in touch with the community:

* Slack: [#thanos](https://slack.cncf.io/)
* Issue Tracker: [GitHub Issues](https://github.com/thanos-io/thanos/issues)

## Adopters

See [`Adopters List`](https://github.com/thanos-io/thanos/blob/main/website/data/adopters.yml.

## Maintainers

See [MAINTAINERS.md](https://github.com/thanos-io/thanos/blob/main/MAINTAINERS.md)
