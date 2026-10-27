# pgvillage.rustfs – Role API

This document describes all variables that can be set to configure the `pgvillage.rustfs` role.
Defaults are defined in [`defaults/main.yml`](../defaults/main.yml).

The role:

1. fails when variables of the Minio role (`minio_*`) are set, and names their replacements (see [Migrating from the Minio role](#migrating-from-the-minio-role)),
2. optionally asserts that certificates are set up (`rustfs_cert_managed`), and checks that the host runs systemd,
3. fails when Minio is installed, unless `rustfs_remove_minio` is true: then it stops and removes Minio, keeping its data (see [Replacing Minio](#replacing-minio)),
4. creates the RustFS user, group, data, working and client config directories,
5. installs the packages and generates the server environment file and systemd unit,
6. deploys the TLS certificates for the server and client,
7. starts RustFS and waits until the endpoint responds,
8. configures the RustFS client (`rc`) with a `rustfs` alias and creates buckets.

## Installation

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_package_names` | `[rustfs, rustfs-cli]` | Packages to install: the RustFS server and the RustFS client (rc), built by PgVillage. The client is installed as `rustfs-cli`, not `rc`: on hosts with EPEL, `rc` installs EPEL's Plan 9 shell. |
| `rustfs_server_bin` | `/usr/local/bin/rustfs` | Path to the RustFS server binary, used by the systemd unit. |
| `rustfs_client_bin` | `/usr/bin/rc` | Path to the RustFS client binary, used to create buckets. |
| `rustfs_service_name` | `rustfs` | Name of the RustFS server service. The role writes `/etc/systemd/system/<name>.service`, which overrides the unit shipped in the package. For ports below 1024 (`rustfs_server_addr`, `rustfs_server_console_addr`), it gets `CAP_NET_BIND_SERVICE`. |

## User and directories

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_user` | `rustfs` | System user that runs the RustFS server service and owns the data. Hosts migrated from Minio set this to `minio`. |
| `rustfs_group` | `{{ rustfs_user }}` | Group of the RustFS server service. |
| `rustfs_server_workdir` | `/opt/rustfs` | Working directory of the RustFS server service, as in the upstream unit. |

## Server configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_server_envfile` | `/etc/default/rustfs` | Path to the file with the environment variables for the RustFS server. It sets `RUSTFS_VOLUMES`, `RUSTFS_ADDRESS`, `RUSTFS_CONSOLE_ENABLE`, `RUSTFS_CONSOLE_ADDRESS`, `RUSTFS_TLS_PATH` and `RUSTFS_ACCESS_KEY` / `RUSTFS_SECRET_KEY`. |
| `rustfs_server_addr` | `:9091` | S3 API listen address (`[host]:port`). The same port as the Minio role, so S3 clients don't change. |
| `rustfs_server_console_enable` | `true` | Enable the RustFS console (web UI). |
| `rustfs_server_console_addr` | `:9001` | Console listen address (`[host]:port`). |
| `rustfs_server_datadirs` | `[/var/lib/rustfs]` | Data directories. Created by the role (mode `0750`) and used as `RUSTFS_VOLUMES` when `rustfs_server_cluster_nodes` is empty. Hosts migrated from Minio set this to Minio's data directories (`[/var/lib/minio]`). |
| `rustfs_server_cluster_nodes` | `[]` | Cluster node list (e.g. `https://rustfs{1...4}/var/lib/rustfs`). When set, it is used as `RUSTFS_VOLUMES` instead of `rustfs_server_datadirs`. |
| `rustfs_server_env_extra` | `""` | Extra environment variables, added verbatim to the env file (e.g. `RUSTFS_REGION=eu-west-1`). |
| `rustfs_access_key` | `AWSKEY` | Access key: `RUSTFS_ACCESS_KEY` for the server, and used by the client. **Override this.** |
| `rustfs_secret_key` | `AWSSECRET` | Secret key: `RUSTFS_SECRET_KEY` for the server, and used by the client. **Override this.** |

## Client configuration and buckets

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_endpoint` | `{}` | Overrides for the client endpoint config, merged on top of the defaults below. Used for the rc `rustfs` alias and for the startup health check. |
| `rustfs_buckets` | `[mybucket]` | Buckets to create with rc after RustFS has started. Existing buckets are left alone. |
| `rustfs_client_configdir` | `<user home>/.config/rc` | Config directory of rc, in the home dir of `rustfs_user`. The role writes `config.toml` there. |
| `rustfs_insecure` | `false` | Skip TLS certificate verification in rc. |
| `rustfs_cli_options` | `--ignore-existing` | Options added to rc commands, such as when creating buckets. |
| `rustfs_client_ca_bundle` | `<rustfs_client_configdir>/ca.pem` when `rustfs_cert_from_string`, else `""` | CA bundle (PEM) rc uses to verify the server certificate. Empty means the system CAs. |

### `rustfs_endpoint`

| Key | Default | Description |
|-----|---------|-------------|
| `url` | `https://<fqdn>:<port of rustfs_server_addr>` | Endpoint URL. |
| `access_key` | `{{ rustfs_access_key }}` | Access key. |
| `secret_key` | `{{ rustfs_secret_key }}` | Secret key. |
| `bucket_lookup` | `auto` | Bucket addressing: `auto`, `path` or `dns`. |

## Certificates

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_tls_path` | `/etc/rustfs/tls` | Directory with the server certificate, key and CA chain, set as `RUSTFS_TLS_PATH`. Outside the home dir, so the service runs with `ProtectHome=true`. |
| `rustfs_cert_managed` | `true` | Whether certificates are managed (e.g. by [chainsmith](https://github.com/MannemSolutions/chainsmith)). When true, the role asserts that `rustfs_client_cert`, `rustfs_server_cert`, `rustfs_server_key` and `rustfs_server_chain` contain real (multi-line) PEM data. |
| `rustfs_cert_from_string` | `{{ rustfs_cert_managed }}` | Deploy certificates from the `body` strings in `rustfs_cert_files` (true), or copy them from the `source` files (false). |
| `rustfs_cert_remote` | `{{ not rustfs_cert_from_string }}` | When copying from file: `source` is on the managed host (true) or on the Ansible controller (false). |
| `rustfs_client_cert` | `---- CERT ----` | CA chain (PEM) trusted by the rc client. |
| `rustfs_server_cert` | `---- CERT ----` | Server certificate (PEM). |
| `rustfs_server_key` | `---- KEY ----` | Server private key (PEM). |
| `rustfs_server_chain` | `---- CHAIN ----` | Server CA chain (PEM), which RustFS trusts for outgoing connections (e.g. to other cluster nodes). |
| `rustfs_cert_folders` | see below | Directories to create for certificates. |
| `rustfs_cert_files` | see below | Certificate files to deploy. |

### `rustfs_cert_folders`

A dict of directories. Each value has a `path` and an `owner`. The directories are created with mode `0700`. The client config directory is created with the rc config.

```yaml
rustfs_cert_folders:
  server:
    path: "{{ rustfs_tls_path }}"
    owner: "{{ rustfs_user }}"
```

### `rustfs_cert_files`

A dict of certificate files. Each file is deployed with mode `0600`. Changes notify the `Restart rustfs` handler, which runs before the startup check. Each value has these keys:

| Key | Description |
|-----|-------------|
| `path` | Destination path. Entries with an empty path are skipped. |
| `owner` | File owner (default `root`). |
| `body` | Content, used when `rustfs_cert_from_string` is true. |
| `source` | Source file, used when `rustfs_cert_from_string` is false. Entries without a `source` are skipped. |

Defaults:

| Entry | Path | Body | Source |
|-------|------|------|--------|
| `client_chain` | `rustfs_client_ca_bundle` | `rustfs_client_cert` | – |
| `server_cert` | `<rustfs_tls_path>/rustfs_cert.pem` | `rustfs_server_cert` | `/etc/pki/rustfs/server.crt` |
| `server_key` | `<rustfs_tls_path>/rustfs_key.pem` | `rustfs_server_key` | `/etc/pki/rustfs/server.key` |
| `server_chain` | `<rustfs_tls_path>/ca.crt` | `rustfs_server_chain` | `/etc/pki/rustfs/root.crt` |

## Replacing Minio

Minio and RustFS use the same ports, so they can't run side by side. When the role finds Minio (its service, systemd unit, init.d script, env file or server package), it fails unless `rustfs_remove_minio` is true.

With `rustfs_remove_minio: true`, the role stops and disables Minio, and removes its systemd unit or init.d script, env file and packages. Its user, data directories and certificates are kept. To have RustFS serve Minio's data, set `rustfs_user` and `rustfs_server_datadirs` to Minio's user and data directories.

> **Back up the data first.** RustFS reads the Minio data in place and imports the bucket settings and users from `.minio.sys` into its own `.rustfs.sys`. The switch is one-way: Minio only reads `.minio.sys`, so it would not see buckets, users or policies created or changed under RustFS. The Minio role refuses to run on a host where RustFS is or was in use. See [MinIO on-disk format interoperability](https://github.com/rustfs/rustfs/blob/main/docs/architecture/minio-file-format-compat.md).

| Variable | Default | Description |
|----------|---------|-------------|
| `rustfs_remove_minio` | `false` | Stop and remove Minio, keeping its data. |
| `rustfs_minio_service_name` | `minio` | Name of the Minio service to stop and remove. |
| `rustfs_minio_package_names` | `[minio, mcli]` | Minio packages to remove. |
| `rustfs_minio_server_envfile` | `/etc/default/minio` | Minio env file to remove. |
| `rustfs_minio_server_package` | `minio` | Minio server package. When installed, Minio is considered present; the client (`mcli`) alone doesn't conflict with RustFS. |
| `rustfs_minio_unit_files` | `/etc/systemd/system/minio.service`, `/usr/local/lib/systemd/system/minio.service`, `/etc/init.d/minio` | Minio unit files and init.d script outside package management, to detect and remove. Units in `/usr/lib` are removed with the packages. |

## Migrating from the Minio role

The role fails when any `minio_*` variable is set, and lists the variable that replaces each one (from `vars/main.yml`). Most are renamed from `minio_` to `rustfs_`. These change more:

| Minio role | This role |
|------------|-----------|
| `minio_use_rustfs` | `rustfs_remove_minio` |
| `minio_rustfs_*` | `rustfs_*` (e.g. `minio_rustfs_tls_path` → `rustfs_tls_path`) |
| `minio_rustfs_env_extra`, `minio_server_env_extra` | `rustfs_server_env_extra` (`RUSTFS_*` variables) |
| `minio_endpoint` (keys `accessKey`, `secretKey`, `api`, `path`) | `rustfs_endpoint` (keys `access_key`, `secret_key`, `bucket_lookup`) |
| `minio_package_names`, `minio_service_name`, `minio_server_envfile` | `rustfs_minio_*`: what `rustfs_remove_minio` removes |
| `minio_cert_files` | `rustfs_cert_files` (other file names) |
| `minio_package_state`, `minio_server_bin`, `minio_server_configdir`, `minio_server_opts` | no longer used |

A host migrated from Minio typically sets:

```yaml
rustfs_remove_minio: true
rustfs_user: minio
rustfs_server_datadirs:
  - /var/lib/minio
```

## Example

```yaml
- hosts: rustfs
  roles:
    - role: pgvillage.rustfs
      vars:
        rustfs_access_key: "{{ vault_rustfs_access_key }}"
        rustfs_secret_key: "{{ vault_rustfs_secret_key }}"
        rustfs_buckets:
          - backups
        rustfs_cert_managed: false
        rustfs_cert_remote: true
```
