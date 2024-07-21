# DataSourceTransactionManager 클래스를 통해 알아보는 Dynamic Proxy와 Thread Bind Transaction

DataSourceTransactionManger 클래스의 doc에서 중요하다고 생각되는 부분을 함께 살펴봅시다.

> Binds a JDBC Connection (from the specified DataSource) to the current thread, potentially allowing for one thread-bound Connection per DataSource.
<br/>

>This transaction manager will associate Connections with thread-bound transactions, according to the specified propagation behavior. It assumes that a separate, independent Connection can be obtained even during an ongoing transaction.
<br/>


>Application code is required to retrieve the JDBC Connection via DataSourceUtils. getConnection(DataSource) instead of a standard EE-style DataSource. getConnection() call.
<br/> 
요약하면 쓰레드를 커넥션에 바인딩하고, 쓰레드를 트렌젝션에 바인딩하므로 동기화된 트렌잭션을 사용하기 용이해집니다.<br/> 

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
