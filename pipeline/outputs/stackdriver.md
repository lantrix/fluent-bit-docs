# Stackdriver

Stackdriver output plugin allows to ingest your records into [Google Cloud Stackdriver Logging](https://cloud.google.com/logging/) service.

Before to get started with the plugin configuration, make sure to obtain the proper credentials to get access to the service. We strongly recommend to use a common JSON credentials file, reference link:

* [Creating a Google Service Account for Stackdriver](https://cloud.google.com/logging/docs/agent/authorization#create-service-account)

> Your goal is to obtain a credentials JSON file that will be used later by Fluent Bit Stackdriver output plugin.

## Configuration Parameters

| Key                        | Description                                                                                                                                                                                                                                                                                   | default                                                                                                                             |
| -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| google_service_credentials | Absolute path to a Google Cloud credentials JSON file                                                                                                                                                                                                                                         | Value of environment variable _$GOOGLE_SERVICE_CREDENTIALS_                                                                         |
| service_account_email      | Account email associated to the service. Only available if **no credentials file** has been provided.                                                                                                                                                                                         | Value of environment variable _$SERVICE_ACCOUNT_EMAIL_                                                                              |
| service_account_secret     | Private key content associated with the service account. Only available if **no credentials file** has been provided.                                                                                                                                                                         | Value of environment variable _$SERVICE_ACCOUNT_SECRET_                                                                             |
| metadata_server            | Prefix for a metadata server. Can also set environment variable _$METADATA_SERVER_.                                                                                                                                                                                                           | [http://metadata.google.internal](http://metadata.google.internal)                                                                  |
| location                   | The GCP or AWS region in which to store data about the resource. If the resource type is one of the _generic_node_ or _generic_task_, then this field is required.                                                                                                                            |                                                                                                                                     |
| namespace                  | A namespace identifier, such as a cluster name or environment. If the resource type is one of the _generic_node_ or _generic_task_, then this field is required.                                                                                                                              |                                                                                                                                     |
| node_id                    | A unique identifier for the node within the namespace, such as hostname or IP address. If the resource type is _generic_node_, then this field is required.                                                                                                                                   |                                                                                                                                     |
| job                        | An identifier for a grouping of related task, such as the name of a microservice or distributed batch. If the resource type is _generic_task_, then this field is required.                                                                                                                   |                                                                                                                                     |
| task_id                    | A unique identifier for the task within the namespace and job, such as a replica index identifying the task within the job. If the resource type is _generic_task_, then this field is required.                                                                                              |                                                                                                                                     |
| export_to_project_id       | The GCP project that should receive these logs.                                                                                                                                                                                                                                               | Defaults to the project ID of the google_service_credentials file, or the project_id from Google's metadata.google.internal server. |
| resource                   | Set resource type of data. Supported resource types: _k8s_container_, _k8s_node_, _k8s_pod_, _global_, _generic_node_, _generic_task_, and _gce_instance_.                                                                                                                                    | global, gce_instance                                                                                                                |
| k8s_cluster_name           | The name of the cluster that the container (node or pod based on the resource type) is running in. If the resource type is one of the _k8s_container_, _k8s_node_ or _k8s_pod_, then this field is required.                                                                                  |                                                                                                                                     |
| k8s_cluster_location       | The physical location of the cluster that contains (node or pod based on the resource type) the container. If the resource type is one of the _k8s_container_, _k8s_node_ or _k8s_pod_, then this field is required.                                                                          |                                                                                                                                     |
| labels_key                 | The value of this field is used by the Stackdriver output plugin to find the related labels from jsonPayload and then extract the value of it to set the LogEntry Labels.                                                                                                                     | logging.googleapis.com/labels                                                                                                       |
| tag_prefix                 | Set the tag_prefix used to validate the tag of logs with k8s resource type. Without this option, the tag of the log must be in format of k8s_container(pod/node).\* in order to use the k8s_container resource type. Now the tag prefix is configurable by this option (note the ending dot). | k8s_container., k8s_pod., k8s_node.                                                                                                 |
| severity_key               | Specify the name of the key from the original record that contains the severity information.                                                                                                                                                                                                  |                                                                                                                                     |
| tag_prefix                 | Set the tag_prefix used to validate the tag of logs with k8s resource type. Without this option, the tag of the log must be in format of k8s_container(pod/node).\* in order to use the k8s_container resource type. Now the tag prefix is configurable by this option.                       | k8s_container., k8s_pod., k8s_node.                                                                                                 |

### Configuration File

If you are using a _Google Cloud Credentials File_, the following configuration is enough to get started:

```
[INPUT]
    Name  cpu
    Tag   cpu

[OUTPUT]
    Name        stackdriver
    Match       *
```

Example configuration file for k8s resource type:

local_resource_id is used by stackdriver output plugin to set the labels field for different k8s resource types. Stackdriver plugin will try to find the local_resource_id field in the log entry. If there is no field logging.googleapis.com/local_resource_id in the log, the plugin will then construct it by using the tag value of the log.

The local_resource_id should be in format:

* `k8s_container.<namespace_name>.<pod_name>.<container_name>`
* `k8s_node.<node_name>`
* `k8s_pod.<namespace_name>.<pod_name>`

This implies that if there is no local_resource_id in the log entry then the tag of logs should match this format. Note that we have an option tag_prefix so it is not mandatory to use k8s_container(node/pod) as the prefix for tag.

```
[INPUT]
    Name               tail
    Tag_Regex          var.log.containers.(?<pod_name>[a-z0-9](?:[-a-z0-9]*[a-z0-9])?(?:\\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-(?<docker_id>[a-z0-9]{64})\.log$
    Tag                custom_tag.<namespace_name>.<pod_name>.<container_name>
    Path               /var/log/containers/*.log
    Parser             docker
    DB                 /var/log/fluent-bit-k8s-container.db

[OUTPUT]
    Name        stackdriver
    Match       custom_tag.*
    Resource    k8s_container
    k8s_cluster_name test_cluster_name
    k8s_cluster_location  test_cluster_location
    tag_prefix  custom_tag.
```

## Troubleshooting Notes

### Upstream connection error

> Github reference: [#761](https://github.com/fluent/fluent-bit/issues/761)

An upstream connection error means Fluent Bit was not able to reach Google services, the error looks like this:

```
[2019/01/07 23:24:09] [error] [oauth2] could not get an upstream connection
```

This belongs to a network issue by the environment where Fluent Bit is running, make sure that from the Host, Container or Pod you can reach the following Google end-points:

* [https://www.googleapis.com](https://www.googleapis.com)
* [https://logging.googleapis.com](https://logging.googleapis.com)

### Fail to process local_resource_id

The error looks like this:

```
[2020/08/04 14:43:03] [error] [output:stackdriver:stackdriver.0] fail to process local_resource_id from log entry for k8s_container
```

Do following check:

* If the log entry does not contain the local_resource_id field, does the tag of the log match for format?
*   If tag_prefix is configured, does the prefix of tag specified in the input plugin match the tag_prefix?

    **Other implementations**

Stackdriver officially supports a [logging agent based on Fluentd](https://cloud.google.com/logging/docs/agent).

We plan to support some [special fields in structured payloads](https://cloud.google.com/logging/docs/agent/configuration#special-fields). Use cases of special fields is [here](stackdriver_special_fields.md).
