# PostgreSQL High Availability and automatic failover using repmgr
In this article, I will set up a Postgresql high-availability cluster with automatic failover using the REPMGR tool. This setup consists of two servers. One primary and one standby. We can also use three servers. One primary server and two standby servers. With the help of the repmgr tool, when the primary server fails, it will automatically elect a new primary server. This ensures that our PostgreSQL cluster remains highly available and capable of handling failover scenarios.

The primary server is registered to the cluster, and the standby server is created by cloning the primary server. repmgrd monitors the cluster and facilitates automatic failover. It constantly checks the health of the primary server and the standby servers to ensure they are up and running. If the primary server goes down, repmgrd will automatically elect a new primary server from the available standby servers.

Servers:
Pnode = primary server
Snode = standby server

Install PostgreSQL, repmgr:
In this article I used Debian 10 and Postgresql version 15. Install postgresql and repmgr on both servers.

# sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'
# wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
# sudo apt-get update
# sudo apt-get -y install postgresql-15 postgresql-15-repmgr

Set up the primary server:
We need to add some parameters to config file in order to set up high availability.
# update /etc/postgresql/15/main/postgresql.conf with the following values:
 
listen_address = '*'
max_wal_senders = 10
max_replication_slots = 10
wal_level = 'hot_standby'
hot_standby = on
archive_mode = on
archive_command = '/bin/true
wal_log_hints = on
wal_keep_size = 100    
shared_preload_libraries = 'repmgr'

# Then restart postgresql
Systemctl restart postgresql@15-main.service                 
 
# Create database and user
createuser -s repmgr
createdb repmgr -O repmgr
 
# enable remote connection to the primary server
# editing /etc/postgresql/15/main/pg_hba.conf
# this will enable our subnet ip's to connect to the primary server
 
Local	replication	repmgr 				trust
Host	replication	repmgr		192.168.86.0/24	trust
Local	repmgr		repmgr					trust
Host	repmgr		repmgr		192.168.86.0/24	trust
 
Register the primary server into the cluster:
we can now use the repmgr tool to register our primary server node into the cluster. By running the repmgr primary register command on the primary server node, we will create a record of the primary node in the repmgr metadata.

# create a /etc/repmgr.conf file with this content:
 
node_id=1
node_name=Pnode
conninfo='host=<primary node ip> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/data/postgresql/15/main'
failover=automatic
promote_command='repmgr -f /etc/postgresql/13/main/repmgr.conf standby promote'
service_start_command='pg_ctlcluster 15 main start'
service_stop_command='pg_ctlcluster 15 main stop'
service_restart_command='pg_ctlcluster 15 main restart'
service_reload_command='pg_ctlcluster 15 main reload'
 
# and register the node into the cluster
 
repmgr -f /etc/repmgr.conf primary register
 
# check if the cluster is available via
repmgr -f /etc/repmgr.conf cluster show –compact
 
# output should be similar to:
ID | Name | Role    | Status    | Upstream | Location | Priority | Timeline | Connection string
 
----+------+---------+-----------+----------+----------+----------+----------+-------------------------------------------------------------
 
 1  | pnode  | primary | * running |          | default  | 100      | 1        | host=172.7.7.11 user=repmgr dbname=repmgr connect_timeout=2

Set up the secondary server

If the primary node is running the next step is to clone the primary server to create a standby server.

# update /etc/postgresql/15/main/postgresql.conf with the following values:
 
listen_address = '*'
max_wal_senders = 10
max_replication_slots = 10
wal_level = 'hot_standby'
hot_standby = on
archive_mode = on
archive_command = '/bin/true
wal_log_hints = on
wal_keep_size = 100    
shared_preload_libraries = 'repmgr'
 
# enable remote connection to the standby server
# editing /etc/postgresql/15/main/pg_hba.conf
# this will enable our subnet ip's to connect to the standby server
 
Local	replication	repmgr 				trust
Host	replication	repmgr		192.168.86.0/24	trust
Local	repmgr		repmgr					trust
Host	repmgr		repmgr		192.168.86.0/24	trust

# create a clone from the primary server to act as a standby server, 
# create a file repmgr.conf at /etc with this content:
node_id=2
node_name=Snode
conninfo='host=<standby node ip> user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/postgresql/data'
failover=automatic
promote_command='repmgr -f /etc/repmgr.conf standby promote'
service_start_command='pg_ctlcluster 15 main start'
service_stop_command='pg_ctlcluster 15 main stop'
service_restart_command='pg_ctlcluster 15 main restart'
service_reload_command='pg_ctlcluster 15 main reload'

# create a replica / clone from the primary server
# dry run in case you want to test it before calling the clone
repmgr -h <primary node ip> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone --dry-run

# clone
repmgr -h <primary node ip> -U repmgr -d repmgr -f /etc/repmgr.conf standby clone

# register the standby server
repmgr -f /etc/repmgr.conf standby register
Enable repmgrd for monitoring and failover handling

To enable monitoring and automatic failover handling, we need to set up repmgrd on all the PostgreSQL nodes

repmgrd -f /etc/repmgr.conf -d

References: 
https://www.repmgr.org/
https://www.linkedin.com/pulse/postgresql-high-availability-automatic-failover-using-vanzuita
https://maggiminutes.com/install-postgresql-15-on-linux/
