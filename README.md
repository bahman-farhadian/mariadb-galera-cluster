# MariaDB Galera Cluster - Ansible Deployment

Production-ready Ansible automation for deploying highly available MariaDB Galera Cluster on Debian 12.

## Features

- **Multi-node cluster**: Deploy 3, 5, 7+ node clusters
- **High availability**: Synchronous multi-master replication
- **Load balancing**: Optional HAProxy with stats dashboard
- **Arbitrator support**: Optional Garbd for even-numbered setups
- **Single playbook**: All operations in one `mariadb.yml` with tags
- **Idempotent**: Safe to run multiple times - detects existing cluster state
- **Parallel execution**: Fast deployment with parallel install/configure
- **Automated recovery**: One-command cluster recovery

## Quick Start

```bash
# 1. Install required Ansible collection
ansible-galaxy collection install -r requirements.yml

# 2. Configure your cluster in group_vars/all.yml
vim group_vars/all.yml

# 3. Deploy the cluster
ansible-playbook mariadb.yml --tags deploy
```

## Commands

| Command | Description |
|---------|-------------|
| `ansible-playbook mariadb.yml --tags deploy` | Deploy full cluster (MariaDB + HAProxy) |
| `ansible-playbook mariadb.yml --tags deploy-mariadb` | Deploy MariaDB nodes only |
| `ansible-playbook mariadb.yml --tags deploy-haproxy` | Deploy HAProxy load balancers only |
| `ansible-playbook mariadb.yml --tags deploy-garbd` | Deploy Galera Arbitrator only |
| `ansible-playbook mariadb.yml --tags status` | Check cluster status |
| `ansible-playbook mariadb.yml --tags status-detailed` | Detailed per-node status |
| `ansible-playbook mariadb.yml --tags recover` | Recover from cluster failure |

## Execution Flow

The playbook is optimized for speed with parallel execution where safe:

```
PARALLEL:  Prepare all remote hosts
PARALLEL:  Install MariaDB packages (all nodes simultaneously)
PARALLEL:  Configure MariaDB (all nodes simultaneously)
PARALLEL:  Stop MariaDB (all nodes)
SEQUENTIAL: Bootstrap first node
SEQUENTIAL: Join remaining nodes (one by one for safe sync)
SEQUENTIAL: Secure cluster
PARALLEL:  Deploy HAProxy (all LBs simultaneously)
PARALLEL:  Deploy Garbd (if enabled)
```

## Configuration

Edit `group_vars/all.yml`:

```yaml
# MariaDB nodes (use ODD number: 3, 5, 7...)
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }

# HAProxy load balancers (optional, set to [] to disable)
haproxy_nodes:
  - { ip: "192.168.1.20", name: "mariadb-lb-1" }
  - { ip: "192.168.1.21", name: "mariadb-lb-2" }

# Galera Arbitrator (optional, for even node counts)
garbd_enabled: false
garbd_nodes: []

# Credentials (CHANGE THESE!)
mariadb_root_password: "YourSecurePassword"
admin_user: "admin"
admin_password: "YourSecurePassword"
```

## Node Count Guidelines

| Data Nodes | Garbd | Total Voters | Tolerates |
|------------|-------|--------------|-----------|
| 3 | No | 3 | 1 failure |
| 4 | Yes (1) | 5 | 2 failures |
| 5 | No | 5 | 2 failures |
| 7 | No | 7 | 3 failures |

**Rule**: Always maintain an ODD number of voters to prevent split-brain.

## Network Ports

| Port | Service |
|------|---------|
| 3306 | MySQL client connections |
| 4567 | Galera cluster replication |
| 4568 | IST / Garbd |
| 4444 | SST (State Snapshot Transfer) |
| 8404 | HAProxy stats dashboard |

## Connecting to the Cluster

```bash
# Via HAProxy (recommended)
mysql -h <haproxy-ip> -u admin -p

# Direct to any node
mysql -h <node-ip> -u admin -p
```

## HAProxy Stats Dashboard

Access at: `http://<haproxy-ip>:8404/stats`

Default credentials (change in `group_vars/all.yml`):
- Username: `admin`
- Password: `ChangeMe_StatsPassword123!`

## Recovery

### Single Node Failure
No action needed. The node will automatically rejoin when restarted:
```bash
systemctl start mariadb
```

### Complete Cluster Failure
```bash
ansible-playbook mariadb.yml --tags recover
```
This will:
1. Stop all services
2. Find the most up-to-date node (highest seqno)
3. Bootstrap from that node
4. Rejoin all other nodes

## Monitoring

Key metrics to watch:
```sql
SHOW STATUS LIKE 'wsrep_cluster_size';      -- Should match node count
SHOW STATUS LIKE 'wsrep_cluster_status';    -- Should be 'Primary'
SHOW STATUS LIKE 'wsrep_local_state_comment'; -- Should be 'Synced'
SHOW STATUS LIKE 'wsrep_ready';             -- Should be 'ON'
```

## File Structure

```
mariadb-galera-cluster/
├── mariadb.yml           # Main playbook (all operations)
├── ansible.cfg           # Ansible configuration
├── requirements.yml      # Ansible Galaxy dependencies
├── group_vars/
│   └── all.yml           # Cluster configuration
├── inventory/
│   └── hosts.yml         # SSH settings
└── templates/
    ├── 60-galera.cnf.j2  # Galera configuration
    ├── garb.j2           # Garbd configuration
    └── haproxy.cfg.j2    # HAProxy configuration
```

## License

MIT
