# SpringBootAppToAzurePaaS
Develop a Spring Boot web application.
Connect your application to a MySQL database.
Deploy the web application to Azure App Service.

## Prerequisites
<ul style="list-style-type: circle;">
  <li>An Azure subscription</li>
  <li>Local installations of Java JDK (1.8 or later)</li>
  <li>Maven (3.0 or later)</li>
  <li>Azure CLI (2.12 or later)</li>
</ul>

## Setup Steps

```bash
az account show
```

```bash
AZ_RESOURCE_GROUP=azure-spring-workshop
AZ_DATABASE_NAME=<YOUR_DATABASE_NAME>
AZ_LOCATION=<YOUR_AZURE_REGION>
AZ_MYSQL_USERNAME=spring
AZ_MYSQL_PASSWORD=<YOUR_MYSQL_PASSWORD>
AZ_LOCAL_IP_ADDRESS=<YOUR_LOCAL_IP_ADDRESS>
```

```bash
# install jq (jq is a lightweight and flexible command-line JSON processor akin to sed,awk,grep, and friends for JSON data.)
wget -O /usr/local/bin/jq https://github.com/stedolan/jq/releases/download/jq-1.6/jq-win64.exe
```

```bash
az group create \
    --name $AZ_RESOURCE_GROUP \
    --location $AZ_LOCATION \
    | jq
```

```bash
#Obsolete (create mysql db ref: https://learn.microsoft.com/en-us/azure/mysql/migrate/whats-happening-to-mysql-single-server):
az mysql server create \
--resource-group $AZ_RESOURCE_GROUP \
--name $AZ_DATABASE_NAME \
--location $AZ_LOCATION \
--sku-name B_Gen5_1 \
--admin-user $AZ_MYSQL_USERNAME \
--admin-password $AZ_MYSQL_PASSWORD \
| jq
```

```bash
# To see defaults and min max:
az mysql flexible-server create --help
az mysql flexible-server list-skus --location "northeurope"
```

```bash
# Recommended (create sql db):
az mysql flexible-server create \
--resource-group $AZ_RESOURCE_GROUP \
--name $AZ_DATABASE_NAME \
--location $AZ_LOCATION \
--storage-size 5 \
--admin-user $AZ_MYSQL_USERNAME \
--admin-password $AZ_MYSQL_PASSWORD \
--public-access $AZ_LOCAL_IP_ADDRESS \
| jq
```

```bash
# Create firewall rule (Obsolete  ref: https://learn.microsoft.com/en-us/azure/mysql/migrate/whats-happening-to-mysql-single-server)
az mysql server firewall-rule create \
    --resource-group $AZ_RESOURCE_GROUP \
    --name $AZ_DATABASE_NAME \
    --server-name $AZ_DATABASE_NAME \
    --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0 \
    | jq
```
# If not using a ssl certificate, diable require_secure_transport
```bash
 az mysql flexible-server parameter set \
--resource-group $AZ_RESOURCE_GROUP \
--server-name $AZ_DATABASE_NAME \
--name require_secure_transport \
--value OFF
```

# Create firewall rule (flexible)

```bash
# Create firewall rule (flexible)
az mysql flexible-server firewall-rule create \
--name $AZ_DATABASE_NAME \
--resource-group $AZ_RESOURCE_GROUP \
--rule-name AllowAzureIPs \
--start-ip-address 0.0.0.0 \
--end-ip-address 0.0.0.0 | jq
```
## Configure db (init db)

```bash
#Obsolete (ref: https://learn.microsoft.com/en-us/azure/mysql/migrate/whats-happening-to-mysql-single-server)
az mysql db create \
--resource-group $AZ_RESOURCE_GROUP \
--name demo \
--server-name $AZ_DATABASE_NAME \
| jq
```

```bash
az mysql flexible-server db create \
--resource-group $AZ_RESOURCE_GROUP \
--server-name $AZ_DATABASE_NAME \
--database-name demo | jq
```

## Verify that the Azure MySQL server is running
```bash
az mysql flexible-server show --name $AZ_DATABASE_NAME --resource-group $AZ_RESOURCE_GROUP
```

## Test db Connection
```bash
mysql -h $AZ_DATABASE_NAME.mysql.database.azure.com \
--user $AZ_MYSQL_USERNAME@tenant.onmicrosoft.com \
--password=$(az account get-access-token --resource-type oss-rdbms --output tsv --query accessToken)

mysql -h $AZ_DATABASE_NAME.mysql.database.azure.com -u $AZ_MYSQL_USERNAME -p 
#--ssl-mode=REQUIRED --ssl-ca=DigiCertGlobalRootCA.crt.pem
```

## Generate the application by using Spring Initializr
```bash
#Find a compatible bootversion for the spring project
curl -s https://start.spring.io/actuator/info

BOOT_RELEASE=$(curl -s https://start.spring.io/actuator/info | jq -r '.build.versions["spring-boot"]')
JAVA_HOME_VERSION=$(echo $JAVA_HOME | grep -o '[0-9]\+' | tail -n 1)

curl https://start.spring.io/starter.tgz -d type=maven-project -d dependencies=web,data-jpa,mysql -d baseDir=azure-spring-workshop -d bootVersion=$BOOT_RELEASE -d javaVersion=$JAVA_HOME_VERSION | tar -xzvf -
```

```bash
cd ./azure-spring-workshop
vi src/main/resources/application.properties
```
Update application.properties with (with the content from the AZ_ .. envs (default port is 3306))
```bash
logging.level.org.hibernate.SQL=DEBUG

spring.datasource.url=jdbc:mysql://$AZ_DATABASE_NAME.mysql.database.azure.com:3306/demo?serverTimezone=UTC
spring.datasource.username=spring@$AZ_DATABASE_NAME
spring.datasource.password=$AZ_MYSQL_PASSWORD

spring.jpa.show-sql=true
spring.jpa.hibernate.ddl-auto=create-drop
```

## Run App

```bash
mvn spring-boot:run
```

```bash
2025-02-18T08:21:47.488+01:00  INFO 25480 --- [demo] [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
2025-02-18T08:47:32.204+01:00  INFO 25480 --- [demo] [ionShutdownHook] o.s.b.w.e.tomcat.GracefulShutdown        : Commencing graceful shutdown. Waiting for active requests to complete
2025-02-18T08:47:32.221+01:00  INFO 25480 --- [demo] [tomcat-shutdown] o.s.b.w.e.tomcat.GracefulShutdown        : Graceful shutdown complete
2025-02-18T08:47:32.230+01:00  INFO 25480 --- [demo] [ionShutdownHook] j.LocalContainerEntityManagerFactoryBean : Closing JPA EntityManagerFactory for persistence unit 'default'
2025-02-18T08:47:32.234+01:00  INFO 25480 --- [demo] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown initiated...
2025-02-18T08:47:32.243+01:00  INFO 25480 --- [demo] [ionShutdownHook] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Shutdown completed.
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  30:20 min
[INFO] Finished at: 2025-02-18T08:47:32+01:00
[INFO] ------------------------------------------------------------------------
```
## Trobleshoot missing dependencies (eg java update)

Add Dependencies to pom.xml
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <version>2.5.4</version>
</dependency>
```

test Maven
```bash
mvn test
```

## Test Program (mvn spring-boot:run)
```bash
curl --header "Content-Type: application/json" \
--request POST \
--data '{"description":"configuration","details":"congratulations, you have set up your Spring Boot application correctly!","done":true}' \
http://127.0.0.1:8080/
```
Success result
```bash
{"id":1,"description":"configuration","details":"congratulations, you have set up your Spring Boot application correctly!","done":true}
```
## Clean up resources when done
```bash
az group delete --name $AZ_RESOURCE_GROUP
az group delete --name <your resource group name> --yes
```


