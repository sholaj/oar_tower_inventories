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

## Branching

This repo is single-trunk (`main`). Environment and region are encoded in
the inventory filename and in the per-group `ssc_sn_environment` /
`ssc_sn_region` variables — not in branch names.

## Inventory File Naming

```
{BU}_{ENVIRONMENT}_{REGION}_Inv_InSpec_Database
```

Example: `ALPHA_DEVTEST_NA_Inv_InSpec_Database`

A single BU directory holds one inventory file per environment + region
combination. AAP2 selects the right file per job template.

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
