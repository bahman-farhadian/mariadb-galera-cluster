# MariaDB Galera Cluster - Ansible Deployment

Automated deployment of a highly available MariaDB Galera Cluster on Debian 12.

## Features

- **Variable node count**: Deploy 3, 5, 7+ node clusters
- **High Availability**: Synchronous multi-master replication
- **Optional HAProxy**: Configurable load balancers
- **Optional Garbd**: Galera Arbitrator for even-numbered setups
- **Idempotent**: Safe to run multiple times
- **Rolling updates**: No cluster downtime for config changes

## Quick Start

```bash
# 1. Install required collection
ansible-galaxy collection install -r requirements.yml

# 2. Edit group_vars/all.yml with your configuration

# 3. Deploy
ansible-playbook deploy.yml

# 4. Check status
ansible-playbook status.yml
```

## Configuration

Edit `group_vars/all.yml`:

```yaml
# MariaDB nodes (KEEP ODD NUMBER: 3, 5, 7...)
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }

# HAProxy load balancers (optional)
haproxy_nodes:
  - { ip: "192.168.1.20", name: "mariadb-lb-1" }
  - { ip: "192.168.1.21", name: "mariadb-lb-2" }

# Garbd - disabled by default
garbd_enabled: false
garbd_nodes: []

# Credentials
mariadb_root_password: "YourSecurePassword"
admin_user: "admin"
admin_password: "YourSecurePassword"
```

## Playbooks

| Playbook | Description |
|----------|-------------|
| `deploy.yml` | Full cluster deployment |
| `recover.yml` | Recover from cluster failure |
| `status.yml` | Check all nodes status |

## Usage Examples

```bash
# Full deployment
ansible-playbook deploy.yml

# Deploy only MariaDB
ansible-playbook deploy.yml --tags mariadb

# Deploy only HAProxy
ansible-playbook deploy.yml --tags haproxy

# Check status
ansible-playbook status.yml

# Detailed status
ansible-playbook status.yml --tags detailed

# Recover failed cluster
ansible-playbook recover.yml
```

## Node Count Guidelines

| Data Nodes | Garbd Needed | Total Voters | Max Failures |
|------------|--------------|--------------|--------------|
| 3 | No | 3 | 1 |
| 4 | Yes (1) | 5 | 2 |
| 5 | No | 5 | 2 |
| 7 | No | 7 | 3 |

## Network Ports

| Port | Service |
|------|---------|
| 3306 | MySQL connections |
| 4567 | Galera replication |
| 4568 | IST / Garbd |
| 4444 | SST |

## Recovery

### Single Node Failure
No action needed - node auto-rejoins on restart.

### Full Cluster Failure
```bash
ansible-playbook recover.yml
```

## Monitoring

```bash
mysql -u admin -p -e "SHOW STATUS LIKE 'wsrep_%';"
```

Key metrics:
- `wsrep_cluster_size` - Should match node count
- `wsrep_cluster_status` - Should be `Primary`
- `wsrep_local_state_comment` - Should be `Synced`
- `wsrep_ready` - Should be `ON`
