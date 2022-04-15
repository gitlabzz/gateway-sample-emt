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
- CREATE USER 'gateway_user'@'%' IDENTIFIED BY 'changeme';
- GRANT ALL PRIVILEGES ON database_name.* TO 'gateway_user'@'%';
- exit
- mysql -u gateway -p
- changeme
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
- mkdir -p ~/merge-dir/apigateway/logs
- mkdir -p ~/merge-dir/apigateway/events


- Build anm image: 
    - ./build_anm_image.py --domain-cert=./certs/mydomain/mydomain-cert.pem --domain-key=./certs/mydomain/mydomain-key.pem --domain-key-pass-file=./pass.txt --anm-username=admin --anm-pass-file=./pass.txt --parent-image=apim_base_202204:7.7 --metrics --merge-dir=/home/asim/merge-dir/apigateway/ --out-image=apim_anm_202204:7.7 --license=/home/asim/merge-dir/apigateway/lic20.lic
    
- touch nopass.txt

-  copy gateway-sample-emt/docker-apim-vanilla-1cass.env
-  gateway-sample-emt/docker-apim-vanilla-3cass-smtp.pol

-  Build gwt image:
    -  ./build_gw_image.py --license=/home/asim/merge-dir/apigateway/lic20.lic --domain-cert=./certs/mydomain/mydomain-cert.pem --domain-key=./certs/mydomain/mydomain-key.pem --domain-key-pass-file=./pass.txt --parent-image=apim_base_202204:7.7 --pol=./docker-apim-vanilla-3cass-smtp.pol --env=./docker-apim-vanilla-1cass.env --fed-pass-file=./nopass.txt --group-id=mydomain-group --merge-dir=/home/asim/merge-dir/apigateway/ --out-image=apim_apig_202204:7.7
               

docker tag apim_apig_202202:7.7.KT romaicus/gateway:7.7
docker push romaicus/gateway:7.7


To run locally:


Run anm: docker run -d -p 8090:8090 --name=anm77 --network=api-gateway-domain -v /home/asim/mydomain/apim77.202202/emt/apigateway/events:/opt/Axway/apigateway/events -e METRICS_DB_URL=jdbc:mysql://172.16.63.175:3306/matrix?useSSL=false -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes apim_anm_202202:7.7.KT


Run gwt: docker run -d --name=apimgr77  --network=api-gateway-domain -p 8075:8075 -p 8065:8065 -p 8080:8080 -v /home/asim/mydomain/apim77.202202/emt/apigateway/events:/opt/Axway/apigateway/events -v /home/asim/mydomain/apim77.202202/emt/apigateway/logs:/opt/Axway/apigateway/logs -v /home/asim/mydomain/apim77.202202/emt/apigateway/conf/licenses:/opt/Axway/apigateway/conf/licenses -e EMT_ANM_HOSTS=anm77:8090 -e EMT_DEPLOYMENT_ENABLED=true -e CASS_HOST1=172.16.63.174 -e CASS_HOST2=172.16.63.174 -e CASS_HOST3=172.16.63.174 -e CASS_PORT1=9042 -e CASS_PORT2=9042 -e CASS_PORT3=9042 -e CASS_USERNAME=cassandra -e CASS_PASS=changeme -e METRICS_DB_URL=jdbc:mysql://172.16.63.175:3306/matrix?useSSL=false -e METRICS_DB_USERNAME=gateway -e METRICS_DB_PASS=changeme -e EMT_TRACE_LEVEL=INFO -e ACCEPT_GENERAL_CONDITIONS=yes apim_apig_202202:7.7.KT





./dbsetup --dburl=jdbc:mysql://172.16.63.175:3306/matrix?allowPublicKeyRetrieval=false --dbuser=gateway --dbpass=changeme




