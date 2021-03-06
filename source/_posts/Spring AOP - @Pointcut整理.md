### AOP 是什么
AOP 即面向切面编程，简单地说就是将代码中重复的部分抽取出来，在需要执行的时候使用动态代理技术，在不修改源码的基础上对方法进行增强。

Spring 根据类是否实现接口来判断动态代理方式，如果实现接口会使用 JDK 的动态代理，核心是 InvocationHandler 接口和 Proxy 类，如果没有实现接口会使用 CGLib 动态代理，CGLib 是在运行时动态生成某个类的子类，如果某个类被标记为 final，不能使用 CGLib 。

JDK 动态代理主要通过重组字节码实现，首先获得被代理对象的引用和所有接口，生成新的类必须实现被代理类的所有接口，动态生成Java 代码后编译新生成的 .class 文件并重新加载到 JVM 运行。JDK 代理直接写 Class 字节码，CGLib 是采用 ASM 框架写字节码，生成代理类的效率低。但是 CGLib 调用方法的效率高，因为 JDK 使用反射调用方法，CGLib 使用 FastClass 机制为代理类和被代理类各生成一个类，这个类会为代理类或被代理类的方法生成一个 index，这个 index 可以作为参数直接定位要调用的方法。

### 常用场景
权限认证、自动缓存、错误处理、日志、调试和事务等。

### AOP相关术语
- Aspect：切面，一个关注点的模块化，这个关注点可能会横切多个对象。
- Joinpoint：连接点，程序执行过程中的某一行为，即业务层中的所有方法。。
- Advice：通知，指切面对于某个连接点所产生的动作，**包括前置通知、后置通知、返回后通知、异常通知和环绕通知。**
- Pointcut：切入点，指被拦截的连接点，**切入点一定是连接点，但连接点不一定是切入点。**
- Proxy：代理，Spring AOP 中有 JDK 动态代理和 CGLib 代理，**目标对象实现了接口时采用 JDK 动态代理，反之采用 CGLib 代理。**
- Target：代理的目标对象，指一个或多个切面所通知的对象。
- Weaving ：织入，指把增强应用到目标对象来创建代理对象的过程。

### 相关注解

- @Aspect：声明被注解的类是一个切面 Bean。
- @Before：前置通知，指在某个连接点之前执行的通知。
- @After：后置通知，指某个连接点退出时执行的通知（不论正常返回还是异常退出）。
- @AfterReturning：返回后通知，指某连接点正常完成之后执行的通知，返回值使用returning属性接收。
- @AfterThrowing：异常通知，指方法抛出异常导致退出时执行的通知，和@AfterReturning只会有一个执行，异常使用throwing属性接收。







































