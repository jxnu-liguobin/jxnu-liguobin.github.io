---
title: JDBC原理&浅析HIVE-JDBC
categories:
- 数据库
---

* 目录
{:toc}

# 什么是JDBC？

JDBC是Java DataBase Connectivity的缩写，它是Java程序访问数据库的标准接口。[Java8 JDBC API](https://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/)

使用Java程序访问数据库时，Java代码并不是直接通过TCP连接去访问数据库，而是通过JDBC接口来访问，而JDBC接口则通过JDBC驱动来实现真正对数据库的访问。

例如，我们在Java代码中如果要访问MySQL，那么必须编写代码操作JDBC接口。注意到JDBC接口是Java标准库自带的，所以可以直接编译。而具体的JDBC驱动是由数据库厂商提供的，例如，MySQL的JDBC驱动由Oracle提供。因此，访问某个具体的数据库，我们只需要引入该厂商提供的JDBC驱动，就可以通过JDBC接口来访问，这样保证了Java程序编写的是一套数据库访问代码，却可以访问各种不同的数据库，因为他们都提供了标准的JDBC驱动：

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐

│  ┌───────────────┐  │
   │   Java App    │
│  └───────────────┘  │
           │
│          ▼          │
   ┌───────────────┐
│  │JDBC Interface │<─┼─── JDK
   └───────────────┘
│          │          │
           ▼
│  ┌───────────────┐  │
   │  JDBC Driver  │<───── Vendor
│  └───────────────┘  │
           │
└ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ┘
           ▼
   ┌───────────────┐
   │   Database    │
   └───────────────┘
```

从代码来看，Java标准库自带的JDBC接口其实就是定义了一组接口，而某个具体的JDBC驱动其实就是实现了这些接口的类：

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐

│  ┌───────────────┐  │
   │   Java App    │
│  └───────────────┘  │
           │
│          ▼          │
   ┌───────────────┐
│  │JDBC Interface │<─┼─── JDK
   └───────────────┘
│          │          │
           ▼
│  ┌───────────────┐  │
   │ MySQL Driver  │<───── Oracle
│  └───────────────┘  │
           │
└ ─ ─ ─ ─ ─│─ ─ ─ ─ ─ ┘
           ▼
   ┌───────────────┐
   │     MySQL     │
   └───────────────┘
```

实际上，一个MySQL的JDBC的驱动就是一个jar包，它本身也是纯Java编写的。我们自己编写的代码只需要引用Java标准库提供的java.sql包下面的相关接口，由此再间接地通过MySQL驱动的jar包通过网络访问MySQL服务器，所有复杂的网络通讯都被封装到JDBC驱动中，因此，Java程序本身只需要引入一个MySQL驱动的jar包就可以正常访问MySQL服务器：

```
┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
   ┌───────────────┐
│  │   App.class   │  │
   └───────────────┘
│          │          │
           ▼
│  ┌───────────────┐  │
   │  java.sql.*   │
│  └───────────────┘  │
           │
│          ▼          │
   ┌───────────────┐     TCP    ┌───────────────┐
│  │ mysql-xxx.jar │──┼────────>│     MySQL     │
   └───────────────┘            └───────────────┘
└ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
          JVM
```

使用JDBC的好处是：

* 各数据库厂商使用相同的接口，Java代码不需要针对不同数据库分别开发；
* Java程序编译期仅依赖java.sql包，不依赖具体数据库的jar包；
* 可随时替换底层数据库，访问数据库的Java代码基本不变。

# JDBC的有哪些种类？

## 1.JDBC-ODBC桥

这种类型的驱动把所有JDBC的调用传递给ODBC，再让后者调用数据库本地驱动代码（也就是数据库厂商提供的数据库操作二进制代码库，例如Oracle中的oci.dll）。

- 优点
    - 只要有对应的ODBC驱动（大部分数据库厂商都会提供），几乎可以访问所有的数据库。
- 缺点
    - 执行效率比较低，不适合大数据量访问的应用；
    - 由于需要客户端预装对应的ODBC驱动，不适合Internet/Intranet应用。

## 2.本地API驱动

这种类型的驱动通过客户端加载数据库厂商提供的本地代码库（C／C++等）来访问数据库，而在驱动程序中则包含了Java代码。

- 优点
    - 速度快于第一类驱动（但仍比不上第3、第4类驱动）。
- 缺点
    - 由于需要客户端预装对应的数据库厂商代码库，仍不适合Internet/Intranet应用。

## 3.网络协议驱动

这种类型的驱动给客户端提供了一个网络API，客户端上的JDBC驱动程序使用套接字（Socket）来调用服务器上的中间件程序，后者再将其请求转化为所需的具体API调用。

- 优点
    - 不需要在客户端加载数据库厂商提供的代码库，单个驱动程序可以对多个数据库进行访问，可扩展性较好。
- 缺点
    - 在中间件层仍需对最终数据进行配置；
    - 由于多出一个中间件层，速度不如第四类驱动程序。

## 4.本地协议驱动

这种类型的驱动使用Socket，直接在客户端和数据库间通信。

- 优点
    - 访问速度最快；
    - 这是最直接、最纯粹的Java实现。

- 缺点
    - 几乎只有数据库厂商自己才能提供这种类型的JDBC驱动；
    - 需要针对不同的数据库使用不同的驱动程序。

# HIVE JDBC 源码分析

有了上面的预备知识，看HIVE源码就容易理解了。HIVE JDBC使用第三和第四两种方式。首先，Java既然提供了标准接口，那么所有JDBC程序自然必须实现，才能保证操作数据库是透明的，先找到这个实现类：`HiveDriver`。
与其他JDBC相同，它实现`java.sql.Driver`接口，重写了抽象方法。

1.第一步就是必须在`HiveDriver`中注册自己的实例对象：
```java
  static {
    try {
      java.sql.DriverManager.registerDriver(new HiveDriver());
    } catch (SQLException e) {
      throw new RuntimeException("Failed to register driver", e);
    }
  }
```

必须向上面那样注册自己，在`java.sql.Driver`的接口文档上强调了这点，这样注册后才能以下面方式加载驱动，并在接下来使用该驱动来建立连接：

```java
Class.forName("org.apache.hive.jdbc.HiveDriver");
```

2.建立连接当然需要使用到URL，而URL必须是经过校验的，由于是HIVE的驱动，所以使用的格式是`jdbc:hive://[host[:port]]`：
```java
  /**
   * Checks whether a given url is in a valid format.
   *
   * The current uri format is: jdbc:hive://[host[:port]]
   *
   * jdbc:hive:// - run in embedded mode jdbc:hive://localhost - connect to
   * localhost default port (10000) jdbc:hive://localhost:5050 - connect to
   * localhost port 5050
   *
   * TODO: - write a better regex. - decide on uri format
   */

  @Override
  public boolean acceptsURL(String url) throws SQLException {
    return Pattern.matches(Utils.URL_PREFIX + ".*", url);
  }
```

3.校验成功返回Connection对象：
```java
  /*
   * As per JDBC 3.0 Spec (section 9.2)
   * "If the Driver implementation understands the URL, it will return a Connection object;
   * otherwise it returns null"
   */
  @Override
  public Connection connect(String url, Properties info) throws SQLException {
    return acceptsURL(url) ? new HiveConnection(url, info) : null;
  }
```

`HiveConnection`类是HIVE中`java.sql.Connection`接口的实现。info是连接所需的一些属性键值对。HIVE中所有可用的键在`org.apache.hive.jdbc.Utils`都被定义成了常量，如user，password，retries，token等等。

除了该方法，还有一些获取版本和信息的方法需要重写，这些是根据场景实现的，一般是从指定文件读取的，暂时可以不用太关心这些。

4.`HiveConnection`对象的获取：

可有两种方式，第一种是通过`DriverManager#getConnection`获取，下面是`getConnection`的具体内部实现：

```java
    //  Worker method called by the public getConnection() methods.
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) throws SQLException {
        /*
         * When callerCl is null, we should check the application's
         * (which is invoking this class indirectly)
         * classloader, so that the JDBC driver class outside rt.jar
         * can be loaded from here.
         */
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) { // 线程安全获取类加载器
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }

        if(url == null) {
            throw new SQLException("The url cannot be null", "08001");
        }

        println("DriverManager.getConnection(\"" + url + "\")");

        // Walk through the loaded registeredDrivers attempting to make a connection.
        // Remember the first exception that gets raised so we can reraise it.
        SQLException reason = null;

        // 这里是遍历了registeredDrivers，所以前面说的必须注册就好理解了，不注册这里就拿不到了。
        for(DriverInfo aDriver : registeredDrivers) {
            // If the caller does not have permission to load the driver then
            // skip it.
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    Connection con = aDriver.driver.connect(url, info); // 看，这里同样开始使用重写的connect方法，获取Connection对象。
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }

        // if we got here nobody could connect.
        if (reason != null)    {
            println("getConnection failed: " + reason);
            throw reason;
        }

        println("getConnection: no suitable driver found for "+ url);
        throw new SQLException("No suitable driver found for "+ url, "08001");
    }


}
```

另一种通过`Driver#connect`获取。

```java
    Connection conn = driver.connect(connString, connectionProps);
```

两者的注释有点不同，`getConnection`多抛出了一个异常`SQLTimeoutException`。通过分析`getConnection`的源码也可以看出，最好使用Java给我们提供的`DriverManager#getConnection`获取连接。起码人家给我们判断了一下连接是否存在，同时也保证拿到的连接是否为之前driver注册的。

5.`Connection`连接建立

嵌入式使用thrift RPC，浏览器端使用ServerSocket。上面提到，第四种JDBC直接使用socket连接，性能最高。


```java
  protected HiveConnection(String uri, Properties info,
      IJdbcBrowserClientFactory browserClientFactory) throws SQLException {
    try {
    // 连接URL和所需的属性信息经过处理得出`jdbcUriString`和`sessConfMap`
      connParams = Utils.parseURL(uri, info);// 需要zooKeeper
    } catch (ZooKeeperHiveClientException e) {
      throw new SQLException(e);
    }
    jdbcUriString = connParams.getJdbcUriString();
    // JDBC URL: jdbc:hive2://<host>:<port>/dbName;sess_var_list?hive_conf_list#hive_var_list
    // each list: <key1>=<val1>;<key2>=<val2> and so on
    // sess_var_list -> sessConfMap
    // hive_conf_list -> hiveConfMap
    // hive_var_list -> hiveVarMap
    sessConfMap = connParams.getSessionVars();
    setupLoginTimeout();//设置登录超时，由JdbcConnectionParams.SOCKET_TIMEOUT指定
    if (isKerberosAuthMode()) {// 根据不同模式设置host用于建立socket
      host = Utils.getCanonicalHostName(connParams.getHost());
    } else if (isBrowserAuthMode() && !isHttpTransportMode()) {
      throw new SQLException("Browser auth mode is only applicable in http mode");
    } else {
      host = connParams.getHost();
    }
    port = connParams.getPort();
    isEmbeddedMode = connParams.isEmbeddedMode();

    initFetchSize = Integer.parseInt(sessConfMap.getOrDefault(JdbcConnectionParams.FETCH_SIZE, "0"));

    if (sessConfMap.containsKey(JdbcConnectionParams.INIT_FILE)) {
      initFile = sessConfMap.get(JdbcConnectionParams.INIT_FILE);
    }
    wmPool = sessConfMap.get(JdbcConnectionParams.WM_POOL);
    // 指定应用程序名称 找到一个即可
    for (String application : JdbcConnectionParams.APPLICATION) {
      wmApp = sessConfMap.get(application);
      if (wmApp != null) {
        break;
      }
    }

    // add supported protocols  支持的thrift版本
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V1);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V2);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V3);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V4);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V5);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V6);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V7);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V8);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V9);
    supportedProtocols.add(TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V10);

    if (isBrowserAuthMode()) {
      try {
      // 浏览器模式 创建client，构建ServerSocket实例，属于第四种JDBC
        browserClient = browserClientFactory.create(connParams);
      } catch (HiveJdbcBrowserException e) {
        throw new SQLException("");
      }
    } else {
      browserClient = null;
    }
    if (isEmbeddedMode) {
      // 嵌入式模式 创建client，构建org.apache.hive.service.cli.thrift.EmbeddedThriftBinaryCLIService的实例。属于第三种JDBC
      client = EmbeddedCLIServicePortal.get(connParams.getHiveConfs());
      connParams.getHiveConfs().clear();
      // open client session
      if (isBrowserAuthMode()) {
        throw new SQLException(new IllegalArgumentException(
            "Browser mode is not supported in embedded mode"));
      }
      openSession();
      executeInitSql();
    } else {
      long retryInterval = 1000L;
      try {
        String strRetries = sessConfMap.get(JdbcConnectionParams.RETRIES);
        if (StringUtils.isNotBlank(strRetries)) {
          maxRetries = Integer.parseInt(strRetries);
        }
        String strRetryInterval = sessConfMap.get(JdbcConnectionParams.RETRY_INTERVAL);
        if(StringUtils.isNotBlank(strRetryInterval)){
          retryInterval = Long.parseLong(strRetryInterval);
        }
      } catch(NumberFormatException e) { // Ignore the exception
      }
        // 重试
      for (int numRetries = 0;;) {
        try {
          // open the client transport
          openTransport();
          // set up the client
          client = new TCLIService.Client(new TBinaryProtocol(transport));
          // 开启session并执行初始化SQL
          openSession();
          executeInitSql();

          break;
        } catch (Exception e) {
          LOG.warn("Failed to connect to " + connParams.getHost() + ":" + connParams.getPort());
          String errMsg = null;
          String warnMsg = "Could not open client transport with JDBC Uri: " + jdbcUriString + ": ";
          try {
            close(); //即使有异常连接也可能已经打开，所以先关闭?
          } catch (Exception ex) {
            // Swallow the exception
            LOG.debug("Error while closing the connection", ex);
          }
          if (ZooKeeperHiveClientHelper.isZkDynamicDiscoveryMode(sessConfMap)) {
            errMsg = "Could not open client transport for any of the Server URI's in ZooKeeper: ";
            // 尝试zookeeper中的下一个可用服务器，如果启用了“重试”，将重试所有服务器
            while(!Utils.updateConnParamsFromZooKeeper(connParams) && ++numRetries < maxRetries) {
              connParams.getRejectedHostZnodePaths().clear();
            }
            // 获取重试后的jdbcUriString，因为机器已经更新
            jdbcUriString = connParams.getJdbcUriString();
            if (isKerberosAuthMode()) {
              host = Utils.getCanonicalHostName(connParams.getHost());
            } else {
              host = connParams.getHost();
            }
            port = connParams.getPort();
          } else {
            errMsg = warnMsg;
            ++numRetries; //记录重试次数
          }

          if (numRetries >= maxRetries) {
            throw new SQLException(errMsg + e.getMessage(), " 08S01", e);
          } else {
            LOG.warn(warnMsg + e.getMessage() + " Retrying " + numRetries + " of " + maxRetries+" with retry interval "+retryInterval+"ms");
            try {
              Thread.sleep(retryInterval);// 每次重试的间隔时间，停顿等待
            } catch (InterruptedException ex) {
              //Ignore
            }
          }
        }
      }
    }

    // 使用线程安全代理包装client以序列化RPC调用
    client = newSynchronizedClient(client);
  }
```

最复杂的是`Connection`的建立，之后使用`Connection`对象创建`HiveStatement`对象操作SQL即可。`HiveStatement`也是Java标准库提供的JDBC接口`java.sql.Statement`的实现。
对于`EmbeddedMode`模式，如何使用thrift存储session并保持连接，是核心所在。HIVE中，session在服务器使用`ThreadLocal<ServerContext>`来存储。client调用`OpenSession` RPC 方法，服务端收到请求后会创建session：

```java
  @Override
  public TOpenSessionResp OpenSession(TOpenSessionReq req) throws TException {
    LOG.info("Client protocol version: " + req.getClient_protocol());
    TOpenSessionResp resp = new TOpenSessionResp();
    String userName = null;
    try {
      userName = getUserName(req);
      final SessionHandle sessionHandle = getSessionHandle(req, resp, userName);

      final int fetchSize = hiveConf.getIntVar(HiveConf.ConfVars.HIVE_SERVER2_THRIFT_RESULTSET_DEFAULT_FETCH_SIZE);

      Map<String, String> map = new HashMap<>();
      map.put(HiveConf.ConfVars.HIVE_SERVER2_THRIFT_RESULTSET_DEFAULT_FETCH_SIZE.varname, Integer.toString(fetchSize));
      map.put(HiveConf.ConfVars.HIVE_DEFAULT_NULLS_LAST.varname,
          String.valueOf(hiveConf.getBoolVar(ConfVars.HIVE_DEFAULT_NULLS_LAST)));
      resp.setSessionHandle(sessionHandle.toTSessionHandle());
      resp.setConfiguration(map);
      resp.setStatus(OK_STATUS);
      // 获取当前线程的ServerContext对象，
      ThriftCLIServerContext context = (ThriftCLIServerContext) currentServerContext.get(); 
      if (context != null) {
        context.setSessionHandle(sessionHandle);
      }
      LOG.info("Login attempt is successful for user : " + userName);
    } catch (Exception e) {
      // Do not log request as it contains password information
      LOG.error("Login attempt failed for user : {}", userName, e);
      resp.setStatus(HiveSQLException.toTStatus(e));
    }
    return resp;
  }
```

而关闭session，只需要client调用`CloseSession`：
```java
  @Override
  public TCloseSessionResp CloseSession(TCloseSessionReq req) throws TException {
    TCloseSessionResp resp = new TCloseSessionResp();
    try {
      SessionHandle sessionHandle = new SessionHandle(req.getSessionHandle());
      cliService.closeSession(sessionHandle);
      resp.setStatus(OK_STATUS);
      ThriftCLIServerContext context = (ThriftCLIServerContext) currentServerContext.get();
      if (context != null) {
        context.clearSessionHandle(); //删除session
      }
    } catch (Exception e) {
      LOG.error("Failed to close the session", e);
      resp.setStatus(HiveSQLException.toTStatus(e));
    }
    return resp;
  }
```

有了session后续就可以使用`HiveStatement`对象执行SQL。每个操作都是直接发送RPC请求，比如`execute`方法：
```java
  @Override
  public boolean execute(String sql) throws SQLException {
    runAsyncOnServer(sql);
    TGetOperationStatusResp status = waitForOperationToComplete();

    // The query should be completed by now
    if (!status.isHasResultSet() && stmtHandle.isPresent() && !stmtHandle.get().isHasResultSet()) {
      return false;
    }
    resultSet = new HiveQueryResultSet.Builder(this).setClient(client)
        .setStmtHandle(stmtHandle.get()).setMaxRows(maxRows).setFetchSize(fetchSize)
        .setScrollable(isScrollableResultset)
        .build();
    return true;
  }

// runAsyncOnServer方法是实现异步执行SQL，后面就能再使用stmtHandle获取结果
// 获取结果调用`getResultSet`方法，获取数据前校验connect是否有效
  private void runAsyncOnServer(String sql) throws SQLException {
    checkConnection("execute");

    reInitState();

    TExecuteStatementReq execReq = new TExecuteStatementReq(sessHandle, sql);
    /**
     * Run asynchronously whenever possible
     * Currently only a SQLOperation can be run asynchronously,
     * in a background operation thread
     * Compilation can run asynchronously or synchronously and execution run asynchronously
     */
    execReq.setRunAsync(true);
    execReq.setConfOverlay(sessConf);
    execReq.setQueryTimeout(queryTimeout);
    try {
      LOG.debug("Submitting statement [{}]: {}", sessHandle, sql);
      TExecuteStatementResp execResp = client.ExecuteStatement(execReq);
      Utils.verifySuccessWithInfo(execResp.getStatus());
      List<String> infoMessages = execResp.getStatus().getInfoMessages();
      if (infoMessages != null) {
        for (String message : infoMessages) {
          LOG.info(message);
        }
      }
      stmtHandle = Optional.of(execResp.getOperationHandle());
      LOG.debug("Running with statement handle: {}", stmtHandle.get());
    } catch (SQLException eS) {
      isLogBeingGenerated = false;
      throw eS;
    } catch (Exception ex) {
      isLogBeingGenerated = false;
      throw new SQLException("Failed to run async statement", "08S01", ex);
    }
  }
```

HIVE service-rpc模块的`TCLIService.thrift`文件定义了JDBC所需的所有方法和生成的多种语言代码，与grpc的protocol buffer类似。
service模块提供了rpc的实现。

# 引用

- [JDBC简介](https://www.liaoxuefeng.com/wiki/1252599548343744/1305152088703009)
- [Java数据库连接](https://zh.wikipedia.org/wiki/Java%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5)
- [HIVE JDBC](https://github.com/apache/hive/tree/master/jdbc)
