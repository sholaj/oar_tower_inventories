# Inventory Converters (tech-group + BU sub-group output)

Convert database source data (flat files or CMDB CSV exports) into the
Ansible inventory format used by the `oar_tower_inventories` repo and
consumed by `linux-inspec` scan playbooks.

## Output Format

Each converter run produces a **single-BU fragment** laid out as
technology groups (`mssql_databases`, `oracle_databases`, etc.) each
owning a **platform-specific** BU subgroup (`db_<bu>_mssql`,
`db_<bu>_oracle`, …). A top-level `db_<bu>` aggregator holds the
BU-wide vars and aggregates the per-platform subgroups so
`--limit db_<bu>` still targets every host in a BU across platforms.
The platform is implicit from the parent tech group — **no
`database_platform` var on individual hosts**.

Why platform-specific subgroup names: Ansible inventory YAML treats
group names as global keys. A single `db_<bu>` placed as a direct
child of multiple tech groups would be one merged group, polluting
`mssql_databases` with Oracle/Sybase hosts (and vice versa).

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
      children:
        db_alpha_mssql:
        db_alpha_oracle:

    mssql_databases:
      children:
        db_alpha_mssql:
          hosts:
            dbserver01.example.internal:
              mssql_server: dbserver01.example.internal
              mssql_port: 1433
              mssql_version: "2019"
              mssql_database: master
              mssql_username: nist_scan_user
              use_winrm: false

    oracle_databases:
      children:
        db_alpha_oracle:
          hosts:
            oradb01.example.internal:
              oracle_server: oradb01.example.internal
              oracle_port: 1521
              oracle_service: ORCL
              oracle_version: "19"
              oracle_username: nist_scan_user
```

## Single-BU fragment → env inventory

For an env-level file containing corp + affiliates, run the converter
once per BU and merge the fragments into a single file like
`DEVTEST_NA_Inv_InSpec_Database` in the repo root. The tech groups
(`mssql_databases`, ...) stay the same; each merge adds a new
`db_<bu>` aggregator at the top level and new `db_<bu>_<plat>`
subgroup entries under each tech group.

## Required Parameters (both converters)

| Parameter | Description | Example |
|-----------|-------------|---------|
| `target_bu` | Business unit identifier (lowercase) | `alpha` |
| `ssc_environment` | `ssc_sn_environment` value | `test`, `prod` |
| `ssc_region` | `ssc_sn_region` value | `na`, `eu`, `apac` |
| `inventory_output` | Output path | `fragments/alpha_devtest_na.yml` |

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
| MODE_SUFFIX | No | Free-text qualifier appended to the FQDN so the same server can have multiple inventory entries with distinct ids (e.g. `direct`, `winrm`, `cdb1`, `pdb2`). When set, `CONNECTION` must also be set (use `DIRECT` as a placeholder). Also needed when you want multiple databases on the same FQDN to each get their own inventory entry (without a suffix they'd all collide on the same key). |

#### Multi-mode example

```text
# Same MSSQL server, two scan variants
MSSQL mssql01.example.internal master svc 1733 2019 DIRECT direct
MSSQL mssql01.example.internal master svc 1733 2019 WINRM  winrm

# Same Oracle server, two services
ORACLE ora01.example.internal cdb1 cdb1 1528 19 DIRECT cdb1
ORACLE ora01.example.internal pdb2 pdb2 1528 19 DIRECT pdb2

# No suffix — single entry
SYBASE syb01.example.internal master master 5000 16
```

Resulting `host_id` values (inventory keys):

```
mssql01.example.internal_direct
mssql01.example.internal_winrm
ora01.example.internal_cdb1
ora01.example.internal_pdb2
syb01.example.internal
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

| Platform | inventory_hostname | Dedup behaviour on same-FQDN collision |
|----------|-------------------|-----------------------------------------|
| MSSQL | `<fqdn>` (or `<fqdn>_<mode_suffix>` when given) | first row wins, later rows logged + ignored |
| Oracle / Sybase / Postgres | `<fqdn>` (or `<fqdn>_<mode_suffix>` when given) | last row wins, earlier entries overwritten — flatfile processor logs a warning |

The `inventory_hostname` is the FQDN as-is (dots and dashes preserved). If the same FQDN hosts multiple databases / instances and you want each as its own inventory entry, provide a `MODE_SUFFIX` (flatfile 8th column, e.g. `direct`, `winrm`, `cdb1`, `pdb2`) — the suffix is appended to the FQDN so each row yields a distinct key.

The underlying connection metadata (`mssql_port`, `oracle_service`, `sybase_database`, etc.) is still recorded on the host vars — only the YAML key has changed from `<short-host>_<port>`-style to `<fqdn>`-style.

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
      children:
        db_alpha_mssql:
        db_alpha_oracle:

    oracle_databases:
      children:
        db_alpha_oracle:
          hosts:
            ora01.example.internal:
              oracle_server: ora01.example.internal
              oracle_service: CDB1
              oracle_port: 1521
              oracle_version: "19c"
              oracle_username: nist_scan_user

    mssql_databases:
      children:
        db_alpha_mssql:
          hosts:
            mssql01.example.internal:
              mssql_server: mssql01.example.internal
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
- The contract each single-platform playbook relies on is simply
  `hosts: <plat>_databases` (e.g. `mssql_databases`). The top-level
  `db_<bu>` aggregator lets you `--limit db_alpha` to scope to one
  BU across all platforms; combined with the playbook's
  `hosts: <plat>_databases` Ansible intersects the two groups and
  the scan runs against `db_alpha_<plat>` only.
