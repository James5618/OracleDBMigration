# Oracle Schema Migration Playbook - Dynamic Version Features

### 1. Single Schema Migration
- Prompts for a single schema name during execution
- Good for testing or migrating individual schemas

### 2. Bulk Migration
- Uses the `schemas.yml` configuration file
- Migrates multiple schemas based on priority
- Provides detailed reporting and error handling
- Continues with remaining schema        "Source server: oracle-source"
        "Destination server: oracle-destination"if one fails

### 3. Oracle Data Pump File Splitting
- **Automatic file splitting**: Large schemas are automatically split into multiple dump files (e.g., `schema_01.dmp`, `schema_02.dmp`)
- **Configurable file size**: Maximum file size controlled by `max_file_size` parameter (default: 2GB)
- **Parallel processing**: Uses `%U` substitution variable for parallel dump file generation
- **Smart transfer**: All split files are automatically detected and transferred using the selected method
- **Pattern-based import**: Uses wildcard patterns to import all related dump files

## Transfer Methods

This Ansib## Prerequisites

1. **Oracle Database Access**: Ensure you have SYSDBA access to both source and destination Oracle servers
2. **Data Pump Directory**: Ensure DATA_PUMP_DIR directory is configured on both servers
3. **Network Connectivity**: Ansible control machine should be able to reach both Oracle servers
4. **SSH Access**: SSH key-based authentication configured for the ansible user on both servers
5. **Sudo Access**: The ansible user must have sudo privileges to become the oracle user
6. **Server-to-Server Communication** (for SCP/Rsync): Direct SSH access between Oracle servers (optional, only needed for transfer methods 2 and 3)
7. **Version Compatibility**: Destination Oracle version must be equal or higher than source versionook migrates Oracle schemas between different Oracle database versions using Oracle Data Pump utilities. It automatically detects Oracle versions and adapts the migration process accordingly.

## Structure

```
oracleschemamigration/
├── main.yml                     # Main playbook with dynamic version detection
├── migrate_single_schema.yml    # Individual schema migration tasks
├── schemas.yml                  # Schema configuration for bulk migration
├── inventory.ini               # Inventory file with source and destination servers
├── host_vars/                  # Host-specific variables
│   ├── oracle-source.yml       # Source server configuration
│   └── oracle-destination.yml  # Destination server configuration
├── group_vars/
│   └── all.yml                # Global configuration settings
├── VERSION_COMPATIBILITY.md   # Version compatibility guide
├── TRANSFER_METHODS.md        # Transfer method comparison
└── README.md                   # This file
```

## Key Features

### **Dynamic Version Detection**
- Automatically detects Oracle versions on both source and destination servers
- Supports Oracle 11g, 12c, 18c, 19c, 21c, and 23c
- Adapts export/import parameters based on detected versions
- Validates migration compatibility (prevents downgrades)

### **Migration Types Supported**
- **Upgrade migrations**: 11g→19c, 18c→19c, 19c→21c, etc.
- **Same-version migrations**: Server-to-server migration
- **Cross-platform migrations**: With compatible Oracle versions

### **Intelligent Migration Adaptation**
- Sets appropriate export version for maximum compatibility
- Adjusts Data Pump parameters based on source/destination versions
- Provides version-specific warnings and recommendations

### **Pre-Migration Validation**
- **Disk Space Analysis**: Automatically checks available space on both source and destination servers
- **Space Requirements**: Estimates dump file sizes based on schema sizes with compression factors
- **Proactive Warnings**: Alerts if disk space is insufficient before starting migration
- **Smart Recommendations**: Suggests cleanup options and batch migration strategies

## Migration Modes

### 1. Single Schema Migration
- Prompts for a single schema name during execution
- Good for testing or migrating individual schemas

### 2. Bulk Migration
- Uses the `schemas.yml` configuration file
- Migrates multiple schemas based on priority
- Provides detailed reporting and error handling
- Continues with remaining schemas if one fails

## Transfer Methods

The playbook supports three different methods for transferring dump files between servers:

### 1. Ansible fetch/copy (Default)
- **How it works**: Export → Fetch to Ansible control machine → Copy to destination
- **Pros**: Simple, works with any Ansible setup, good for smaller files
- **Cons**: Uses Ansible control machine storage, slower for large files
- **Best for**: Small to medium schemas, environments where direct server-to-server communication isn't available

### 2. SCP (Secure Copy)
- **How it works**: Direct server-to-server transfer using SCP
- **Pros**: Faster than fetch/copy, doesn't use control machine storage, secure
- **Cons**: Requires SSH access between source and destination servers
- **Best for**: Medium to large schemas, when servers can communicate directly

### 3. Rsync
- **How it works**: Direct server-to-server transfer using rsync with compression
- **Pros**: Fastest transfer, compression, progress indication, resume capability
- **Cons**: Requires rsync installed on both servers, SSH access between servers
- **Best for**: Large schemas, when maximum transfer speed is needed

## Prerequisites

1. **Oracle Database Access**: Ensure you have SYSDBA access to both Oracle 18 (source) and Oracle 19c (destination) servers
2. **Data Pump Directory**: Ensure DATA_PUMP_DIR directory is configured on both servers
3. **Network Connectivity**: Ansible control machine should be able to reach both Oracle servers
4. **SSH Access**: SSH key-based authentication configured for the ansible user on both servers
5. **Sudo Access**: The ansible user must have sudo privileges to become the oracle user
6. **Server-to-Server Communication** (for SCP/Rsync): Direct SSH access between Oracle servers (optional, only needed for transfer methods 2 and 3)

## Configuration

### 1. Update Inventory (inventory.ini)

Edit the inventory file to match your environment. The inventory defines two groups (`oracle_source` and `oracle_destination`) with their respective hostnames:

```ini
[oracle_source]
oracle-source ansible_host=YOUR_ORACLE_SOURCE_IP ansible_user=ansible

[oracle_destination]
oracle-destination ansible_host=YOUR_ORACLE_DEST_IP ansible_user=ansible

[oracle_servers:vars]
ansible_become=yes
ansible_become_user=oracle
ansible_become_method=sudo
```

**Important**: The hostnames (`oracle-source`, `oracle-destination`) must match the corresponding `host_vars` filenames. If you change the hostnames in the inventory, rename the `host_vars` files accordingly.

### 2. Configure Host Variables

The host_vars files correspond to the hostnames defined in your inventory file. Update these files to match your environment:

#### Source Server (host_vars/oracle-source.yml)
*This corresponds to the host in the `oracle_source` inventory group*
```yaml
oracle_home: "/opt/oracle/product/18.0.0/dbhome_1"
oracle_sid: "ORCL"  # Default: ORCL, can be customized (e.g., PROD, DEV, TEST)
# oracle_connect_string is automatically derived from inventory: {{ ansible_host }}:{{ oracle_port }}/{{ oracle_service_name }}
# oracle_version is automatically detected at runtime via SQL*Plus
oracle_port: 1521
oracle_service_name: "ORCL"
sysdba_user: "sys"
sysdba_password: "your_actual_sys_password"
```

#### Destination Server (host_vars/oracle-destination.yml)
*This corresponds to the host in the `oracle_destination` inventory group*
```yaml
oracle_home: "/opt/oracle/product/19.0.0/dbhome_1"
oracle_sid: "ORCL"  # Default: ORCL, can be customized (e.g., PROD19, NEWDB)
# oracle_connect_string is automatically derived from inventory: {{ ansible_host }}:{{ oracle_port }}/{{ oracle_service_name }}
# oracle_version is automatically detected at runtime via SQL*Plus
oracle_port: 1521
oracle_service_name: "ORCL"
sysdba_user: "sys"
sysdba_password: "your_actual_sys_password"
```

### 3. Dynamic Oracle Connect String

The playbook automatically constructs Oracle connect strings from the inventory file and host variables, eliminating the need to maintain duplicate host information:

**Format**: `{{ ansible_host }}:{{ oracle_port }}/{{ oracle_service_name }}`

**Example**:
- Inventory: `oracle18-server ansible_host=192.168.1.10`
- Host vars: `oracle_port: 1521`, `oracle_service_name: "ORCL"`
- Resulting connect string: `192.168.1.10:1521/ORCL`

### 4. Dynamic Oracle Version Detection

The playbook automatically detects Oracle versions on both source and destination servers at runtime, eliminating the need for static version configuration:

**Detection Process**:
- Connects to each Oracle server using SQL*Plus
- Queries `v$version` and `v$instance` for version information
- Extracts major version (e.g., 18, 19, 21) and detailed version (e.g., 19.3.0.0.0)
- Auto-determines compatible export version for Data Pump operations


**What You'll See**:
```
Oracle Version: Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 (detected)
Detailed Version: 19.3.0.0.0
Export Version: 19.3.0.0.0 (auto-determined)
```

### 5. Oracle SID Configuration

The playbook uses a default Oracle SID of "ORCL" for both source and destination servers. You can customize this in the host_vars files:

#### Common SID Examples:
- **Default**: `ORCL` (standard Oracle default)
- **Environment-based**: `PROD`, `DEV`, `TEST`, `UAT`
- **Version-based**: `ORCL18`, `ORCL19`, `ORCL21`
- **Application-based**: `HRDB`, `FINDB`, `SALESDB`

#### Multi-Instance Example:
If you have multiple Oracle instances on the same server:

```yaml
# Source server with custom SID
oracle_sid: "PRODDB"
oracle_port: 1521
oracle_service_name: "PRODDB"
# Connect string automatically becomes: {{ ansible_host }}:1521/PRODDB

# Destination server with different SID
oracle_sid: "PRODDB19"
oracle_port: 1521
oracle_service_name: "PRODDB19"
# Connect string automatically becomes: {{ ansible_host }}:1521/PRODDB19
```

**Note**: The Oracle connect string is automatically derived from the inventory `ansible_host` and the `oracle_port`/`oracle_service_name` variables. This ensures consistency and eliminates the need to maintain duplicate host information. Ensure the `oracle_sid`, `oracle_port`, and `oracle_service_name` are all consistent with your actual Oracle database configuration.

### 6. Configure Schemas for Bulk Migration (schemas.yml)

```yaml
schemas:
  - name: "HR"
    description: "Human Resources schema"
    priority: 1
    tablespace_mapping:
      USERS: "USERS"
      TEMP: "TEMP"
    exclude_objects: []
    
  - name: "FINANCE"
    description: "Finance and Accounting schema"
    priority: 2
    exclude_objects:
      - "TEMP_TABLE_*"
      - "LOG_TABLE_*"
```

## Usage

### Schema Discovery and Pre-Migration Checks
Before prompting for migration options, the playbook will automatically:
1. **Version Detection**: Connect to both source and destination servers to detect Oracle versions
2. **Schema Analysis**: Query all available user schemas (excluding system schemas) and calculate sizes in GB
3. **Disk Space Analysis**: Check available disk space on both servers' Data Pump directories
4. **Space Validation**: Estimate dump file sizes and validate sufficient space is available
5. **Compatibility Check**: Verify migration compatibility and display warnings if needed

**Sample Output:**
```
================== DISK SPACE ANALYSIS ==================
Source Server (/u01/app/oracle/dpdump):
  Total: 100G  Used: 45G  Available: 55G  Usage: 45%

Destination Server (/u01/app/oracle/dpdump):
  Total: 200G  Used: 80G  Available: 120G  Usage: 40%

Estimated Requirements:
  Total Schema Size: 25.6 GB
  Estimated Dump Size: 10.2 GB (with compression)
```

This comprehensive analysis helps you make informed decisions about which schemas to migrate, which transfer method to use, and whether additional disk space is needed.

### Single Schema Migration
Run the playbook and choose option 1:

```bash
ansible-playbook -i inventory.ini main.yml
```

When prompted:
1. Choose migration mode: **1** (Single schema) or **2** (Bulk migration)
2. Choose transfer method: **1** (Ansible), **2** (SCP), or **3** (Rsync)
3. Enter the schema name (for single mode)
4. Confirm the migration

### Bulk Migration
Run the playbook and choose option 2:

```bash
ansible-playbook -i inventory.ini main.yml
```

When prompted:
1. Choose migration mode: **2** (Bulk migration)
2. Choose transfer method: **1** (Ansible), **2** (SCP), or **3** (Rsync)
3. Confirm the migration

The playbook will read the schema configuration from `schemas.yml` and migrate all configured schemas based on their priority order.

## What the Playbook Does

### For Each Schema:
1. **Validation**: Checks if the schema exists on the source server
2. **Size Calculation**: Determines the schema size for planning
3. **Export**: Uses `expdp` to export the schema from Oracle 18 with version compatibility
4. **Transfer**: Copies the dump file from source to destination server
5. **Import**: Uses `impdp` to import the schema into Oracle 19c
6. **Verification**: Verifies the migration by counting migrated objects
7. **Cleanup**: Cleans up temporary files for each schema

### Bulk Migration Features:
- **Priority-based processing**: Schemas are processed based on their priority value
- **Error resilience**: Continues with remaining schemas if one fails
- **Detailed reporting**: Generates a comprehensive migration summary
- **Progress tracking**: Shows progress for each schema with detailed logging
- **Flexible configuration**: Each schema can have custom settings

## Schema Configuration Options

In `schemas.yml`, each schema can be configured with:

- **name**: Schema name (required)
- **description**: Human-readable description
- **priority**: Processing order (lower numbers processed first)
- **tablespace_mapping**: Map source tablespaces to destination tablespaces
- **exclude_objects**: List of objects to exclude from migration (supports wildcards)

### Global Migration Settings

The `migration_settings` section controls overall migration behavior:

```yaml
migration_settings:
  parallel_processes: 4         # Number of parallel export/import processes
  compression: "ALL"            # Data Pump compression level
  max_file_size: "2G"          # Maximum size per dump file (triggers automatic splitting)
  version_compatibility: "18.0.0"  # Export version for compatibility
```

**File Size Management**:
- Large schemas are automatically split into multiple files when they exceed `max_file_size`
- Oracle uses the `%U` substitution variable to create sequentially numbered files
- Example: `HR_export_123456_01.dmp`, `HR_export_123456_02.dmp`, etc.
- All split files are automatically detected and transferred together
- Import process uses wildcard patterns to process all related files

## Features

- **Interactive prompts** for migration mode selection
- **Bulk processing** with priority-based ordering
- **Size calculation** of schemas before migration
- **Version compatibility** handling for Oracle 18 to 19c migration
- **Error resilience** - continues with other schemas if one fails
- **Comprehensive reporting** with success/failure tracking
- **Automatic cleanup** of temporary files
- **Detailed logging** for troubleshooting

## Transfer Method Recommendations

### Schema Size Guidelines:
- **< 1 GB**: Ansible fetch/copy is fine
- **1-10 GB**: SCP recommended for better performance
- **> 10 GB**: Rsync recommended for optimal speed and reliability

### Network Considerations:
- **High latency/slow networks**: Use Rsync for compression and resume capability
- **Restricted networks**: Use Ansible fetch/copy if direct server communication isn't allowed
- **Fast internal networks**: SCP or Rsync both work well

## Security Considerations

1. **Encrypt passwords**: Use Ansible Vault to encrypt the host_vars files:
   ```bash
   ansible-vault encrypt host_vars/oracle-source.yml
   ansible-vault encrypt host_vars/oracle-destination.yml
   ```

2. **SSH Keys**: Ensure SSH private keys are properly secured

3. **Sudo Configuration**: Configure sudo for the ansible user:
   ```
   # In /etc/sudoers.d/ansible
   ansible ALL=(oracle) NOPASSWD: ALL
   ```

## Example Bulk Migration Run

```bash
$ ansible-playbook -i inventory.ini main.yml

PLAY [Oracle Schema Discovery - Display Available Schemas] ********************

TASK [Display available schemas on source server] ****************************
ok: [localhost] => {
    "msg": "================== AVAILABLE SCHEMAS ON SOURCE SERVER ==================\nServer: oracle18-server (Oracle 18)\n\nSchema Name                    Size (GB)\n---------------------------------------------\nHR                                    2.34\nEXAMPLE1                              8.67\nEXAMPLE2                               1.23\nEXAMPLE3                          5.45\n---------------------------------------------\nTotal schemas found: 4\n========================================================================"
}

PLAY [Oracle Schema Migration from Oracle 18 to Oracle 19c] ******************

Choose migration mode: (1) Single schema (2) Bulk migration from schemas.yml: 2
Choose transfer method: (1) Ansible fetch/copy (2) SCP (3) Rsync: 3
Confirm migration? (yes/no): yes
Confirm migration? (yes/no): yes

PLAY [Oracle Schema Migration from Oracle 18 to Oracle 19c (Bulk Migration)] ***

TASK [Display migration information] *******************************************
ok: [localhost] => {
    "msg": [
        "Migration mode: Bulk migration",
        "Schemas to migrate: EXAMPLE1, EXAMPLE2, EXAMPLE3, EXAMPLE4",
        "Source server: oracle18-server", 
        "Destination server: oracle19-server"
    ]
}

TASK [[HR] Check if schema exists on source server] ***************************
...

TASK [Display final migration results] ****************************************
ok: [localhost] => {
    "msg": [
        "==================== MIGRATION COMPLETED ====================",
        "Total schemas: 4",
        "Successful: 3", 
        "Failed: 1",
        "Success rate: 75.0%",
        "Summary report: /tmp/migration_summary_1625097600.txt"
    ]
}
```

## Troubleshooting

### Common Issues

1. **Connection Issues**: Verify SSH connectivity and Oracle listener status
2. **Permission Issues**: Ensure the ansible user has proper sudo permissions
3. **Data Pump Issues**: Verify DATA_PUMP_DIR is properly configured
4. **Schema Dependencies**: Consider schema dependencies when setting priorities

### Disk Space Issues

**Error: "INSUFFICIENT DISK SPACE ON SOURCE SERVER"**
- Check current space: `df -h /path/to/datapump/directory`
- Free up space by removing old dump files
- Consider migrating schemas in smaller batches
- Enable `cleanup_source_files: true` in host_vars for automatic cleanup

**Error: "INSUFFICIENT DISK SPACE ON DESTINATION SERVER"**
- Verify destination tablespace capacity
- Clean up old backup files and logs
- Consider using different dump file location with more space
- Adjust Oracle tablespace autoextend settings

**Warning: "DISK SPACE IS GETTING LOW"**
- Monitor space usage during migration with: `watch df -h`
- Use smaller `max_file_size` setting to create smaller dump chunks
- Enable compression with `compression: "ALL"` in schemas.yml
- Consider using rsync transfer method for better compression

### Space Optimization Tips

1. **Estimate Before Migration**:
   ```bash
   # Check current schema sizes
   SELECT owner, ROUND(SUM(bytes)/1024/1024/1024,2) as GB 
   FROM dba_segments 
   WHERE owner = 'YOUR_SCHEMA' 
   GROUP BY owner;
   ```

2. **Optimize Dump Files**:
   - Use `compression: "ALL"` (reduces size by 60-70%)
   - Set appropriate `max_file_size` (default 2GB)
   - Enable `parallel_processes` for faster compression

3. **Cleanup Strategy**:
   - Enable automatic cleanup: `cleanup_source_files: true`
   - Set cleanup retention: `cleanup_destination_files: true`
   - Manual cleanup: `rm -f /datapump/dir/*_export_*.dmp`
