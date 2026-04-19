# oar_tower_inventories

Ansible Tower / AAP2 inventory files for database compliance scanning, organized by business unit.

## Structure

Each business unit has its own directory containing inventory files per environment:

> Business-unit names in this repo use NATO-phonetic codenames as
> placeholders (`ALPHA`, `BRAVO`, `CHARLIE`, `DELTA`, `ECHO`). Real BU
> identifiers are intentionally not committed.

```
oar_tower_inventories/
├── ALPHA/                   # Example BU (placeholder name)
├── BRAVO/                   # Example BU
├── CHARLIE/                 # Example BU
├── DELTA/                   # Example BU
├── ECHO/                    # Example BU
└── tools/                   # Inventory management utilities
```

## Branch Strategy

| Branch | Environment | Description |
|--------|-------------|-------------|
| `main` | All | Default branch with templates |
| `NA_DEVTEST` | Dev/Test | North America development and test |
| `NA_PROD` | Production | North America production |

## Inventory File Naming

```
{BU}_{ENVIRONMENT}_{REGION}_Inv_InSpec_Database
```

Example: `ALPHA_DEVTEST_NA_Inv_InSpec_Database`

## Required Variables

Each inventory group must define:

| Variable | Description | Example |
|----------|-------------|---------|
| `ssc_sn_environment` | Environment identifier | `test`, `prod` |
| `ssc_sn_region` | Region identifier | `na`, `eu`, `apac` |
| `ssc_sn_bu` | Business unit identifier | `alpha`, `bravo`, `charlie` |
| `database_platform` | Per-host: database type | `mssql`, `oracle`, `sybase`, `postgres` |

## Usage

```bash
# Scan all databases in the ALPHA BU
ansible-playbook -i ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database run_compliance_scans.yml

# Scan only MSSQL databases in the ALPHA BU
ansible-playbook -i ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database run_compliance_scans.yml \
  -e "target_platform=mssql"
```
