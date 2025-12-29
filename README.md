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

## Architecture

```
                    ┌─────────────┐
                    │   Clients   │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
      ┌───────▼───────┐         ┌───────▼───────┐
      │  HAProxy LB1  │         │  HAProxy LB2  │
      │ (192.168.1.20)│         │ (192.168.1.21)│
      └───────┬───────┘         └───────┬───────┘
              │                         │
              └────────────┬────────────┘
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
┌──────▼──────┐     ┌──────▼──────┐     ┌──────▼──────┐
│  MariaDB 1  │◄───►│  MariaDB 2  │◄───►│  MariaDB 3  │
│   (Node 1)  │     │   (Node 2)  │     │   (Node 3)  │
└─────────────┘     └─────────────┘     └─────────────┘
       │                   │                   │
       └───────────────────┴───────────────────┘
                    Galera Replication
```

### Galera Arbitrator (Garbd)

Garbd is a lightweight daemon that participates in Galera voting **without storing data**. It runs on a **separate VM** from your MariaDB nodes.

**When to use Garbd:**
- You have an **EVEN** number of MariaDB nodes (2, 4, 6)
- Garbd adds 1 vote to make total voters ODD
- Prevents split-brain scenarios

**When NOT needed:**
- You have an **ODD** number of MariaDB nodes (3, 5, 7)
- Odd count already provides proper quorum

```
Example: 2 MariaDB nodes + 1 Garbd = 3 voters (tolerates 1 failure)
Example: 4 MariaDB nodes + 1 Garbd = 5 voters (tolerates 2 failures)
```

**Important:** Garbd must run on a **separate VM** from MariaDB nodes for proper failure domain isolation. If Garbd runs on the same VM as a MariaDB node, losing that VM loses 2 voters.

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

# Components separately
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
# -----------------------------------------------------------------------------
# MariaDB Cluster Nodes (use ODD number: 3, 5, 7)
# -----------------------------------------------------------------------------
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }

# -----------------------------------------------------------------------------
# HAProxy Load Balancers (optional - set to [] to disable)
# -----------------------------------------------------------------------------
haproxy_nodes:
  - { ip: "192.168.1.20", name: "mariadb-lb-1" }
  - { ip: "192.168.1.21", name: "mariadb-lb-2" }

# -----------------------------------------------------------------------------
# Galera Arbitrator (optional - for EVEN node counts)
# IMPORTANT: Must be on SEPARATE VMs from mariadb_nodes!
# -----------------------------------------------------------------------------
garbd_enabled: false  # Set to true if using even number of MariaDB nodes
garbd_nodes: []       # Example: [{ ip: "192.168.1.30", name: "garbd-1" }]

# -----------------------------------------------------------------------------
# Credentials (CHANGE THESE!)
# -----------------------------------------------------------------------------
mariadb_root_password: "YourSecurePassword"
admin_user: "admin"
admin_password: "YourSecurePassword"

# -----------------------------------------------------------------------------
# HAProxy Stats
# -----------------------------------------------------------------------------
haproxy_stats_user: "admin"
haproxy_stats_password: "YourStatsPassword"
```

### Example: 2 MariaDB nodes + Garbd

```yaml
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }

garbd_enabled: true
garbd_nodes:
  - { ip: "192.168.1.30", name: "garbd-1" }  # Separate VM!
```

### Example: 5 MariaDB nodes (no Garbd needed)

```yaml
mariadb_nodes:
  - { ip: "192.168.1.10", name: "mariadb-node-1" }
  - { ip: "192.168.1.11", name: "mariadb-node-2" }
  - { ip: "192.168.1.12", name: "mariadb-node-3" }
  - { ip: "192.168.1.13", name: "mariadb-node-4" }
  - { ip: "192.168.1.14", name: "mariadb-node-5" }

garbd_enabled: false
garbd_nodes: []
```

## Idempotency

| State | `--tags deploy` | Result |
|-------|-----------------|--------|
| No cluster | Fresh install | Bootstrap new cluster |
| Running cluster | Update | Rolling restart if config changed |
| Running cluster + `-e force_bootstrap=yes` | Force | Destroys and recreates |

## Node Count & Quorum

| MariaDB Nodes | Garbd | Total Voters | Tolerates Failures |
|---------------|-------|--------------|-------------------|
| 2 | 1 (required) | 3 | 1 |
| 3 | 0 | 3 | 1 |
| 4 | 1 (required) | 5 | 2 |
| 5 | 0 | 5 | 2 |
| 6 | 1 (required) | 7 | 3 |
| 7 | 0 | 7 | 3 |

**Rule:** Total voters must be ODD for proper split-brain protection.

## Ports

| Port | Service |
|------|---------|
| 3306 | MySQL client connections |
| 4567 | Galera replication |
| 4568 | IST / Garbd |
| 4444 | SST (State Snapshot Transfer) |
| 8404 | HAProxy stats dashboard |

## HAProxy Stats

- URL: `http://<haproxy-ip>:8404/stats`
- Credentials: Set in `group_vars/all.yml`

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
├── ansible.cfg           # Ansible configuration
├── requirements.yml      # Galaxy dependencies
├── group_vars/all.yml    # Cluster configuration
├── inventory/hosts.yml   # SSH settings
└── templates/
    ├── 60-galera.cnf.j2  # Galera configuration
    ├── haproxy.cfg.j2    # HAProxy configuration
    └── garb.j2           # Garbd configuration
```

## License

MIT
