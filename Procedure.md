# Procedure
## Step 1: Create the PDB

Connect as `sysdba` and create the new PDB using `FILE_NAME_CONVERT`.  
*Note: The seed PDB files are located in `/data1/oradata/enabling/pdbseed`; new files go to `/data4/oradata/enabling/ytlc_preprod_cc`.*

```sql
ALTER SESSION SET CONTAINER = CDB$ROOT;

CREATE PLUGGABLE DATABASE ytlc_preprod_cc    
  ADMIN USER imp_user IDENTIFIED BY "oracle" 
  ROLES = (CONNECT, DBA)                          
  FILE_NAME_CONVERT = (                         
    '/data1/oradata/enabling/pdbseed',    
    '/data4/oradata/enabling/ytlc_preprod_cc' 
  );

ALTER PLUGGABLE DATABASE ytlc_preprod_cc OPEN;

ALTER SESSION SET CONTAINER = ytlc_preprod_cc;
```
## Step 2: Create Tablespaces
Two tablespaces are created: TAB_CC for data, IDX_CC for indexes.
Multiple data files of 30 GB each are added. Adjust the list according to your space estimates.

```sql
-- Data tablespace
CREATE TABLESPACE tab_cc DATAFILE 
  '/data4/oradata/enabling/ytlc_preprod_cc/tab_cc01.dbf' SIZE 30G AUTOEXTEND OFF;

ALTER TABLESPACE tab_cc ADD DATAFILE '/data4/oradata/enabling/ytlc_preprod_cc/tab_cc02.dbf' SIZE 30G;
-- ... repeat for tab_cc03 through tab_cc20 (some on /data3)

-- Index tablespace
CREATE TABLESPACE idx_cc DATAFILE 
  '/data4/oradata/enabling/ytlc_preprod_cc/idx_cc01.dbf' SIZE 30G AUTOEXTEND OFF;

ALTER TABLESPACE idx_cc ADD DATAFILE '/data4/oradata/enabling/ytlc_preprod_cc/idx_cc02.dbf' SIZE 30G;
-- ... up to idx_cc06
```

Current sizing: TAB_CC = 20 × 30 GB = 600 GB; IDX_CC = 6 × 30 GB = 180 GB.

## Step 3: Create Target Users

Create each schema user with default tablespace TAB_CC and temporary tablespace TEMP, then set the same password as in production (<password>).

```sql
-- Create users with a temporary password
CREATE USER cc IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER crm IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER pot IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER qmdb IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER jnc IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER pcc IDENTIFIED BY smart DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;
CREATE USER apig IDENTIFIED BY "<password>" DEFAULT TABLESPACE tab_cc TEMPORARY TABLESPACE temp QUOTA UNLIMITED ON tab_cc;

-- Set production password
ALTER USER cc IDENTIFIED BY "<password>";
ALTER USER crm IDENTIFIED BY "<password>";
ALTER USER pot IDENTIFIED BY "<password>";
ALTER USER qmdb IDENTIFIED BY "<password>";
ALTER USER jnc IDENTIFIED BY "<password>";
ALTER USER pcc IDENTIFIED BY "<password>";
-- apig already has the correct password
```

Create the Data Pump directory and grant access:

```sql
CREATE OR REPLACE DIRECTORY CC_DIR AS '/data4';
GRANT READ, WRITE ON DIRECTORY CC_DIR TO PUBLIC;
```

## Step 4: Export Data from Production (Already Done)
The following exports were executed on the production environment. Dump files are stored in CC_DIR. Statistics are excluded, and compression is enabled.

| Schema | Dumpfile Pattern | Special Handling |
|--------|------------------|-------------------|
| `cc`   | `expdp_cc_20260323_%U.dmp` | – |
| `crm`  | `expdp_crm_20260323_%U.dmp` | – |
| `pot`  | `expdp_pot_20260323_%U.dmp` | – |
| `qmdb` | `expdp_qmdb_20260323_%U.dmp` | – |
| `jnc`  | `expdp_jnc_20260323_%U.dmp` | – |
| `pcc`  | `expdp_pcf_20260323_%U.dmp` | – |
| `apig` | `expdp_apig_20260323_%U.dmp` (main) + `expdp_apig_metadata_20260323.dmp` | Excluded large log tables from data; metadata only for those tables. |

Example export command:

```bash
expdp cc/<password> \
  directory=CC_DIR \
  dumpfile=expdp_cc_20260323_%U.dmp \
  logfile=expdp_cc_20260323.log \
  schemas=cc \
  version=12.2 \
  parallel=4 \
  compression=all \
  exclude=statistics
```

## Step 5: Import Data into Pre‑Production
Use the imp_user account (password oracle) to import. For schemas that originally used different tablespace names, remap them to TAB_CC and IDX_CC.

```bash
# Normal imports (no remap needed)
impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_cc_20260323_%U.dmp \
  logfile=impdp_cc_20260323.log schemas=cc parallel=4

impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_crm_20260323_%U.dmp \
  logfile=impdp_crm_20260323.log schemas=crm parallel=4

impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_pot_20260323_%U.dmp \
  logfile=impdp_pot_20260323.log schemas=pot parallel=4

# Imports with tablespace remapping
impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_pcf_20260323_%U.dmp \
  logfile=impdp_pcc_20260323.log \
  REMAP_TABLESPACE=tab_pcc:tab_cc,idx_pcc:idx_cc schemas=pcc parallel=4

impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_qmdb_20260323_%U.dmp \
  logfile=impdp_qmdb_20260323.log \
  REMAP_TABLESPACE=tab_mdb:tab_cc,idx_mdb:idx_cc schemas=qmdb parallel=4

impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_jnc_20260323_%U.dmp \
  logfile=impdp_jnc_20260323.log \
  REMAP_TABLESPACE=tab_jnc:tab_cc,idx_jnc:idx_cc schemas=jnc parallel=4

impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_apig_20260323_%U.dmp \
  logfile=impdp_apig_20260323.log \
  REMAP_TABLESPACE=tab_apig:tab_cc,idx_apig:idx_cc schemas=apig parallel=4

# Additional import for apig metadata (only for excluded tables)
impdp imp_user/oracle@ytlc_preprod_cc \
  directory=CC_DIR dumpfile=expdp_apig_metadata_20260323.dmp \
  logfile=impdp_apig_metadata_20260323.log \
  REMAP_TABLESPACE=tab_apig:tab_cc,idx_apig:idx_cc schemas=apig
```

**Monitor progress**: During import, you can query DBA_DATAPUMP_JOBS to see the status. For large schemas like crm, the import may take a while (e.g., ~37 GB at 26% in the example log).

## Step 6: Post‑Import Tasks
After all imports complete successfully, perform these steps inside the PDB:

1. Recompile invalid objects
```sql
EXEC DBMS_UTILITY.COMPILE_SCHEMA('CC');
EXEC DBMS_UTILITY.COMPILE_SCHEMA('CRM');
EXEC DBMS_UTILITY.COMPILE_SCHEMA('POT');
-- ... repeat for each schema
```

2. Gather statistics (optional but recommended for performance)
```sql
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('CC');
EXEC DBMS_STATS.GATHER_SCHEMA_STATS('CRM');
-- ... other schemas
```

3. Check for import errors – review the .log files for any ORA- or FAILED lines.
   
4. Test connectivity from application side using <password> password.
   
5. Secure the import user (optional):
```sql
ALTER USER imp_user PASSWORD EXPIRE;
-- or REVOKE DBA FROM imp_user;
```
