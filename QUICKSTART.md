# Quick Start Guide

Get up and running in 5 minutes.

## Step 1: Install Dependencies

```bash
pip install requests --break-system-packages
```

## Step 2: Configure Clusters

Create `es_clusters_config.json`:

```json
{
  "clusters": {
    "prod": {
      "url": "https://your-prod-cluster:9200",
      "api_key": "your-prod-api-key",
      "verify_ssl": false
    },
    "qa": {
      "url": "https://your-qa-cluster:9200",
      "api_key": "your-qa-api-key",
      "verify_ssl": false
    },
    "ccs": {
      "url": "https://your-ccs-cluster:9200",
      "kibana_url": "https://your-ccs-kibana:5601",
      "api_key": "your-ccs-api-key",
      "verify_ssl": false
    }
  },
  "defaults": {
    "remote_inject_patterns": ["partial-*", "restored-*"],
    "ccs_inject_patterns": ["partial-*", "restored-*", "elastic-cloud-logs-*"],
    "ccs_kibana_privileges": [
      "feature_discover.all",
      "feature_dashboard.all",
      "feature_visualize.all"
    ],
    "remote_clusters": ["prod", "qa"],
    "ccs_cluster": "ccs"
  }
}
```

## Step 3: Test Connection (Dry Run)

```bash
# Preview changes for all matching roles
python es_role_auto_update.py --all-matching --dry-run
```

## Step 4: Apply Updates

```bash
# Update all matching roles
python es_role_auto_update.py --all-matching

# Or update specific roles
python es_role_auto_update.py role_name_1 role_name_2
```

## Step 5: Verify in Kibana

1. Go to **Stack Management → Roles**
2. Open an updated role
3. Verify:
   - Index patterns include `partial-*`, `restored-*`
   - Kibana privileges show `.all` for Discover, Dashboard, Visualize
   - No remote patterns like `prod:filebeat-*` (for CCS roles)

## Common Commands

```bash
# Dry run (preview only)
python es_role_auto_update.py --all-matching --dry-run

# Update with report
python es_role_auto_update.py --all-matching --report updates.json

# Skip Kibana updates
python es_role_auto_update.py role1 --skip-kibana-privileges

# Remote clusters only
python es_role_auto_update.py --all-matching --skip-ccs

# CCS cluster only
python es_role_auto_update.py --all-matching --skip-remote

# Debug mode
python es_role_auto_update.py role1 --log-level DEBUG
```

## What Gets Updated?

### Remote Clusters (prod, qa, etc.)

| Action | Example |
|--------|---------|
| Add inject patterns | `+ partial-*`, `+ restored-*` |
| Remove subset patterns | `- partial-restored-*` (covered by `partial-*`) |

### CCS Cluster

| Action | Example |
|--------|---------|
| Add inject patterns | `+ partial-*`, `+ restored-*`, `+ elastic-cloud-logs-*` |
| Sync from remotes | `+ filebeat-*` (if exists in prod/qa) |
| Remove subset patterns | `- apm-*` (covered by `apm*`) |
| Remove remote patterns | `- prod:filebeat-*` (now local) |
| Add Kibana `.all` | `+ feature_discover.all`, etc. |
| Replace `space_read` | `space_read` → explicit privileges |

## Troubleshooting

### Connection Failed

```
ERROR - Failed to connect to prod cluster
```

**Fix:** Check URL and API key in config.

### Role Not Found

```
WARNING - Role 'my_role' not found in prod
```

**Fix:** Verify role exists in all clusters, or use `--skip-ccs` / `--skip-remote`.

### Kibana Shows "Invalid Role"

**Fix:** Already handled by script. Re-run the update.

## Next Steps

- Read [README.md](README.md) for full documentation
- See [WORKFLOW.md](WORKFLOW.md) for visual diagrams
