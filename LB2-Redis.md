## Redis를 이용하는 Session Manager 설정하기

Session Cluster (Http Session)가 필요한 경우에 자주 사용되는 서비스가 ElastiCache Redis입니다. Redis의 짧은 latency와 throughput이 Session manager에 매우 적합하기 때문입니다. 

Spring에서는 Redis session manager가 기본적으로 제공되기 때문에, Redis Cluster를 구성하고, 이후, Configuration 수정을 통하여, 적용해 보도록 하겠습니다. 

1. Redis Cluster 생성을 수행하여야 합니다. Console 상에서 수행을 하여도 되며, 아래와 같이 CLI를 수행하여 구성할 수 있습니다. 
 Cloud 9 Terminal에서 아래 내용을 수행하십시요. 
 
```
DEFAULT_VPC=`aws ec2 describe-vpcs | jq ' .Vpcs[] | select(.IsDefault == true) | .VpcId' | sed '1,$s/"//g'`
SUBNETS=`aws ec2 describe-subnets --filters Name=vpc-id,Values=${DEFAULT_VPC} | jq ' .Subnets[] | .SubnetId' | sed '1,$s/"//g' | xargs`
aws elasticache create-cache-subnet-group \
--cache-subnet-group-name redis-subnetgroup \
--cache-subnet-group-description "Redis Test Cluster Subnets" \
--subnet-ids ${SUBNETS}

C9_SG=`aws ec2 describe-instances --instance-id | jq ' .Reservations[0].Instances[0].SecurityGroups[0].GroupId ' | sed '1,$s/"//g'`

aws elasticache create-cache-cluster \
--cache-cluster-id redis-session-manager-cluster \
--az-mode single-az \
--num-cache-nodes 1 \
--cache-node-type cache.t3.small \
--engine redis \
--engine-version 5.0.6 \
--cache-subnet-group-name redis-subnetgroup \
--security-group-ids ${C9_SG} \
--port 6379
```

2. Redis Cluster를 생성할 때, Cloud9 Instance에서 사용하는 SecurityGroup을 Redis Group에서 사용하는 SecurityGroup으로 지정하였습니다. 
   따라서 Cloud9 Instance에서 Redis에 접근할 수 있도록 Ingress Rule을 추가합니다. 
   (Cloud 9 Terminal에서 수행하십시요. ) 
   추가적으로 VPC내에 있는 Instance에서 Redis에 접근할 수 있도록 Ingress Rule을 추가하는 방법도 수행해 보겠습니다.
   
```
aws ec2 authorize-security-group-ingress \
--group-id ${C9_SG} \
--protocol tcp \
--port 6379 \
--source-group ${C9_SG}

DEFAULT_VPC_CIDR=`aws ec2 describe-vpcs | jq ' .Vpcs[] | select(.IsDefault == true) | .CidrBlock' | sed '1,$s/"//g'`

aws ec2 authorize-security-group-ingress \
--group-id ${C9_SG} \
--protocol tcp \
--port 6379 \
--cidr ${DEFAULT_VPC_CIDR}


```

3. 이것으로 인프라에서 Redis Cluster를 생성하고, Security Group을 설정하여, Cloud9 Instance가 Redis에 접근할 수 있게 구성을 완료하였습니다. 
  이후에는 Spring Framework에서 Redis session manager를 설정해야 합니다. 

4. pom 파일에서 Spring redis session package를 추가합니다. Cloud 9 에서 project root에 있는 pom.xml 파일을 엽니다. 

![image](https://user-images.githubusercontent.com/9047122/85365998-6b027600-b561-11ea-816d-02f1b24c9e4b.png)

5. Dependencies 안에, 아래 내용을 추가합니다. 

```
  <dependency>
			<groupId>org.springframework.session</groupId>
			<artifactId>spring-session-data-redis</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-redis</artifactId>
		</dependency>
  
```

![image](https://user-images.githubusercontent.com/9047122/85366344-20352e00-b562-11ea-834a-1e73ebbe5230.png)

6. Redis Cluster 정보를 추가합니다. 

```
REDIS_ENDPOINT=`aws elasticache describe-cache-clusters --cache-cluster-id redis-session-manager-cluster --show-cache-node-info | jq ' .CacheClusters[0].CacheNodes[0].Endpoint.Address' | sed '1,$s/"//g'`

echo "spring.redis.host=${REDIS_ENDPOINT}" >> src/main/resources/application.properties

```

7. src/main/resources/application.properties 파일에 위에서 설정한 host 정보가 정상적으로 들어가 있는지 확인합니다. 

8. Redis CLI를 설치하고, redis cluster 에 붙어서, 현재 KEY 목록을 가지고 옵니다. (Cloud 9 Terminal에서 수행) 현재 아무것도 표시되지 않는 것이 정상입니다.
   이후, Redis Session이 생성되면 KEYS가 추가되는 것을 볼 수 있습니다. 

```
$ npm install -g redis-cli
$ rdcli -h ${REDIS_ENDPOINT}
redis-session-manager-cluster.xxxxx.0001.apn2.cache.amazonaws.com:6379> KEYS *

```

** 만약 연결이 되지 않으면, Security Group Setting이 제대로 되지 않은 것입니다. Redis Cluster console화면에서 Security Group를 확인해 보십시요. 연결이 되어 있지 않다면, Elastic BeanStalk Instance가 사용하는 Default SG값을 Redis Cluster에 적용하면 됩니다. 

현재는 어떤 세션 정보도 가지고 있지 않기 때문에, 아무 값이 나오지 않는 것을 볼 수 있습니다. exit명령어로 나오십시요. 

```
redis-session-manager-cluster.xxxxx.0001.apn2.cache.amazonaws.com:6379> exit
```

10 ElastiCache Redis는 Redis on EC2와는 다른 방식을 취합니다. 예를 들어 CONFIG와 같이 보안적으로 사용을 지양하는 Command는 ElastiCache Redis에서 사용할 수 없습니다. 문제는 Spring Redis Session Manager 는 초기 Connection 구성시에 CONFIG 명령어를 사용하도록 되어 있습니다. 이를 Disable하기 위해서는 Class 방식의 환경 설정이 필요합니다. 이를 위하여, 아래 내용을 Cloud 9 Terminal에서 수행해 주십시요. 

```
cd ~/environment/mybatis-spring-boot-jpetstore
echo "package com.kazuki43zoo.jpetstore.config;" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "import org.springframework.context.annotation.Configuration;" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "import org.springframework.session.data.redis.config.ConfigureRedisAction;" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "import org.springframework.context.annotation.Bean;" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "@Configuration" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "public class RedisConfig {" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "	@Bean" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "	ConfigureRedisAction configureRedisAction() {" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "		return ConfigureRedisAction.NO_OP;" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "	}" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java
echo "}" >> src/main/java/com/kazuki43zoo/jpetstore/config/RedisConfig.java

```

11. 반영된 패키지를 만들기 위하여, maven으로 재구성하고, Application 을 수행합니다.

```
./mvnw clean 
./mvnw package -Dmaven.test.skip=true
java -jar target/mybatis-spring-boot-jpetstore-2.0.0-SNAPSHOT.jar

```

12. 앞선 장과 마찬가지로 Preview Running Application 기능을 이용하여 Browser에서 Application에 접근합니다. 

13. 로그인을 수행합니다. 

14. Cloud 9 Terminal 에서 다음과 같은 명령어를 이용하여, Redis에 접속합니다. 이후 KEYS * 명령어를 이용하여 Session 정보가 들어가 있는 것을 확인합니다. 

```
$ rdcli -h ${REDIS_ENDPOINT}
redis-session-manager-cluster.xxxxx.0001.apn2.cache.amazonaws.com:6379> KEYS *
redis-session-manager-cluster.xxxxx.0001.apn2.cache.amazonaws.com:6379> KEYS *
1) spring:session:sessions:32e02a17-d30e-494a-8ddd-8646a5f7ec30
2) spring:session:index:org.springframework.session.FindByIndexNameSessionRepository.PRINCIPAL_NAME_INDEX_NAME:honggildong
3) spring:session:sessions:9e6a3a5e-c638-4ee3-a603-0b19c1f05ac5

```

15. 여기까지 수행하였으면, 아래 아키텍처를 완료한 상태가 됩니다. 

![image](https://user-images.githubusercontent.com/9047122/85513558-18df5480-b636-11ea-8878-83e56e4a3787.png)


수고하셨습니다. 다음장으로 이동하여 Aurora MySQL을 설치하고, Spring Application에 연결하도록 해 보겠습니다. [Lab3-Aurora.md](Lab3-Aurora.md)
