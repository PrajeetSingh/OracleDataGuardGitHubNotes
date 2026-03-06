# Far Sync Init Parameters

```
DB_NAME = 'PROD'
DB_UNIQUE_NAME = 'PROD_FS'
CONTROL_FILES = ('/u01/oradata/PROD_FS/control01.ctl', '/u01/oradata/PROD_FS/control02.ctl')
LOG_ARCHIVE_CONFIG = 'DG_CONFIG=(PROD, PROD_FS, PROD_STBY)'
LOG_ARCHIVE_DEST_1 = 'LOCATION=/u01/oradata/PROD_FS/arch/ VALID_FOR=(STANDBY_LOGFILES,STANDBY_ROLE) DB_UNIQUE_NAME=PROD_FS'
LOG_ARCHIVE_DEST_2 = 'SERVICE=prod_stby ASYNC VALID_FOR=(STANDBY_LOGFILES,STANDBY_ROLE) DB_UNIQUE_NAME=PROD_STBY'
LOG_ARCHIVE_DEST_STATE_1 = 'ENABLE'
LOG_ARCHIVE_DEST_STATE_2 = 'ENABLE'
FAL_SERVER = 'PROD'
SGA_TARGET = 500M
PGA_AGGREGATE_TARGET = 100M

Example:

*.db_name='cdbapp1'
*.db_unique_name='FSInstScd'
*.compatible='19.0.0'
*.control_files='/u01/app/oracle/oradata/FSInstScd/control01.ctl'
*.db_block_size=8192
*.enable_pluggable_database=true

*.db_recovery_file_dest='/u01/app/oracle/fast_recovery_area'
*.db_recovery_file_dest_size=5368709120
*.diagnostic_dest='/u01/app/oracle'
*.audit_file_dest='/u01/app/oracle/admin/FSInstScd/adump'

*.dg_broker_start=TRUE
*.dg_broker_config_file1='/u01/app/oracle/oradata/FSInstScd/dr1FSInstScd.dat'
*.dg_broker_config_file2='/u01/app/oracle/oradata/FSInstScd/dr2FSInstScd.dat'

*.remote_login_passwordfile='EXCLUSIVE'
*.log_archive_format='%t_%s_%r.dbf'
*.processes=300
*.sga_target=500m
*.pga_aggregate_target=100m


mkdir -p /u01/app/oracle/oradata/FSInstScd
ls -lrth /u01/app/oracle/fast_recovery_area
mkdir -p /u01/app/oracle/fast_recovery_area
ls -lrth /u01/app/oracle
ls -lrth /u01/app/oracle/admin/FSInstScd/adump
mkdir -p /u01/app/oracle/admin/FSInstScd/adump
ls -lrth /u01/app/oracle/oradata/FSInstScd/
cp controlfs.ctl /u01/app/oracle/oradata/FSInstScd/control01.ctl
ls -lrth /u01/app/oracle/oradata/FSInstScd/control01.ctl


ALTER DATABASE ADD STANDBY LOGFILE GROUP 4 ('/u01/oradata/PROD_FS/srl04.log') SIZE 1G;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 5 ('/u01/oradata/PROD_FS/srl05.log') SIZE 1G;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 6 ('/u01/oradata/PROD_FS/srl06.log') SIZE 1G;
ALTER DATABASE ADD STANDBY LOGFILE GROUP 7 ('/u01/oradata/PROD_FS/srl07.log') SIZE 1G;

SELECT GROUP#, TYPE, MEMBER FROM V$LOGFILE WHERE TYPE = 'STANDBY';

