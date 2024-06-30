# ClassLoader(.class -> 메타정보)

자바 런타임 환경(JRE)의 필수 구성 요소인 클래스 로더는 자바 클래스를 자바 가상 머신(JVM)에 동적으로 로드한다. 

클래스는 일반적으로 특별히 요청된 경우에만 로드되어 효율적인 리소스 활용과 최적화된 성능을 제공합니다.

클래스 로더 덕분에 자바 런타임 시스템은 파일 및 파일 시스템과 독립으로 작동합니다.

**Delegation**, a crucial concept handled by the loader, ensures efficient handling of class loading tasks.

The class loader is tasked with locating libraries, reading their contents, and loading the classes they contain. 

This loading process usually occurs **“on demand,”** meaning it waits until the program requests the class. 

Each named class can only be loaded once by a specific class loader.

<img src="https://github.com/beginerer/java-spring/assets/96945728/a4c26e47-403c-448d-898b-f6746fd704c8.png" width="400" height="400"/>

[이미지 출처] : (https://medium.com/@alxkm/java-jvm-minimum-what-every-developer-should-know-226321cdffd0)

<br/>

지금까지 간략하게 클래스 로더에 대해서 설명했습니다. 이제 소스코드를 분석해보고, 커스텀 클래스 로더를 구현해보겠습니다.
<br/>
## CustomClassLoader
```java
@Component
public class CustomClassLoader extends ClassLoader{


    private String classPath;

    public void setClassPath(String classPath) {
        this.classPath = classPath;
    }

    protected CustomClassLoader(ClassLoader parent) {
        super(parent);
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] classBytes = loadClassData(name);
        return defineClass(name, classBytes, 0, classBytes.length);
    }
    private byte[] loadClassData(String className) throws ClassNotFoundException {
        String fileName = className.replace('.', File.separatorChar) + ".class";
        String filePath = classPath + File.separator + fileName;

        try (FileInputStream fis = new FileInputStream(filePath);
             ByteArrayOutputStream bos = new ByteArrayOutputStream()) {
            byte[] buffer = new byte[1024];
            int bytesRead;
            while ((bytesRead = fis.read(buffer)) != -1) {
                bos.write(buffer, 0, bytesRead);
            }
            return bos.toByteArray();
        } catch (IOException e) {
            throw new ClassNotFoundException("Class '" + className + "' not found.", e);
        }
    }
}
```
커스텀 클래스 로더를 구현하기 위해선 ClassLoader를 상속받은 후 findClass()를 오버라이딩하면 됩니다. 

loadClassData함수는 클래스파일의 이름이 매개변수로 주어지면 약속된 경로에서 클래스파일을 찾고, 이를 바이트배열로 변환해주는 역할을 합니다. 

바이트배열이 인자로 주어지면 클래스 로더에서 메타정보를 만드는데 이러한 프로세스는 상위 클래스에서 위임해서 처리합니다.

# UploadController

```java
@Controller
public class UploadController {

    @Value("${classPath}")
    String UPLOAD_DIR;

    @Autowired
    CustomClassLoader classLoader; 

    @ResponseBody
    @PostMapping(value = "/upload")
    public String uploadFile(@RequestParam("file")MultipartFile file) {
        if(file.isEmpty() ||!file.getOriginalFilename().endsWith(".java")) { // 파일 확장자 검사
            return "파일이 비었거나 올바른 형식의 파일을 업로드 하지 않았습니다.";
        }
        try {
            Files.createDirectories(Paths.get(UPLOAD_DIR)); // 파일 저장 디렉토리 생성

            Path path = Paths.get(UPLOAD_DIR + File.separator+  file.getOriginalFilename()); // 파일 저장 경로 설정

            Files.copy(file.getInputStream(),path);

            if(!isVaildJavaFile(path)) {
                return "형식에 맞는 파일을 업로드 해주세요";
            }
            // 파일명을 기반으로 클래스 이름을 추출
            String className = file.getOriginalFilename().replace(".java", "");

            // CustomClassLoader를 통해 클래스를 로드
            Class<?> myClass = classLoader.findClass(className);
            //생성자 가져오기
            Constructor<?> constructor = myClass.getConstructor(String.class, int.class, String.class, int.class);
            //인스턴스 생성
            Object instance = constructor.newInstance("privateValue", 10, "publicValue", 10);

            //리플렉션
            // 수정전 인스턴스 출력
            Method printMethod = myClass.getMethod("print");
            printMethod.invoke(instance);

            Field privateStringField = myClass.getDeclaredField("privateString");
            privateStringField.setAccessible(true); // private 필드에 접근 허용
            privateStringField.set(instance, "modifiedPrivateValue");

            Field publicString = myClass.getDeclaredField("publicString");
            publicString.set(instance,"modifiedPublicValue");

            Field privateInt = myClass.getDeclaredField("privateInt");
            privateInt.setAccessible(true);
            privateInt.set(instance,100);
            Field publicIntField = myClass.getDeclaredField("publicInt");
            publicIntField.set(instance, 100);


            //수정 후 인스턴스 출력
            System.out.println("--------");
            printMethod.invoke(instance);

            return "success";

        } catch (IOException | ClassNotFoundException | NoSuchMethodException | InstantiationException |
                 IllegalAccessException | InvocationTargetException | NoSuchFieldException e) {
            throw new RuntimeException(e);
        }

    }
    private boolean isVaildJavaFile(Path filePath) { //소스 파일 업로드

    try {
        JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
        if(compiler==null) {
            throw new IllegalStateException("컴파일러 생성 실패");
        }
        ByteArrayOutputStream errStream = new ByteArrayOutputStream();
        int result = compiler.run(null, null, new PrintStream(errStream), filePath.toString());

        // 컴파일 오류가 있는지 확인합니다.
        if (result != 0) {
            System.out.println("컴파일 실패: " + errStream.toString());
            return false;
        }
        return true;
    } catch (IllegalStateException e) {
        throw new RuntimeException(e);
    }


    }
}
```
Upload컨트롤러는 클라이언트가 웹에서 자바 소스파일을 업로드하면 생성된 자바 컨파일러로 소스파일을 컴파일하고, 부수효과로 클래스파일이 생성됩니다.

자바 컴파일러를 통해 파일에 대한 유효성 검증이 완료되면  CustomClassLoader를 통해 클래스에대 한 메타정보를 생성합니다.

마지막으로 지금까지의 과정이 제대로 작동하였는지 확인해 보기 위하여 리플렉션을 사용하여 출력해보는 코드를 추가하였습니다.

리플렉션으로는 public 접근지정자에만 접근할 수 있기 때문에 private을 접근자로하는 메서드나 필드에  접근하기 위해서는 field.setAccessible(true) 코드를 추가해 주어야합니다.

## Hello.java(웹에서 업로드한 소스파일)
```java
public class hello {

    private String privateString;
    private int privateInt;

    public String publicString;
    public int publicInt;

    public void print() {
        System.out.println("privateString = " + privateString);
        System.out.println("privateInt = " + privateInt);
        System.out.println("publicString = " + publicString);
        System.out.println("publicInt = " + publicInt);
    }

    public hello(String privateString, int privateInt, String publicString, int publicInt) {
        this.privateString = privateString;
        this.privateInt = privateInt;
        this.publicString = publicString;
        this.publicInt = publicInt;
    }
}
```
![스크린샷 2024-07-01 000512](https://github.com/beginerer/java-spring/assets/96945728/40d651fa-c618-4e77-9725-f37004e97a88)
![스크린샷 2024-07-01 001901](https://github.com/beginerer/java-spring/assets/96945728/60b09905-3dbe-4ad9-bb0c-d40ecb5dfa75)

위의 그림에서 보듯이 출력이 잘되는것을 확인할 수 있습니다.
