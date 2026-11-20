# mybatis执行

依赖注入：Mapper声明 → Spring启动注入Mapper实例（MapperProxy）

Mybatis调用：Mapper调用 → JdkDynamicProxy.invoke() target就是实例化的MapperProxy → ReflectiveMethodInvocation.proceed → AopUtils.`*invokeJoinpointUsingReflection` →* MapperProxy代理方法调用 → `mapperMethod.execute(sqlSession, args)` 

SqlSession执行：SqlSession执行Mapper方法（DefaultSqlSession）→ 通过Configuration拿到mapper方法的MappedStatement → Executor.doQuery → 组装真正的Statement执行 → ResultHandler解析结果

### 整体

接口层：提供对外API，封装底层操作；例如SqlSession封装了CRUD操作

- SqlSession：默认实现DefaultSqlSession。核心数据库操作：insert，selectOne
- SqlSessionFactory：基于Configuration构建，用于创建SqlSession
- **SqlSessionFactoryBuilder：解析Mybatis配置，构建**SqlSessionFactory
- Mapper：定义的Mapper通过扫描或注解，实现依赖注入生成MapperProxy代理类

核心处理：SQL执行，结果映射，缓存管理

- Executor：SimpleExecutor创建Statement并执行
- MapperProxy：Mapper接口的代理类实现
- MapperRegistery：管理Mapper的注册
- MappedStatement：封装SQL语句的信息，参数结果类型
- StatementHandler：根据ms创建JDBC的Statement
- ResultSetHandler：将JDBC查询结果映射到Java对象
- Interceptor：拦截Executor和StatementHandler，实现语句拦截功能扩展（日志，加密，分页）在SqlSessionFactory的Plugin属性配置（获取MappedStatement，修改参数）

基础层：全局基础设施

- COnfiguration：全局配置，存储DataSource，事务管理器，映射器，类型处理器，插件
- DataSource：JDBC数据源
- Transaction：事务接口
- TypeHandler：类型映射处理