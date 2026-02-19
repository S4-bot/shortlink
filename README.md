##### shortlink
#### java微服务项目


### 一、项目的搭建
  首先引入依赖，但是有的依赖下载失败，我刚开始以为是镜像和网络的问题，但是我并没有解决，后来就直接到maven中心手动去下载。
  
  在父工程下创建一个文件夹叫libs，把从中心下载的依赖放进去，然后在pom文件中使用<systemPath>标签去指定文件的路径，这里本来用的是${project.basedir}（ ${project.basedir} 是 Maven 内置变量，表示当前模块的根目录），但是因为这是微服务，用有不同的模块，所以这个不适用，之后我直接用的绝对路径。这里肯定是要优化的，现在先跳过。最后再加上<scope>system</scope>，这样就不会默认去repository文件夹中去找了。
  
  该项目分为三个模块：admin（8002）,gateway（8000）,project（8001）。分别创建好对应的包结构。最后使用apifox去调试。

### 二、用户管理

## 2.1根据用户名字查找信息接口设计

  先去用户接口层去声明一个方法，接口需要继承IService<UserDO>，然后去实现层去实现，并且也需要继承ServiceImpl<UserMapper, UserDO>。因为mp中有默认的方法可以直接使用，无需我们去实现。最后去控制层去编写业务。
  
  值得一提的是，在控制层中之前使用的是@Autowired去引入service，这样会涉黄，所以不推荐。而是是使用@RequiredArgsConstructor（@RAC）。
  
  这里在做的时候遇到个错误，在用户持久层中我定义了一个实体类，并去继承了mp，报错了，应该是定义一个接口。
  
  遇到两个问题，一个说是数据源的问题，这个后来发现是配置文件的顺序没对。第二个是请求头格式不匹配，原因是我在UserRespDTO中没有加上@Data注解（确保 UserRespDTO 类中有适当的 getter 和 setter 方法，Jackson 需要这些方法来序列化对象）。

  在controller中禁止写业务代码。

## 2.2错误码的设计

  1.在admin\common\convention\errorcode包里面先定义一个接口IErrorCode，里面定义了定义了错误码的通用契约，所有具体的错误码实现类都必须提供 code() 和 message() 方法。 提供了一种标准化的方式，使得不同模块或服务可以通过统一的接口获取错误码和错误信息。
  
   2.再定义一个枚举类BaseErrorCode并继承IErrorCode。这里是将所有系统级错误码集中在一个枚举中，方便查找和维护。负责系统级通用错误码。
  
  3.再去admin\common\enums包里定义一个枚举类UserErrorCodeEnum并继承IErrorCode。将用户模块的错误码独立出来，避免与其他模块的错误码混淆，提升模块化程度。负责用户模块特有错误码。

## 2.3全局异常拦截器

  2.创建common.convention.exception包，里面定义抽象规约异常，客服端异常，服务端异常，远程调用异常。其他三个需要全部继承抽象规约异常。
  
  1.创建一个web包，在里面定义全局异常拦截器GlobalExceptionHandler。需要加上注解@RestControllerAdvice，用于定义全局异常处理器，能够捕获整个应用中未被 try-catch 处理的异常。结合 @ExceptionHandler 注解，可以针对不同类型的异常进行定制化处理。
  
  @ExceptionHandler(value = {T})，里面是可以直接写父类，这样也可以扫描到子类


## 2.4用户敏感信息脱敏展示

  利用JOSN的机制来完成序列化。
  
  1.在common包里创建一个有关序列化的包serialize。在里面创建PhoneDesensitizationSerializer类，在类中实现序列化的逻辑。

  2.在对应的字段加上@JsonSerialize(using = PhoneDesensitizationSerializer.class)注解。要在UserRespDTO中加上注解，而不是UserDO。

DTO和DO的区别
DO

是与数据库表直接映射的实体类，用于持久化操作（如增删改查）。

它的职责是准确反映数据库中的数据结构，不应对数据做额外处理。

如果在 UserDO 中添加 @JsonSerialize，会导致持久化层和业务逻辑耦合，违反单一职责原则。

DTO

是用于对外暴露的数据传输对象，通常用于 API 接口返回给前端或其他服务。

它可以根据业务需求对数据进行加工、脱敏或格式化。

在 UserRespDTO 中添加 @JsonSerialize 是为了在序列化时对敏感信息（如手机号）进行脱敏处理，保护用户隐私。

  如果想要看到真实的数据，就再创建一个即UserActualRespDTO。这里用到一个方法,使用 Hutool 工具类中的 BeanUtil.toBean 方法，将 UserRespDTO 对象转换为 UserActualRespDTO 类型的对象。
  return Results.success(BeanUtil.toBean(userService.getUserByUsername(username), UserActualRespDTO.class));

使用@Data生成的set和get方法的返回值是void，不能使用链式编程。需要加上@Accessors(chain = true)
  
解释注解@Accessors(chain = true)
当 chain = true 时，Lombok 会为类中的字段生成返回当前对象（this）的 setter 方法，从而支持链式调用

在 Results.java 文件中，Result 类的 setter 方法（如 setCode、setData 等）目前是标准的 Java setter 形式，返回值为 void。如果希望支持链式调用，可以在 Result 类上添加 @Accessors(chain = true) 注解。



## 2.5用户名全局唯一

 这里涉及到了缓存穿透。缓存穿透是指客户端请求的数据在数据库中根本不存在，从而导致请求穿透缓存，直接打到数据库的问题。

 常见的解决方案有两种（这里采用布隆过滤器）：
 1.缓存空对象

 2.布隆过滤器。布隆过滤器是一种数据统计的算法，用于检索一个元素是否存在一个集合中

 ---创建布隆过滤器
   1.引入redis依赖，如下。

   2，创建admin.config包，在里面定义布隆过滤器RBloomFilterConfiguration，并做一些初始化
   
       @Configuration
    public class RBloomFilterConfiguration {
    
        /**
         * 防止用户注册查询数据库的布隆过滤器
         */
        @Bean
        public RBloomFilter<String> userRegisterCachePenetrationBloomFilter(RedissonClient redissonClient) {
            RBloomFilter<String> cachePenetrationBloomFilter = redissonClient.getBloomFilter("xxx");
            cachePenetrationBloomFilter.tryInit(0, 0);
            return cachePenetrationBloomFilter;
        }
    }

tryInit 有两个核心参数：
● expectedInsertions：预估布隆过滤器存储的元素长度。
● falseProbability：运行的误判率。
错误率越低，位数组越长，布隆过滤器的内存占用越大。
错误率越低，散列 Hash 函数越多，计算耗时较长。

使用布隆过滤器的两种场景：
● 初始使用：注册用户时就向容器中新增数据，就不需要任务向容器存储数据了。
● 使用过程中引入：读取数据源将目标数据刷到布隆过滤器。

------

 补充：这里在定义布隆过滤器时，需要引入redis依赖。课程上的依赖是
   
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
                                            
    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson-spring-boot-starter</artifactId>
    </dependency>                            

但是我出现了，找不到包的报错。需要再引入redis的核心依赖

    <dependency>
        <groupId>org.redisson</groupId>
        <artifactId>redisson</artifactId>
        <version>3.21.3</version>
    </dependency>

redisson-spring-boot-starter 已经是一个用于 Spring Boot 项目的启动器，它封装了对 Redisson 库的支持。但是，尽管如此，它可能并不会直接引入 Redisson 核心库（即 redisson）。

为什么需要显式添加 Redisson 核心依赖？

启动器和核心库的关系：

1. redisson-spring-boot-starter 只是 Redisson 的一个 Spring Boot 启动器，它本身依赖于 Redisson 核心库，但它不一定会显式地在项目中引入 Redisson 核心库。如果某些 Redisson 的 API（如 RBucket、RBloomFilter 等）在项目中被调用，且这些 API 所需的类没有加载，那么会出现 Cannot resolve symbol 'api' 的错误。

2. 依赖关系的传递性：
启动器可能会通过某些依赖间接引入核心库，但这并不总是能确保所有所需的类被正确加载。因此，显式添加 Redisson 核心库可以避免缺少类的问题。

3. 版本不兼容：
如果 Spring Boot 启动器与 Redisson 核心库的版本不兼容，手动指定版本可以确保使用正确的版本。

要想实现真正的全局唯一，还需要为username加上唯一索引。

## 2.6 海量用户注册

  baseMapper 是 MyBatis-Plus 提供的通用 Mapper 接口。

-----

  1. 用户注册的代码实现，先在service定义方法，再去实现。最后编写controller。

          int insert = baseMapper.insert(BeanUtil.toBean(userparam, UserDO.class));
          
解释：insert 方法用于向数据库中插入一条记录，参数是一个实体对象（这里是 UserDO）。
返回值 int 表示受影响的行数，通常为 1（表示成功插入一行数据）

遇到了bug,maven的配置文件出现了警告,原因是<repositories> 标签应该只出现在项目的 pom.xml 文件中，而不应该出现在 settings.xml 文件中。在pom中配置<repositories> 标签（<repositories> 标签 用于声明项目所需的远程仓库，Maven 会从这些仓库中下载依赖项）。
<repositories>
        <repository>
            <id>aliyun-maven</id>
            <url>https://maven.aliyun.com/repository/public</url>
        </repository>
        <repository>
            <id>apache-repo</id>
            <url>https://repo.maven.apache.org/maven2</url>
        </repository>
  </repositories>

改正之后还是依赖的问题，原因是使用 system 作用域会导致 Maven 无法自动解析传递依赖（transitive dependencies），因此 redisson-spring-data 及其相关依赖可能未被正确引入。之后就是把相关属性删除就对了。

2.实现自动填充（参考baomidou.com）

-----
解释注解

@RequestParam 适用于查询参数（URL 中的 ?key=value 形式）。

@RequestBody 适用于请求体中的 JSON 数据（常见于 POST/PUT 请求）

-----

## 2.7 如何防止恶意请求毫秒级触发大量请求去一个未注册的用户名

  <img width="1962" height="856" alt="image" src="https://github.com/user-attachments/assets/b3519ab1-1735-4044-ac35-06c9ba5e4ad5" />

  1.在common.constant里创建RedisCacheConstant（短连接后管 Redis 缓存常量类），在里面定义常量
  
   public static final String LOCK_USER_REGISTER_KEY = "short-link:lock_user-register:";

  2.在impl层里实现逻辑。

     try{
              if(lock.tryLock()){
                  int insert = baseMapper.insert(BeanUtil.toBean(requestParam, UserDO.class));
                  if(insert<1){
                      throw new ClientException(USER_SAVE_ERROR);
                  }
                  userRegisterCachePenetrationBloomFilter.add(requestParam.getUsername());
                  return;
              }
              throw new ClientException(USER_NAME_EXIST);
          }finally{
              lock.unlock();
          }

          
 -----
    // 阻塞式获取锁
    lock.lock();           // 一直等待直到获取锁
  
    // 非阻塞式获取锁
    boolean result = lock.tryLock();  // 立即返回true/false
  
    lock.unlock();  //  必须释放锁

 
  使用静态导入，可以不加前缀。普通导入不行。
  静态导入：
  import static com.nageoffer.shortlink.admin.common.enums.UserErrorCodeEnum.*;

  普通导入：
  import com.nageoffer.shortlink.admin.common.enums.UserErrorCodeEnum;

  -----

  ## 2.8海量用户如何分库分表

   采用Shardingsphere来实现分表
  
   1，创建多个用户表（0-15）

        CREATE TABLE `t_user_0` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
      `username` varchar(256) DEFAULT NULL COMMENT '用户名',
      `password` varchar(512) DEFAULT NULL COMMENT '密码',
      `real_name` varchar(256) DEFAULT NULL COMMENT '真实姓名',
      `phone` varchar(128) DEFAULT NULL COMMENT '手机号',
      `mail` varchar(512) DEFAULT NULL COMMENT '邮箱',
      `deletion_time` bigint(20) DEFAULT NULL COMMENT '注销时间戳',
      `create_time` datetime DEFAULT NULL COMMENT '创建时间',
      `update_time` datetime DEFAULT NULL COMMENT '修改时间',
      `del_flag` tinyint(1) DEFAULT NULL COMMENT '删除标识 0：未删除 1：已删除',
      PRIMARY KEY (`id`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

2.在相应的模块里引入依赖

    <dependency>
        <groupId>org.apache.shardingsphere</groupId>
        <artifactId>shardingsphere-jdbc-core</artifactId>
        <version>5.3.2</version>
    </dependency>

3.修改配置文件，并新建shardingsphere-config.yaml文件

    spring:
      datasource:
      	# ShardingSphere 对 Driver 自定义，实现分库分表等隐藏逻辑
        driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
        # ShardingSphere 配置文件路径
        url: jdbc:shardingsphere:classpath:shardingsphere-config.yaml

shardingsphere-config.yaml

    # 数据源集合
    dataSources:
      ds_0:
        dataSourceClassName: com.zaxxer.hikari.HikariDataSource
        driverClassName: com.mysql.cj.jdbc.Driver
        jdbcUrl: jdbc:mysql://127.0.0.1:3306/link?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai
        username: root
        password: root
    
    rules:
      - !SHARDING
        tables:
          t_user:
            # 真实数据节点，比如数据库源以及数据库在数据库中真实存在的
            actualDataNodes: ds_0.t_user_${0..15}
            # 分表策略
            tableStrategy:
              # 用于单分片键的标准分片场景
              standard:
                # 分片键
                shardingColumn: username
                # 分片算法，对应 rules[0].shardingAlgorithms
                shardingAlgorithmName: user_table_hash_mod
        # 分片算法
        shardingAlgorithms:
          # 数据表分片算法
          user_table_hash_mod:
            # 根据分片键 Hash 分片
            type: HASH_MOD
            # 分片数量
            props:
              sharding-count: 16
    # 展现逻辑 SQL & 真实 SQL
    props:
      sql-show: true

  -----

  String.format() 是 Java 中用于字符串格式化的强大工具，能够将数据按照指定格式转换为格式化的字符串。
  
主要作用：

1. 数据格式化
2. 动态字符串构建
3. 跨平台换行
4. 参数复用

-----
  





  

