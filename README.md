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
- The `rustfs` and `rustfs-cli` (rc) packages, built by PgVillage, from a configured package repository.
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

Testing
-------

[Molecule](https://ansible.readthedocs.io/projects/molecule/) tests the role on Rocky Linux 9 in a systemd container with Podman.
It installs the packages from the latest [rpmbuilder release](https://github.com/pgvillage-build/rpmbuilder/releases), with TLS from a test CA, and checks idempotency, the service, TLS and bucket creation.
CI runs it on every pull request.

    pip install ansible-core molecule "molecule-plugins[podman]"
    molecule test

License
-------

PostgreSQL
