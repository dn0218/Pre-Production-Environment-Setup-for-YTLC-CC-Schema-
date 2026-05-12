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
