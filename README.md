
# Redis session cache & Redis Result Cache & Aurora MySQL HoL

Redis를 이용한 Session Cache와, Result Cache를 적용해 보는 HoL 가이드입니다. 

** 원래 목적은, DynamoDB Session / Redis Session 의 차이가 주 목적이었으나, 현재 DyanamoDB Session Management Project가 Deprecated된 상태입니다. 최신 버젼의 Tomcat/Spring에서는 사용상 어려운 점이 있으므로, DynamoDB 를 이용한 Http Session Clustering은 해당 HoL에서 제외되었습니다.

![image](https://user-images.githubusercontent.com/9047122/85213506-41e3b900-b39a-11ea-9936-7a8defb3934f.png)

# 테스트 환경 설정

 모든 HoL은, 기본적으로 Cloud9 IDE와 Linux 환경을 기본으로 수행합니다. Cloud 9 은 Browser에서 직접 Coding을 수행할 수 있는 Browser기반 IDE(Integrated Development Environment)이며, 추가적으로 Linux EC2 Instance기반하에 작업을 수행하기 때문에, 별도의 Linux Machine이 필요하지 않습니다. 
 
1. 먼저 Cloud 9 서비스를 구성하기 위하여, Console에서 위와 같이 Cloud9 서비스를 입력하고 엔터를 치면, 해당 서비스 화면으로 이동합니다. 
![image](https://user-images.githubusercontent.com/9047122/85213530-a7d04080-b39a-11ea-929e-e97b33dc2626.png)


2. 화면 상단에 보이는 "Create Environment" 버튼을 클릭하십시요. 
![image](https://user-images.githubusercontent.com/9047122/85213580-48266500-b39b-11ea-983c-144b2a67d1bd.png)

3. 아래와 같이, 이름/설명에 대한 부분을 입력하시고 (임의로 입력하셔도 상관 없습니다. ) "Next Step"을 클릭하세요.

![image](https://user-images.githubusercontent.com/9047122/85213598-8c196a00-b39b-11ea-8c40-f3547adf6d55.png)

4. "환경 설정" 부분에서 특별하게 수정할 사항이 없습니다. 제일 하단에 있는 "Next Step"을 클릭하세요. 

5. 화면 하단에 있는 "Create Environment"를 클릭하시면, 아래와 같은 개발 환경 생성 화면이 보이고, 5분 후에, Cloud 9 IDE가 Browser에 표시됩니다. 

![image](https://user-images.githubusercontent.com/9047122/85213643-fb8f5980-b39b-11ea-99d8-d7730f84b34d.png)

6. 테스트에 필요한 일부 Toolkit을 설치합니다. Cloud 9 Terminal에서 아래 Command를 수행합니다. 

```
$ sudo yum install jq -y
```


# Example Application 을 Cloud 9 Instance에 설치하기, 

해당 HoL에서는 Spring JPetStore Example을 이용하여, DynamoDB / Redis session을 테스트해 보도록 하겠습니다.

1. 먼저 Cloud 9에 있는 Terminal을 이용하여 아래 내용을 입력하여, Java, JPetStore를 설치합니다. (COPY & PASTE). 

```
# Install java 11 - 5 minues
sudo rpm -ivh https://corretto.aws/downloads/latest/amazon-corretto-11-x64-linux-jdk.rpm
# Install sample application (Spring JPetStore) - 20 minutes
git clone https://github.com/kpyopark/mybatis-spring-boot-jpetstore
cd mybatis-spring-boot-jpetstore
./mvnw clean 
./mvnw package -Dmaven.test.skip=true
```

2. JpetStore를 Cloud 9이 설치되어 있는 Instance에서 실행합니다. 

```
java -jar target/mybatis-spring-boot-jpetstore-2.0.0-SNAPSHOT.jar
```

3. 상단에 있는 Preview - "Preview running application" 내용을 (아래 그림 참조) 클릭하면, JPetStore 화면이 표시됩니다. 

![image](https://user-images.githubusercontent.com/9047122/85239960-30b9ab80-b471-11ea-9861-c65733b79b24.png)

** 화면을 키우기 위하여, Preview 화면 오른쪽 상단의 버튼을 클릭하면, 전체 화면으로 전환됩니다. 


4. 먼저 Sample 프로그램의 기능을 확인해 보기 위하여, Enter the Store를 클릭합니다. 

![image](https://user-images.githubusercontent.com/9047122/85240016-82facc80-b471-11ea-94d6-e6fedde009bf.png)

5. 상단에 있는 "Sign in"을 클릭하고, Login 화면에 표시되는, "Register Now" 링크를 클릭하세요. 

![image](https://user-images.githubusercontent.com/9047122/85240059-af164d80-b471-11ea-9693-506e3b53f6c5.png)

6. 모든 항목을 입력하고, 하단에 있는 "Enable MyList", "Enable MyBanner"를 체크합니다. 
  (Phone 항목은 000-0000-0000 형태로 입력해 주셔야 합니다. )

![image](https://user-images.githubusercontent.com/9047122/85240087-d10fd000-b471-11ea-92c2-619a0dc735d3.png)

![image](https://user-images.githubusercontent.com/9047122/85240134-f7357000-b471-11ea-8024-86f087c4571a.png)

7. 다시 로그인 화면으로 이동하여 위에서 생성한 사용자 정보를 이용하여 로그인을 수행합니다.

8. 이후 몇가지 애완동물을 선택하여 카드에 담아 보시기 바랍니다. 

![image](https://user-images.githubusercontent.com/9047122/85240276-935f7700-b472-11ea-978f-7a20b23bae6e.png)

9. 카트에서 Proceed to checkout을 선택하여 구매 화면으로 이동합니다. 

10. 구매화면에서 Fake Card Number 를 이용하여, Card Number, Expiry Date 를 입력합니다. 

![image](https://user-images.githubusercontent.com/9047122/85240386-f4874a80-b472-11ea-8885-fd8111fb514e.png)

11. "Continue"를 선택하여, 구매확정 화면으로 이동하고, "Submit"을 클릭하여, 구매합니다. 

![image](https://user-images.githubusercontent.com/9047122/85240438-27314300-b473-11ea-9d82-3471d2494d9a.png)

12. 수고하셨습니다. 해당 내용을 통하여, 기본적인 JPetStore와 해당 기능에 대하여 알아보았습니다. 구성한 아키텍처는 단일 EC2 Instance에 Spring Boot와 H2 Standalone Database를 이용하여 구성한 경우며, 아래와 같은 구성으로 생각하면 됩니다. 

![image](https://user-images.githubusercontent.com/9047122/85339790-d1ff3b00-b51f-11ea-9161-28a49d5a6d4b.png)


다음 장으로 이동하시기 바랍니다. [Lab1-EB](Lab1-EB.md)
