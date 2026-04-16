# Java 代理模式

## 静态代理

Java 中的静态代理模式通常有两种实现：

**1. 通过继承重写实现** — 代理类继承目标类

```java
public class Proxy extends Target {
    @Override
    public void method() {
        // 代理逻辑
        super.method();
    }
}
```

**2. 通过关联实现** — 代理类持有目标类的引用，通过调用目标类的方法来实现代理逻辑

```java
public class Proxy {
    private Target target;

    public Proxy(Target target) {
        this.target = target;
    }

    public void method() {
        // 代理逻辑
        target.method();
    }
}
```

> 如果程序运行前就在 Java 代码中定义好代理类（Proxy），这种方式叫做**静态代理**；若代理类在程序运行时创建则叫做**动态代理**。

静态代理适合为特定类的特定方法生成固定的代理，但当需要为大量不同类的不同方法生成代理时，需要编写非常多且冗余的代理类。

---

## JDK 动态代理

代理类不需要手动编写，由 JDK 在运行过程中自动生成代理类对象。在不改动原有类逻辑的基础上，织入额外的业务逻辑。

典型场景：AOP、事务管理、RPC 远程调用封装、权限控制、日志记录、延迟加载、缓存代理

程序在运行时生成一个"假实现类"或"假的子类"，对外暴露相同的方法，但内部会拦截方法调用：

```
调用代理对象的方法
  -> 进入统一拦截逻辑
  -> 执行前置处理
  -> 调用真实对象的方法
  -> 执行后置处理
  -> 返回结果
```

**示例：**

```java
interface Flyable {
    void fly();
}

class Bird implements Flyable {
    public void fly() {
        System.out.println("I am flying");
    }
}

class BirdProxy implements InvocationHandler {
    private final Flyable flyable;

    BirdProxy(Flyable flyable) {
        this.flyable = flyable;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before method call");
        Object result = method.invoke(flyable, args);
        System.out.println("After method call");
        return result;
    }
}

public class JDKDynamicProxy {
    public static void main(String[] args) {
        ClassLoader classLoader = JDKDynamicProxy.class.getClassLoader();
        Class<?>[] interfaces = Bird.class.getInterfaces();

        Bird bird = new Bird();

        Flyable flyable = (Flyable) Proxy.newProxyInstance(classLoader, interfaces, new BirdProxy(bird));
        flyable.fly();
        bird.fly();
    }
}
```

---

## ByteBuddy

> Byte Buddy 是一个运行时生成和修改 Java 类的库。它不只像 JDK Proxy 那样只能代理接口，也可以基于已有类生成子类，并把方法调用拦截后转发到自定义逻辑。
>
> 相比 **CGLIB**：CGLIB 已不再活跃维护，不支持 Java 17。

相比 JDK Proxy，ByteBuddy 不仅可以代理接口，还可以基于已有类生成子类、做更细粒度的方法匹配、参数绑定和方法增强。

**安装依赖：**

```xml
<dependency>
    <groupId>net.bytebuddy</groupId>
    <artifactId>byte-buddy</artifactId>
    <version>1.18.8</version>
    <scope>compile</scope>
</dependency>
```

**定义 OrderService 与方法拦截器 LogInterceptor：**

```java
public class OrderService {
    public Object query(Long id) {
        return "查询到订单：" + id;
    }

    public final void delete(Long id) {
        System.out.println("已删除订单：" + id);
    }
}

public class LogInterceptor {
    public static Object intercept(
            @Origin Method method,
            @AllArguments Object[] args,
            @SuperCall Callable<?> zuper) throws Exception {
        System.out.println("before: " + method.getName() + ", args=" + Arrays.toString(args));
        Object result = zuper.call();
        System.out.println("after: result=" + result);
        return result;
    }
}
```

**生成代理类：**

```java
public class Main {
    public static void main(String[] args) throws Exception {
        Class<? extends OrderService> proxyClass = new ByteBuddy()
                .subclass(OrderService.class)
                .method(named("query"))
                .intercept(MethodDelegation.to(LogInterceptor.class))
                .make()
                .load(Main.class.getClassLoader())
                .getLoaded();

        OrderService service = proxyClass.getDeclaredConstructor().newInstance();
        System.out.println(service.query(1001L));
    }
}
```

**代理生成工厂类：**

```java
public class ProxyFactory {
    public static class GeneralInterceptor {
        @RuntimeType
        public static Object intercept(
                @Origin Method method,
                @AllArguments Object[] args,
                @SuperCall(nullIfImpossible = true) Callable<?> zuper) throws Exception {
            System.out.println("before " + method.getName());

            Object result;
            if (zuper != null) {
                result = zuper.call();
            } else {
                result = defaultValue(method.getReturnType());
            }

            System.out.println("after " + method.getName() + ", result: " + result);
            return result;
        }

        private static Object defaultValue(Class<?> type) {
            if (type == void.class)    return null;
            if (type == boolean.class) return false;
            if (type == byte.class)    return (byte) 0;
            if (type == short.class)   return (short) 0;
            if (type == int.class)     return 0;
            if (type == long.class)    return 0L;
            if (type == float.class)   return 0f;
            if (type == double.class)  return 0d;
            if (type == char.class)    return '\0';
            return null;
        }
    }

    public static <T> T createProxy(Class<T> type) throws Exception {
        Class<? extends T> proxyClass = new ByteBuddy()
                .subclass(type)
                .method(not(isDeclaredBy(Object.class)))
                .intercept(MethodDelegation.to(GeneralInterceptor.class))
                .make()
                .load(type.getClassLoader())
                .getLoaded();

        return proxyClass.getDeclaredConstructor().newInstance();
    }
}
```

---

参考文章：

- https://www.pkslow.com/docs/zh/jdk-cglib-proxy/
- https://www.cnblogs.com/bytesfly/p/dynamic-proxy-in-java.html
