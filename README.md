# Ansible role "papanito.backup" <!-- omit in toc -->

[![Ansible Role](https://img.shields.io/ansible/role/57847)](https://galaxy.ansible.com/papanito/backup) [![Ansible Quality Score](https://img.shields.io/ansible/quality/57847)](https://galaxy.ansible.com/papanito/backup) [![Ansible Role](https://img.shields.io/ansible/role/d/57847)](https://galaxy.ansible.com/papanito/backup) [![GitHub issues](https://img.shields.io/github/issues/papanito/ansible-role-backup)](https://github.com/papanito/ansible-role-backup/issues) [![GitHub pull requests](https://img.shields.io/github/issues-pr/papanito/ansible-role-backup)](https://github.com/papanito/ansible-role-backup/pulls)

Ansible role do install and setup regular backups with either

- [borg]
- [restic]

The role performs the following steps:

- [Optional] Delete existing repository
- Initialize a repository
  - [borg][borg init]: 
    - `backup_borg_protocol`://`backup_borg_server`:`backup_target_dir` for a remote backup or
    - `backup_target_dir` for a local backup
  - [restic][restic init]:
    - `backup_restic_repo` for a remote backup or
    - `backup_target_dir` for a local backup

  > **Notes**
  >
  > In case the repo `backup_target_dir` already exists, the initalization will be skipped
  > If `backup_borg_server` nor `backup_restic_repo` are not specified role assumes a local backup i.e. to a local directory

- Create a `systemd` service which regularly (according to `backup_schedule`) runs script
  - [borg]: `borg.sh` from [borgbackup.org](https://borgbackup.readthedocs.io/en/stable/quickstart.html#automating-backups)
  - [restic]: `restic.sh` a modified `borg.sh` script considering restic command
- There will be an individual script named `automatic-backup-{{service_name}}.sh` in `/opt/backup` which is customized with

  - `backup_source_dir`
  - `backup_exclude_file` or `backup_exclude_list`
  - `backup_schedule`


## Requirements

None

## Role Variables

To keep sensitive information hidden I recommend to use [`ansible-vault`](https://docs.ansible.com/ansible/latest/user_guide/vault.html)

You can define the password file in `ansible.cfg` so none vault parameter has to be specified. Thus, the encrypted variable `backup_borg_encryption_key` can be created as follows:

```bash
ansible-vault encrypt_string'SupersecretPa$$phrase' --name 'backup_borg_encryption_key'
```

### Common Variables

These are all variables

| Parameter| Description | Default Value |
| -------- | ----------- | ------------- |
| `backup_engine`| Name of engine (`borg` or `restic` to use for your backups | - |
| `backup_name`| [mandatory] Name of backup | |
| `backup_target_dir` | Target directory of the backups on the `backup_borg_server` or `backup_restic_repo` | `"./backups/{{ backup_name }}"` |
| `backup_delete`| **WARNING** If set to `true` then existing backup repository will be [deleted](https://borgbackup.readthedocs.io/en/stable/usage/delete.html)| `false` |
| `backup_create`| Creation of repository. You can use the role to explicitly delete an existing `repository` by running the role with `-e backup_delete=true -e backup_create=false` | `true`|
| `systemd_script_user` | User for permissions of script | `root` |
| `systemd_script_group` | User group for permissions of script | `wheel` |
| `backup_schedule`| Systemd schedule notation for the daily backup to run| `*-*-* 03:00:00`|
| `backup_include_list`| List of source directories to backup | - |
| `backup_exclude_file`| [`EXCLUDEFILE`](https://borgbackup.readthedocs.io/en/stable/usage/create.html) which contains exclude patterns<br>Takes precedence over `backup_exclude_list`| - |
| `backup_exclude_list`| List of patterns which will be added as `--exclude 'PATTERN'`| - |

### Restic specific variables

| Parameter| Description | Default Value |
| -------- | ----------- | ------------- |
| `backup_restic_repo`| Name of repo endpoint. `local` will create a local repo using `backup_target_path`. For anything else specify the `Repo format` according to the list below | `local` |
| `backup_exclude_larger_than` | Specified once to excludes files larger than the given size | - |

Restic supports the following repo types with the expected repo format for `backup_restic_repo`:

| Repo Type | Repo format |
| --------- | ----------- |
| [local](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#local) | `/srv/restic-repo` |
| [rest server](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#rest-server) | `rest:https://user:pass@host:8000/PATH` |
| s3 ([Amazon S3](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#amazon-s3), [minio](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#minio-server), ...) | `s3:s3.amazonaws.com/bucket_name/PATH` |
| [sftp](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#sftp) | `sftp:user@host:PATH` |
| [Alibaba Cloud (Aliyun) <br>Object Storage System (OSS)](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#alibaba-cloud-aliyun-object-storage-system-oss) | `-o s3.bucket-lookup=dns -o s3.region=<OSS-REGION> -r s3:https://<OSS-ENDPOINT>/<OSS-BUCKET-NAME>` |
| Openstack Swift | see [here](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#openstack-swift) |
| [Backblaze B2](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#backblaze-b2) | `b2:bucketname:PATH` |
| [Microsoft Azure Blob Storage](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#microsoft-azure-blob-storage) | `azure:foo:/PATH` |
| [Google Cloud Storage](https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html#google-cloud-storage) | `gs:foo:/PATH` |

Restic also need environment variables for authentication, so they have to be set by defining them in your playbook

| Parameter| Description | Default Value |
| -------- | ----------- | ------------- |
| AWS_ACCESS_KEY_ID | Amazon S3 access key ID | - |
| AWS_SECRET_ACCESS_KEY | Amazon S3 secret access key | - |
| AWS_DEFAULT_REGION | Amazon S3 default region | - |
| ST_AUTH | Auth URL for keystone v1 authentication | - |
| ST_USER | Username for keystone v1 authentication | - |
| ST_KEY | Password for keystone v1 authentication | - |
| OS_AUTH_URL | Auth URL for keystone authentication | - |
| OS_REGION_NAME | Region name for keystone authentication | - |
| OS_USERNAME | Username for keystone authentication | - |
| OS_USER_ID | User ID for keystone v3 authentication | - |
| OS_PASSWORD | Password for keystone authentication | - |
| OS_TENANT_ID | Tenant ID for keystone v2 authentication | - |
| OS_TENANT_NAME | Tenant name for keystone v2 authentication | - |
| OS_USER_DOMAIN_NAME | User domain name for keystone authentication | - |
| OS_USER_DOMAIN_ID | User domain ID for keystone v3 authentication | - |
| OS_PROJECT_NAME | Project name for keystone authentication | - |
| OS_PROJECT_DOMAIN_NAME | Project domain name for keystone authentication | - |
| OS_PROJECT_DOMAIN_ID | Project domain ID for keystone v3 authentication | - |
| OS_TRUST_ID | Trust ID for keystone v3 authentication | - |
| OS_APPLICATION_CREDENTIAL_ID | Application Credential ID (keystone v3) | - |
| OS_APPLICATION_CREDENTIAL_NAME | Application Credential Name (keystone v3) | - |
| OS_APPLICATION_CREDENTIAL_SECRET | Application Credential Secret (keystone v3) | - |
| OS_STORAGE_URL | Storage URL for token authentication | - |
| OS_AUTH_TOKEN | Auth token for token authentication | - |
| B2_ACCOUNT_ID | Account ID or applicationKeyId for Backblaze B2 | - |
| B2_ACCOUNT_KEY | Account Key or applicationKey for Backblaze B2 | - |
| AZURE_ACCOUNT_NAME | Account name for Azure | - |
| AZURE_ACCOUNT_KEY | Account key for Azure | - |
| GOOGLE_PROJECT_ID | Project ID for Google Cloud Storage | - |
| GOOGLE_APPLICATION_CREDENTIALS | Application Credentials for Google Cloud Storage | - |

### Borg specific variables

| Parameter| Description | Default Value |
| -------- | ----------- | ------------- |
| `backup_borg_server`| Name of the backup server - if not defined, it assumes a local backup| - |
| `backup_user`| Name of the user to connect to the server| - |
| `backup_borg_protocol` | backup_borg_protocol used to connect to `backup_borg_server`| `ssh` |
| `backup_port`| Port to connect to `backup_borg_server` | - |
| `backup_borg_encryption_key`| [mandatory] Passphrase for the encryption key using `repokey`| - |
| `backup_borg_encryption_method` | Borg [encryption method](https://borgbackup.readthedocs.io/en/stable/usage/init.html#encryption-modes), currently only `repokey` implemented | `repokey` |

### Systemd Service specific variables

The following parameters are related to the systemd service file:

| Parameter| Description | Default Value|
| -------- | ----------- | ---------- |
| `backup_systemd_backup_target_dir` | Location where to copy `.service`-files| `/etc/systemd/system/` |
| `backup_systemd_user` | User for systemd service | `backup` |
| `backup_systemd_group`| Group for systemd service| `backup` |
| `backup_systemd_on_failure` | If set it will make an [OnFailure] entry in the service file | `-`|
| `systemd_script_mode`| Mode of the script file| `0774`|
| `systemd_service_mode`| Mode of the service file| `0644`|

## Pruning / Cleanup of old backups

The script which is deployed also defines the options for [`prune` (borg)] or [`forget` (restic)]

The variables are not set and have to be explicitly defined in your playbook:

| Parameter | Description | Default Value |
| --------- | ----------- | ------------- |
| `backup_prune_keep_within` | `--keep-within INTERVAL` keep all archives within this time interval | - |
| `backup_prune_keep_last` | `--keep-last, --keep-secondly INTERVAL` number of secondly archives to keep | - |
| `backup_prune_keep_minutely` | `--keep-minutely INTERVAL` number of minutely archives to keep| - |
| `backup_prune_keep_hourly` | `-H, --keep-hourly INTERVAL` number of hourly archives to keep| - |
| `backup_prune_keep_daily`| `-d, --keep-daily INTERVAL` number of daily archives to keep| - |
| `backup_prune_keep_weekly` | `-w, --keep-weekly INTERVAL` number of weekly archives to keep| - |
| `backup_prune_keep_monthly`| `-m, --keep-monthly INTERVAL` number of monthly archives to keep| - |
| `backup_prune_keep_yearly` | `-y, --keep-yearly INTERVAL` number of yearly archives to keep| - |
| `backup_prune_save_space`| `--save-space` work slower, but using less space | `false` |

For [borg][`prune` (borg)] there are some additional variables

| Parameter | Description | Default Value |
| --------- | ----------- | ------------- |
| `backup_prune_dryrun`| `-n, --dry-run` do not change repository | `false` |
| `backup_prune_force` | `--force` force pruning of corrupted archives| `false` |
| `backup_prune_stats` | `-s, --stats` print statistics for the deleted archive | `true`|
| `backup_prune_list`| `--list` output verbose list of archives it keeps/prunes | `true`|

For [restic][`forget` (restic)] there are some additional variables

| Parameter | Description | Default Value |
| --------- | ----------- | ------------- |
| `backup_prune_keep_within_hourly` | `-y, --keep-within-hourly INTERVAL` keep all hourly snapshots made within specified duration of the latest snapshot. The duration is specified in the same way as for `--keep-within` and the method for determining hourly snapshots is the same as for `--keep-hourly` | - |
| `backup_prune_keep_within_daily` | `-y, --keep-within-daily INTERVAL` keep all daily snapshots made within specified duration of the latest snapshot | - |
| `backup_prune_keep_within_weekly` | `-y, --keep-within-weekly INTERVAL` keep all weekly snapshots made within specified duration of the latest snapshot | - |
| `backup_prune_keep_within_monthly` | `-y, --keep-within-monthly INTERVAL` keep all monthly snapshots made within specified duration of the latest snapshot | - |
| `backup_prune_keep_within_yearly` | `-y, --keep-within-yearly INTERVAL` keep all yearly snapshots made within specified duration of the latest snapshot | - |


## Dependencies

None

## Examples

### Example Playbook remote backup

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yaml
- hosts: localhost
vars:
- backup_engine: borg
- backup_borg_server: borg.intra
- backup_user: borguser
- backup_name: mybackupname
- backup_borg_encryption_key: test
- backup_port: 23
- backup_target_dir: "/var/backups/"
- backup_schedule: "*-*-* 03:00:00"
- backup_exclude_list:
- "*/Downloads"
- "*/google-chrome*"
- backup_include_list:
- /home/papanito
- backup_prune_keep_daily: 7
- backup_prune_keep_weekly: 5
- backup_prune_keep_monthly: 6
- backup_prune_keep_yearly: 1

roles:
- role: papanito.backup
```

This will create a backup at `ssh://borguser@borg.intra:/var/backup/mybackupname` and the following systemd files

- `/opt/borg_backup/automatic-backup-mybackupname-borg.intra.sh` (backup script)
- `/etc/systemd/system/automatic-backup-mybackupname-borg.intra.service` (systemd service file)
- `/etc/systemd/system/automatic-backup-mybackupname-borg.intra.timer` (systemd timers file)

### Example Playbooks

#### local backup with borg

Including an example of how to use your role (for instance, with variables passed in as parameters) is always nice for users too:

```yaml
- hosts: localhost
  vars:
  - backup_engine: borg
  - backup_name: mybackupname
  - backup_borg_encryption_key: test
  - backup_target_dir: "/var/backup/"
  - backup_schedule: "*-*-* 03:00:00"
  - backup_exclude_list:
    - "*/Downloads"
    - "*/google-chrome*"
  - backup_include_list:
    - /home/papanito
  - backup_prune_keep_daily: 7
  - backup_prune_keep_weekly: 5
  - backup_prune_keep_monthly: 6
  - backup_prune_keep_yearly: 1

  roles:
    - papanito.backup
```

This will create a backup at `/var/backup/mybackupname` and the following systemd files

- `/opt/borg_backups/automatic-backup-mybackupname-local.sh` (backup script)
- `/etc/systemd/system/automatic-backup-mybackupname-local.service` (systemd service file)
- `/etc/systemd/system/automatic-backup-mybackupname-local.timer` (systemd timers file)

#### Remote backup with restic

```yaml
- hosts: localhost
  remote_user: root
  vars:
  - backup_engine: restic
  - backup_include_list :
    - /home/aedu/Downloads/Fotos
  - backup_engine: restic
  - backup_systemd_user: root
  - backup_name: test
  - backup_borg_encryption_key: test
  - backup_target_dir: "/test"
  - backup_delete: true
  - backup_create: true
  - backup_source_dir: ./defaults
  - B2_ACCOUNT_ID: XXXX
  - B2_ACCOUNT_KEY: XXXX

  roles:
    - papanito.backup
```

## License

This is Free Software, released under the terms of the Apache v2 license.

## Author Information

Written by [Papanito](https://wyssmann.com) - [Gitlab](https://gitlab.com/papanito) / [Github](https://github.com/papanito)


[OnFailure]: https://dyn.manpages.debian.org/buster/systemd/systemd.unit.5.en.html#%5BUNIT%5D_SECTION_OPTIONS[OnFailure]
[borg]: https://github.com/borgbackup/borg
[borg init]: https://borgbackup.readthedocs.io/en/stable/usage/init.html
[restic]: https://restic.readthedocs.io
[`prune` (borg)]: https://borgbackup.readthedocs.io/en/stable/usage/prune.x
html
[`forget` (restic)]: https://restic.readthedocs.io/en/stable/060_forget.html