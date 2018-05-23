---
id: logging
title: Logging
permalink: guides/sync-gateway/logging/index.html
---

Continuous logging is a new feature in Sync Gateway 2.1 that allows the console log output to be separated from log files collected by Couchbase Support.

This allows system administrators running Sync Gateway to tweak log level, and log keys for the console output to suit their needs, whilst maintaining the level of logging required by Couchbase Support for investigation of issues.

## Console Log output

The console output of Sync Gateway can be filtered down via log level and log keys, and you can tweak this as much as you like without impacting Support's ability to analyze the log files described in [Log File Outputs](#log-file-outputs).
There is an additional option to color log output based on log level if `logging.console.color_enabled` is set to `true` (Linux/MacOS only).

### Log Levels

The console log output can be configured with the following log levels, ordered from least verbose, to most:

Also note that log levels are additive, so if you enable `info` level, `warn` and `error` logs are also enabled.

| Log Level | Appearance  | Description                                                            |
| ---------:|:----------- | ---------------------------------------------------------------------- |
|    `none` | -           | Disables log output                                                    |
|   `error` | `[ERR]`     | Displays errors that need urgent attention                             |
|    `warn` | `[WRN]`     | Displays warnings that need some attention                             |
|    `info` | `[INF]`     | Displays information about normal operations that don't need attention |
|   `debug` | `[DBG]`     | Displays verbose output that might be useful when debugging            |
|   `trace` | `[TRC]`     | Displays extremely verbose output that might be useful when debugging  |

### Log Keys

Log keys are used to provide finer-grained control over what is logged. By default, only `HTTP` is enabled.

| Log Key     | Description                                                                                                                                                                                                                                                                                            |
| ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `*`         | Enables all log keys                                                                                                                                                                                                                                                                                   |
| `Admin`     | Admin processes in Sync Gateway                                                                                                                                                                                                                                                                        |
| `Accel`     | Sync Gateway Accelerator                                                                                                                                                                                                                                                                               |
| `Access`    | Anytime an `access()` call is made in the sync function                                                                                                                                                                                                                                                |
| `Auth`      | Authentication                                                                                                                                                                                                                                                                                         |
| `Bucket`    | Sync Gateway interactions with the bucket (`trace` level only)                                                                                                                                                                                                                                         |
| `Cache`     | Interactions with Sync Gateway's in-memory channel cache                                                                                                                                                                                                                                               |
| `Changes`   | Processing of _changes requests                                                                                                                                                                                                                                                                        |
| `CRUD`      | Updates made by Sync Gateway to documents                                                                                                                                                                                                                                                              |
| `DCP`       | DCP-feed processing                                                                                                                                                                                                                                                                                    |
| `Events`    | Event processing (webhooks)                                                                                                                                                                                                                                                                            |
| `gocb`      |                                                                                                                                                                                                                                                                                                        |
| `HTTP`      | All requests made to the Sync Gateway REST APIs                                                                                                                                                                                                                                                        |
| `HTTP+`     | Additional information about HTTP requests (response times, status codes)                                                                                                                                                                                                                              |
| `Import`    | Introduced in Sync Gateway 1.5 to help troubleshoot the import process of a document (this is the Sync Gateway process to make a document that was added through N1QL or the Server SDKs mobile-aware). This log key can be useful to troubleshoot why a given document was not successfully imported. |
| `Migrate`   |                                                                                                                                                                                                                                                                                                        |
| `Query`     |                                                                                                                                                                                                                                                                                                        |
| `Replicate` | Log messages related to replications between Sync Gateways (using sg-replicate). This tag cannot be used for replications initiated by Couchbase Lite                                                                                                                                                  |
| `Shadow`    |                                                                                                                                                                                                                                                                                                        |
| `Sync`      | Activity which relates to synchronization between Couchbase Lite and Sync Gateway                                                                                                                                                                                                                      |
| `SyncMsg`   | Additional information about Sync                                                                                                                                                                                                                                                                      |
| `WS`        |                                                                                                                                                                                                                                                                                                        |
| `WSFrame`   |                                                                                                                                                                                                                                                                                                        |

### Console Output redirection

The log files described below are intended for Couchbase Support, and users are urged not to rely on these.

If you have special requirements for logs, such as centralized logging, you can redirect the console output to a file, and apply your own log collection mechanism to feed that data elsewhere.
For example:

    # Start Sync Gateway and redirect console output to a file
    ./sync-gateway > my_sg_logs.txt 2>&1
    
    # Start log collection to send to a centralized log aggregator.
    logcollector my_sg_logs.txt


## Log File Outputs

These are 4 log files split by log level, with a guaranteed retention period for each.
The log files can be collected with [SGCollect Info](../../../guides/sync-gateway/sgcollect-info/index.html), and can be analyzed by Couchbase Support for diagnosing issues in Sync Gateway.
As described above, it is recommended to use [Console Output redirection](#console-output-redirection) if you require special handling of log files from Sync Gateway, as these files are intended for Couchbase Support.

| Log File       | Default `enabled` | Default `max_age` | Minimum `max_age` |
| --------------:| ----------------- | -----------------:| -----------------:|
| `sg_error.log` | `true`            |          360 Days |          180 Days |
|  `sg_warn.log` | `true`            |          180 Days |           90 Days |
|  `sg_info.log` | `true`            |            6 Days |            3 Days |
| `sg_debug.log` | `false`           |            2 Days |             1 Day |

 
Each log level and its parameters are described in the [`logging.$level`](../../../guides/sync-gateway/config-properties/index.html#2.1/logging-$level) property reference.

### Log File Rotation

These four log files will be rotated once each exceeds a threshold size, defined by `max_size` in megabytes.
Once rotated, the log files will be compressed with gzip, to reduce the disk space taken up by older logs.
These old logs will then be cleaned up once the age exceeds `max_age` in days.

## Log Redaction

All log outputs can be redacted, this means that user-data, considered to be private, is removed. This feature is optional and can be enabled in the configuration with the [`logging.redaction_level`](../../../guides/sync-gateway/config-properties/index.html#2.1/logging-redaction_level) property.