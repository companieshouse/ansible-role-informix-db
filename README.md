# Ansible Role: Informix Databases

An [Ansible Galaxy](https://galaxy.ansible.com/) role for configuring IBM Informix server instances, including configuration files (i.e. `onconfig` and `sqlhosts`), as well as initialising a root dbspace and preparing additional dbspaces and chunks according to configuration provided.

> [!WARNING]
> There may be a **risk of data loss** when using this role against previously provisioned hosts. For additional safety, this role creates a lock file in `/etc` and will refuse to execute again so long as the lock file exists. See [Reprovisioning Hosts](#reprovisioning-hosts) for more information should this role need to be applied to hosts that contain existing data.

## Table of contents

* [Role Variables][1]
* [Informix Database Configuration][2]
  * [Server Connections Configuration][3]
    * [Default Server Connections Configuration][4]
    * [Host Specific Server Connections][5]
  * [DBSpaces Configuration][6]
  * [Chunk Configuration][7]
  * [Informix User Configuration][8]
  * [Informix Event Alarms Configuration][9]
  * [Example Configuration][10]
* [Reprovisioning Hosts][11]
* [Example Requirements File][12]
* [Example Playbook][13]
* [License][14]

[1]: #role-variables
[2]: #informix-database-configuration
[3]: #server-connections-configuration
[4]: #default-server-connections-configuration
[5]: #host-specific-server-connections
[6]: #dbspaces-configuration
[7]: #chunk-configuration
[8]: #informix-user-configuration
[9]: #informix-event-alarms-configuration
[10]: #example-configuration
[11]: #reprovisioning-hosts
[12]: #example-requirements-file
[13]: #example-playbook
[14]: #license

## Role Variables

The following role variables are required, however default values may be used where applicable:

| Name                             | Default | Description                                            |
|----------------------------------|---------|--------------------------------------------------------|
| `informix_db_chunk_store_path`   | `/data/informix/chunks` | The path of the directory for storing cooked files for dbspace chunks (when not using raw disks). This variable should be referenced in the [Dbspaces configuration][6] `path` option for any dbspace chunks that are to be represented using cooked files (e.g. `{{ informix_db_chunk_store_path }}/rootdbs`). |
| `informix_db_dbdate`             | `DMY4`     | The value to be used for the `DBDATE` environment variable. |
| `informix_db_install_path`       | `/opt/informix/14.10` | The path to the Informix installation directory. |
| `informix_db_logs_path`          | `/var/log/informix` | The path to the directory where Informix logs will be stored. |
| `informix_db_server_name`        |            | The name of the database server to be provisioned, for which a key exists in the `informix_db_config` dictionary with suitable configuration. |
| `informix_db_server_name_suffix` |            | A suffix value that will be appended to the Informix server name in configuration files and environment variables to differentiate servers when using High Availability Data Replication (HDR). For example, the values `_primary` and `_secondary`. |
| `informix_db_service_group`      | `informix` | The user to be used for ownership of configuration files and directories. |
| `informix_db_service_user`       | `informix` | The group to be used for ownership of configuration files and directories. |
| `informix_db_tape_device_path`   | `/db_dump` | The path to be used for the Informix tape device setting. |

## Informix Database Configuration

The `informix_db_config` dictionary variable controls the configuration of Informix databases, dbspaces, chunks and server connections. This variable should be defined as a dictionary of dictionaries, whose keys represent unique database server instances (such keys are analogous to the 'server name' in this context):

```yaml
informix_db_config:
  server1:
    ...
  server2:
    ...
```

:notebook: All configuration keys that follow are required unless marked _optional_.

Each nested dictionary within `informix_db_config` represents an individual database server (e.g. `server1` and `server2` in the above example) and supports the following key:

| Name                 | Default | Description                                            |
|----------------------|---------|--------------------------------------------------------|
| `server_id`          |         | A unique numeric identifier for this server instance   |
| `server_port`        |         | _Optional_. The port number this server instance will bind to for TCP/IP connections when using the default `server_connections` value. If a `server_connections` key has been defined then `server_port` should be omitted, and the port number should be specified in the relevant field of the `server_connections` list (if required for that connection type). |
| `server_aliases`     |         | _Optional_. A comma-separated string containing one or more database server aliases. |
| `server_connections` | See [Server Connections Configuration][3] for defaults. | A list of dictionaries representing client/server connections for this server (for constructing the `sqlhosts` configuration file). See [Server Connections Configuration][3] for more details. Connections specified for this key are common to all remote hosts provisioned by this role. To specify host-specific connections see [Host Specific Server Connections][5]. |
| `dbspaces`           |         | A dictionary of uniquely named dbspaces. Must include at least a `root` dbspace. See [Dbspaces configuration][6] for more details.
| `users`              |         | _Optional_. A list of dictionaries specifying user accounts for Informix connections. See [Informix User Configuration][8] for more information. |
| `logical_log_size`   | `10000` | _Optional_. The size of each logical log in kilobytes. |
| `logical_log_number_files` | `100` | _Optional_. The number of logical log files. |
| `event_alarms_enabled` | `false` | _Optional_. A boolean value representing whether event alarms are enabled or not. See [Informix Event Alarms Configuration][9] for more information. |

### Server Connections Configuration

If defined, the _optional_ `server_connections` key must specify a list of dictionaries. If omitted, a default set of server connections will be used; see [Default Server Connections Configuration][4] for more information. Each dictionary supports the following keys:

| Name                 | Default |                         | `sqlhosts` configuration file field name |
|----------------------|---------|-------------------------|------------------------------|
| `server_name`        |         | A unique database server name for this connection. | `dbservername` |
| `connection_type`    |         | The connection type to use (e.g. `onipcshm` for shared memory segment or `onsoctcp` for TCP/IP connection) | `nettype` |
| `host`               |         | The host computer for the database server. | `hostname` |
| `service_or_port`    |         | The service name or port number dependent upon `connection_type`. | `servicename` |
| `options`            |         | Any options that describe or limit the connection | `options` |

#### Default Server Connections Configuration

A default set of server connections will be used if no `server_connections` are specified for a given database server (see [Informix Database Configuration][2]) _and_ no host-specific connections have been specified for the same database server in the `informix_db_host_connections` variable (see [Host Specific Server Connections][5]). These default connections include:

1. A _shared memory_ connection — the `server_name` value for this connection will be set using the parent dictionary key that this connection belongs to (e.g. `server1` in the example given during the introductory section of [Informix Database Configuration][2])
2. A socket connection using TCP/IP for client applications — the `server_name` value for this connection will be set using a combination of the parent dictionary key that this connection belongs to and a `tcp` suffix (e.g. `server1tcp` in the example given during the introductory section of [Informix Database Configuration][2])

#### Host Specific Server Connections

Server connections specified using the `server_connections` key (see [Informix Database Configuration][2]) are common to _all_ remote hosts provisioned by this role. To define host-specific connections, add the variable `informix_db_host_connections` to one or more group or host vars files. This variable should be defined as a dictionary of dictionaries whose keys represent the same database server instances as those used in the `informix_db_config` variable (see [Informix Database Configuration][2]).

For example, to define a connection that is specific to a remote EC2 host whose name tag matches the value `fil-tuxedo-staging-1`, add the following configuration to the group vars file named `tag_Name_fil_tuxedo_staging_1`:

```yaml
informix_db_host_connections:
  ef:
    - server_name: example
      connection_type: onsoctcp
      host: instance-1.fil.tuxedo.staging.heritage.aws.internal
      service_or_port: 1234
```

In this example, the single connection above would be combined with any connections already specified in the `server_connections` key of the `informix_db_config` variable for the same top-level database server named `ef`. If the `informix_db_config` contains no shared `server_connections`, then only the single host-specific connection above will be used for the matching remote host.

### DBSpaces Configuration

The `dbspaces` key must be a dictionary of dictionaries whose keys represent uniquely named dbspaces for the server that they belong to. A `root` key must be defined for each dbspace—representing the root dbspace—and each dbspace dictionary may specify the following options:

| Name                 | Default |                                             |
|----------------------|---------|---------------------------------------------|
| `initial_chunk`      |         | A dictionary representing the initial chunk for the root dbspace. See [Chunk Configuration][7] for more details. |
| `additional_chunks`  |         | _Optional_. A list of one or more dictionaries representing additional chunks to be added to the dbspace. See [Chunk Configuration][7] for more details. |

### Chunk Configuration

The `initial_chunk` and `additional_chunks` keys both represent chunks belonging to a dbspace. The former is mandatory and takes the form of a dictionary, while the later is an optional list of dictionaries. In either case, the following keys are supported:

| Name           | Default |                                                                |
|----------------|---------|----------------------------------------------------------------|
| `path`         |         | The path to a character device or file on disk for this chunk. |
| `offset_in_kb` |         | The offset in kB for this chunk.                               |
| `size_in_kb`   |         | The size in kB for this chunk.                                 |

Observations to consider when configuring dbspace chunks:

* Chunks are assumed to be cooked files if the `path` does not refer to a block device, and a suitable file will be created at the specified path using `informix:informix` ownership and `0660` permissions before adding the chunk to a dbspace.
* Chunks that belong to different cooked files should use an offset value of `0`. Chunks that belong to the same cooked file as other chunks should typically use an `offset_in_kb` value equal the sum of the `offset_in_kb + size_in_kb` of the previous chunk with the same path.
* Chunks that belong to raw disks should use an offset sufficient to ensure that they do not overlap with existing data on the disk or other chunks.

### Informix User Configuration

If defined, the _optional_ `users` key must specify a list of dictionaries. Each dictionary supports the following keys:

| Name                 | Default |                                                                                                 |
|----------------------|---------|-------------------------------------------------------------------------------------------------|
| `name`               |         | A unique user account name for connectivity to the database server under which this key exists. |


User configuration will be read from HashiCorp Vault using the path specified by the `informix_db_vault_path` variable, and this configuration is expected to contain a JSON object with a top-level key corresponding to the database server name. User names should be specified within a `users` key as an object containing a single `password` key and string value. For example, given a single database server named `example-server` and user named `example-user`:

```json
{
  "example-server": {
    "users": {
      "example-user": {
        "password": "..."
      }
    }
  }
}
```

All user accounts are created as non-login accounts using `/sbin/nologin` as the default shell, with no password expiry.

### Informix Event Alarms Configuration

Event alarms configuration will be read from HashiCorp Vault using the path specified by the `informix_db_vault_path` variable, and this configuration is expected to contain a JSON object with a top-level key corresponding to the database server name. An `event_alarms` key should be set to an object containing the keys `email_addresses` and `level` which are used to configure the event alarms. The value for `level` should correspond to the desired [event severity](https://www.ibm.com/docs/en/informix-servers/14.10?topic=alarms-event-alarm-parameters#ids_adr_0674__title__3) level for alarms. For example, given a single database server named `example-server` and using the email address `user@nowhere.com` for events of severity level `3` or greater:

```json
{
  "example-server": {
    "event_alarms": {
      "email_addresses": "user@nowhere.com",
      "level": "3"
    }
  }
}
```

### Example Configuration

The example that follows shows how to define the configuration for a single server instance named `server1` that meets the following criteria:

* One Informix server instance `server1` with server identifier `1`
* A `root` dbspace composed of one initial cooked file chunk of size 1GiB
* An additional `data` dbspace with initial cooked file chunk of size 1GiB and an additional chunk of size 2GiB (both belonging to the same filesystem object)
* The two [default connection types][4] (i.e. a shared memory segment and TCP/IP connection respectively) with port `1234` used for TCP/IP connectivity

```yaml
informix_db_config:
  server1:
    server_id: 1
    server_port: 1234
    dbspaces:
      root:
        initial_chunk:
          path: "/informixchunks/rootdbs"
          offset_in_kb: 0
          size_in_kb: 1048576
      data:
        initial_chunk:
          path: "/informixchunks/datadbs"
          offset_in_kb: 0
          size_in_kb: 1048576
        additional_chunks:
          - path: "/informixchunks/datadbs"
            offset_in_kb: 1048576
            size_in_kb: 1048576
```

## Reprovisioning Hosts

> [!WARNING]
> This role is _not_ idempotent and there is a **risk of data loss** if the role is executed against hosts that contain existing dbspaces and chunks. A lock file is created in `/etc` for each database server instance provisioned by this role to provide a basic level of protection against this. These lock files are used to distinguish whether a database server was previously provisioned with this role, and the role will refuse to execute again until manual intervention is taken.

To provision a database server on a remote host with this role a second time:

* Confirm that there are no dbpsaces or chunks present for the database server(s) on the remote host(s) (i.e. check all `path` references in the `informix_db_config` configuration and confirm that those paths—whether they reference cooked files or raw disks—do not contain actual data on the remote host(s))
* Remove the corresponding lock file(s) for the database server(s) affected from `/etc`—these follow the naming convention `informix-db-provisioned-<server>`
* Stop any active `oninit` processes on the remote host(s) for the affected database server(s)
* Remove any shared memory segments that were created by `oninit` processes for the affected database server(s)
* Rerun this role against the same database server(s) and host(s)

## Example Requirements File

```yml
- src: https://github.com/companieshouse/ansible-role-informix-db
  name: informix-db
  version: "n.n.n"
```

## Example Playbook

```yml
    - hosts: servers
      roles:
        - informix-db
```

## License

This project is subject to the terms of the [MIT License](/LICENSE).
