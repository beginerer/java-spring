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
![스크린샷 2024-07-21 160301](https://github.com/user-attachments/assets/8e875efa-8274-46ba-a641-169e0364dd6d)

 
