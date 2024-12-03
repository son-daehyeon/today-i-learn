# 2024년 12월 2일 (월)
> Annotation VS Decorator (Annotation 편)

## 개요
어제<del><i>(지각해서 오늘이긴 한데...)</i></del>는 Decorator에 대해 알아봤다. ([링크](https://github.com/son-daehyeon/today-i-learn/tree/main/2024-12-02))

오늘은 Annotation에 대해 알아보겠다.

이전 설명에도 있지만, 어노테이션은 주로 컴파일 언어에서 사용되고, 이런 특성에 따라 동작을 수정할 수 없어 단지 메타데이터만 수정할 수 있다.

그리고 주로 스프링과 같이 메타코딩 계열은 ClassLoader로 모든 클래스를 뒤지면서 Reflection API를 통해 특정 속성, 메소드, 클래스 등등 에 어노테이션이 달려있는지 확인한다.

만약, 확인될 경우 특정 작업을 수행한다. <i>(스프링에서는 주로 빈 컨테이너에 프록시된 인스턴스를 삽입한다.)</i>

아래 내용을 이해하기 위해선 자바의 Proxy와 Dynamic Proxy의 개념을 알아야 한다.

> 추천 게시글... <br/>
> https://sondaehyeon.tistory.com/entry/Java-Proxy <br/>
> https://sondaehyeon.tistory.com/entry/Java-Dynamic-Proxy

## ClassLoader
위에도 있지만, 컴파일의 특성상, 런타임 당시 어노테이션만 가지고 동작을 수정할 수 없다.

따라서 런타임 당시에 모든 클래스를 검색하여야 하는데, 아래 코드가 그 부분이다.

일단, ClassLoader란 JVM에서 바이트코를 읽어 class 객체를 생성하는 일을 한다.

즉, 우리가 import하면, JVM에서 ClassLoader을 활용하여 `어쩌고.class` 파일을 찾고, 그 파일의 바이트코드를 읽어 메모리에 올린다.

예를들어, 아래와 같은 코드를 컴파일 후 실행해보자.

```java
public class Main {
    public static void main(String[] args) {
        System.out.println("Hi.");
    }
}
```

```bash
javac Main.java
java -verbose:class Main
```

그럼 아래와 같이 필요한 모든 클래스를 불러오게된다.

```
[0.046s][info][class,load] java.lang.Object source: shared objects file
[0.047s][info][class,load] java.io.Serializable source: shared objects file
[0.047s][info][class,load] java.lang.Comparable source: shared objects file
[0.047s][info][class,load] java.lang.CharSequence source: shared objects file
[0.047s][info][class,load] java.lang.String source: shared objects file
[0.047s][info][class,load] java.lang.reflect.AnnotatedElement source: shared objects file
[0.047s][info][class,load] java.lang.reflect.GenericDeclaration source: shared objects file
[0.047s][info][class,load] java.lang.reflect.Type source: shared objects file
[0.048s][info][class,load] java.lang.invoke.TypeDescriptor source: shared objects file
[0.048s][info][class,load] java.lang.invoke.TypeDescriptor$OfField source: shared objects file
(생략)
Hi.
```

사실 ClassLoader만으로 TIL 이틀치 분량을 뽑아먹을 수 있어서, 간단하게 이정도로만 소개하겠다.

---

즉, 모든 패키지의 모든 클래스를 불러오기 위해선 ClassLoader를 사용하면 된다.

```java
public List<Class<?>> getAllClasses() {
    List<Class<?>> classes = new ArrayList<>();
    ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
    
    Enumeration<URL> resources = classLoader.getResources("");
    
    while (resources.hasMoreElements()) {
        findClassesInDirectory(resources.nextElement().getFile(), "", classes);
    }
    
    return classes;
}

private void findClassesInDirectory(File directory, String packageName, List<Class<?>> classes) {
    for (File file : directory.listFiles()) {
        if (file.isDirectory()) {
            findClassesInDirectory(file, packageName + (packageName.isEmpty() ? "" : ".") + file.getName(), classes);
        } else if (file.getName().endsWith(".class")) {
            String className = packageName + "." + file.getName().substring(0, file.getName().length() - 6);
            classes.add(Class.forName(className));
        }
    }
}
```

## 구현

위에서 모든 클래스를 탐색하고, 그 클래스를 프록시화한다.

```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface TimeLoggingAnnotation() {
}

public static void main(String[] args) {
    List<Class<?>> classes = getAllClasses();

    for (Class<?> clazz : classes) {
        Object instance = Proxy.newProxyInstance(
            Thread.currentThread().getContextClassLoader(),
            new Class[] {clazz},
            (proxy, method, args) -> {
                if (!method.isAnnotationPresent(TimeLoggingAnnotation.class)) {
                    return method.invoke(proxy, args);
                }

                long start = System.currentTimeMillis();
                Object result = method.invoke(proxy, args);
                long end = System.currentTimeMillis();
                System.out.printf("elapsed time : %dms%n", end - start);
                
                return result;
            }
        );
        
        diContainer.put(clazz, instance);
    }
}
```

이러한 개념을 기반으로 스프링 부트에서 어노테이션이 달린 메소드들을 처리한다.

> lombok은 약간 다른 로직을 가지는데, 이는 annotationProcessor를 통해 수정된다. (다음 시간에)