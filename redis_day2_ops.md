# Redis Day 2 Operations — Ansible Playbook

A single-playbook runbook for Redis routine operations: status check, service lifecycle management, backup, restore, and ACL user creation.

---

## Requirements

| Requirement | Details |
|---|---|
| Ansible | ≥ 2.12 |
| Target OS | RHEL / Debian with `systemd` |
| Redis | ≥ 6.0 (ACL support required for `create_user`) |
| Privileges | `become: true` (sudo) on target hosts |
| Inventory group | `redis` |

---

## Inventory

Add target hosts to the `redis` group in your inventory file:

```ini
[redis]
redis-prod-01 ansible_host=10.0.1.10
redis-prod-02 ansible_host=10.0.1.11
```

---

## Variables

### Core variables (set in `vars` block)

| Variable | Default | Description |
|---|---|---|
| `redis_service_name` | `redis` | systemd unit name |
| `redis_cli_path` | `/usr/bin/redis-cli` | Path to `redis-cli` binary |
| `redis_data_dir` | `/var/lib/redis` | Directory containing the RDB file |
| `redis_rdb_file` | `dump.rdb` | RDB filename |
| `redis_backup_dir` | `/var/backups/redis` | Destination for backup copies |
| `redis_port` | `6379` | Redis port |
| `operation` | `status` | Operation to run (see below) |
| `restore_source_file` | `{{ redis_backup_dir }}/dump.rdb` | RDB file used as restore source |
| `redis_acl_user` | `app_user` | ACL username to create or update |
| `redis_acl_permissions` | `on ~* +@all` | ACL permissions string (without password) |

### Sensitive variables (runtime prompts)

Both passwords are collected interactively at runtime via `vars_prompt` so they are **never stored in plaintext** in the playbook or inventory.

| Prompt variable | When used |
|---|---|
| `redis_password` | All operations that call `redis-cli` (backup, create_user) |
| `redis_acl_password` | `create_user` only |

**For unattended/CI runs**, replace the prompts with `ansible-vault` encrypted vars:

```bash
# Encrypt each secret individually and paste into a vault file
ansible-vault encrypt_string 'YourRedisPassword' --name 'redis_password'
ansible-vault encrypt_string 'YourAclPassword'   --name 'redis_acl_password'
```

Then run with `--vault-password-file` or `--ask-vault-pass`.

---

## Supported Operations

| Operation | Description | Tag |
|---|---|---|
| `status` | Print systemd `ActiveState` and `SubState` | `status` |
| `start` | Start Redis and enable it at boot | `start` |
| `stop` | Stop Redis | `stop` |
| `restart` | Restart Redis | `restart` |
| `backup` | Trigger `BGSAVE`, wait for completion, copy RDB with timestamp | `backup` |
| `restore` | Stop Redis, replace RDB file, restart — with auto-recovery on failure | `restore` |
| `create_user` | Create or update an ACL user and persist with `ACL SAVE` | `create_user` |

---

## Usage

### Running by operation (recommended)

Pass the operation as an extra variable. Ansible also accepts `--tags` as an alternative:

```bash
# Check service status
ansible-playbook redis_day2.yml -e "operation=status"

# Start the service
ansible-playbook redis_day2.yml -e "operation=start"

# Stop the service
ansible-playbook redis_day2.yml -e "operation=stop"

# Restart the service
ansible-playbook redis_day2.yml -e "operation=restart"

# Create a backup
ansible-playbook redis_day2.yml -e "operation=backup"

# Restore from a specific file
ansible-playbook redis_day2.yml -e "operation=restore" \
  -e "restore_source_file=/var/backups/redis/dump-20250318T120000Z.rdb"

# Create an ACL user
ansible-playbook redis_day2.yml -e "operation=create_user" \
  -e "redis_acl_user=reporting_user" \
  -e "redis_acl_permissions='on ~reports:* +@read'"
```

### Using Ansible tags

```bash
# Alternative — run using tags
ansible-playbook redis_day2.yml --tags backup
ansible-playbook redis_day2.yml --tags create_user
```

### Limiting to a single host

```bash
ansible-playbook redis_day2.yml -e "operation=backup" --limit redis-prod-01
```


---

## Operation Details

### Backup

The backup flow correctly waits for BGSAVE to finish before copying the RDB file:

1. Record the current `LASTSAVE` Unix timestamp.
2. Issue `BGSAVE` to trigger an asynchronous save.
3. Poll `INFO persistence` every 3 seconds (up to 60 seconds) until:
   - `rdb_bgsave_in_progress` equals `0`, **and**
   - `rdb_last_save_time` is greater than the pre-BGSAVE timestamp.
4. Copy the RDB to `{{ redis_backup_dir }}/dump-<ISO8601_timestamp>.rdb` with mode `0640`.

This two-condition check prevents copying a stale RDB that existed before the current save cycle.

### Restore

The restore uses a `block/rescue` pattern to guarantee Redis is always brought back online, even if the file copy fails:

```
block:
  stop Redis → copy RDB → start Redis → print success
rescue:
  start Redis (recovery) → fail with diagnostic message
```

If the copy step fails, the rescue block restarts Redis with its original data and surfaces a clear error message.

### Create user

ACL management follows three steps:

1. `ACL SETUSER <user> <permissions>` — creates or updates the user in memory.
2. `ACL SAVE` — writes the rules to the `aclfile` configured in `redis.conf`, making them survive restarts.
3. Confirmation message printed (no sensitive data exposed).

> **Prerequisite:** `redis.conf` must include `aclfile /etc/redis/users.acl` (or equivalent). Without an `aclfile` directive, `ACL SAVE` has no effect.

---

## Security Notes

| Control | Implementation |
|---|---|
| No plaintext secrets | Passwords collected via `vars_prompt` (private) or ansible-vault |
| No credential leakage | `no_log: true` on all tasks that build or use passwords |
| Least-privilege file ownership | Backup and restore copies use `owner: redis, group: redis` |
| ACL persistence | `ACL SAVE` called after every user creation/update |

---

