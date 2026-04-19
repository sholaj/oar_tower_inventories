# Inventory Converters (BU-based output)

Convert database source data (flat files or CMDB CSV exports) into the
**business-unit-based** Ansible inventory format used by the
`oar_tower_inventories` repo and consumed by `linux-inspec` scan playbooks.

## Output Format

All converters emit a single `db_<bu>` group with `ssc_sn_*` group vars and
**`database_platform` per host** (the field the multi-platform scan playbook
uses to dispatch to the correct role: `mssql`, `oracle`, `sybase`, `postgres`).

Example:

```yaml
all:
  children:
    db_alpha:
      vars:
        ssc_sn_environment: test
        ssc_sn_region: na
        ssc_sn_bu: alpha
        exec_timeout: 600
        ansible_connection: local
        inspec_delegate_host: "[DELEGATE_HOST]"
      hosts:
        DBSERVER01_1433:
          database_platform: mssql
          mssql_server: "[DB_SERVER].example.internal"
          mssql_port: 1433
          mssql_version: "2019"
          mssql_database: master
          mssql_username: nist_scan_user
          use_winrm: false
```

## Required Parameters (both converters)

| Parameter | Description | Example |
|-----------|-------------|---------|
| `target_bu` | Business unit identifier (lowercase) | `alpha` |
| `ssc_environment` | `ssc_sn_environment` value | `test`, `prod` |
| `ssc_region` | `ssc_sn_region` value | `na`, `eu`, `apac` |
| `inventory_output` | Output path | `ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database` |

## Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `inspec_delegate_host` | Delegate host for InSpec execution | `""` (local) |
| `exec_timeout` | Per-host scan timeout (seconds) | `600` |

---

## Flat-File Converter

### Input Format

```
PLATFORM SERVER DATABASE SERVICE PORT VERSION [CONNECTION] [MODE_SUFFIX]
```

| Field | Required | Description |
|-------|----------|-------------|
| PLATFORM | Yes | `MSSQL`, `ORACLE`, `SYBASE`, `POSTGRES` |
| SERVER | Yes | Database server hostname |
| DATABASE | Yes | Database name (use `master` for MSSQL) |
| SERVICE | Yes | Service name (use `null` if N/A) |
| PORT | Yes | Database port |
| VERSION | Yes | Database version |
| CONNECTION | No | `WINRM` or `DIRECT` (default: `DIRECT`) |
| MODE_SUFFIX | No | Free-text qualifier appended to `host_id` so the same `server:port` can have multiple inventory entries with distinct ids (e.g. `direct`, `winrm`, `EE`, `DELEGATE_CONNECTION`). When set, `CONNECTION` must also be set (use `DIRECT` as a placeholder). |

#### Multi-mode example (matches the live env naming convention)

```text
# Same MSSQL server, two scan variants
MSSQL gdctwvc0007 master svc 1733 2019 DIRECT direct
MSSQL gdctwvc0007 master svc 1733 2019 WINRM  winrm

# Same Oracle DB, two execution modes
ORACLE o02dil0 mydb mydb 1528 19 DIRECT EE
ORACLE o02dil0 mydb mydb 1528 19 DIRECT DELEGATE_CONNECTION

# No suffix — single entry
SYBASE s01dms0 master master 5000 16
```

Resulting `host_id` values:

```
gdctwvc0007_1733_direct
gdctwvc0007_1733_winrm
o02dil0_mydb_1528_EE
o02dil0_mydb_1528_DELEGATE_CONNECTION
s01dms0_master_5000
```

The suffix is purely a label — it does **not** auto-set `use_winrm`,
`inspec_delegate_host`, or any other behaviour-changing variable. The
caller is responsible for setting those (per-host or via inventory
group vars).

### Usage

```bash
ansible-playbook convert_flatfile_to_inventory.yml \
  -e "target_bu=alpha" \
  -e "ssc_environment=test" \
  -e "ssc_region=na" \
  -e "flatfile_input=databases.txt" \
  -e "inventory_output=ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database"
```

### MSSQL WinRM (mixed environment)

Set the platform default and override per-host as needed:

```bash
ansible-playbook convert_flatfile_to_inventory.yml \
  -e "target_bu=alpha" \
  -e "ssc_environment=test" \
  -e "ssc_region=na" \
  -e "flatfile_input=databases.txt" \
  -e "inventory_output=ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database" \
  -e "mssql_connection_mode=winrm"
```

```text
# Most hosts use default (winrm); one Linux MSSQL overrides to direct
MSSQL winserver01 master svc1 1433 2019
MSSQL winserver02 master svc1 1433 2019
MSSQL linuxsql01 master svc1 1433 2019 DIRECT
```

### Optional flat-file flags

| Parameter | Description | Default |
|-----------|-------------|---------|
| `mssql_connection_mode` | Default MSSQL connection mode (`winrm`/`direct`) | `direct` |
| `sybase_ssh_user_default` | SSH user used for Sybase scans | `sybase_admin` |

### Deduplication

| Platform | Dedup key |
|----------|-----------|
| MSSQL | `server:port` (server-level scanning) |
| Oracle / Sybase / Postgres | per database |

---

## CMDB CSV Converter

Converts CMDB CSV exports (Oracle and/or MSSQL) using the `DATABASE@SERVER`
naming convention.

### Usage

```bash
# Oracle only
ansible-playbook convert_cmdb_to_inventory.yml \
  -e "target_bu=alpha" \
  -e "ssc_environment=test" \
  -e "ssc_region=na" \
  -e "cmdb_oracle_csv=/path/to/oracle_cmdb.csv" \
  -e "inventory_output=ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database"

# MSSQL only
ansible-playbook convert_cmdb_to_inventory.yml \
  -e "target_bu=alpha" \
  -e "ssc_environment=test" \
  -e "ssc_region=na" \
  -e "cmdb_mssql_csv=/path/to/mssql_cmdb.csv" \
  -e "inventory_output=ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database"

# Both combined
ansible-playbook convert_cmdb_to_inventory.yml \
  -e "target_bu=alpha" \
  -e "ssc_environment=test" \
  -e "ssc_region=na" \
  -e "cmdb_oracle_csv=/path/to/oracle_cmdb.csv" \
  -e "cmdb_mssql_csv=/path/to/mssql_cmdb.csv" \
  -e "inventory_output=ALPHA/ALPHA_DEVTEST_NA_Inv_InSpec_Database"
```

### CMDB Input Columns

**Oracle CSV:**

| Column | Example |
|--------|---------|
| Name | `CDB1@[DB_SERVER].example.internal` |
| Version | `19.0.0.0` |
| Validation Status | `Validated` |
| Comments | (free text) |

**MSSQL CSV:**

| Column | Example |
|--------|---------|
| Name | `MSSQLSERVER@[DB_SERVER].example.internal` |
| Version name | `2019` |
| Validation Status | `Validated` |
| TCP port(s) | `1433` |
| Comments | (free text) |

### Filtering

- Only rows whose `Validation Status` contains `validated` (case-insensitive)
  are included.
- Rows with comments matching `unable to connect` / `unable to validate` are
  **still included** in the inventory but recorded in a sidecar file
  `<inventory_output>.diagnostics.yml` for triage.
- CMDB metadata fields (validation status, permissions status, comments) are
  intentionally **not** written into the generated inventory.

### Oracle Version Mapping

| CMDB | InSpec |
|------|--------|
| `12.x.x.x` | `12c` |
| `19.x.x.x` | `19c` |
| `21.x.x.x` | `21c` |
| `23.x.x.x` | `23c` |

### Example Output

```yaml
all:
  children:
    db_alpha:
      vars:
        ssc_sn_environment: test
        ssc_sn_region: na
        ssc_sn_bu: alpha
        exec_timeout: 600
        ansible_connection: local
        inspec_delegate_host: "[DELEGATE_HOST]"
      hosts:
        CDB1_EXAMPLEDB1:
          database_platform: oracle
          oracle_server: "[DB_SERVER].example.internal"
          oracle_service: CDB1
          oracle_port: 1521
          oracle_version: "19c"
          oracle_username: nist_scan_user

        EXAMPLEHOST_1433:
          database_platform: mssql
          mssql_server: "[DB_SERVER].example.internal"
          mssql_port: 1433
          mssql_version: "2019"
          mssql_instance: MSSQLSERVER
          mssql_database: master
          mssql_username: nist_scan_user
```

---

## File Naming Convention

Output files follow the repo convention:

```
{BU}_{ENVIRONMENT}_{REGION}_Inv_InSpec_Database
```

The converters do not auto-generate the filename — pass it explicitly via
`inventory_output` so the caller (typically the AAP2 job template) keeps full
control of placement.

## Notes

- All hostnames/identifiers in this README are placeholders. Real source data
  is never committed to git.
- Credentials (passwords) are injected by AAP2 Custom Credential at runtime;
  inventories carry usernames only.
- The `db_<bu>` group + per-host `database_platform` is the contract the
  multi-platform scan playbook (`run_compliance_scans.yml`) relies on to
  dispatch to the correct role.
