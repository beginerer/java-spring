# DataSourceTransactionManager 클래스를 통해 알아보는 Dynamic Proxy와 Thread Bind Transaction

DataSourceTransactionManger 클래스의 doc에서 중요하다고 생각되는 부분을 함께 살펴봅시다.

> Binds a JDBC Connection (from the specified DataSource) to the current thread, potentially allowing for one thread-bound Connection per DataSource.
<br/>

>This transaction manager will associate Connections with thread-bound transactions, according to the specified propagation behavior. It assumes that a separate, independent Connection can be obtained even during an ongoing transaction.
<br/>


>Application code is required to retrieve the JDBC Connection via DataSourceUtils. getConnection(DataSource) instead of a standard EE-style DataSource. getConnection() call.

<br/> 
요약하면 쓰레드를 커넥션에 바인딩하고, 쓰레드를 트렌젝션에 바인딩하므로 동기화된 트렌잭션을 사용하기 용이해집니다. 
<br/>
<br/> 
간단하게 DataSourceUtils 클래스의 Connection을 반환하는 doConnection()메서드를 살펴보겠습니다.

```java
public static Connection doGetConnection(DataSource dataSource) throws SQLException {

		ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
		if (conHolder != null && (conHolder.hasConnection() || conHolder.isSynchronizedWithTransaction())) {
			conHolder.requested();
			if (!conHolder.hasConnection()) {
				logger.debug("Fetching resumed JDBC Connection from DataSource");
				conHolder.setConnection(fetchConnection(dataSource));
			}
			return conHolder.getConnection();
		}
		// Else we either got no holder or an empty thread-bound holder here.
		Connection con = fetchConnection(dataSource);

....

		return con;
	}
```
이 부분은 해당 쓰레드에 바인딩된 커넥션이 있는지 조회하는 로직입니다. 
이는 TransactionSynchronizationManager클래스에서 ThreadLocal을 사용하여 구현됩니다.
<br/> <br/> 

ThreadLoacl에 대한 공부가 아직 부족해서 나중에 더 자세히 알아보도록하고, 정말 쓰레드에 바인딩된 커넥션을 반환하는지 테스트코드를 통해 확인해 봅시다.

```java
@BeforeEach
    void before() {
        dataSource = new HikariDataSource();
        dataSource.setJdbcUrl(URL);
        dataSource.setUsername(USERNAME);
        dataSource.setPassword(PASSWORD);
        dataSource.setMaximumPoolSize(10);
        dataSource.setPoolName("MyPool");
    }
```
커넥션 풀의 사이즈를 10으로 설정해놓았습니다.

```java
    @Test
    void dataSourceUtilsCon() {
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
        Thread thread = new Thread(new Runnable() {
            @Override
            public void run() {
                Connection con = DataSourceUtils.getConnection(dataSource);
                System.out.println(con.toString());
                TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
                transactionInfo(status);
            }
        });
        threadTest(thread);
    }
    
    private void threadTest(Thread thread) {
        IntStream.range(0,100).forEach(i -> {
            try {
                Thread.sleep(1000);
                System.out.println("time = "+(i+1));
                thread.run();
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        });
    }
```
커넥션을 얻기만하고 커넥션을 닫지 않았습니다.
</br>

 만약 쓰레드에 커넥션이 바인딩되지 않는다면 매번 다른 커넥션을 얻을 것이므로 커넥션풀 사이즈인 10개 이상의 커넥션 객체를 얻을 수 없을 것입니다. </br>
 반대로 쓰레드에 커넥션이 바인딩 되어있다면 매번 같은 커넥션이 리턴될 것이므로 문제없이 작동할 것이라고 생각할 수 있습니다.
<img src="https://github.com/user-attachments/assets/8e875efa-8274-46ba-a641-169e0364dd6d" width="800" height="400"/>

 위의 그림에서 보듯이 문제없이 실행이 된다는 것을 확인할 수 있습니다. 이로써 DataSourceUtils.getConnection()은 쓰레드에 바인딩된 커넥션을 반환한다는 것을 알 수 있습니다.

그렇다면 표준 EE-Style에서는 어떤 결과가 나올까요?

doc에 따르면 쓰레드에 바인딩된 커넥션을 반환하지 않기 때문에 쓰레드 풀 사이즈를 초과해서 커넨션 반환 함수를 실행할 수 없다는 것을 예상할 수 있습니다.
```java
@Test
    void EEStyleDataSource() throws InterruptedException {
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);


        Thread thread = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                Connection con = dataSource.getConnection();
                System.out.println(con.toString());
                TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
                transactionInfo(status);

            }
        });
        Thread.sleep(1000);
        threadTest(thread);
    }
```
![스크린샷 2024-07-21 160733](https://github.com/user-attachments/assets/9e0a0b99-027a-44f5-9fd7-1687766f511d)
datasource.getConnection()함수를 호출할 때마다 다른 커넥션을 반환하기 때문에 커넥션풀 사이즈 이상의 함수를 실행할 수 없다는 것을 확인할 수 있습니다.

로그 Connection is not avaible. (total = 10, active=10, idle=0, waiting=0)  보시면 커넥션풀에 사용가능한 커넥션이 없는데 datasource.getConnection()을 해서 오류가 난것을 확인할 수 있습니다.

## Dynamic Proxy

java에서는 proxy를 사용하여 표준 EE-Style코드를 리팩토링하지 않아도 쓰레드 바인딩 커넥션을 사용할 수 있게 해줍니다.

헷갈렸던 부분은 쓰레드 바인딩된 커넥션을 반환하는 것이 아니고, 커넥션의 함수를 호출할 때마다 쓰레드에 바인딩된 커넥션을 생성하고  Task를 수행 후 닫는 방식이라는 사실이었습니다.

자세하게 알아보기 전에 잘 작동하는지  테스트 코드를 통해 확인해봅시다.
```java
    @Test
    void dataSourceProxy() throws SQLException {

        TransactionAwareDataSourceProxy dataSourceProxy = new TransactionAwareDataSourceProxy(dataSource);
        DataSourceTransactionManager transactionManager = new DataSourceTransactionManager(dataSourceProxy);

        long start = System.currentTimeMillis();
        Thread thread = new Thread(new Runnable() {
            @SneakyThrows
            @Override
            public void run() {
                Connection con = dataSourceProxy.getConnection();
                System.out.println(con.toString());
                TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());
                transactionInfo(status);
            }
        });
        Thread.sleep(1000);
        threadTest(thread);
        long end = System.currentTimeMillis();
        System.out.println("소요 시간 = " + (end-start));
    }
```
<img src="https://github.com/user-attachments/assets/916a376c-f44f-4f66-9d67-4ff71fc437fb" width="800" height="400"/>

위의 그림에서 보듯이 커넥션 프록시를 사용해서 쓰레드에 바인딩된 커넥션을 사용할 수 있게되었습니다. 

주의할점은 프록시로 생성된 커넥션 객체가 쓰레드에 바인딩 된 것이아니고, 커낵션을 함수를 호출할때 dynamic proxy의 invoke메서드가 호출되면서 동적으로 쓰레드 바인딩된 커넥션을 만들고 Clinet의 Task를 수행 후 커넥션을 닫습니다.

## TransactionAwareDataSourceProxy
```java
@Override
	public Connection getConnection() throws SQLException {
		DataSource ds = obtainTargetDataSource();
		Connection con = getTransactionAwareConnectionProxy(ds);
		if (!this.lazyTransactionalConnections && shouldObtainFixedConnection(ds)) {
			((ConnectionProxy) con).getTargetConnection();
		}
		return con;
	}
protected Connection getTransactionAwareConnectionProxy(DataSource targetDataSource) {
		return (Connection) Proxy.newProxyInstance(
				ConnectionProxy.class.getClassLoader(),
				new Class<?>[] {ConnectionProxy.class},
				new TransactionAwareInvocationHandler(targetDataSource));
	}
```
```java
@Override
		public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
			// Invocation on ConnectionProxy interface coming in...

			...
			
			if (this.target == null) {
				if (method.getName().equals("getWarnings") || method.getName().equals("clearWarnings")) {
					// Avoid creation of target Connection on pre-close cleanup (e.g. Hibernate Session)
					return null;
				}
				if (this.closed) {
					throw new SQLException("Connection handle already closed");
				}
				if (shouldObtainFixedConnection(this.targetDataSource)) {
					this.target = DataSourceUtils.doGetConnection(this.targetDataSource);
				}
			}
			Connection actualTarget = this.target;
			if (actualTarget == null) {
				actualTarget = DataSourceUtils.doGetConnection(this.targetDataSource);
			}

			if (method.getName().equals("getTargetConnection")) {
				// Handle getTargetConnection method: return underlying Connection.
				return actualTarget;
			}

			// Invoke method on target Connection.
			try {
				Object retVal = method.invoke(actualTarget, args);

				// If return value is a Statement, apply transaction timeout.
				// Applies to createStatement, prepareStatement, prepareCall.
				if (retVal instanceof Statement statement) {
					DataSourceUtils.applyTransactionTimeout(statement, this.targetDataSource);
				}

				return retVal;
			}
			catch (InvocationTargetException ex) {
				throw ex.getTargetException();
			}
			finally {
				if (actualTarget != this.target) {
					DataSourceUtils.doReleaseConnection(actualTarget, this.targetDataSource);
				}
			}
		}
	}
```
프록시의 invoke 메서드를 보시면 DataSourceUtils.doGetConnection(this.targetDataSource)를 통해 쓰레드 바인드된 커넥션을 얻고, Task를 수행후 finally를 통해 DataSourceUtils.doReleaseConnection()으로 커넥션을 닫습니다.

마지막으로 dynamicProxy는 리플렉션을 사용하기때문에 성능 오버헤드가 발생한다는것을 예상할 수 있습니다. 

이를 테스트 해보았는데요 커넥션 프록시를 사용할 경우 위의 그림에서 보듯이 소요시간은 102039였고,
![스크린샷 2024-07-21 191211](https://github.com/user-attachments/assets/d71ce9d8-bcc2-4cfe-ad31-b08af39b7170)

프록시를 사용하지 않고 DataSourceUtils.getConnection()을 사용할 경우 소요시간은 101287걸리는 것을 확인 할 수 있었습니다.

약 1%차이로 생각보다 큰 차이는 아닌 것 같지만 프록시를 사용했을때 소요시간이 더 걸린다는 것을 확인할 수 있었습니다.

여기서는 코드 리팩토링을 줄이기 위해 proxy패턴이 사용되었는데요 소스코드를 분석해보면서 dynamic proxy패턴에 대해 공부해 보는 계기가 되었습니다. dynamic proxy는 리플렉션이 사용되기때문에 성능 오버헤드가 필연적으로 발생하므로 무분별하게 사용하지 않고 꼭 필요한 곳에 사용하는 것이 중요합니다.
