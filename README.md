# Postgres HA Master-Slave arch 
Postgres clustring means a different concept from what's know a cluster. It doesnt mean multiple machines/servers running togather to provide resources and High Availablity. Postgres clustring referes to a cluster of databases reside on one host in which it can provide some featers sich as:
1. Text fixture eg. for automated tests.
2. Bringing up a new database without disturbing existing postgres configuration. 
3. Running multiple versions of postgres at the same time.  

Postgres clustring provides many features but not related to HA concept. 
Postgres cluster consist of two parts:
1. **The Data Directory**: contains Databases, tables, and other logical entities used by postgres tools/resources.
2. **Postgres**: the master process which controls the data director & provide interfaces for manipulating data directory content. 

## Postgres HA
Postgres haigh availablity in Master-Slave archeticture relys on data continous archiving known as log-shipping. Breifly, the master database writes its transaction to WAL file (write-ahead log). When the file reach a certain threshold it will allow it to be sent to the standby database server. As it appears, its asyncronouse process and cnsiderd a legacy method. 
In this Tutorial, we will considre a streaming replication with syncronous process. 

### What's Streaming Replication ?
Streaming replication is a method of replicating data between two or more servers. Standby servers can be kept current by reading a stream of write-ahead log (WAL) records from the Primary Server. This can be synchronous or asynchronous.
A standby server tracks changes made to a primary server, and can be promoted to a primary server if needed. A hot standby server can accept connections and serve read-only queries.
This Method allows for near-real-time replication of data. This means that if one server goes down, the data can be quickly and easily replicated to the other servers.

## Procedure
### Components 
1. One Primary psql database server.
2. One Secondray psql database server.
PS: this procedures uses fedora based system. you can modify it to work for other linux flavors 
### Steps:
**On Primary Server**   
1. Run postgres_installation.sh script
```
sudo yum module install postgresql-server -y 
sudo postgres-setup initdb 
```
2.  Edit host-based authentication file: 
```
vim /etc/postgresql/14/data/pg_hba.conf
```

```
# At the End of the file 
host    replication     rep_user         <IP_Address>/32        scram-sha-256
```

3. Edit the postgres.conf file as follow:
```
/var/lib/pgsql/data/postgresql.conf 
```
```
listen_addresses = '<Your Ip_Address/Hostname>'
port = 5432                             
wal_level = replica
password_encryption = scram-sha-256
max_wal_senders = 10
wal_keep_size = 32
synchronous_commit = remote_apply
synchronous_standby_names = '*'
```
4. Create replication user with the correspond plivirege
```
sudo su postgres 
createuser rep_user --replication -P

#Output:
#Enter password for new role: # Enter any password
#Enter it again:
```

5. Start and enable postgresql service
```
systemctl enable postgresql.service
systemctl start postgresql.service
```

**On Standby Server** 
6. Run postgres_installation.sh script
```
sudo yum module install postgresql-server -y 
sudo postgres-setup initdb 
```
7. Using ```pg_basebackup``` command to sync database from primary db to the standby db:
```
pg_basebackup -R -h <primary_db_ip> -U rep_user -D /var/lib/postgresql/14/data -P
```
-   `-R`  or  `--write-recovery-conf`  - Creates a standby.signal file and appends connection settings to the ‘**postgresql.auto.conf**’ file, easing the process of setting up a standby server.
- 
8. Edit the postgres.conf file as follow:
```
/var/lib/pgsql/data/postgresql.conf 
```
```
listen_addresses = 'Standby_db_ip'
#hot_standby = on		<-- unncomment if you want reading transaction to be from the standby 
primary_conninfo = 'host=<primary_db_ip> port=5432 user=rep_user'
```

9. Start and enable postgresql service:
```
systemctl enable postgresql.service
systemctl restart postgresql.service
```

10. Test Replication [ Execute on the Primary database ]: 
```bash
psql -c "select usename, application_name, client_addr, state, sync_priority, sync_state from pg_stat_replication;"
```
