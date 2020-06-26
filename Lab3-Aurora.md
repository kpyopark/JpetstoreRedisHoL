# Aurora MySQL 이용하여, 공통 Database 생성하고, 연결하기

현재는 개별 JPetStore Application에 있는 Local Storage (Hyper SQL)을 이용하여 자료를 저장하고 있습니다. 이럴 경우, 데이터가 서로 다르기 때문에
문제가 발생할 수 밖에 없습니다. 이를 해결하기 위하여, Aurora MySQL을 별도의 외부 Resource로 등록하고, JPetStore Application을 연결해 보도록 하겠습니다. 

** 주의. Default Parameter Group으로 생성된 Aurora MySQL의 경우 한글 처리가 되지 않습니다. Parameter Group 변경과 서버 재시작을 수행하여 Character Set을 utf8mb4로 변경하면 수행 가능합니다만, 수행단계가 복잡해져, 현재 HoL 목적에 맞게 Parameter Group 생성 부분은 추가하지 않았습니다. 
** 주의. 사용자 등록시점에 한글을 사용하지 마십시요. 해당 Application은 Example용이며, 운영 수준의 에러헨들링을 수행하지 않습니다. 

0. 현재 기동되어 있는 Jpetstore Application이 있다면, ctrl+c 로 중지시킵니다. 앞으로 진행되는 내용은 원래 있던 terminal에서 수행해야 합니다. (환경 변수)

1. RDS Aurora MySQL 에서 사용할 user / password 에서 사용할 정보를 Cloud 9 에 아래와 같이 저장합니다. !!! 주의 - <what as you want> 부분을 반드시 수정하십시요. password의 경우 8글자 이상이어야 합니다.

```
DB_USER=<what as you want>
DB_PASS=<what as you want>
```

1. RDS MySQL 생성을 하기 위하여, 아래 Command를 Cloud 9 Terminal에서 입력하시기 바랍니다. 
   해당 내용은, JPetStore Instance에서 RDS에 접근할 수 있게 Security Group을 설정하고, Aurora Cluster 생성 이후, Aurora instance를 생성합니다. 


```
C9_SG=`aws ec2 describe-instances | jq ' .Reservations[0].Instances[0].SecurityGroups[0].GroupId ' | sed '1,$s/"//g'`
DEFAULT_VPC=`aws ec2 describe-vpcs | jq ' .Vpcs[] | select(.IsDefault == true) | .VpcId' | sed '1,$s/"//g'`
DEFAULT_VPC_CIDR=`aws ec2 describe-vpcs | jq ' .Vpcs[] | select(.IsDefault == true) | .CidrBlock' | sed '1,$s/"//g'`
DEFAULT_VPC=`aws ec2 describe-vpcs | jq ' .Vpcs[] | select(.IsDefault == true) | .VpcId' | sed '1,$s/"//g'`
DB_PORT=3306
DB_ENGINE=aurora-mysql
SUBNET_AZ=`aws ec2 describe-subnets --filters Name=vpc-id,Values=${DEFAULT_VPC} | jq ' .Subnets[0:3] | .[].AvailabilityZone' | sed '1,$s/"//g' | xargs`
SUBNETS=`aws ec2 describe-subnets --filters Name=vpc-id,Values=${DEFAULT_VPC} | jq ' .Subnets[0:3] | .[].SubnetId' | sed '1,$s/"//g' | xargs`

aws ec2 authorize-security-group-ingress \
--group-id ${C9_SG} \
--protocol tcp \
--port ${DB_PORT} \
--source-group ${C9_SG}

aws ec2 authorize-security-group-ingress \
--group-id ${C9_SG} \
--protocol tcp \
--port ${DB_PORT} \
--cidr ${DEFAULT_VPC_CIDR}

aws rds create-db-subnet-group \
          --db-subnet-group-name jpetstore-db-subnet \
          --db-subnet-group-description 'jpetstore database subnet. example' \
          --subnet-ids ${SUBNETS}

aws rds create-db-cluster \
          --availability-zones ${SUBNET_AZ} \
          --database-name dev \
          --db-cluster-identifier jpetstoredb \
          --vpc-security-group-ids ${C9_SG} \
          --db-subnet-group-name jpetstore-db-subnet \
          --engine ${DB_ENGINE} \
          --master-username ${DB_USER} \
          --master-user-password ${DB_PASS} \
          --no-enable-iam-database-authentication

aws rds create-db-instance \
--db-instance-identifier jpetinstance \
--db-instance-class db.t3.medium \
--engine ${DB_ENGINE} \
--db-subnet-group-name jpetstore-db-subnet \
--no-auto-minor-version-upgrade \
--db-cluster-identifier jpetstoredb

```

2. 일단 생성될 때까지 대기를 하여야 합니다. 이를 위해서 wait 명령어를 이용해 봅니다. 

```
aws rds wait db-instance-available --filters Name=db-cluster-id,Values=jpetstoredb
```

3. 생성이 완료되었기 때문에, Database Endpoint를 확인하고 해당 값을, JPetStore에 설정합니다. 

```
DB_ENDPOINT=`aws rds describe-db-clusters | jq ' .DBClusters[] | select(.DBClusterIdentifier == "jpetstoredb").Endpoint' | sed '1,$s/"//g'`
cd ~/environment/mybatis-spring-boot-jpetstore
cat src/main/resources/application.properties | sed "s/jdbc:hsqldb:file:~\/db\/jpetstore/jdbc:mysql:\/\/${DB_ENDPOINT}:${DB_PORT}\/dev/g" > src/main/resources/application.properties.new
rm src/main/resources/application.properties
mv src/main/resources/application.properties.new src/main/resources/application.properties
echo "spring.datasource.username=${DB_USER}" >> src/main/resources/application.properties
echo "spring.datasource.password=${DB_PASS}" >> src/main/resources/application.properties
echo "spring.datasource.driver-class-name=org.mariadb.jdbc.Driver" >> src/main/resources/application.properties

```

4. JDBC를 패키징해야 하기 때문에, pom.xml 파일에 maria jdbc library를 추가합니다. project root 디렉토리에 있는 pom.xml 파일을 엽니다. 이후 아래 내용을 추가합니다. 

```
<dependency>
    <groupId>org.mariadb.jdbc</groupId>
    <artifactId>mariadb-java-client</artifactId>
    <version>2.5.4</version>
</dependency>

```

4. 새로 프로그램을 Packaging 합니다. 

```
./mvnw clean 
./mvnw package -Dmaven.test.skip=true

```
5. Application 을 기동합니다.  

```
java -jar target/mybatis-spring-boot-jpetstore-2.0.0-SNAPSHOT.jar
```

6. 다시 JPetStore Application 을 Preview Running Application 기능을 이용합니다. 이후 로그인 및 기능 테스트를 수행하십시요. 
   

7. 이제는 정상적으로 MySQL 데이터가 들어가 있는지 볼 차례입니다. 아래 명령어를 이용하여 MySQL 서버에 접근합니다. 

```
DB_ENDPOINT=`aws rds describe-db-clusters | jq ' .DBClusters[] | select(.DBClusterIdentifier == "jpetstoredb").Endpoint' | sed '1,$s/"//g'`
cd ~/environment/mybatis-spring-boot-jpetstore

```
만약 DB_USER 가 환경 변수에서 사라졌을 수 있기 때문에 다시 DB_USER를 입력하고, 이후 패스워드는 DB Password를 입력합니다. 

```
DB_USER=<앞서 지정한 DB User>

$ mysql -h ${DB_ENDPOINT} -u ${DB_USER} -p
password : 

```

8. 접속이 정상적으로 되어 있다면, 사용자 테이블을 조회하여 신규 사용자가 등록되어 있는 지 확인해 봅니다. 

```sql
mysql> use dev;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
mysql> show tables;
+-----------------------+
| Tables_in_dev         |
+-----------------------+
| ACCOUNT               |
| BANNERDATA            |
..
..

| flyway_schema_history |
+-----------------------+
14 rows in set (0.01 sec)

mysql> select * from ACCOUNT;
+-------------+-------------------------+-----------+----------+--------+----------------------+---------------+-----------+-------+-------+-------------+--------------+
| USERID      | EMAIL                   | FIRSTNAME | LASTNAME | STATUS | ADDR1                | ADDR2         | CITY      | STATE | ZIP   | COUNTRY     | PHONE        |
+-------------+-------------------------+-----------+----------+--------+----------------------+---------------+-----------+-------+-------+-------------+--------------+
| ACID        | acid@yourdomain.com     | ABC       | XYX      | OK     | 901 San Antonio Road | MS UCUP02-206 | Palo Alto | CA    | 94303 | USA         | 555-555-5555 |
| honggildong | hongildong@gmail.com    | 1         | 1        | NULL   | 1                    | 1             | 1         | 1     | 07204 | South Korea | 01081667716  |
| j2ee        | yourname@yourdomain.com | ABC       | XYX      | OK     | 901 San Antonio Road | MS UCUP02-206 | Palo Alto | CA    | 94303 | USA         | 555-555-5555 |
+-------------+-------------------------+-----------+----------+--------+----------------------+---------------+-----------+-------+-------+-------------+--------------+
3 rows in set (0.00 sec)

mysql> exit
Bye
TeamRole:~/environment/mybatis-spring-boot-jpetstore (master) $ 
```

9. 수고하셨습니다. 해당 기능을 통하여, Java application 과 Aurora MySQL 연결에 성공하셨습니다.
