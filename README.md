# MariaDB Galera Cluster - Ansible Deployment

Production-ready Ansible automation for MariaDB Galera Cluster on Debian 12.

## Features

- **Multi-node cluster**: 3, 5, 7+ nodes
- **High availability**: Synchronous multi-master replication
- **Load balancing**: Optional HAProxy with stats dashboard
- **Arbitrator**: Optional Garbd for even node counts
- **Idempotent**: Safe to run multiple times
- **Parallel execution**: Fast deployment
- **CLI switches**: No interactive prompts

## Quick Start

```bash
# 1. Install Ansible collection
ansible-galaxy collection install -r requirements.yml

# 2. Configure cluster
vim group_vars/all.yml

# 3. Deploy
ansible-playbook mariadb.yml --tags deploy
```

## Commands

### Deploy

```bash
# Full cluster
ansible-playbook mariadb.yml --tags deploy

# Components
ansible-playbook mariadb.yml --tags deploy-mariadb
ansible-playbook mariadb.yml --tags deploy-haproxy
ansible-playbook mariadb.yml --tags deploy-garbd

# Force fresh install (destroys cluster!)
ansible-playbook mariadb.yml --tags deploy -e force_bootstrap=yes
```

### Status

```bash
# Quick status
ansible-playbook mariadb.yml --tags status

# Detailed wsrep status
ansible-playbook mariadb.yml --tags status-detailed
```

### Recovery

```bash
# Recover failed cluster
ansible-playbook mariadb.yml --tags recover -e confirm=yes
```

## Configuration

Edit `group_vars/all.yml`:

```yaml
# Cluster nodes (use ODD number)
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }

# HAProxy (optional)
haproxy_nodes:
  - { ip: "192.168.1.20", name: "mariadb-lb-1" }

# Credentials
mariadb_root_password: "YourSecurePassword"
admin_user: "admin"
admin_password: "YourSecurePassword"

# HAProxy stats
haproxy_stats_user: "admin"
haproxy_stats_password: "YourStatsPassword"
```

## Idempotency

| State | `--tags deploy` | Result |
|-------|-----------------|--------|
| No cluster | Fresh install | Bootstrap new cluster |
| Running cluster | Update | Rolling restart if config changed |
| Running cluster + `-e force_bootstrap=yes` | Force | Destroys and recreates |

## Ports

| Port | Service |
|------|---------|
| 3306 | MySQL |
| 4567 | Galera replication |
| 4568 | IST / Garbd |
| 4444 | SST |
| 8404 | HAProxy stats |

## HAProxy Stats

- URL: `http://<haproxy-ip>:8404/stats`
- Credentials: Set in `group_vars/all.yml`

## Monitoring

```sql
SHOW STATUS LIKE 'wsrep_cluster_size';     -- Should match node count
SHOW STATUS LIKE 'wsrep_cluster_status';   -- Should be 'Primary'
SHOW STATUS LIKE 'wsrep_local_state_comment'; -- Should be 'Synced'
SHOW STATUS LIKE 'wsrep_ready';            -- Should be 'ON'
```

## File Structure

```
mariadb-galera-cluster/
├── mariadb.yml           # Main playbook
├── ansible.cfg
├── requirements.yml
├── group_vars/all.yml    # Configuration
├── inventory/hosts.yml
└── templates/
    ├── 60-galera.cnf.j2
    ├── haproxy.cfg.j2
    └── garb.j2
```

## License

MIT
