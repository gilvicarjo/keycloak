# keycloak

# Setup Keycloak

## Installing JDK 17

```
sudo dnf install --assumeyes java-17-openjdk-devel.x86_64
sudo dnf install java-17-openjdk
javac -version
```

## Configure Initial KEYCLOAK_ADMIN and KEYCLOAK_ADMIN_PASSWORD

Open the file vim /etc/environment and define the Values for the Keys

```
KEYCLOAK_ADMIN=admin.${env}
KEYCLOAK_ADMIN_PASSWORD=VjA5bUNSUVN5blRqZ0E9PQo=YOUR_PASSWORD
```
After that:
```
source /etc/environment
```
And then, validate the values:

```
$ echo $KEYCLOAK_ADMIN
admin.qld
```
Make sure, you have the values at the session you will run Keycloak.

## Installing Keycloak

Create the path and go there to get the Keycloak package:
```
mkdir -p /opt/keycloak && cd /opt/keycloak
wget https://github.com/keycloak/keycloak/releases/download/23.0.6/keycloak-23.0.6.tar.gz
tar -xvzf keycloak-23.0.6.tar.gz
mv keycloak-23.0.6 23.0.6
ln -s /opt/keycloak/23.0.6 /opt/keycloak/current
groupadd keycloak
useradd -r -g keycloak -d /opt/keycloak -s /sbin/nologin keycloak
chown -R keycloak: /opt/keycloak
chmod o+x /opt/keycloak/23.0.6/bin/
```
## Copy Certificates

Attach certificates from the folder /etc/ssl/certs/ from any other keycloak server in the organization.
```
tmlmobilidade.pt.crt.pem
tmlmobilidade.pt.key.pem
```

## Copy Themes Directory (after migrate Database)
Ref: https://www.keycloak.org/docs/latest/upgrading/index.html

If you have created any custom themes they must be migrated to the new server. Any changes to the built-in themes might need to be reflected in your custom themes, depending on which aspects you have customized.

You must copy your custom themes from the old server themes directory to the new server themes directory. After that you need to review the changes below and consider if the changes need to be applied to your custom theme.

In summary:

If you have customized any of the changed templates listed below you need to compare the template from the base theme to see if there are changes you need to apply.

If you have customized any of the styles and are extending the Keycloak themes you need to review the changes to the styles. If you are extending the base theme you can skip this step.

If you have customized messages you might need to change the key or value or to add additional messages.

## Copy Themes Directory

Copy all files from 

## Configure keycloak.conf

For a first build add the following content for the /opt/keycloak/23.0.6/conf/keycloak.conf file

```
db=postgres
db-username=sa_keycloak_${env}
db-password=YOUR_:PASSWORD
db-url=jdbc:postgresql://10.129.42.10:5432/db_keycloak_${env}
https-certificate-file=/etc/ssl/certs/tmlmobilidade.pt.crt.pem
https-certificate-key-file=/etc/ssl/certs/tmlmobilidade.pt.key.pem
http-relative-path=/auth
#proxy=reencrypt
http-enabled=true
http-port=8180
hostname-url=http://localhost:8180/auth/
hostname-admin-url=http://localhost:8180/auth/
log=console,file
log-level=TRACE
log-file=/var/log/keycloak/server.log
log-console-output=default
features=admin-fine-grained-authz,scripts,token-exchange
```

Now go to /opt/keycloak/23.0.6/conf/ and run:

```
./kc.sh build
./kc.sh show-config
./kc.sh start-dev
```
Make sure you tested all the connections maily with Postgre before move forward with the next steps.

After that you can procced with:

```
./kc.sh build
./kc.sh start
```
And after validate a reasonable production setup, you can setup the service in the next section

## Configure keycloak.service





## Copy 

# Setup Database

## Install PostgreSQL 15
Ref: https://medium.com/@umairhassan27/setting-up-postgresql-replication-on-slave-server-a-step-by-step-guide-1ff36bb9a47f
```
vim /etc/yum.repos.d/pgdg-oraclelinux.repo
```
Add the following content to the file pgdg-oraclelinux.repo file:
```
[pgdg15]
name=PostgreSQL 15 for Oracle Linux $releasever - $basearch
baseurl=https://download.postgresql.org/pub/repos/yum/15/redhat/rhel-$releasever-$basearch
enabled=1
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-PGDG-15
```
Save and exit the text editor.
Install PostgreSQL 15 Server:
```
sudo dnf install postgresql15-server -y
```
Initialize the database and start the PostgreSQL service:
```
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```

Until here, you can do the same steps in the Standby PostgreSQL Server

## Create a user for Replication (Now at the PRIMARY Server)

Only at the MASTER/PRIMARY PostgreSQL Server create a user for Replication purposes:
```
sudo -u postgres psql
CREATE USER replicator_${env} WITH REPLICATION ENCRYPTED PASSWORD 'YOUR_PASSWORD'; _(generated by openssl rand -base64 10 | base64)_
```
Set a password for the default PostgreSQL user:
```
sudo passwd postgres
```
You can now access PostgreSQL using the psql command-line tool or connect to it using a GUI tool like DBeaver or pgAdmin

## Configure Network Connectivity in PostgreSQL

### postgresql.conf file

In the file /var/lib/pgsql/15/data/postgresql.conf
Add the following line:
```
listen_addresses = '*'
```
The file should have this parameters:
```
max_connections = 100                   # (change requires restart)
shared_buffers = 128MB                  # min 128kB
dynamic_shared_memory_type = posix      # the default is usually the first option
max_wal_size = 1GB
min_wal_size = 80MB
listen_addresses = '*'
log_directory = 'log'                   # directory where log files are written,
log_filename = 'postgresql-%a.log'      # log file name pattern,
log_rotation_age = 1d                   # Automatic rotation of logfiles will
log_rotation_size = 0                   # Automatic rotation of logfiles will
log_truncate_on_rotation = on           # If on, an existing log file with the
log_line_prefix = '%m [%p] '            # special values:
log_timezone = 'Europe/Lisbon'
datestyle = 'iso, mdy'
timezone = 'Europe/Lisbon'
lc_messages = 'en_US.UTF-8'                     # locale for system error message
lc_monetary = 'en_US.UTF-8'                     # locale for monetary formatting
lc_numeric = 'en_US.UTF-8'                      # locale for number formatting
lc_time = 'en_US.UTF-8'                         # locale for time formatting
default_text_search_config = 'pg_catalog.english'
```

### pg_hba.conf file 

In the file /var/lib/pgsql/15/data/pg_hba.conf
Add the following lines in the end of the file.

For example:
```
#Replication From Postgres Backup
host    replication     replicator_qld  10.129.42.11/32         md5
#Jumpserver
host    db_keycloak_qld sa_keycloak_qld 10.129.61.11/32         md5
host    db_keycloak_qld sa_keycloak_qld ::/0                    md5
host    postgres        sa_keycloak_qld 10.129.61.11/32         md5
host    postgres        sa_keycloak_qld ::/0                    md5
host    postgres        postgres        10.129.61.11/32         md5
host    postgres        postgres        ::/0                    md5
#KeycloakPrimary
host    db_keycloak_qld sa_keycloak_qld 10.129.41.10/32         md5
host    db_keycloak_qld sa_keycloak_qld ::/0                    md5
host    postgres        sa_keycloak_qld 10.129.41.10/32         md5
host    postgres        sa_keycloak_qld ::/0                    md5
host    postgres        postgres        10.129.41.10/32         md5
host    postgres        postgres        ::/0                    md5
#KeycloakBackup
host    db_keycloak_qld sa_keycloak_qld 10.129.41.11/32         md5
host    db_keycloak_qld sa_keycloak_qld ::/0                    md5
host    postgres        sa_keycloak_qld 10.129.41.11/32         md5
host    postgres        sa_keycloak_qld ::/0                    md5
host    postgres        postgres        10.129.41.11/32         md5
host    postgres        postgres        ::/0                    md5
```
Make sure, both PRIMARY and STANDBY Servers have the appropriate configs for this file.

### Configure Replication (NOW in the Standby Server)

In the STANDBY Server remove the data folder content:

```
rm -rf /var/lib/pgsql/15/data/*
```
Next, make sure both server have the same permission configs
```
sudo chown -R postgres:postgres /var/lib/pgsql/15/data
sudo chmod -R 700 /var/lib/pgsql
```
Then, Copy files from the master using pg_basebackup.

For example:
```
pg_basebackup -h 10.129.42.10 -D /var/lib/pgsql/15/data -U replicator_qld -P -v -R -X stream
```
After that, restart the service:
```
sudo systemctl restart postgresql-15
```
To validate, create a database.

For example:
```
CREATE DATABASE test_replication;
```

And check in the both sides.


## Configure Keycloak Database

```
sudo -u postgres psql
CREATE DATABASE db_keycloak_${env};
CREATE USER sa_keycloak_${env} WITH PASSWORD 'YOUR_PASSWORD'; _(generated by openssl rand -base64 10 | base64)_
GRANT ALL PRIVILEGES ON DATABASE db_keycloak_${env} TO sa_keycloak_${env};
ALTER USER sa_keycloak_${env} WITH SUPERUSER;
```

## Connect to the Database using GUI tool (DBeaver)

In this case it´s need to set a tunnel behind a Jumpserver:
```
ssh -L localhost:5433:<PostgresIP>:<ServicePort> -p 8022 -i ~/.ssh/<pem key> <user>@<JumpServerIP>
```
Here the parameters for the GUI Tool following the tunnel above:
```
Host: localhost
Port: 5433
Database: db_keycloak_${env}
Username: sa_keycloak_${env}
Password: YOUR_PASSWORD
```

# Setup Webserver

## Configure Certificates

In the path /etc/ssl/certs/
```
openssl genrsa -out tmlmobilidade.key 2048
openssl req -new -key tmlmobilidade.key -out tmlmobilidade.csr
openssl x509 -req -days 365 -in tmlmobilidade.csr -signkey tmlmobilidade.key -out tmlmobilidade.crt
cat tmlmobilidade.crt tmlmobilidade.key > tmlmobilidade.pem
```

## Install HAProxy

```
sudo dnf update -y
sudo dnf install haproxy -y
mkdir /var/log/haproxy
touch /var/log/haproxy/haproxy.log
chown -R haproxy:haproxy /var/log/haproxy/
vim /etc/haproxy/haproxy.cfg

```
The haproxy.cfg, should look like this:
```
global
    log         /var/log/haproxy/haproxy.log        local0
    log         /var/log/haproxy/haproxy.log        local1  notice
    chroot      /var/lib/haproxy
    stats       socket          /run/haproxy/admin.sock         mode    660     level   admin
    stats       timeout         30s
    user        haproxy
    group       haproxy
    daemon
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    timeout connect         10s
    timeout client          1m
    timeout server          1m
frontend keycloak_http
    bind *:8179
    redirect scheme https if !{ ssl_fc }
frontend keycloak_https
    bind *:9444 ssl crt /etc/ssl/certs/tmlmobilidade.pem
    mode http
    default_backend keycloak_nodes

backend keycloak_nodes
    mode http
    balance roundrobin
    server node1 10.129.41.10:8180 check
    server node2 10.129.41.11:8180 check
```
After configure haproxy.cfg, it'a time to validate the config
```
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
Configuration file is valid
sudo systemctl restart haproxy
```

## Configure HTTPD

```
sudo dnf install httpd mod_ssl
```
# Database Migration

## Realms migration
Rule: Create realms from scratch on the new Keycloak instance.

In the database the Realms as stored in the table realm, be aware of the new Realm IDs created which you will need for the next steps.

## Clients migration
Rule: When creating the new Realms, Keycloak automatically create some default resources, one of them are the clients. Be aware of the new client IDs for this default initial clients, which you will need for the next steps.

### Customized Clients
The idea is to migrate only the customized clients, and the good news here is, once migrated they will remain with the same client_id.

The users are linked with clients by the service_account_client_link column which is the same client_id column in client table.

### Default Clients
**Yes, after the migration of users and clients, the users that have link with default clients will need to have this registers updated or recreated.**

## Users migration
Users are stored in the user_entity table. They only need to set the column realm_id for the new realm ID created in the previous step.

**One thing is missing here: verify the accounts.**

## Credentials migration
The credential table is really simple it's a source of truth and only the table user_entity (Users) has link with it.

The column id in the user_entity must match the column user_id in credential table.

## Groups migration

Groups are stored in the table keycloak_group

Each register only refers for a upstream folder (or not) and a realm_id.

So the idea here in Groups is the same for Users, It just need to have the realm_id updated for the new ones in the new Keycloak setup.

After migrate Users and Groups keeping theirs source Ids, now It's important to migrate the table user_group_membership. The name explains itself.

## Roles migration

The roles are stored in keycloak_role table. 

### Customized Roles (167 cases)
This query give us the customized roles 

```
SELECT id, client_realm_constraint, client_role, description, "name", realm_id, client, realm
FROM public.keycloak_role
and remove the ones that has variables refereces in description column.
```
So, this table has columns
client_realm_constraint: that can refer to the Realm or the client with this role.
When it refers to clients, no worries, you already migrate the clients in the step before! It will be there waiting for this role to be attached.
When it refers to the Realm, you only need to replace it with the new Realm ID.

#### Inside the Realm, How is a User associated with a Role?

A: by the table user_role_mapping. So let´s migrate this table too.

And this query will bring this for us:

```
SELECT role_id, user_id
FROM public.user_role_mapping
where role_id in (''id','id2','id3',...)
```
These IDs, are only IDs we imported in our keycloak_role table. We dont wanna have user_role associations with roles (default ones) that we didnt imported yet. 

With that, we will now apply a query like that:

```
INSERT INTO public.user_role_mapping (role_id,user_id) VALUES
	 ('role_id1','user_id1'),
	 ('role_id1','user_id2'),
```

### Default Roles

For the default roles, the steps are:

1) Filter in the user_role_mapping (from the old keycloak) by realm_id the ids from the roles (manage-account, offline_access, uma_authorization, view-profile, manage-clients, manage-users).

this 2 queries are a example of that and will help you 

```
SELECT id, "name"
FROM public.keycloak_role
where realm_id = 'viva' or client_realm_constraint = 'viva';


SELECT role_id, user_id
FROM public.user_role_mapping
where role_id in (
'b552b24d-130f-42aa-a060-86080b84337c',  (manage-account)
'a193107e-7815-4c48-8bad-11ad587520a2',  (offline_access)
'd8bd32ff-a3d4-4bdc-ba2e-500f2b5f3f11',  (uma_authorization)
'962620cb-f894-4ceb-892a-9d0fc3dbd503',  (view-profile)
'abde0cbf-0ce4-41d7-b890-930dd9764f42',  (manage-clients)
'93cbe46f-bb7d-4098-a049-49fdff4410df'   (manage-users)
)
```

2) Then, export this result (export data - in Dbeaver) in a SQL format
3) Now the idea is to replace the role_id of the same roles in the new keycloak.
4) After that, apply this INSERT in the new keycloak database.

# Configure Zabbix Monitoring

For **DEV Environment**, the dashboard created: http://monitorizacao.tmlmobilidade.pt/zabbix/zabbix.php?action=dashboard.view&dashboardid=298

For **QLD Environment**, the dashboard created: http://monitorizacao.tmlmobilidade.pt/zabbix/zabbix.php?action=dashboard.view&dashboardid=299

For **PRD Environment**, the dashboard created: 




  
