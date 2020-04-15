# 基于 - LCN (5.0.2.RELEASE) 版本开发

#### 第一步： 执行 sql/init.sql 里面的脚本

### 第二步 新建 springboot项目，并依赖以下包，并在主类添加@EnableTransactionManagerServer
```
<dependency>
    <groupId>com.codingapi.txlcn</groupId>
    <artifactId>txlcn-tm</artifactId>
    <version>5.0.2.RELEASE</version>
</dependency>
```
# 若需要修改源码，第二步不走， 走 第三 步到 第五步，其他不变，否则直接走第二步，然后直接走第六步

#### 第三步：修改 txlcn-tm 模块的 配置文件 application.properties
```
spring.application.name=tx-manager
server.port=7970

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/tx-manager?characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=123456

mybatis.configuration.map-underscore-to-camel-case=true
mybatis.configuration.use-generated-keys=true


# 开启日志 ,默认为false
tx-lcn.logger.enabled=true
logging.level.com.codingapi=debug
tx-lcn.logger.driver-class-name=${spring.datasource.driver-class-name}
tx-lcn.logger.jdbc-url=${spring.datasource.url}
tx-lcn.logger.username=${spring.datasource.username}
tx-lcn.logger.password=${spring.datasource.password}

# redis 的设置信息. 线上请用Redis Cluster
#redis 主机
spring.redis.host=lancaiwu.cn
#redis 端口
spring.redis.port=9736
#redis 密码
spring.redis.password=123456qq.!@#QWEE

#tx-lcn.logger.enabled=true
# TxManager Host Ip
#tx-lcn.manager.host=127.0.0.1
# TxClient连接请求端口
#tx-lcn.manager.port=8070
# 心跳检测时间(ms)
#tx-lcn.manager.heart-time=15000
# 分布式事务执行总时间
#tx-lcn.manager.dtx-time=30000
#参数延迟删除时间单位ms
#tx-lcn.message.netty.attr-delay-time=10000
#tx-lcn.manager.concurrent-level=128
#redis 主机
#spring.redis.host=127.0.0.1
#redis 端口
#spring.redis.port=6379
#redis 密码
#spring.redis.password=

# TM后台登陆密码，默认值为codingapi
#tx-lcn.manager.admin-key=codingapi

# 分布式事务锁超时时间 默认为-1，当-1时会用tx-lcn.manager.dtx-time的时间
#tx-lcn.manager.dtx-lock-time=${tx-lcn.manager.dtx-time}

# 雪花算法的sequence位长度，默认为12位.
#tx-lcn.manager.seq-len=12

# 异常回调开关。开启时请制定ex-url
#tx-lcn.manager.ex-url-enabled=false

# 事务异常通知（任何http协议地址。未指定协议时，为TM提供内置功能接口）。默认是邮件通知
#tx-lcn.manager.ex-url=/provider/email-to/***@**.com
```
#### 第四步，进入到 txlcn模块，进行 maven 打包 ,获得 tx-manager的 jar 包
``` mvn clean  package  -Dmaven.test.skip=true ```
#### 第五步，新建springboot项目并依赖该 tx-manager的jar，并在主类添加@EnableTransactionManagerServer
```
@SpringBootApplication
@EnableTransactionManagerServer
public class TransactionManagerApplication {

  public static void main(String[] args) {
      SpringApplication.run(TransactionManagerApplication.class, args);
  }

}
```

#### 第五步,在业务微服务 增加以下 maven 依赖,当然也可以自己修改这个项目的tc源码，再进行打包后依赖打包后的jar
```
<dependency>
   <groupId>com.codingapi.txlcn</groupId>
   <artifactId>txlcn-tc</artifactId>
   <version>5.0.2.RELEASE</version>
</dependency>
<dependency>
   <groupId>com.codingapi.txlcn</groupId>
   <artifactId>txlcn-txmsg-netty</artifactId>
   <version>5.0.2.RELEASE</version>
</dependency>
```
#### 第六步，在业务微服务的application.properties配置文件增加以下配置
```
# 默认之配置为TM的本机默认端口
tx-lcn.client.manager-address=127.0.0.1:8070 
```
#### 第七步,在微服务开启分布式事务，主类上使用@EnableDistributedTransaction,如下代码
```
@SpringBootApplication
@EnableDistributedTransaction
public class DemoAApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoDubboClientApplication.class, args);
    }

}
```
#### 第八步，TC微服务A业务方法配置
```
@Service
public class ServiceA {
    
    @Autowired
    private ValueDao valueDao; //本地db操作
    
    @Autowired
    private ServiceB serviceB;//远程B模块业务
    
    @LcnTransaction //分布式事务注解
    @Transactional //本地事务注解
    public String execute(String value) throws BusinessException {
        // step1. call remote service B
        String result = serviceB.rpc(value);  // (1)
        // step2. local store operate. DTX commit if save success, rollback if not.
        valueDao.save(value);  // (2)
        valueDao.saveBackup(value);  // (3)
        return result + " > " + "ok-A";
    }
}
```
#### 第九步，TC微服务B业务方法配置
```
@Service
public class ServiceB {
    
    @Autowired
    private ValueDao valueDao; //本地db操作
    
    @LcnTransaction //分布式事务注解
    @Transactional  //本地事务注解
    public String rpc(String value) throws BusinessException {
        valueDao.save(value);  // (4)
        valueDao.saveBackup(value);  // (5)
        return "ok-B";
    }
}
```

### tm项目启动，访问http://localhost:7970，密码为 codingapi