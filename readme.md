
# Improving Appication Performance with AWS ElastiCache #

- DB Result Caching
- Session Store
- Eliminating DB Hot Blocks when heavy transcation occurs in small range of data. 

In this blog, We are going to go through how to eliminate database hotblock with amazon elasticache redis and
java springboot web application.

We don't deal with application implementation details in here, but If you are either developer or someone who can read
java language, you can easily catch up with details.


### Business Problems ###


## ElastiCache Briefs ##

Before diving deeply, I want to just introduce about AWS elastiCache for your understanding about this article.

## Architecture ##

![infra](https://github.com/gnosia93/demo-cache/blob/master/document/infra-architecture.png)

ALB has two endpoint which port number is 80, 8080. 
Port 80 is service endpoint for web user interface, in there you can add and select procuct, order information.
On the other hand, port 8080 is rest api endpoint to perform stress test with apache bench(AB).

Under the ALB, there are two EC2 instnaces which contain react for ui and springboot for api.
At the data layer, we have Amazon Elasticache for redis which have cache objects counting product selling
and auroa database cluster which is composed of one master node, no replica.
and then Aurora database have two DB tables which name is product and order.

* /site-address/order/add
* /site-address/order/event-add

<< architecture >>



## Infrastructure Building ##

### Infra Provisioning with CloudFormation ###

Here, we will use AWS cloudformation to automate painfull and error-prone infrastucture building. 
You can find cloudformation configuration file which name is stack-build.yaml in the subdirectory of this project.

Go to AWS Cloudformation console, and build infrasture of this project with stack-build.yaml.
Normally it takes about 10 minitues for all infra provisioning.
If you are not good at AWS Cloudformation, refer to this URL (https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/GettingStarted.html)

Below screen is the sample outputs of cloudformation which can be founded at stack menu.
You can easily identify web, api endpoint, and provisioned EC2 instances public address, etc.

![cf-outputs](https://github.com/gnosia93/demo-cache/blob/master/document/cf-outputs.png)

AuroraCluster and Redis URL is used at JAVA springboot application configuration.
Both WebEndPoint and ApiEndPoint is load balancer url having public ip address, served at port 80.

### Configure API Server ###

In order to configure API server, log into your api-server with ssh or compatible ssh client and then set up your backend connection for both redis and aurora database.
Please refer following instruction to do your settings.

```
$ ssh -i <your-pem-file> ec2-user@your-api-instance-dnsname

The authenticity of host 'your-api-instance-dnsname (your-api-ip)' can't be established.
ECDSA key fingerprint is SHA256:f1leNwUtSQdTwHqsusHlzEef812DWDtqgJ7oVwlUOzg.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added 'ec2-13-114-101-172.ap-northeast-1.compute.amazonaws.com,13.114.101.172' (ECDSA) to the list of known hosts.

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
$ 
$ cd ~/demo-cache/src/main/resources
$ vi application-prod.properties
```
You have to change <your-aurora-writer-endpoint> and <your-redis-cluster-endpoint> to yours.
You can find all required connection address from cloudformation stack outputs tab like above. 

[application-prod.properties]
```
spring.datasource.jdbc-url=jdbc:mysql://<your-aurora-writer-endpoint>:3306/shop?serverTimezone=UTC
spring.datasource.url=jdbc:mysql://<your-aurora-writer-endpoint>:3306/shop?serverTimezone=UTC
spring.datasource.username=demo
spring.datasource.password=demo12345
spring.datasource.maximum-pool-size=100

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

spring.redis.lettuce.pool.max-active=10
spring.redis.lettuce.pool.max-idle=10
spring.redis.lettuce.pool.min-idle=2
spring.redis.port=6379
spring.redis.host=<your-redis-cluster-endpoint>        
```

### Initialize Database ###

In this project, we are using Aurora MySQL datbase which has only primary node,
and making database tables, procedure and buidling sample data of product table.
Please execute a command like below at ec2 instance console at either api server instances and
you need to confirm product table's row count is 10000. 

If you are good at MySQL database and compatibles, you can login Auroa RDS using mysql client and check sample schema,
auto generated datas.

Goto init/sql sub directory under the demo-cache project root,  
```
$ cd ~/demo-cache/init/sql
$ vi create-schema.sh 
````

and change `<your-aurora-address>` into yours.(refer to cloudformation outputs)
As you see, We will database user which name is demo and password demo12345
Don't modify this database user login information. 

[create-schema.sh]
```
AURORA=<your-aurora-address>
mysql -u demo -pdemo12345 -h $AURORA < aurora.sql
mysql -u demo -pdemo12345 -h $AURORA -e "select count(1) as 'gen_product_cnt' from shop.product"
```    

You can check schema build result from execution create-schema.sh like below.
```
$ sh create-schema.sh 
mysql: [Warning] Using a password on the command line interface can be insecure.
mysql: [Warning] Using a password on the command line interface can be insecure.
+-----------------+
| gen_product_cnt |
+-----------------+
|           10000 |
+-----------------+                                                 
```

### Execute Application and Check the right result ###

Now, We have done all the configuration and sample database building.
Finally, execute run.sh to start spring boot java application and check if java web application is working properly.
The result of curl must be like belows, it must have to return json health checking result output.
If you have any problem, please check [application-prod.properties] configuration file and tomcat.log in project root.

```
$ cd ~/demo-cache
$ mvn clean; mvn package
$ sh scripts/run.sh

$ curl localhost
{"localDateTime":"2020-06-10T12:00:40.45","code":200,"message":"ok","data":{"localDateTime":"2020-06-10T12:00:40.45","code":200,"message":"I am working!","data":""}}
```

Additionaly check web browser output which connect to api load balancer url. 


## BenchMark Test ##

![bench-arch](https://github.com/gnosia93/demo-cache/blob/master/document/benchmark-architecture.png)

* AB 에 대한 간략한 설명 및 노트북 인스톨

* json 데이터 설명, .sh 설명

* 테스트 실행.

- AB

```
$ yum install -y httpd-tools

```

<< Performance Graph >>


### Addtional Contents (Planned) ##

- develop material in HA cases, how it works and how many times is required for completion of HA

- deep dive to spring boot implementaion for transaction processing between redis and rdbms.

- pricing comparison (Well Architected view)
