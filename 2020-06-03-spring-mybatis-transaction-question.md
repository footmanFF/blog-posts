---
layout: post
title: spring一例事务执行问题
date: 2020-06-03
tags: spring
---

#### 问题

事务执行的代码：

```java
    @Transactional
    @Override
    public CheckResult check(TradeCheckDO tradeCheckDO) {
    		// 一次查询，查询库A A1表
        // 一次查询，查询库B B1表
    }
```

这里的问题第二次查询，在执行 select ... B1 时，没有在 B 库上执行，而是在 A 库上查 B1 表，就报错了。这个原因应该是数据源（DataSource）用的是动态数据源，导致没有重新获取连接。

<!-- more -->

数据源使用的是动态数据源，继承自 AbstractRoutingDataSource：

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    private static final ThreadLocal<String> contextHolder = new ThreadLocal<>();
    //设置数据源
    static void setDataSourceType(String dataSource) {
        contextHolder.set(dataSource);
    }
    //清除数据源
    static void clearDataSourceType() {
        contextHolder.remove();
    }
    //每执行一次数据库，动态获取DataSource
    @Override
    protected Object determineCurrentLookupKey() {
        return contextHolder.get();
    }
}
```

那么问题来了，spring 对事务处理有自己的抽象，在上面的例子中底层用的是 mybatis，获取的连接其实是 mybatis 的 sqlsession。

这个问题的关键是第二次查询没有获取新的连接，那么这里是不是获取新连接是取决于什么的？是不是因为第二个执行 sql 用的 mapper 内部注入的 DataSource 和当前事务属于同一个数据源，所以才不获取新的连接？

想找到答案只需要去看执行 sql 代码前，spring 是怎么处理连接的就行了。

spring 和 mybatis 会对 Mapper 接口实现一个代理，代理作为 Mapper 的实现类，真实的承接具体 sql 执行。那么决定开不开新连接，是由 mapper 代理实现的，还是由外层的 spring 事务抽象实现的？

先看看 mapper 代理是怎么生成的，有哪些东西。

mapper 的扫描，扫描到了以后注册到 spring 中是 MapperScannerConfigurer 做的，并且最终由 ClassPathMapperScanner 来处理：

```java
private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
  for (BeanDefinitionHolder holder : beanDefinitions) {
  	// ...
    GenericBeanDefinition definition = (GenericBeanDefinition) holder.getBeanDefinition();
    // 此处更换了BeanDefinition的beanClass，是关键
    definition.setBeanClass(this.mapperFactoryBean.getClass());
    definition.getPropertyValues().add("sqlSessionFactory", new RuntimeBeanReference(this.sqlSessionFactoryBeanName));
    // ...
  }
}
```

注册到 spring 的是一个 FactoryBean，实现类是 org.mybatis.spring.mapper.MapperFactoryBean。那么也就是说，真实被使用的 mapper 是由这个 FactoryBean 构造返回的。

那么 FactoryBean 是如何构造的？

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
  @Override
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
}
```

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {
  @Override
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }
}
```

```java
public class Configuration {
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    return mapperRegistry.getMapper(type, sqlSession);
  }
}
```

```java
public class MapperRegistry {
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    // ...
    return mapperProxyFactory.newInstance(sqlSession);
  }
}
```

```java
public class MapperProxyFactory<T> {
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
}
```

最终调了 java 的动态代理，传的 InvocationHandler 是 MapperProxy，生成的代理类实现 mapper 接口。那么接着看 MapperProxy 是怎么代理 mapper 接口的：

```java
public class MapperProxy<T> implements InvocationHandler, Serializable {
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
  	// .. 缓存MapperMethod的逻辑
    
    // 简化了代码
    MapperMethodInvoker invoker = new PlainMethodInvoker(
      new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
  	return invoker.invoke(proxy, method, args, sqlSession);
  }
}
```

MapperMethodInvoker：

```java
  private static class PlainMethodInvoker implements MapperMethodInvoker {
    private final MapperMethod mapperMethod;
    @Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      return mapperMethod.execute(sqlSession, args);
    }
  }
```

MapperMethod，只看一个场景的例子，省略其他代码：

```java
public class MapperMethod {
  public Object execute(SqlSession sqlSession, Object[] args) {
    switch (command.getType()) {
      //..
      case INSERT: {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = rowCountResult(sqlSession.insert(command.getName(), param));
        break;
      }
      //..
    }
  }
}
```

SqlSessionTemplate：

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {
  @Override
  public int insert(String statement, Object parameter) {
    return this.sqlSessionProxy.insert(statement, parameter);
  }
}
```

sqlSessionProxy 其实也是一个代理，代理的是 SqlSession 接口：

```java
    this.sqlSessionProxy = (SqlSession) newProxyInstance(
        SqlSessionFactory.class.getClassLoader(),
        new Class[] { SqlSession.class },
        new SqlSessionInterceptor());
```

SqlSessionInterceptor：

```java
 /**
   * Proxy needed to route MyBatis method calls to the proper SqlSession got
   * from Spring's Transaction Manager
   * It also unwraps exceptions thrown by {@code Method#invoke(Object, Object...)} to
   * pass a {@code PersistenceException} to the {@code PersistenceExceptionTranslator}.
   */
	private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    	// （1）
      SqlSession sqlSession = SqlSessionUtils.getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      
      try {
        // (2)
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        // ... 省略代码
        throw t;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  } 
```

（1）处获取实际执行逻辑的对象，SqlSessionInterceptor 的这个 invoker 方法到底还是帮别人代理了逻辑，总归需要找到一个真实的类去处理逻辑。

（2）找到类真实的 SqlSession 以后，反射执行 Java 的 Method。

最初的问题是执行 sql 前对连接是怎么处理的，那么就应该跟进 SqlSessionUtils.getSqlSession 方法看个究竟。

```java
  public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ExecutorType executorType, 
    PersistenceExceptionTranslator exceptionTranslator) 
  {
    //（1）
    SqlSessionHolder holder = (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);
    if (session != null) {
      return session;
    }
    //（2）
    session = sessionFactory.openSession(executorType);

    registerSessionHolder(sessionFactory, executorType, exceptionTranslator, session);

    return session;
  }
```

（1）：从一个ThreadLocal中获取 SqlSessionHolder，SqlSessionHolder 内部封装了 SqlSession，实现类是 DefaultSqlSession。

（2）：从 SqlSessionHolder 中拿到了 DefaultSqlSession。

debug 到这个地方，TransactionSynchronizationManager.getResource 是直接返回了个 SqlSessionHolder 对象，而不是 null。真实的 sql 执行在 DefaultSqlSession 中，由他的 Executor 属性执行。

那么这里问题来了，DefaultSqlSession 是什么时候被创建，又被塞进 TransactionSynchronizationManager 的 ThreadLocal 中去的？

TransactionSynchronizationManager 的 ThreadLocal：

```java
public abstract class TransactionSynchronizationManager {
	private static final ThreadLocal<Map<Object, Object>> resources =
			new NamedThreadLocal<Map<Object, Object>>("Transactional resources");
}
```

这里可以猜测，因为测试的 demo 是用 @Transaction 注解的，也就是用到的 spring 申明式事，一定是在 aop 的切面里通过 TransactionSynchronizationManager 的方法设置进去的。

@Transactional 注解标注的申明式事务是用 AOP 实现的，那么事务逻辑是怎么织入的呢，织入的逻辑中包含了开启 session 的代码了吗？是不是用 TransactionSynchronizationManager 打开并暂存了 session ?

事务逻辑的织入见 「spring-声明式事务如何织入事务逻辑.md」。但是织入的时，往 TransactionSynchronizationManager 里设置的不是 SqlSessionHolder。而是 org.springframework.jdbc.datasource.ConnectionHolder，且 key 是 DataSource。

那么 SqlSessionHolder 又是什么时候设置进 TransactionSynchronizationManager 的呢？这个问题暂时搁置，继续 debug 进上面的 SqlSessionInterceptor.invoke 方法。获取的 SqlSession 是 DefaultSqlSession，反射执行的方法是 public int insert(String, Object)。

```java
public class DefaultSqlSession implements SqlSession {
  @Override
  public int update(String statement, Object parameter) {
    MappedStatement ms = configuration.getMappedStatement(statement);
    return executor.update(ms, wrapCollection(parameter));
  }
}
```

executor.update：

```java
public class CachingExecutor implements Executor {
  @Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    return delegate.update(ms, parameterObject);
  }
}
```

delegate.update：

```java
public abstract class BaseExecutor implements Executor {
	  @Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
		//..
    return doUpdate(ms, parameter);
  }
}
```

doUpdate：

```java
public class SimpleExecutor extends BaseExecutor {
  @Override
  public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(this, ms, parameter, RowBounds.DEFAULT, null, null);
      //(1)
      Statement stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.update(stmt);
    } finally {
      closeStatement(stmt);
    }
  }
}
```

prepareStatement：

```java
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    handler.parameterize(stmt);
    return stmt;
  }
```

getConnection：

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

transaction.getConnection：

```java
public class SpringManagedTransaction implements Transaction {
  @Override
  public Connection getConnection() throws SQLException {
    if (this.connection == null) {
      openConnection();
    }
    return this.connection;
  }
  
  private void openConnection() throws SQLException {
    this.connection = DataSourceUtils.getConnection(this.dataSource);
    this.autoCommit = this.connection.getAutoCommit();
    this.isConnectionTransactional = DataSourceUtils.isConnectionTransactional(this.connection, this.dataSource);
  }
}
```

DataSourceUtils.getConnection：

```java
// 省略部分代码
public abstract class DataSourceUtils {
  public static Connection doGetConnection(DataSource dataSource) throws SQLException {
    ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    if (conHolder != null) {
			// ..
			return conHolder.getConnection();
		}
    Connection con = dataSource.getConnection();
    // ..
    return con;
  }
}
```

至此就很明确了，最终的连接来自 TransactionSynchronizationManager。

#### 总结

1. 这个问题的关键是第二次查询没有获取新的连接，那么这里是不是获取新连接是取决于什么的？是不是因为第二个执行 sql 用的 mapper 内部注入的 DataSource 和当前事务属于同一个数据源，所以才不获取新的连接？

   是的，AbstractRoutingDataSource 在内部做了连接数据库的路由，但是在外层的 spring 看来，依然只是一个数据源。当前操作的数据源已有连接的情况，复用连接。

2. spring 和 mybatis 会对 Mapper 接口实现一个代理，代理作为 Mapper 的实现类，真实的承接具体 sql 执行。那么决定开不开新连接，是由 mapper 代理实现的，还是由外层的 spring 事务抽象实现的？

   由 spring 实现，spring 开启的连接放在 TransactionSynchronizationManager 的 ThreadLocal 中，Mybatis 用到连接的时候通过 TransactionSynchronizationManager 获取。