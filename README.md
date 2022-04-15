# gateway-sample-emt
gateway-sample-emt



## Cassandra Setup
- Install JDK 1.8
- Unzip cassandra.zip
- Update cassandra/bin/cassandra.in.sh:
    update JAVA_HOME
    
- Update cassandra/conf/cassandra.yaml:
    - start_rpc: true
    - rpc_address: 0.0.0.0
    - broadcast_rpc_address: node_ip
    - listen_address:node_ip
    - seed_provider -> class_name -> seeds -> node_ip


## MySQL Setup
- sudo apt update -y
- sudo apt install mysql-server
- sudo systemctl start mysql.service
- sudo systemctl status mysql.service
- sudo systemctl enable mysql.service
- sudo mysql_secure_installation
- sudo vi /etc/mysql/mysql.conf.d/mysqld.cnf -> bind-address = 0.0.0.0
- sudo systemctl restart mysql
- sudo mysql
- create database metrics;
- CREATE USER 'gateway_user'@'%' IDENTIFIED BY 'changeme';
- GRANT ALL PRIVILEGES ON metrics.* TO 'gateway_user'@'%';
- exit
- mysql -u gateway_user -p
- changeme
- show databases;
- exit
- sudo ufw disable


## Test MySQL Remote Connectivity
- Go to Gateway EMT machine that has Docker installed already
- docker run --name adminer-db -p 8086:8080 -d adminer (this will run Adminer container)
- In browser http://localhost:8086 (enter database IP and gateway_user/ changeme to connect)


## Gateway Setup

- unzip APIGateway_7.7.20220228-DockerScripts-2.4.0-1.tar in any folder let's say ~/gateway_emt_scripts
- copy APIGateway_7.7.20220228_Install_linux-x86-64_BN02.run to ~/gateway_emt_scripts
- tar -xf APIGateway_7.7.20220228-DockerScripts-2.4.0-1.tar
- cd apigw-emt-scripts-2.4.0/

- Create dummy password file: 
    - echo changeme > pass.txt

- Generate cert: 
    - ./gen_domain_cert.py --domain-id=mydomain --pass-file=./pass.txt

- Build base image: 
    - ./build_base_image.py --installer=../APIGateway_7.7.20220228_Install_linux-x86-64_BN02.run --os=centos7 --out-image=apim_base_202204:7.7

- mkdir -p ~/merge-dir/apigateway/ext/lib
- copy lic20.lic in ~/merge-dir/apigateway
- copy mysql-connector-java-8.0.28.jar in ~/merge-dir/apigateway/ext/lib 
- mkdir -p ~/merge-dir/apigateway/conf


- Build anm image: 
    - ./build_anm_image.py --domain-cert=./certs/mydomain/mydomain-cert.pem --domain-key=./certs/mydomain/mydomain-key.pem --domain-key-pass-file=./pass.txt --anm-username=admin --anm-pass-file=./pass.txt --parent-image=apim_base_202204:7.7 --metrics --merge-dir=/home/asim/merge-dir/apigateway/ --out-image=apim_anm_202204:7.7 --license=/home/asim/merge-dir/apigateway/lic20.lic
    
- docker tag apim_anm_202204:7.7 romaicus/anm:7.7
- docker push romaicus/anm:7.7
    
- touch nopass.txt

-  copy docker-apim-vanilla-1cass.env
-  copy docker-apim-vanilla-3cass-smtp.pol

-  Build gwt image:
    -  ./build_gw_image.py --license=/home/asim/merge-dir/apigateway/lic20.lic --domain-cert=./certs/mydomain/mydomain-cert.pem --domain-key=./certs/mydomain/mydomain-key.pem --domain-key-pass-file=./pass.txt --parent-image=apim_base_202204:7.7 --pol=./docker-apim-vanilla-3cass-smtp.pol --env=./docker-apim-vanilla-1cass.env --fed-pass-file=./nopass.txt --group-id=mydomain-group --merge-dir=/home/asim/merge-dir/apigateway/ --out-image=apim_apig_202204:7.7
               

- docker tag apim_apig_202204:7.7 romaicus/gateway:7.7
- docker push romaicus/gateway:7.7


To run locally:

- mkdir -p ~/merge-dir/apigateway/logs
- mkdir -p ~/merge-dir/apigateway/events
- mkdir -p ~/merge-dir/apigateway/conf/licenses
- cp lic20.lic conf/licenses/
- docker network create api-gateway-domain

Run anm: 

docker run -d -p 8090:8090 --name=anm77 --network=api-gateway-domain -v /home/asim/merge-dir/apigateway/events:/opt/Axway/apigateway/events -e METRICS_DB_URL=jdbc:mysql://172.16.63.178:3306/metrics?useSSL=false -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes apim_anm_202204:7.7

OR 

docker run -d -p 8090:8090 --name=anm77 --network=api-gateway-domain -v /home/asim/merge-dir/apigateway/events:/opt/Axway/apigateway/events -e METRICS_DB_URL=jdbc:mysql://172.16.63.178:3306/metrics?useSSL=false -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes romaicus/anm:7.7


Run gwt: 

docker run -d --name=apimgr77 --network=api-gateway-domain -p 8075:8075 -p 8065:8065 -p 8080:8080 -v /home/asim/merge-dir/apigateway/events:/opt/Axway/apigateway/events -v /home/asim/merge-dir/apigateway/logs:/opt/Axway/apigateway/logs -v /home/asim/merge-dir/apigateway/conf/licenses:/opt/Axway/apigateway/conf/licenses -e EMT_ANM_HOSTS=anm77:8090 -e EMT_DEPLOYMENT_ENABLED=true -e CASS_HOST1=172.16.63.177 -e CASS_HOST2=172.16.63.177 -e CASS_HOST3=172.16.63.177 -e CASS_PORT1=9042 -e CASS_PORT2=9042 -e CASS_PORT3=9042 -e CASS_USERNAME=cassandra -e CASS_PASS=changeme -e METRICS_DB_URL=jdbc:mysql://172.16.63.178:3306/metrics?useSSL=false -e METRICS_DB_USERNAME=gateway_user -e METRICS_DB_PASS=changeme -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes apim_apig_202204:7.7


OR

docker run -d --name=apimgr77 --network=api-gateway-domain -p 8075:8075 -p 8065:8065 -p 8080:8080 -v /home/asim/merge-dir/apigateway/events:/opt/Axway/apigateway/events -v /home/asim/merge-dir/apigateway/logs:/opt/Axway/apigateway/logs -v /home/asim/merge-dir/apigateway/conf/licenses:/opt/Axway/apigateway/conf/licenses -e EMT_ANM_HOSTS=anm77:8090 -e EMT_DEPLOYMENT_ENABLED=true -e CASS_HOST1=172.16.63.177 -e CASS_HOST2=172.16.63.177 -e CASS_HOST3=172.16.63.177 -e CASS_PORT1=9042 -e CASS_PORT2=9042 -e CASS_PORT3=9042 -e CASS_USERNAME=cassandra -e CASS_PASS=changeme -e METRICS_DB_URL=jdbc:mysql://172.16.63.178:3306/metrics?useSSL=false -e METRICS_DB_USERNAME=gateway_user -e METRICS_DB_PASS=changeme -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes romaicus/gateway:7.7


https://localhost:8075/home
https://localhost:8090/

- docker exec -it apimgr77 /bin/bash
- cd /opt/Axway/apigateway/posix/bin
- ./dbsetup --dburl=jdbc:mysql://172.16.63.178:3306/metrics?allowPublicKeyRetrieval=false --dbuser=gateway_user --dbpass=changeme

look for message:
New database
About to upgrade schema. Please note that this operation may take some time for very large databases
Schema successfully upgraded to: 002-leaf

mysql -u gateway_user -p
changeme
show databases;
use metrics;
mysql> show tables;
+------------------------+
| Tables_in_metrics      |
+------------------------+
| audit_log_points       |
| audit_log_sign         |
| audit_message_payload  |
| metric_group_types     |
| metric_group_types_map |
| metric_groups          |
| metric_types           |
| metrics_alerts         |
| metrics_data           |
| process_groups         |
| processes              |
| time_window_types      |
| transaction_data       |
| versions               |
+------------------------+
14 rows in set (0.00 sec)





