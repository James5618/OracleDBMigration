# Oracle SID Configuration Guide

## Overview
The Oracle System Identifier (SID) is a unique identifier for each Oracle database instance. This playbook allows you to configure custom SIDs for both source and destination servers through the host_vars files.

## Default Configuration

By default, the playbook uses "ORCL" as the Oracle SID for both servers:

```yaml
oracle_sid: "ORCL"
oracle_connect_string: "localhost:1521/ORCL"
oracle_service_name: "ORCL"
```

## Customizing Oracle SIDs

### Single Instance Configuration

For a standard single-instance Oracle database:

```yaml
# host_vars/oracle-source.yml
oracle_sid: "PROD"
oracle_connect_string: "localhost:1521/PROD"
oracle_service_name: "PROD"

# host_vars/oracle-destination.yml
oracle_sid: "PROD19"
oracle_connect_string: "localhost:1521/PROD19"
oracle_service_name: "PROD19"
```

### Multi-Instance Configuration

If you have multiple Oracle instances on the same server:

```yaml
# Primary instance
oracle_sid: "ORCL1"
oracle_connect_string: "localhost:1521/ORCL1"
oracle_service_name: "ORCL1"

# Secondary instance (different port)
oracle_sid: "ORCL2"
oracle_connect_string: "localhost:1522/ORCL2"
oracle_service_name: "ORCL2"
```

### RAC (Real Application Clusters) Configuration

For Oracle RAC environments:

```yaml
# Node 1
oracle_sid: "ORCL1"
oracle_connect_string: "scan-cluster:1521/ORCLPDB"
oracle_service_name: "ORCLPDB"

# Node 2
oracle_sid: "ORCL2"
oracle_connect_string: "scan-cluster:1521/ORCLPDB"
oracle_service_name: "ORCLPDB"
```

### Container Database (CDB/PDB) Configuration

For Oracle 12c+ with Pluggable Databases:

```yaml
# Container Database
oracle_sid: "ORCL"
oracle_connect_string: "localhost:1521/ORCLPDB1"
oracle_service_name: "ORCLPDB1"
```

## Common SID Naming Conventions

### Environment-Based
```yaml
# Development
oracle_sid: "DEV"

# Test/UAT
oracle_sid: "TEST"

# Production
oracle_sid: "PROD"
```

### Version-Based
```yaml
# Oracle 18c
oracle_sid: "ORCL18"

# Oracle 19c
oracle_sid: "ORCL19"

# Oracle 21c
oracle_sid: "ORCL21"
```

### Application-Based
```yaml
# HR System
oracle_sid: "HRDB"

# Finance System
oracle_sid: "FINDB"

# Sales System
oracle_sid: "SALESDB"
```

### Geographic-Based
```yaml
# US East
oracle_sid: "USEAST"

# EU West
oracle_sid: "EUWEST"

# Asia Pacific
oracle_sid: "APAC"
```

## Configuration Variables Explained

### oracle_sid
The Oracle System Identifier that uniquely identifies the database instance on the server.

```yaml
oracle_sid: "PROD"
```

### oracle_connect_string
The connection string used by SQL*Plus and other Oracle tools to connect to the database.

Format: `hostname:port/service_name`

```yaml
oracle_connect_string: "localhost:1521/PROD"
```

### oracle_service_name
The Oracle service name that clients use to connect to the database.

```yaml
oracle_service_name: "PROD"
```

### oracle_port
The port number on which the Oracle listener is running (default: 1521).

```yaml
oracle_port: 1521
```

## Validation Steps

Before running the migration, validate your SID configuration:

### 1. Check Oracle Instance Status
```bash
ps -ef | grep pmon
```
Should show: `ora_pmon_YOURSID`

### 2. Check Listener Status
```bash
lsnrctl status
```
Should show your service name in the services list.

### 3. Test Connection
```bash
sqlplus sys/password@localhost:1521/YOURSID as sysdba
```

### 4. Verify Service Name
```sql
SELECT name FROM v$database;
SELECT instance_name FROM v$instance;
```

## Troubleshooting SID Issues

### Common Error Messages

#### ORA-12514: TNS:listener does not currently know of service
**Cause**: Service name not registered with listener
**Solution**: 
- Check oracle_service_name matches actual service
- Restart the listener: `lsnrctl reload`

#### ORA-12541: TNS:no listener
**Cause**: Listener not running or wrong port
**Solution**:
- Check listener status: `lsnrctl status`
- Verify oracle_port matches listener configuration

#### ORA-01034: ORACLE not available
**Cause**: Database instance not started
**Solution**:
- Start the instance: `startup` in SQL*Plus
- Check oracle_sid matches running instance

### Configuration File Locations

#### Oracle Network Configuration
- **tnsnames.ora**: `$ORACLE_HOME/network/admin/tnsnames.ora`
- **listener.ora**: `$ORACLE_HOME/network/admin/listener.ora`

#### Environment Variables
Ensure these are set correctly:
```bash
export ORACLE_HOME=/opt/oracle/product/19.0.0/dbhome_1
export ORACLE_SID=PROD
export PATH=$ORACLE_HOME/bin:$PATH
```

## Best Practices

### 1. Consistent Naming
Use consistent naming conventions across environments:
```yaml
# Development
oracle_sid: "APPDEV"

# Testing  
oracle_sid: "APPTEST"

# Production
oracle_sid: "APPPROD"
```

### 2. Documentation
Document your SID naming convention and maintain an inventory:

| Environment | Server | SID | Service Name | Port |
|-------------|--------|-----|--------------|------|
| DEV | dev-db-01 | APPDEV | APPDEV | 1521 |
| TEST | test-db-01 | APPTEST | APPTEST | 1521 |
| PROD | prod-db-01 | APPPROD | APPPROD | 1521 |

### 3. Validation
Always validate SID configuration before migration:
- Test connections to both source and destination
- Verify listener status on both servers
- Confirm service names are registered

### 4. Backup Configuration
Keep backups of your host_vars files:
```bash
cp host_vars/oracle-source.yml host_vars/oracle-source.yml.backup
cp host_vars/oracle-destination.yml host_vars/oracle-destination.yml.backup
```

## Example Complete Configuration

### Production Environment
```yaml
# host_vars/oracle-source.yml
---
oracle_home: "/opt/oracle/product/18.0.0/dbhome_1"
oracle_sid: "HRPROD"
oracle_connect_string: "prod-db-18:1521/HRPROD"
oracle_port: 1521
oracle_service_name: "HRPROD"
sysdba_user: "sys"
sysdba_password: "{{ vault_sys_password }}"

# host_vars/oracle19-server.yml  
---
oracle_home: "/opt/oracle/product/19.0.0/dbhome_1"
oracle_sid: "HRPROD19"
oracle_connect_string: "prod-db-19:1521/HRPROD19"
oracle_port: 1521
oracle_service_name: "HRPROD19"
sysdba_user: "sys"
sysdba_password: "{{ vault_sys_password }}"
```

This configuration supports a migration from an Oracle 18c HR production database to a new Oracle 19c instance with clear naming that indicates both the application and version.
