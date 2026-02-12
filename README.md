# shortlink
java微服务项目


一、项目的搭建
  首先引入依赖，但是有的依赖下载失败，我刚开始以为是镜像和网络的问题，但是我并没有解决，后来就直接到maven中心手动去下载。
  
  在父工程下创建一个文件夹叫libs，把从中心下载的依赖放进去，然后在pom文件中使用<systemPath>标签去指定文件的路径，这里本来用的是${project.basedir}（ ${project.basedir} 是 Maven 内置变量，表示当前模块的根目录），但是因为这是微服务，用有不同的模块，所以这个不适用，之后我直接用的绝对路径。这里肯定是要优化的，现在先跳过。最后再加上<scope>system</scope>，这样就不会默认去repository文件夹中去找了。
  
  该项目分为三个模块：admin（8002）,gateway（8000）,project（8001）。分别创建好对应的包结构。最后使用apifox去调试。

二、用户管理

2.1根据用户名字查找信息接口设计

  先去用户接口层去声明一个方法，接口需要继承IService<UserDO>，然后去实现层去实现，并且也需要继承ServiceImpl<UserMapper, UserDO>。因为mp中有默认的方法可以直接使用，无需我们去实现。最后去控制层去编写业务。
  
  值得一提的是，在控制层中之前使用的是@Autowired去引入service，这样会涉黄，所以不推荐。而是是使用@RequiredArgsConstructor（@RAC）。
  
  这里在做的时候遇到个错误，在用户持久层中我定义了一个实体类，并去继承了mp，报错了，应该是定义一个接口。
  
  遇到两个问题，一个说是数据源的问题，这个后来发现是配置文件的顺序没对。第二个是请求头格式不匹配，原因是我在UserRespDTO中没有加上@Data注解（确保 UserRespDTO 类中有适当的 getter 和 setter 方法，Jackson 需要这些方法来序列化对象）。

  在controller中禁止写业务代码。

2.2错误码的设计

  1.在admin\common\convention\errorcode包里面先定义一个接口IErrorCode，里面定义了定义了错误码的通用契约，所有具体的错误码实现类都必须提供 code() 和 message() 方法。 提供了一种标准化的方式，使得不同模块或服务可以通过统一的接口获取错误码和错误信息。
  
  2.再定义一个枚举类BaseErrorCode并继承IErrorCode。这里是将所有系统级错误码集中在一个枚举中，方便查找和维护。负责系统级通用错误码。
  
  3.再去admin\common\enums包里定义一个枚举类UserErrorCodeEnum并继承IErrorCode。将用户模块的错误码独立出来，避免与其他模块的错误码混淆，提升模块化程度。负责用户模块特有错误码。

2.3全局异常拦截器

  2.创建common.convention.exception包，里面定义抽象规约异常，客服端异常，服务端异常，远程调用异常。其他三个需要全部继承抽象规约异常。
  
  1.创建一个web包，在里面定义全局异常拦截器GlobalExceptionHandler。需要加上注解@RestControllerAdvice，用于定义全局异常处理器，能够捕获整个应用中未被 try-catch 处理的异常。结合 @ExceptionHandler 注解，可以针对不同类型的异常进行定制化处理。
  
  @ExceptionHandler(value = {T})，里面是可以直接写父类，这样也可以扫描到子类


2.4用户敏感信息脱敏展示

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

  如果想要看到真实的数据，就再创建一个即UserActualRespDTO。这里用到一个方法,使用 Hutool 工具类中的 BeanUtil.toBean 方法，将 UserRespDTO 对象转换为 UserActualRespDTO 类型的对象。return Results.success(BeanUtil.toBean(userService.getUserByUsername(username), UserActualRespDTO.class));
  

