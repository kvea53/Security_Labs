# HDF 3.0 Active Directory Lab guide

## Install HDF 3.0 (plus Druid)

- First decide which node will be ambari-server

- Run on non-amabri nodes to install agents
```
export ambari_server=ip-172-30-0-206.us-west-2.compute.internal
curl -sSL https://raw.githubusercontent.com/seanorama/ambari-bootstrap/master/ambari-bootstrap.sh | sudo -E sh
```

- run on ambari node to install ambari-server
```
export install_ambari_server=true
curl -sSL https://raw.githubusercontent.com/seanorama/ambari-bootstrap/master/ambari-bootstrap.sh | sudo -E sh
```

- run remaining steps on ambari-server node

- install 
```
sudo yum localinstall -y https://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm
sudo yum install -y git python-argparse epel-release mysql-connector-java* mysql-community-server

# MySQL Setup to keep the new services separate from the originals
echo Database setup...
sudo systemctl enable mysqld.service
sudo systemctl start mysqld.service
```

- Run below to:
  - 1. reset Mysql password to temp value and create druid/superset/registry/streamline schemas and users
  - 2. sets passwords for druid/superset/registry/streamline users to StrongPassword
```
#extract system generated Mysql password
oldpass=$( grep 'temporary.*root@localhost' /var/log/mysqld.log | tail -n 1 | sed 's/.*root@localhost: //' )

#create sql file 
cat << EOF > mysql-setup.sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'Secur1ty!'; 
uninstall plugin validate_password;
CREATE DATABASE druid DEFAULT CHARACTER SET utf8; CREATE DATABASE superset DEFAULT CHARACTER SET utf8; CREATE DATABASE registry DEFAULT CHARACTER SET utf8; CREATE DATABASE streamline DEFAULT CHARACTER SET utf8; 
CREATE USER 'druid'@'%' IDENTIFIED BY 'StrongPassword'; CREATE USER 'superset'@'%' IDENTIFIED BY 'StrongPassword'; CREATE USER 'registry'@'%' IDENTIFIED BY 'StrongPassword'; CREATE USER 'streamline'@'%' IDENTIFIED BY 'StrongPassword'; 
GRANT ALL PRIVILEGES ON *.* TO 'druid'@'%' WITH GRANT OPTION; GRANT ALL PRIVILEGES ON *.* TO 'superset'@'%' WITH GRANT OPTION; GRANT ALL PRIVILEGES ON registry.* TO 'registry'@'%' WITH GRANT OPTION ; GRANT ALL PRIVILEGES ON streamline.* TO 'streamline'@'%' WITH GRANT OPTION ; 
commit; 
EOF

#run sql file
mysql -h localhost -u root -p"$oldpass" --connect-expired-password < mysql-setup.sql
```

- change Mysql password to StrongPassword
```
mysqladmin -u root -p'Secur1ty!' password StrongPassword

#test password and confirm dbs created
mysql -u root -pStrongPassword -e 'show databases;'

```

- Install mysql jar and HDF mpack and restart 
```
sudo ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
sudo ambari-server install-mpack --verbose --mpack=http://public-repo-1.hortonworks.com/HDF/centos7/3.x/updates/3.0.0.0/tars/hdf_ambari_mp/hdf-ambari-mpack-3.0.0.0-453.tar.gz
sudo ambari-server restart
```

- wait 30s then open ambari UI via browser on port 8080


- Add Services HDFS, Storm, Kafka, NiFi, Registry, Streaming Analytics Manager, Druid
- Assign masters
  - keep SAM/registry/Druid on Ambari node (where Mysql was installed)
  - move Storm, Smartsense, Metrics related to seperate nodes 
  - add nifi on all nodes

- keep default clients

- Customize Services: 
  - Password/Secrets are 'StrongPassword' for all services
  - All Database types are MySql; 
  - All hostnames are FDQDN of ambari node, mysql port 3306; 
  - Update the mysql URLs with FQDN manually; In SAM registry.url to http://FQDN:7788/api/v1; 
  - In SAM, update streamline.dashboard.url to http://FQDN:9089
  - Druid
    - druid.storage.storageDirectory = /user/druid/data
    - druid.storage.type = local
    - Superset: email: a@b.c, firstname: admin, lastname: jones; Change SUPERSET_WEBSERVER_PORT from 9088 to 9089;


