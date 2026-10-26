RustFS
======

[RustFS](https://rustfs.com) is an S3 compatible object storage server.
PgVillage uses it to store backups in S3 buckets on a filesystem.
This role installs and configures RustFS, and can replace Minio by RustFS.
This role is part of PgVillage, which is an opinated PostgreSQL deployment for Virtual Machines.

It replaces the [pgvillage.minio](https://github.com/pgvillage/ansible-role-minio) role.
See [Migrating from the Minio role](docs/api.md#migrating-from-the-minio-role).

Requirements
------------

- systemd
- The `rustfs` and `rc` packages, built by PgVillage, from a configured package repository.
- EL 9, EL 10, Ubuntu or Debian.

Role Variables
--------------

Please see the [API docs](docs/api.md) for a description of all variables.
Defaults can be found in [defaults](defaults/main.yml).

Dependencies
------------

No dependencies

Example Playbook
----------------

    - hosts: servers
      roles:
         - role: pgvillage.rustfs
           vars:
             rustfs_access_key: "{{ vault_rustfs_access_key }}"
             rustfs_secret_key: "{{ vault_rustfs_secret_key }}"
             rustfs_buckets:
               - backups

License
-------

PostgreSQL
