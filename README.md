# qb.backup - Backuped client setup role

This repository contains the role setting up Debian (>= 8) servers that
need to be backuped.

**Important note**: This role saves on the Ansible controller running the
playbook one borg repository encryption key passphrase per backuped host. By
default, those passphrases are saved in `/tmp/backup-passphrases/` which is a
temporary folder erased at reboot, so make sure to move those passphrases to a
secure place as they will be needed to recover backups in case of a loss of
data!

You can see an example playbook implementing this role here:
[ansible-playbook-qb.backup](https://github.com/quarkslab/ansible-playbook-qb.backup).

## Manual install

On a Debian server (>= 9.0) you can manually run the following commands as root
to let the backup server handle your host. Then, provide the person running the
backup server with your server IP address, ssh port, the file
`/root/.ssh/backup.pub` and the value of `.location.repositories` in the file
`/etc/qb.backup/borgmatic.yml`.

You need to ask the person running the backup server for the SSH public key of
the backup server so that you can authorize it to connect as root to your
server to do the backups.

The minimum level of authentication we support for `PermitRootLogin` is
`forced-commands-only`, but any level above would work (`prohibit-password` or
`yes`).

Note: borg 1.1.10 is required.

```console
# echo PermitRootLogin forced-commands-only > /set/ssh/sshd_config # This can be set to a higher level of access instead
# echo PermitUserEnvironment yes > /set/ssh/sshd_config
# # Remove any sshd configuration options that may collide with those above.
# systemctl restart sshd
# apt install -y libc6 python3-dev python3-pip python3-setuptools
# wget https://github.com/borgbackup/borg/releases/download/1.1.10/borg-linux64 -O /usr/local/bin/borg1.1.10
# ln -sf borg1.1.10 /usr/local/bin/borg
# pip install --user borgmatic
# ssh-keygen -t ed25519 -f /root/.ssh/backup -N "" -C "backup@$(cat /etc/hostname)"
# mkdir -m 700 /etc/qb.backup
# dd if=/dev/urandom bs=1 count=24 |base64 >/etc/qb.backup/BACKUP_PASSPHRASE
# # Create the file /etc/qb.backup/borgmatic.yml based on the example below
# echo 'environment="BORG_PASSCOMMAND=cat /etc/qb.backup/BACKUP_PASSPHRASE",command="/root/.local/bin/borgmatic -c /etc/qb.backup/borgmatic.yml --init --encryption repokey --create --check --prune" (SSH Public key of the backup server here without the parenthesis!) backup-master' > /root/.ssh/authorized_keys
# echo $(cat /etc/qb.backup/BACKUP_PASSPHRASE) " < keep this passphrase in a safe place (eg. your password manager), it will be needed to restore backups for this host"
```

### Example borgmatic.yml file

```yaml
location:
  local_path: /usr/local/bin/borg
  source_directories: ["/boot", "/etc", "/home", "/opt", "/root", "/srv", "/usr", "/var"]
  repositories:
    - backup-master:/mnt/backup/ # followed by an unique value across servers (hostname or random string), such as backup-master:/mnt/backup/EMx2ft3tsA
  exclude_patterns: ["*.pyc", "*.swp", "*.swo", "*/lost+found", "*/tmp", "*/.cache", "/var/cache", "re:^/var/lib/docker/(?!volumes)", "/var/lock", "/var/run", "/var/tmp"]
  exclude_caches: true
  exclude_if_present: .nobackup
storage:
  ssh_command: ssh -i /root/.ssh/backup -o BatchMode=yes -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o HostName=localhost -p 64064 -l backup -i /root/.ssh/backup
  archive_name_format: "{hostname}-{now:%Y-%m-%dT%H:%M:%S}"
retention:
  keep_daily: 7
  keep_weekly: 4
  keep_monthly: 12
  keep_yearly: 10
  prefix: "{hostname}-"
hooks:
  before_backup: []
  after_backup: []
  on_error: []
```

Note: since the server works in `append-only` mode for security reasons, the
retention settings will only mark expired blocks as deletable, but they won't
actually be deleted! You will need to manually run borg without `append-only` so
that they are actually deleted.
