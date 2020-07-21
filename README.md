#### Logical backup   
https://www.postgresql.org/docs/12/app-pgdump.html  
Contains logical instructions (SQL commands) to recreate the database.
Not enough information to do point in time recovery.
1. dump  (from the database server)   
   * `pg_dump demo > backup.dump`   
2. create new db and restore  
   * `createdb demo_new`  
   * `psql -d demo_new -f backup.dump`

---

#### Physical backup + Point in time recovery   
https://www.postgresql.org/docs/12/continuous-archiving.html  
Contains physical (bytes) information of the database. Combine with WAL (write ahead logs) to do point in time recovery.
1. Setup WAL archiving (in `postgresql.conf`)  
   * `archive_mode = on`  
   * Using /tmp for the demo, but in practice the data in the path should persist (via nfs mount etc.) even if the db server is wiped out  
   `archive_command = 'test ! -f /tmp/wal_archive/%f && cp %p /tmp/wal_archive/%f`
   * `pg_ctl restart`
2. Create physical backup  
   * `pg_basebackup -D /tmp/basebackup` again using tmp for demo only
3. Indicate to Postgres that you are doing a restore  
   * `touch recovery.signal`
4. Wipe and restore database cluster (PGDATA) using basebackup and WAL
   * `pg_ctl stop`
   * store recent WAL  
     `cp -r ./pg_wal/* /tmp/previous_wal/` 
   * `rm -rf ./*`
   * `cp -r /tmp/basebackup/* ./`
   * delete stale WAL from the basebackup  
     `rm -r ./pg_wal/*` 
   * copy recent WAL from the wiped DB  
     `cp -r /tmp/previous_wal/* ./pg_wal/` 
5. Restore
   * Set recovery config (in `postgresql.conf`)  
     * `restore_command = 'cp /tmp/wal_archive/%f %p'`
     * `recovery_target_time = 'Monday, July 20, 2020 4:20:00 PM GMT-07:00'`
   * `pg_ctl start`
   * Resume reading from pg_wal folder after recovering from archive    
     `psql -c 'SELECT pg_wal_replay_resume()'` 

---

#### AWS RDS
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html   
Automated backup and point in time recovery
1. Actions -> Restore to point in time 


---

###### Demo Setup
* `initdb postgres_demo` create cluster  
* `createdb demo` create db  
* `export PGDATA=postgres_demo && export PGDATABASE=demo`  
* `pgbench -i -s 100` generate 100x (ensures some WAL is archived) data  

