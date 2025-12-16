# Elasticsearch Role Auto-Updater

A Python tool for automating Elasticsearch role management across multiple clusters in a Cross-Cluster Search (CCS) environment.

## Features

### Index Pattern Management

| Feature | Remote Clusters | CCS Cluster |
|---------|-----------------|-------------|
| Inject patterns (`partial-*`, `restored-*`) | ✓ | ✓ |
| Inject `elastic-cloud-logs-*` | - | ✓ |
| Sync patterns from remote to CCS | - | ✓ |
| Remove subset patterns | ✓ | ✓ |
| Remove redundant remote references | - | ✓ |

### Kibana Privilege Management (CCS Only)

| Feature | Description |
|---------|-------------|
| Add `.all` privileges | Adds `feature_discover.all`, `feature_dashboard.all`, `feature_visualize.all` |
| Replace `space_read` | Converts to explicit feature privileges with disabled feature detection |
| Skip `space_all` | Leaves entries with full access unchanged |
| Remove superseded privileges | Removes `.read` when adding `.all` to avoid conflicts |

## Installation

```bash
# Clone or download the scripts
# Install dependencies
pip install requests --break-system-packages
```

## Configuration

Create `es_clusters_config.json`:

```json
{
  "clusters": {
    "prod": {
      "url": "https://prod-elasticsearch:9200",
      "api_key": "YOUR_PROD_API_KEY",
      "verify_ssl": false,
      "description": "Production cluster"
    },
    "qa": {
      "url": "https://qa-elasticsearch:9200",
      "api_key": "YOUR_QA_API_KEY",
      "verify_ssl": false,
      "description": "QA cluster"
    },
    "ccs": {
      "url": "https://ccs-elasticsearch:9200",
      "kibana_url": "https://ccs-kibana:5601",
      "api_key": "YOUR_CCS_API_KEY",
      "verify_ssl": false,
      "description": "Cross-Cluster Search cluster"
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

### Configuration Options

| Field | Description |
|-------|-------------|
| `url` | Elasticsearch endpoint URL |
| `kibana_url` | Kibana endpoint URL (CCS only, for disabled feature detection) |
| `api_key` | API key for authentication |
| `verify_ssl` | Whether to verify SSL certificates |
| `remote_inject_patterns` | Patterns to inject into remote cluster roles |
| `ccs_inject_patterns` | Patterns to inject into CCS cluster roles |
| `ccs_kibana_privileges` | Kibana privileges to add to CCS roles |

## Usage

### Basic Commands

```bash
# Dry run for specific roles
python es_role_auto_update.py role1 role2 --dry-run

# Update specific roles
python es_role_auto_update.py role1 role2

# Update all matching roles across clusters
python es_role_auto_update.py --all-matching --dry-run

# Skip Kibana privilege updates
python es_role_auto_update.py role1 --skip-kibana-privileges

# Skip CCS cluster updates
python es_role_auto_update.py role1 --skip-ccs

# Skip remote cluster updates
python es_role_auto_update.py role1 --skip-remote

# Generate report
python es_role_auto_update.py --all-matching --dry-run --report updates.json
```

### Command-Line Options

| Option | Description |
|--------|-------------|
| `role_names` | Specific role names to update |
| `--all-matching` | Update all roles that exist in all clusters |
| `--dry-run` | Preview changes without applying |
| `--skip-ccs` | Skip CCS cluster updates |
| `--skip-remote` | Skip remote cluster updates |
| `--skip-kibana-privileges` | Skip Kibana privilege updates |
| `--continue-on-error` | Continue processing if a role update fails |
| `--report FILE` | Generate JSON report |
| `--config FILE` | Path to config file (default: `es_clusters_config.json`) |
| `--log-level` | Logging level (DEBUG, INFO, WARNING, ERROR) |

## How It Works

### 1. Index Pattern Injection

Adds missing patterns to role index permissions:

**Remote Clusters:**
```
Before: ['filebeat-*', 'metricbeat-*']
After:  ['filebeat-*', 'metricbeat-*', 'partial-*', 'restored-*']
```

**CCS Cluster:**
```
Before: ['filebeat-*']
After:  ['filebeat-*', 'partial-*', 'restored-*', 'elastic-cloud-logs-*']
```

### 2. Pattern Sync (Remote → CCS)

Syncs patterns from remote clusters to CCS:

```
Remote (prod): ['filebeat-*', 'metricbeat-*', 'apm-*']
Remote (qa):   ['filebeat-*', 'logs-*']
CCS Before:    ['filebeat-*']
CCS After:     ['filebeat-*', 'metricbeat-*', 'apm-*', 'logs-*']
```

### 3. Subset Pattern Cleanup

Removes patterns that are subsets of other patterns:

| Before | After | Removed |
|--------|-------|---------|
| `apm*`, `apm-*` | `apm*` | `apm-*` |
| `partial-*`, `partial-restored-*` | `partial-*` | `partial-restored-*` |
| `restored-*`, `restored-filebeat-*` | `restored-*` | `restored-filebeat-*` |
| `filebeat-*`, `filebeat-7*` | `filebeat-*` | `filebeat-7*` |

**Logic:** Pattern A is a subset of Pattern B if A's prefix starts with B's prefix (for `prefix*` patterns).

### 4. Remote Pattern Removal (CCS Only)

Removes remote index references when the base pattern exists locally:

| Remote Pattern | Local Patterns | Action |
|----------------|----------------|--------|
| `prod:filebeat-*` | `filebeat-*` | **Remove** (exact match) |
| `prod:filebeat-7*` | `filebeat-*` | **Remove** (subset) |
| `prod:partial-restored-*` | `partial-*` | **Remove** (subset) |
| `*:logs-*` | `filebeat-*` | **Keep** (not covered) |

**Why?** After syncing patterns to CCS, remote references like `prod:filebeat-*` become redundant. Access to remote indices is granted through the synced local pattern.

### 5. Kibana Privilege Updates (CCS Only)

#### Adding `.all` Privileges

Adds `feature_discover.all`, `feature_dashboard.all`, `feature_visualize.all` to enable CSV report generation:

```
Before: ['feature_discover.read', 'feature_dashboard.read']
After:  ['feature_discover.all', 'feature_dashboard.all', 'feature_visualize.all']
```

Superseded privileges (`.read`, `.minimal_read`, `.url_create`, etc.) are automatically removed.

#### Handling `space_read`

Entries with `space_read` are converted to explicit feature privileges:

```
Before: ['space_read']
After:  ['feature_discover.all', 'feature_dashboard.all', 'feature_visualize.all',
         'feature_canvas.read', 'feature_maps.read', 'feature_ml.read', ...]
```

**Disabled Feature Detection:** If `kibana_url` is configured, the script queries the Kibana Spaces API to exclude disabled features from the replacement list.

#### Handling `space_all`

Entries with `space_all` are skipped entirely (already have full access).

## Order of Operations

### Remote Clusters
1. Add inject patterns (`partial-*`, `restored-*`)
2. Cleanup subset patterns
3. Update role via API

### CCS Cluster
1. Add inject patterns (`partial-*`, `restored-*`, `elastic-cloud-logs-*`)
2. Sync patterns from remote clusters
3. Cleanup subset patterns
4. Remove redundant remote patterns
5. Update Kibana privileges
6. Update role via API

## Files

| File | Description |
|------|-------------|
| `es_role_auto_update.py` | Main script |
| `es_role_manager_utils.py` | Utility library with `ElasticsearchRoleManager` and `KibanaClient` |
| `es_clusters_config.json` | Cluster configuration |
| `README.md` | This documentation |
| `QUICKSTART.md` | Quick setup guide |
| `WORKFLOW.md` | Visual workflow diagrams |

## API Requirements

### Elasticsearch API Key Permissions

```json
{
  "cluster": ["manage_security"],
  "indices": [
    {
      "names": ["*"],
      "privileges": ["monitor"]
    }
  ]
}
```

### Kibana API Key Permissions

The same API key used for Elasticsearch should have access to Kibana APIs for:
- `GET /api/spaces/space/{space_id}` - Read space configurations

## Error Handling

| Scenario | Behavior |
|----------|----------|
| Cluster connection fails | Script exits (or continues with `--continue-on-error`) |
| Role update fails | Logged and skipped (or exits without `--continue-on-error`) |
| Kibana API fails | Falls back to required privileges only for `space_read` replacement |
| Reserved role | Automatically skipped |

## Examples

### Example 1: Full Update with Dry Run

```bash
python es_role_auto_update.py --all-matching --dry-run --report preview.json
```

### Example 2: Update Specific Roles

```bash
python es_role_auto_update.py analyst_role developer_role --report update.json
```

### Example 3: Remote Clusters Only

```bash
python es_role_auto_update.py --all-matching --skip-ccs
```

### Example 4: CCS with Kibana Updates Only

```bash
python es_role_auto_update.py analyst_role --skip-remote
```

## Troubleshooting

### "Role definition is invalid" in Kibana

This usually means conflicting privileges. The script now handles:
- Removing superseded privileges when adding `.all`
- Filtering `transient_metadata` field
- Skipping `space_all` entries
- Converting `space_read` to explicit privileges

### Connection Timeouts

Check your cluster URLs and network connectivity. The script uses a 30-second timeout.

### SSL Certificate Errors

Set `verify_ssl: false` in the config for self-signed certificates.

## License

Internal use only.
