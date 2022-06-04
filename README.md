# Training engineer

## Homework 04 - postgresql

### Instructions

- Поднять 3-ю реплику Postgresql.
- Сделать ее с отставанием по времени.
- Сымитировать падение мастер-ноды.
- Превратить слэйв-ноду в мастер-ноду.

### Tiny log

node-a: 192.168.10.10
node-b: 192.168.10.20
node-c: 192.168.10.30

node-a, node-b, node-c:


    # apt update && apt install -y postgresql


#### Part 1. Two-node master-slave replication

node-a:


    # sudo su
    # cat <<EOF> /etc/postgresql/12/main/conf.d/custom.conf
    > listen_addresses = '*'
    > hot_standby = on
    > wal_level = replica
    > wal_log_hints = on
    > max_wal_senders = 10
    > EOF
    # cat <<EOF>> /etc/postgresql/12/main/pg_hba.conf
    > host all all 192.168.10.20/32 md5
    > host all all 192.168.10.30/32 md5
    > host replication postgres 192.168.10.20/32 md5
    > host replication postgres 192.168.10.30/32 md5
    > EOF
    # su postgres
    $ psql
    postgres=# ALTER ROLE postgres PASSWORD 'qwerty42';
    postgres=# CREATE DATABASE testdb;
    postgres=# \c testdb;
    testdb=# create table example (col1 varchar);
    testdb=# insert into example values('first record');
    testdb=# insert into example values('second record');
    testdb=# select * from example;
    ...
    # systemctl restart postgresql


node-b:


    # systemctl stop postgresql
    # su postgres
    $ cd /var/lib/postgresql/12
    $ rm -rf main/*
    $ pg_basebackup -P -R -X stream -c fast -h 192.168.10.10 -U postgres -D ./main
    ...
    # systemctl restart postgresql
    # sudo -u postgres psql -d testdb -c 'select * from example;'
         col1      
    ---------------
    first record
    second record
    (2 rows)


node-a:


    # su postges
    $ psql
    postgres=# \c testdb
    testdb=# insert into example values('third record');


node-b:


    # sudo -u postgres psql -d testdb -c 'select * from example;'
         col1      
    ---------------
    first record
    second record
    third record
    (3 rows)


#### Part 2. Yet another node with time lag replication

node-c:


    # systemctl stop postgresql
    # su postgres
    $ cd /var/lib/postgresql/12
    $ rm -rf main/*
    $ pg_basebackup -P -R -X stream -c fast -h 192.168.10.10 -U postgres -D ./main
    # systemctl start postgresql
    ...
    # echo "recovery_min_apply_delay = '3min'" >> /etc/postgresql/12/main/conf.d/custom.conf
    # systemctl restart postgresql


node-a:


    testdb=# insert into example values('fourth record');


node-b:

    
    testdb=# select * from example;
         col1      
    ---------------
    first record
    second record
    third record
    fourth record
    (4 rows)


node-c:


    testdb=# select * from example;
         col1      
    ---------------
    first record
    second record
    third record
    (3 rows)


...three minutes later:


    testdb=# select * from example;
         col1      
    ---------------
    first record
    second record
    third record
    fourth record
    (4 rows)


#### Part 3. Shit happens

node-a:


    # systemctl stop postgresql


node-b:


    # cat <<EOF> /etc/postgresql/12/main/conf.d/custom.conf
    > listen_addresses = '*'
    > hot_standby = on
    > wal_level = replica
    > wal_log_hints = on
    > max_wal_senders = 10
    > EOF
    # cat <<EOF>> /etc/postgresql/12/main/pg_hba.conf
    > host all all 192.168.10.30/32 md5
    > host replication postgres 192.168.10.30/32 md5
    > EOF
    # su postgres
    $ /usr/lib/postgresql/12/bin/pg_ctl promote -D /var/lib/postgresql/12/main
    ...
    $ psql    
    postgres=# ALTER ROLE postgres PASSWORD '42qwerty';
    postgres=# \c testdb;
    testdb=# create table example2 (col1 varchar);
    testdb=# insert into example2 values('i am alive!');


node-c:


    # rm /etc/postgresql/12/main/conf.d/custom.conf
    $ psql
    postgres=# ALTER SYSTEM set primary_conninfo='user=postgres password=42qwerty host=192.168.10.20 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any';
    ...
    # systemctl restart postgresql
    # sudo -u postgres psql -d testdb -c 'select * from example2;'
        col1     
    -------------
    i am alive!
    (1 row)


done
