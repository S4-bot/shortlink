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
        <version>5.4.1</version>
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
bug修复

1. 这里我用的版本是5.4.1报错了，这个错误表明HASH_MOD算法不能用于自动分片配置。这是ShardingSphere 5.4.1版本的一个已知限制。需要手动去分片

        # 分片算法
        shardingAlgorithms:
          # 数据表分片算法
          user_table_hash_mod:
            # 根据分片键 Hash 分片
            type: INLINE
              # 自定义分片算法
            props:
              algorithm-expression: t_user_${Math.abs(username.hashCode()) % 16}
          
2.Java版本兼容性问题
Java 9+的变化：从Java 9开始，JAXB（Java Architecture for XML Binding）被从JDK标准库中移除
ShardingSphere需求：ShardingSphere 5.4.1需要JAXB来进行XML配置解析

   <!-- ✅ JAXB 依赖（Java EE 版本，与 ShardingSphere 兼容） -->
        <dependency>
            <groupId>javax.xml.bind</groupId>
            <artifactId>jaxb-api</artifactId>
            <version>2.3.1</version>
        </dependency>

        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-impl</artifactId>
            <version>2.3.9</version>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>com.sun.xml.bind</groupId>
            <artifactId>jaxb-core</artifactId>
            <version>2.3.0.1</version>
            <scope>runtime</scope>
        </dependency>



  -----

  String.format() 是 Java 中用于字符串格式化的强大工具，能够将数据按照指定格式转换为格式化的字符串。
  
主要作用：

1. 数据格式化
2. 动态字符串构建
3. 跨平台换行
4. 参数复用

-----

## 2.9 敏感数据加密存储

  在shardingsphere-config.yaml文件中来编写加密配置

    #数据源集合
    dataSources:
      # 定义数据源 ds_0
      ds_0:
        # 数据源类名，HikariCP 是一种高效的数据库连接池
        dataSourceClassName: com.zaxxer.hikari.HikariDataSource
        # 使用 MySQL 驱动类
        driverClassName: com.mysql.cj.jdbc.Driver
        # 数据库连接 URL，包含了数据库名称、字符编码、时区等配置
        jdbcUrl: jdbc:mysql://127.0.0.1:3306/link?useUnicode=true&characterEncoding=UTF-8&rewriteBatchedStatements=true&allowMultiQueries=true&serverTimezone=Asia/Shanghai
        # 数据库用户名
        username: root
        # 数据库密码
        password: 123456
    
    # 规则配置，包含分片与加密规则
    rules:
      # 分片规则
      - !SHARDING
        # 分片的表配置
        tables:
          t_user:
            # 定义真实数据节点。ds_0 表示使用 ds_0 数据源，t_user_${0..15} 表示数据表分片的命名规则
            actualDataNodes: ds_0.t_user_${0..15}
            # 分表策略
            tableStrategy:
              # 标准分片策略，使用单分片键进行分片
              standard:
                # 分片键，使用 `username` 字段作为分片依据
                shardingColumn: username
                # 分片算法，使用 `user_table_hash_mod` 算法
                shardingAlgorithmName: user_table_hash_mod
    
        # 定义分片算法
        shardingAlgorithms:
          # 分表时使用的分片算法
          user_table_hash_mod:
            # 根据分片键的 Hash 值进行分片
            type: INLINE
            # 自定义分片算法表达式
            props:
              # 算法表达式，采用 `username` 的 `hashCode()` 值对 16 取模，决定分表编号
              algorithm-expression: t_user_${Math.abs(username.hashCode()) % 16}
    
      # 加密规则
      - !ENCRYPT
        tables:
          # 需要加密的表配置
          t_user:
            columns:
              # 加密字段 `phone`
              phone:
                cipher:
                  # 加密后字段名为 `phone`
                  name: phone
                  # 使用的加密器名为 `common_encryptor`
                  encryptorName: common_encryptor
              # 加密字段 `mail`
              mail:
                cipher:
                  # 加密后字段名为 `mail`
                  name: mail
                  # 使用的加密器名为 `common_encryptor`
                  encryptorName: common_encryptor
    
        # 定义加密器
        encryptors:
          # 加密器配置，使用 AES 算法
          common_encryptor:
            type: AES
            # 配置 AES 加密密钥
            props:
              aes-key-value: d6oadClrrb9A3GWo
    
    #配置全局属性
    props:
      # 开启 SQL 执行日志输出，用于调试与监控
      sql-show: true

注意，因为使用的是5.4.1版本，所以加密字段下要使用 mail:

                    mail:
                      cipher：？
                        name:？
                        encryptorName:？

## 2.10 用户个人信息修改功能

  1.在controller中定义update方法
  方式：PUT 
  返回值：无 
  路径：/api/short-link/v1/user
  
      @PutMapping("/api/short-link/v1/user")
    public Result<Void> update(@RequestBody UserUpdateRepDTO requestParam){
        userService.update(requestParam);
        return Results.success();
    }

  2.在service中定义接口

    void update(UserUpdateRepDTO userparam);

  3.在impl中实现

    @Override
      public void update(UserUpdateRepDTO requestParam) {
          // TODO: 验证当前用户是否是登录用户
          LambdaUpdateWrapper<UserDO> updateWrapper = Wrappers.lambdaUpdate(UserDO.class)
                  .eq(UserDO::getUsername, requestParam.getUsername());
          baseMapper.update(BeanUtil.toBean(requestParam, UserDO.class),updateWrapper);
      }


## 2.11用户登录接口

1.在controller中定义login方法
  方式：Post
  返回值：Results.success(userService.login(requestParam))
  路径：/api/short-link/v1/user/login

      @PostMapping("/api/short-link/v1/user/login")
    public Result<UserLoginRespDTO> login(@RequestBody UserLoginRepDTO requestParam){
        return Results.success(userService.login(requestParam));
    }

 2.在service中定义接口
 
    UserLoginRespDTO login(UserLoginRepDTO requestParam);
 

3.在impl中实现

    @Override
    public void update(UserUpdateRepDTO requestParam) {
        // TODO: 验证当前用户是否是登录用户
        LambdaUpdateWrapper<UserDO> updateWrapper = Wrappers.lambdaUpdate(UserDO.class)
                .eq(UserDO::getUsername, requestParam.getUsername());
        baseMapper.update(BeanUtil.toBean(requestParam, UserDO.class),updateWrapper);
    }

    @Override
    public UserLoginRespDTO login(UserLoginRepDTO requestParam) {
        LambdaQueryWrapper<UserDO> queryWrapper = Wrappers.lambdaQuery(UserDO.class)
                .eq(UserDO::getUsername, requestParam.getUsername())
                .eq(UserDO::getPassword, requestParam.getPassword())
                .eq(UserDO::getDelFlag, 0);
        UserDO userDO = baseMapper.selectOne(queryWrapper);
        if(userDO==null){
           throw new ClientException("用户不存在");
        }
    //        Boolean hasLogin = stringRedisTemplate.hasKey(requestParam.getUsername());
    //        if(hasLogin !=null && hasLogin){
    //            throw new ClientException("用户已登录");
    //        }
    //        String uuid = UUID.randomUUID().toString();
    //        stringRedisTemplate.opsForHash().put("login_"+ requestParam.getUsername(),uuid, JSON.toJSONString(userDO));
    //        stringRedisTemplate.expire("login_"+ requestParam.getUsername(), 30L, TimeUnit.MINUTES);
    //        return new UserLoginRespDTO(uuid);
            String jwt = JwtUtils.generateToken(requestParam.getUsername(), userDO.getId());
            stringRedisTemplate.opsForHash().put("login_"+ requestParam.getUsername(),jwt, JSON.toJSONString(userDO));
            stringRedisTemplate.expire("login_"+ requestParam.getUsername(), 30L, TimeUnit.MINUTES);
            return new UserLoginRespDTO(jwt);
        }

第一个是使用的是用UUID来生成唯一标识token。

# 第二个是使用JWT令牌。

步骤：
1.首先要引入依赖

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt</artifactId>
            <version>0.9.1</version>
        </dependency>

2.在utils包里创建工具类JwtUtils。里面要有创建和解析令牌的方法


    /**
     * JWT工具类
     */
    
    public class JwtUtils {

    /**
     * JWT密钥 - 与测试类保持一致
     */
    private static final String SECRET_KEY = "d6oadClrrb9A3GWo";

    /**
     * 令牌过期时间（30分钟）
     */
    private static final long EXPIRATION_TIME = 30 * 60 * 1000L; // 30分钟

    /**
     * 生成JWT令牌
     *
     * @param username 用户名
     * @param userId 用户ID
     * @return JWT令牌
     */
    public static String generateToken(String username, Long userId) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("username", username);
        claims.put("userId", userId);

        return Jwts.builder()
                .setClaims(claims)
                .setSubject(username)
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
                .signWith(SignatureAlgorithm.HS512, SECRET_KEY)
                .compact();
    }

    /**
     * 解析JWT令牌
     *
     * @param token JWT令牌
     * @return Claims对象，包含令牌中的信息
     * @throws io.jsonwebtoken.JwtException 当令牌无效或过期时抛出异常
     */
    public static Claims parseToken(String token) {
        try {
            return Jwts.parser()
                    .setSigningKey(SECRET_KEY)
                    .parseClaimsJws(token)
                    .getBody();
        } catch (Exception e) {
            log.error("解析JWT令牌失败: {}", e.getMessage());
            throw e;
        }
    }

    /**
     * 验证令牌是否有效
     *
     * @param token JWT令牌
     * @return true表示有效，false表示无效
     */
    public static boolean validateToken(String token) {
        try {
            parseToken(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }

    /**
     * 从令牌中获取用户名
     *
     * @param token JWT令牌
     * @return 用户名
     */
    public static String getUsernameFromToken(String token) {
        Claims claims = parseToken(token);
        return claims.getSubject();
    }

    /**
     * 从令牌中获取用户ID
     *
     * @param token JWT令牌
     * @return 用户ID
     */
    public static Long getUserIdFromToken(String token) {
        Claims claims = parseToken(token);
        return claims.get("userId", Long.class);
    }

    /**
     * 检查令牌是否过期
     *
     * @param token JWT令牌
     * @return true表示已过期，false表示未过期
     */
    public static boolean isTokenExpired(String token) {
        try {
            Claims claims = parseToken(token);
            Date expiration = claims.getExpiration();
            return expiration.before(new Date());
        } catch (Exception e) {
            return true;
        }
    }
    }

## 2.12创建验证用户登录接口

1.在controller中定义login方法
  方式：Get
  返回值：Results.success(userService.checkLogin(username,token))
  路径：/api/short-link/v1/user/check-login

     @GetMapping("/api/short-link/v1/user/check-login")
      public Result<Boolean> checkLogin(@RequestParam("username") String username,@RequestParam("token") String token){
          return Results.success(userService.checkLogin(username,token));
      }


 2.在service中定义接口

    Boolean checkLogin(String username,String  token);

 3.在impl中实现

     @Override
        public Boolean checkLogin(String username,String token) {
            return stringRedisTemplate.opsForHash().get("login_"+ username, token) !=null;
        }

 ## 2.13 创建用户退出登录接口
 
1.在controller中定义login方法
  方式：Delete
  返回值：无
  路径：/api/short-link/v1/user/logout

    @DeleteMapping("/api/short-link/v1/user/logout")
      public Result<Void> logout(@RequestParam("username") String username,@RequestParam("token") String token){
          userService.checkLogout(username,token);
          return Results.success();
      }

 2.在service中定义接口

     void checkLogout(String username, String token);

 3.在impl中实现

     @Override
    public void checkLogout(String username, String token) {
       if(checkLogin(username,token)){
           stringRedisTemplate.delete("login_"+ username);
           return;
       }
       throw new ClientException("用户未登录或登录异常");
    }

### 三、短连接分组

功能分析
  短链接分组就像是谷歌浏览器的收藏栏，标识着不同网站不同的语义。如果大家使用短链接系统，创建不同语义的短链接需要在一个分页里查询，这是一件多么令人抓狂的事。
  按照不同的思路的拆分开，这样查询的时候就会方便很多。也方便统计。
  ● 增加短链接分组
  ● 修改短链接分组（只能修改名称）
  ● 查询短链接分组集合（短链接分组最多10个）
  ● 删除短链接分组
  ● 短链接分组排序

## 3.1创建短链接分组数据库表
  建表语句：
  
    CREATE TABLE `t_group` (
      `id` bigint(20) NOT NULL AUTO_INCREMENT COMMENT 'ID',
      `gid` varchar(32) DEFAULT NULL COMMENT '分组标识',
      `name` varchar(64) DEFAULT NULL COMMENT '分组名称',
      `username` varchar(256) DEFAULT NULL COMMENT '创建分组用户名',
      `sort_order` int(3) DEFAULT NULL COMMENT '分组排序',
      `create_time` datetime DEFAULT NULL COMMENT '创建时间',
      `update_time` datetime DEFAULT NULL COMMENT '修改时间',
      `del_flag` tinyint(1) DEFAULT NULL COMMENT '删除标识 0：未删除 1：已删除',
      PRIMARY KEY (`id`),
      UNIQUE KEY `idx_unique_username_gid` (`gid`,`username`) USING BTREE
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;;


## 3.2 新增用户接口开发

  1.创建控制层，接口层，实现层，持久层

  接口层要继承extends IService<GroupDO>
  实现层要继承extends ServiceImpl<GroupMapper, GroupDO>
  持久层要继承extends BaseMapper<GroupDO>

  2，创建接口并在实现类里实现

    /**
     * 短连接分组实现层
     */
    @Service
    @Slf4j
    public class GroupServiceImpl extends ServiceImpl<GroupMapper, GroupDO> implements GroupService {
    
        @Override
        public void saveGroup(ShortLinkGroupSaveRepDTO requestParam) {
            String gid ;
            do{
                gid = RandomUtils.generateRandom();
            } while (!hasGid(gid));
            GroupDO groupDO = GroupDO.builder()
                    .name(requestParam.getName())
                    .gid(gid)
                    .build();
            baseMapper.insert(groupDO);
        }
    
    
        private boolean hasGid(String gid){
            LambdaQueryWrapper<GroupDO> queryWrapper = Wrappers.lambdaQuery(GroupDO.class)
                    .eq(GroupDO::getDelFlag, 0)
                    .eq(GroupDO::getGid, gid)
                    //TODO: 设置用户名
                    .eq(GroupDO::getUsername, null);
            GroupDO hasGroupFlag = baseMapper.selectOne(queryWrapper);
            return hasGroupFlag == null;
        }
    }

-----

补充：这里需要用随机函数来生成gid。

# 怎么创建一个随机函数？

1.在工具包里定一个个随机数的工具类RandomUtils

2.实现业务逻辑

    // 定义字母和数字的字符集
    private static final String CHAR_SET = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";
    private static final SecureRandom RANDOM = new SecureRandom();
    //方法重载
    public static String generateRandom() {
       return generateRandom(6);
    }
    /**
     * 生成一个包含字母和数字的指定长度的随机字符串
     *
     * @param length 随机字符串的长度
     * @return 生成的随机字符串
     */
    public static String generateRandom(int length) {
        StringBuilder sb = new StringBuilder(length);

        // 从字符集随机选择字符
        for (int i = 0; i < length; i++) {
            int randomIndex = RANDOM.nextInt(CHAR_SET.length());
            sb.append(CHAR_SET.charAt(randomIndex));
        }
        return sb.toString();
    }
    }
-----

3.这里对数据库持久层基础属性进行了优化。把常用的属性单独定义一个类，并继承它。（shortlink.admin.common.database）

    /**
     * 数据库持久层对象基础属性
     */
    @Data
    public class BaseDO {
    
    
        /**
         * 创建时间
         */
        @TableField(fill = FieldFill.INSERT)
        private Date createTime;
    
        /**
         * 修改时间
         */
        @TableField(fill = FieldFill.INSERT_UPDATE)
        private Date updateTime;
    
        /**
         * 删除标识 0：未删除 1：已删除
         */
        @TableField(fill = FieldFill.INSERT)
        private Integer delFlag;
    }

# bug:

  这里是出现了找不到t_group的bug。通过ai发现是shardingspere的配置问题，可能是没有配置该表的属性，默认进行了分片。所以需要配置t_group不分片

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
                # 配置 t_group 表为普通表（不分片）
          t_group:
            actualDataNodes: ds_0.t_group




      

