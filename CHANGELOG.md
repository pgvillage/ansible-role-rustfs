# Changelog

## Unreleased

### Fixed

- `rustfs_package_names` installs the client as `rustfs-cli` (was `rc`). On hosts with EPEL, `rc` installed EPEL's Plan 9 shell instead of the RustFS client.

## v0.1.0

First release of `pgvillage.rustfs`, based on `pgvillage.minio`.

### Breaking changes from `pgvillage.minio`

- All variables use the `rustfs_` prefix. The role fails when `minio_*` variables are set, and names their replacements. See [Migrating from the Minio role](docs/api.md#migrating-from-the-minio-role).
- RustFS only: Minio, `mcli` and init.d support are removed. `minio_use_rustfs` is replaced by `rustfs_remove_minio`.
- New defaults: user `rustfs`, data dir `/var/lib/rustfs`, certificates in `/etc/rustfs/tls`. The ports stay `:9091` (S3 API) and `:9001` (console).
- The certificate variables are renamed to `rustfs_client_cert`, `rustfs_server_cert`, `rustfs_server_key` and `rustfs_server_chain`; update the chainsmith config.
- The rc alias is `rustfs` (was `minio`), and `rustfs_endpoint` uses the keys `url`, `access_key`, `secret_key` and `bucket_lookup`.
- The default endpoint URL includes the port of `rustfs_server_addr`.

### Added

- The role fails when Minio is installed and `rustfs_remove_minio` is false, as both use the same ports.
- The systemd unit runs with `ProtectHome=true`, as upstream.
- Supported platforms: EL 9, EL 10, Ubuntu and Debian.
