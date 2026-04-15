
---  
title: Java 代理模式  
tags:  
    - Java    - 动态代理data: 2026 年 4 月 13 日 16:41:34  
---  
  
## 静态代理  
  
Java 中的静态代理模式通常有两种实现。  
  
1. **通过继承重写实现：**  
  
- 代理类继承目标类  
  
```java  
public class Proxy extends Target {  
    @Override    public void method() {        // 代理逻辑        super.method(); // 调用目标类的方法    }}  
```  
  
2. **通过关联实现：**  
  
- 代理类持有目标类的引用，通过调用目标类的方法来实现代理逻辑  
  
```java  
public class Proxy {  
    private Target target;  
    public Proxy(Target target) {        this.target = target;    }  
    public void method() {        // 代理逻辑        target.method(); // 调用目标类的方法    }}  
```  
  
>如果程序运行前就在Java代码中定义好代理类(Proxy)，那么这种代理方式就叫做静态代理；若代理类在程序运行时创建就叫做动态代理  
  
如果为特定类的特定方法生成固定的代理，当然使用静态代理就能很好满足需求。缺点：在大量不同类的不同方法生成代理，静态代理需要编写非常多且冗余的代理类。  
  
## JDK 的动态代理  
  
**动态代理：** 代理类不需要手动编写，由 JDK 在运行过程中进行自动生产代理类对象。在不改动原有类的逻辑基础上，织入额外的业务逻辑。  
  
典型场景：  
- AOP  
- 事务管理  
- RPC 远程调用封装  
- 权限控制  
- 日志记录  
- 延迟加载  
- 缓存代理  
    
程序在运行时生成一个“假的实现类”或“假的子类”，这个对象长得像原对象，对外暴露相同的方法，但内部会拦截方法调用。  
  
```aiignore  
调用代理对象的方法  
-> 进入统一拦截逻辑  
-> 执行前置处理  
-> 调用真实对象的方法  
-> 执行后置处理  
-> 返回结果  
```  
  
Example:  
  
```java  
interface Flyable {  
    void fly();}  
  
class Bird implements Flyable {  
    public void fly() {        System.out.println("I am flying");    }}  
  
class BirdProxy implements InvocationHandler {  
    private final  Flyable flyable;  
    BirdProxy(Flyable flyable) {        this.flyable = flyable;    }  
    @Override    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {        System.out.println("Before method call");        Object result = method.invoke(flyable, args);        System.out.println("After method call");        return result;    }}  
  
public class JDKDynamicProxy {  
    public static void main(String[] args) {        ClassLoader classLoader = JDKDynamicProxy.class.getClassLoader();        Class<?>[] interfaces = Bird.class.getInterfaces();  
        Bird bird = new Bird();  
        Flyable flyable = (Flyable) Proxy.newProxyInstance(classLoader, interfaces, new BirdProxy(bird));        flyable.fly();        bird.fly();    }}  
```  
  
  
## CGLIB  
  
**Code Generation Library**: 基于字节码生成的动态代理库, 通过继承实现，在代码的运行过程中动态的生成子类，在调用的方法前后织入增强逻辑。