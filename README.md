# MariaDB Galera Cluster - Ansible Deployment

Production-ready Ansible automation for deploying highly available MariaDB Galera Cluster on Debian 12.

## Features

- **Multi-node cluster**: Deploy 3, 5, 7+ node clusters
- **High availability**: Synchronous multi-master replication
- **Load balancing**: Optional HAProxy with stats dashboard
- **Arbitrator support**: Optional Garbd for even-numbered setups
- **Single playbook**: All operations in one `mariadb.yml` with tags
- **Idempotent**: Safe to run multiple times - auto-detects cluster state
- **Parallel execution**: Fast deployment with parallel install/configure
- **CLI switches**: No interactive prompts - all options via command line

## Quick Start

```bash
# 1. Install required Ansible collection
ansible-galaxy collection install -r requirements.yml

# 2. Configure your cluster
vim group_vars/all.yml

# 3. Deploy
ansible-playbook mariadb.yml --tags deploy
```

## Commands

### Deploy

```bash
# Deploy full cluster (MariaDB + HAProxy + Garbd)
ansible-playbook mariadb.yml --tags deploy

# Deploy MariaDB nodes only
ansible-playbook mariadb.yml --tags deploy-mariadb

# Deploy HAProxy load balancers only
ansible-playbook mariadb.yml --tags deploy-haproxy

# Deploy Galera Arbitrator only
ansible-playbook mariadb.yml --tags deploy-garbd

# Force fresh bootstrap (WARNING: destroys existing cluster)
ansible-playbook mariadb.yml --tags deploy -e force_bootstrap=yes
```

### Status

```bash
# Quick cluster status
ansible-playbook mariadb.yml --tags status

# Detailed per-node wsrep status
ansible-playbook mariadb.yml --tags status-detailed
```

### Recovery

```bash
# Recover from complete cluster failure
ansible-playbook mariadb.yml --tags recover -e confirm=yes
```

## Configuration

Edit `group_vars/all.yml`:

```yaml
# Cluster name
cluster_name: "production_cluster"

# MariaDB nodes (use ODD number: 3, 5, 7...)
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }

# HAProxy load balancers (optional - set to [] to disable)
haproxy_nodes:
  - { ip: "192.168.1.20", name: "mariadb-lb-1" }
  - { ip: "192.168.1.21", name: "mariadb-lb-2" }

# Galera Arbitrator (optional - for even node counts)
garbd_enabled: false
garbd_nodes: []

# Credentials (CHANGE THESE!)
mariadb_root_password: "YourSecureRootPassword"
admin_user: "admin"
admin_password: "YourSecureAdminPassword"

# HAProxy stats
haproxy_stats_enabled: true
haproxy_stats_port: 8404
haproxy_stats_user: "admin"
haproxy_stats_password: "YourStatsPassword"
```

## Idempotency Behavior

| Current State | Action | Result |
|---------------|--------|--------|
| No cluster | `--tags deploy` | Fresh install + bootstrap |
| Running cluster | `--tags deploy` | Verify config, rolling restart if changed |
| Running cluster | `--tags deploy -e force_bootstrap=yes` | **DESTROYS** and recreates |
| Failed cluster | `--tags recover -e confirm=yes` | Find best node, bootstrap, rejoin |

## Node Count Guidelines

| Data Nodes | Garbd | Total Voters | Tolerates |
|------------|-------|--------------|-----------|
| 3 | No | 3 | 1 failure |
| 4 | Yes | 5 | 2 failures |
| 5 | No | 5 | 2 failures |
| 7 | No | 7 | 3 failures |

**Rule**: Always maintain an ODD number of voters.

## Network Ports

| Port | Service |
|------|---------|
| 3306 | MySQL client |
| 4567 | Galera replication |
| 4568 | IST / Garbd |
| 4444 | SST |
| 8404 | HAProxy stats |

## Connecting

```bash
# Via HAProxy (recommended)
mysql -h <haproxy-ip> -u admin -p

# Direct to node
mysql -h <node-ip> -u admin -p
```

## HAProxy Stats

- URL: `http://<haproxy-ip>:8404/stats`
- Credentials: Set in `group_vars/all.yml`

## Recovery

### Single Node Failure

Node auto-rejoins on restart:
```bash
systemctl start mariadb
```

### Complete Cluster Failure

```bash
ansible-playbook mariadb.yml --tags recover -e confirm=yes
```

This will:
1. Stop all services
2. Find node with highest seqno
3. Bootstrap from that node
4. Rejoin remaining nodes

## Monitoring

```sql
-- Cluster size (should match node count)
SHOW STATUS LIKE 'wsrep_cluster_size';

-- Cluster status (should be 'Primary')
SHOW STATUS LIKE 'wsrep_cluster_status';

-- Node state (should be 'Synced')
SHOW STATUS LIKE 'wsrep_local_state_comment';

-- Ready for queries (should be 'ON')
SHOW STATUS LIKE 'wsrep_ready';
```

## File Structure

```
mariadb-galera-cluster/
├── mariadb.yml           # Main playbook
├── ansible.cfg           # Ansible config
├── requirements.yml      # Galaxy dependencies
├── group_vars/
│   └── all.yml           # Cluster config
├── inventory/
│   └── hosts.yml         # SSH settings
└── templates/
    ├── 60-galera.cnf.j2  # Galera config
    ├── haproxy.cfg.j2    # HAProxy config
    └── garb.j2           # Garbd config
```

## License

MIT
