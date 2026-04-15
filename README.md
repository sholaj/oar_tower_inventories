# oar_tower_inventories

Ansible Tower / AAP2 inventory files for database compliance scanning, organized by business unit.

## Structure

Each business unit has its own directory containing inventory files per environment:

```
oar_tower_inventories/
├── CORP/                    # Corp business unit
├── UATCORP/                 # UAT Corp business unit
├── IMSWESTUAT/              # IMS West UAT
├── IMSWESTPROD/             # IMS West Production
├── CRDIT/                   # Credit business unit
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

Example: `CORP_DEVTEST_NA_Inv_InSpec_Database`

## Required Variables

Each inventory group must define:

| Variable | Description | Example |
|----------|-------------|---------|
| `ssc_sn_environment` | Environment identifier | `test`, `prod` |
| `ssc_sn_region` | Region identifier | `na`, `eu`, `apac` |
| `ssc_sn_bu` | Business unit identifier | `corp`, `uatcorp`, `crdit` |
| `database_platform` | Per-host: database type | `mssql`, `oracle`, `sybase`, `postgres` |

## Usage

```bash
# Scan all databases in CORP BU
ansible-playbook -i CORP/CORP_DEVTEST_NA_Inv_InSpec_Database run_compliance_scans.yml

# Scan only MSSQL databases in CORP BU
ansible-playbook -i CORP/CORP_DEVTEST_NA_Inv_InSpec_Database run_compliance_scans.yml \
  -e "target_platform=mssql"
```
