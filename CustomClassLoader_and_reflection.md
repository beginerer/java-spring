# ClassLoader(.class -> 메타정보)

자바 런타임 환경(JRE)의 필수 구성 요소인 클래스 로더는 자바 클래스를 자바 가상 머신(JVM)에 동적으로 로드한다. 

클래스는 일반적으로 특별히 요청된 경우에만 로드되어 효율적인 리소스 활용과 최적화된 성능을 제공합니다.

클래스 로더 덕분에 자바 런타임 시스템은 파일 및 파일 시스템과 독립으로 작동합니다.

**Delegation**, a crucial concept handled by the loader, ensures efficient handling of class loading tasks.

The class loader is tasked with locating libraries, reading their contents, and loading the classes they contain. 

This loading process usually occurs **“on demand,”** meaning it waits until the program requests the class. 

Each named class can only be loaded once by a specific class loader.

<img src="https://github.com/beginerer/java-spring/assets/96945728/a4c26e47-403c-448d-898b-f6746fd704c8.png" width="400" height="400"/>
