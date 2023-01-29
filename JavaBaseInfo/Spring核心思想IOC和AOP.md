# Spring核心思想IOC和AOP

## Spring IOC

* IOC(Inversion of Control)控制反转。它只是一个技术思想，不是一个技术实现。在传统的Java开发中，bean的创建和管理一直是一个让人头疼的问题。比如类A依赖于类B，通常会在类A中new一个B的对象。在Ioc思想下，可以不用自己去new对象，而是由Ioc容器去帮助我们实例化对象并且管理它。我们需要使用某个对象，去Ioc容器中要即可。在 Spring 中， IoC 容器是 Spring 用来实现 IoC 的载体， IoC 容器实际上就是个 Map（key，value），Map 中存放的是各种对象。

* 控制：指的是对象的创建（实例化、管理）的权利。
* 反转：控制权又交给外部环境了（Spring框架、Ioc容器）。

**IOC和DI的区别**

IOC和DI描述的是同一件事情（对象实例化及依赖关系维护这件事情），只不过角度不一样罢了。IOC是站在对象的角度，对象实例化及其管理的权利交给了（反转给了）容器；DI是站在容器的角度，容器会把对象依赖的其他对象注入（送进去），比如A对象实例化过程中因为声明了一个B类型的属性，那么就需要容器把B对象注入给A。

## Spring AOP

* 要说AOP，得先从OOP说起，OOP三大特征：封装，继承，多态。OOP是一种垂直纵向的继承体系，它可以解决大多数的代码重复问题，但是有一些情况是处理不了的，比如在一个顶级父类Animal中的多个方法中相同位置出现了**重复代码**，OOP就解决不了。比如需要对接口的性能监控，OOP的思想是需要在每处要监控的地方编写重复代码。
* 横切逻辑（简单来说，就是在业务代码之前，或者之后，或者环绕他们执行）

* 横切逻辑代码：在多个纵向（顺序）流程中出现的相同子流程代码，我们称之为横切逻辑代码。横切逻辑代码的使用场景很有限：一般是事务控制、权限校验和日志。

* 横切逻辑代码存在什么问题： 横切代码重复问题，横切逻辑代码和业务代码混杂在⼀起且代码臃肿，维护不⽅便。AOP独辟蹊径提出横向抽取机制，将横切逻辑代码和业务逻辑代码分离开来。

* 面向切面编程：「切」：指的是横切逻辑，原有业务逻辑代码我们不能动，只能操作横切逻辑代码，所以⾯向横切逻辑。「⾯」：横切逻辑代码往往要影响的是很多个⽅法，每⼀个⽅法都如同⼀个点，多个点构成⾯，有⼀个⾯的概念在⾥⾯。
  
* 每个Bean都会被JDK或者Cglib代理，取决于是否有接口，每个Bean会有多个“方法拦截器”。注意：拦截器分为两层，外层由Spring内核控制流程，内层拦截器是用户设置，也就是AOP。当代理方法被调用时，先经过外层拦截器，外层拦截器根据方法的各种信息判断该方法应该执行哪些“内层拦截器”。

* 代理的创建（按步骤）：

首先，需要创建代理工厂，代理工厂需要 3 个重要的信息：拦截器数组，目标对象接口数组，目标对象；

创建代理工厂时，默认会在拦截器数组尾部再增加一个默认拦截器 —— 用于最终的调用目标方法；

当调用 getProxy 方法的时候，会根据接口数量大于 0 条件返回一个代理对象（JDK or Cglib）；

注意：创建代理对象时，同时会创建一个外层拦截器，这个拦截器就是 Spring 内核的拦截器，用于控制整个 AOP 的流程。

* 代理的调用

当对代理对象进行调用时，就会触发外层拦截器。

外层拦截器根据代理配置信息，创建内层拦截器链。创建的过程中，会根据表达式判断当前拦截是否匹配这个拦截器，而这个拦截器链设计模式就是职责链模式。

当整个链条执行到最后时，就会触发创建代理时那个尾部的默认拦截器，从而调用目标方法，最后返回。

## AOP主要的实现技术

* AOP主要的实现技术有Spring AOP和AspectJ：AspectJ的底层技术是静态代理，即用一种AspectJ支持的特定语言编写切面，通过一个命令来编译，生成一个新的代理类，该代理类增强了业务类，这是在编译时增强，相对于运行时增强而言，编译时增强的性能更好。

* Spring AOP采用的是动态代理，在运行期间对业务方法进行增强，所以不会生成新类，Spring AOP提供了对JDK动态代理的支持以及CGLib的支持。

## 具体实现

* 如下代码块，现在需要统计这个方法执行的耗时情况
```java
public void runTask() {
    doSomething();
}
* 一次性的解决肯定非常简单，直接添加一个时间记录即可，如下代码块
```java
public void runTask() {
    long start = System.currentTimeMillis();
    doSomething();
    System.out.println(System.currentTimeMillis() - start);
}

* 但是这种方法弊端也是非常明显的。直接把非业务功能和业务功能耦合在一起，如果下一次需要把时间记录去掉，换成统计次数调用，那么所有的地方都得改动。

## AOP的几个概念

* Jointpoint（连接点）：具体的切面点点抽象概念，可以是在字段、方法上，Spring中具体表现形式是PointCut（切入点），仅作用在方法上。
* Advice（通知）: 在连接点进行的具体操作，如何进行增强处理的，分为前置、后置、异常、最终、环绕五种情况。
* 目标对象：被AOP框架进行增强处理的对象，也被称为被增强的对象。
* AOP代理：AOP框架创建的对象，简单的说，代理就是对目标对象的加强。Spring中的AOP代理可以是JDK动态代理，也可以是CGLIB代理。
* Weaving（织入）：将增强处理添加到目标对象中，创建一个被增强的对象的过程

**总之，在目标对象（target object）的某些方法（jointpoint）添加不同种类的操作（通知、增强操处理），最后通过某些方法（weaving、织入操作）实现一个新的代理目标对象。**

* 动态代理（Dynamic Proxy）是采用Java的反射技术，在运行时按照某一接口要求创建一个包装了目标对象的新的代理对象，并通过代理对象实现对目标对象的控制操作。
```java
public class HelloInvocationHandle implements InvocationHandler {
    private Object object;
    public HelloInvocationHandle(Object o) {
        this.object = o;
    }
 
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("method: " + method.getName() + " is invoked");
        System.out.println("proxy: " + proxy.getClass().getName());
        Object result = method.invoke(object, args);
        // 反射方法调用
        return result;
    }
}

// HelloWorld 是一个接口，此处没有贴出来
Class<?> proxyClass = Proxy.getProxyClass(HelloWorld.class.getClassLoader(), HelloWorld.class);
Constructor cc = proxyClass.getConstructor(InvocationHandler.class);
InvocationHandler ihs = new HelloInvocationHandle(new HelloWorldImpl());
HelloWorld helloWorld = (HelloWorld) cc.newInstance(ihs);




