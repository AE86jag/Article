# 前言

最近发现项目中跑测试耗时较长，200多个测试需要十几分钟分钟，影响部署和使用，决定定位优化下。看了下代码，因为项目使用了flyway，在跑测试时，为了防止跑测试过程中对数据有修改，影响后面的测试，对有更新操作的测试在测试结束后利用flyway重建数据库，重建过程中既有数据的操作，又有表结构的操作，跑测试过程中对表结构其实是没有影响的，数据修改也不会影响其他表，所以中间做了很多没必要的操作。

因此决定不用flyway，采用事务回滚的方式，每次跑完测试，自动回滚当前测试对数据库的修改内容，保证测试之间不相互影响。具体做法是测试类加上@Transactional注解，或者测试类继承AbstractTransactionalJUnit4SpringContextTests类。

# 事务竟然无法回滚

然而，理想很丰满现实很骨感，当我加上@Transactional注解时，跑完测试并没有回滚。因为我的测试是基于接口的测试，使用了Moscow测试框架，在跑测试的时候会请求Controller的接口，这时候会使用新的线程去执行接口的代码（这块不清楚没关系，后面会写一篇微服务架构下使用Moscow和wiremock进行接口测试），特地在测试代码和Controller代码中打印了当前线程ID，结果如下图

![](https://img-blog.csdnimg.cn/ece195ee16c740f78c097dcc30c9dc22.png#pic_center)

因为事务控制必须在同一个连接内，不同的线程获取的不一定是同一个连接，所以在Controller中对数据的操作是无法回滚的（如果是单元测试，在测试代码中操作数据库是可以回滚的），为了能把所有操作都回滚，那就要保证在这两个线程中获取的是同一个数据库连接。

# 多线程下如何获取同一个数据库连接

因为项目中使用了mybatis框架，可以利用mybatis拦截器，在每次操作数据库获取连接时，将连接替换成同一个连接，这样即使是不同的线程都是同一个连接。下面上相关代码

首先是测试类

```java
/**
 * 测试基类，后续所有测试都继承该测试类，做一些通用的处理逻辑
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = PandoraApplication.class, webEnvironment = SpringBootTest.WebEnvironment.DEFINED_PORT,
//加这个属性是标识是跑测试，后面拦截器有使用到
        properties = {"isRollback=true"})
@Slf4j
public abstract class MyBaseTest extends AbstractTransactionalJUnit4SpringContextTests {
    
    @Autowired
    private DataSource dataSource;
    
    /**
     * 跑每个测试之前，设置要用到的连接
     */
    @Before
    public void setUp() {
        Connection connection = DataSourceUtils.getConnection(this.dataSource);
        CurrentConnectionHolder.setConnection(connection);
    }
    
    /**
     * 跑完测试之后清除当前测试的连接
     */
    @After
    public void destroy() {
        CurrentConnectionHolder.clear();
    }
    
}
```

```java
/**
 * 保存一个全局的数据库链接, 并不是放在ThreadLocal中，只是static修饰的
 */
public class CurrentConnectionHolder {

    private static Connection connection = null;

    public static Connection getConnection() {
        return connection;
    }

    public static void setConnection(Connection connection) {
        CurrentConnectionHolder.connection = connection;
    }

    public static void clear() {
        CurrentConnectionHolder.connection = null;
    }

}
```

```java
/**
 * mybatis拦截器类，只有在本地测试有isRollback属性时才使用，其他开发、测试、线上环境都不用
 * 拦截StatementHandler的prepare方法
 */
@Component
@ConditionalOnProperty(name = "isRollback", havingValue = "true")
@Intercepts({
        @Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})
})
public class ConnectionInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //执行prepare方法之前，如果当前有设置连接，就使用当前的数据库连接，如果没有就无需操作
        Connection connection = CurrentConnectionHolder.getConnection();
        if (Objects.nonNull(connection)) {

            Method method = invocation.getMethod();
            Object target = invocation.getTarget();
            return method.invoke(target, connection, 50000);
        } else {
            return invocation.proceed();
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```

这样就可以多线程下控制事务，进行回滚，在测试代码中对修改的数据进行查询，也是能查询出来的，测试跑完去数据库查看数据是查询不到的，至此多线程下事务回滚就完成了。

原理就是mybatis拦截器替换连接，简单说下拦截器是如何起作用的，如下图

![](https://img-blog.csdnimg.cn/907affc1bc414b41888dee74ce018730.png)

在执行BaseStatementHandler的prepare方式时，会先执行我们定义的拦截器，拦截器中有这段代码：method.invoke(target, connection, 50000); 也就是说调用prepare方法时，传进去的Connection是我们自己的，并不是拦截器前面获取的连接，正常获取的数据库连接是和线程相关的，不符合当前场景的要求。



# 数据库操作日志无法显示

多线程事务回滚的问题解决完，在跑测试的时候发现SQL的操作日志都没有了，像带预处理的SQL、SQL具体参数、更新记录数等都没法显示，如下图

![](https://img-blog.csdnimg.cn/8519d5b93a304cb3978f02cf2dd20729.png)

没有日志可能会影响测试的调试。一种方法是继续利用拦截器打印出要执行的SQL和需要的信息，但是感觉还是有点复杂，总觉得可以更简单一点，继续往下看。

### mybatis正常打印日志流程

因为项目使用的是lombok的slf4j打印日志，而且mybatis打印日志使用的都是org.apache.ibatis.logging.Log类型的实例，直接在Slf4jImpl类的void debug(String s)打上断点，并把拦截器去掉，看正常情况下mybatis是如何打印日志的。

![](https://img-blog.csdnimg.cn/73db52314ef84a808f2395b35b539949.png)

可以看到86行调用了connection的prepareStatement方法，而且注意传进来的Connection的参数是一个代理类，我们点到invoke方法里面

![](https://img-blog.csdnimg.cn/1c6ffb78c6d243a4a475d72496f6c116.png)

在调用connection的prepareStatement方法时，这里其实是调用了ConnectionLogger的invoke，invoke的第53行就是调用Log的实现类去打印日志。这种情况下是日志能正常打印。

### 加了拦截器后日志打印流程

![](https://img-blog.csdnimg.cn/c9b2b11505ed48ee8790e189b989a5a6.png)

加了拦截器以后可以看到传进来的Connection参数是HikariProxyConnection，这样的话86行调用connection.prepareStatement方法就是调用HikariProxyConnection类的prepareStatement方法，这个方法就没有打印mybatis日志相关的逻辑。

到这日志无法打印的原因就找到了，也就是在跑测试设置连接前，连接不能随便拿，而是要用ConnectionLogger代理出来的对象，这样才能进到ConnectionLogger的invoke方法，调用日志实现类打印日志。



# 日志可以打印了

原因找到了，继续顺着堆栈往上找，看传到prepare方法的Connection是怎么来的，在SimpleExecutor类里面看到有个getConnection方法，如下图

![](https://img-blog.csdnimg.cn/f0f0c40321a84a17aca98b81d2b165ba.png)

点进去，代码如下

```java
protected Connection getConnection(Log statementLog) throws SQLException {
    Connection connection = transaction.getConnection();
    if (statementLog.isDebugEnabled()) {
        return ConnectionLogger.newInstance(connection, statementLog, queryStack);
    } else {
        return connection;
    }
}
```

return那句直接复制粘贴，拿来吧你，放到@Before方法里面，这里面还有一个Log参数怎么获取的问题，我直接顺着源码往上找，继续复制粘贴，@Before方法修改后的完整代码如下

```java
@Before
public void setUp() {
    // sqlSessionFactory直接是@Autowired的
    Configuration configuration = SqlSessionUtils.getSqlSession(sqlSessionFactory).getConfiguration();
    /**
     * 源码里面是通过statement拿的，因为日志实现类可以对每个Mapper设置日志实现类，因为我项目没有特殊设置，都是一样的，
     * 这里我随便用了Mapper的statement, 读者可以使用configuration看下有没有其他方式获取Log的实例
     * 获取连接还是和之前一样，这里只是将连接和Log实例封装在一起
     */
    MappedStatement mappedStatement =
        configuration.getMappedStatement("com.pandora.domain.holiday.mapper.TradeDayMapper.isWorkDay");
    Log log = mappedStatement.getStatementLog();
    Connection connection = ConnectionLogger.newInstance(DataSourceUtils.getConnection(this.dataSource), log, 1);
    CurrentConnectionHolder.setConnection(connection);
}
```

用最新代码跑下测试，SQL日志又回来啦！跑完测试数据库是没有数据的

![](https://img-blog.csdnimg.cn/825550e6ab584198958c3f975b2cbb5e.png)

# 总结

- 经过修改，测试时间从十几分钟缩短到了5分钟左右，最短有时候3分钟就可以了，代码提交完不用等十几分钟才能部署完生效了。

- 重点是多线程下使用同一个事务，基于拦截器这个思路可以应用到其他场景，比如异步测试需要保持一致性。

- 日志不打印的问题其实影响不大，只是跑测试没有SQL日志，通过这个问题了解到了mybatis日志打印的原理。