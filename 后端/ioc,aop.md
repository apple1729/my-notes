我们直接用一个真实的业务场景：用户下单接口。

假设我们要开发一个 Spring Boot 接口：

```
POST /orders
```

功能是创建订单，并且需要：

1. 调用订单 Service。
    
2. 记录接口执行日志。
    
3. 统计接口执行时间。
    

我们分别看 IoC 和 AOP 使用前后的区别。

# 一、IoC：不用 IoC 和使用 IoC

## 1. 不使用 IoC：自己创建对象

先写三个类。

OrderMapper：负责操作数据库

```
public class OrderMapper {

    public void insert() {
        System.out.println("向数据库插入订单");
    }
}
```

OrderService：负责业务逻辑

```
public class OrderService {

    private OrderMapper orderMapper = new OrderMapper();

    public void createOrder() {
        System.out.println("执行业务逻辑");
        orderMapper.insert();
    }
}
```

OrderController：接收 HTTP 请求

```
public class OrderController {

    private OrderService orderService = new OrderService();

    public void create() {
        orderService.createOrder();
    }
}
```

调用关系：OrderController自己 new OrderService()OrderService

自己 new OrderMapper()

OrderMapper

操作数据库

对象由程序员自己创建

问题来了：

假设以后你想把 `OrderMapper` 换成另一个实现。

你可能需要修改 `OrderService` 中的创建代码。

如果很多类都依赖它，就需要修改很多地方。

## 2. 使用 IoC：对象交给 Spring 管理

我们改造代码。

OrderMapper

```
@Repository
public class OrderMapper {

    public void insert() {
        System.out.println("向数据库插入订单");
    }
}
```

OrderService

```
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    public void createOrder() {
        System.out.println("执行业务逻辑");
        orderMapper.insert();
    }
}
```

OrderController

```
@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/orders")
    public String create() {
        orderService.createOrder();
        return "订单创建成功";
    }
}
```

现在发生了什么？

Spring IoC 容器

Spring 创建 OrderController

注入 OrderService

Spring 创建 OrderService

注入 OrderMapper

Spring 创建 OrderMapper

你不再需要自己 `new` 这些对象。

Spring 容器负责创建、管理它们，并把依赖注入进去。

注意： `@Autowired` 本身不是 IoC，它是 Spring 实现依赖注入的一种方式。

# 二、AOP：不用 AOP 和使用 AOP

接着还是刚才的下单接口。

现在产品经理提出一个需求：

> 每次有人调用创建订单接口，都需要记录执行日志和执行时间。

## 1. 不使用 AOP：在业务代码中手动写

```
@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/orders")
    public String create() {

        // 记录开始时间
        long start = System.currentTimeMillis();

        System.out.println("开始执行创建订单接口");

        // 真正的业务逻辑
        orderService.createOrder();

        // 计算执行时间
        long end = System.currentTimeMillis();

        System.out.println("接口耗时：" + (end - start) + "ms");

        return "订单创建成功";
    }
}
```

这样确实能实现需求。

但是，如果我们还有：

```
@PostMapping("/pay")
public String pay() {
    // 支付业务
}

@PostMapping("/refund")
public String refund() {
    // 退款业务
}
```

这三个接口都要写日志、计时。

如果有 100 个接口呢？

你就得重复写 100 次。

## 2. 使用 AOP：把公共代码抽出去

我们先让 Controller 保持干净：

```
@RestController
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/orders")
    public String create() {

        orderService.createOrder();

        return "订单创建成功";
    }
}
```

然后单独创建一个切面：

```
@Aspect
@Component
public class LogAspect {

    @Around("execution(* com.example.controller.*.*(..))")
    public Object recordTime(ProceedingJoinPoint joinPoint)
            throws Throwable {

        long start = System.currentTimeMillis();

        System.out.println("开始执行接口");

        // 执行真正的 Controller 方法
        Object result = joinPoint.proceed();

        long end = System.currentTimeMillis();

        System.out.println("接口耗时：" + (end - start) + "ms");

        return result;
    }
}
```

这里最重要的是：

```
Object result = joinPoint.proceed();
```

它的作用是：继续执行被拦截的原始方法。

也就是说，AOP 在原始方法执行前后插入了额外逻辑。

执行过程：

用户发送 POST /orders

AOP 环绕通知

记录开始时间

执行 `joinPoint.proceed()`

执行原始 Controller 方法

记录结束时间、计算耗时

返回「订单创建成功」

现在其他 Controller 接口只要符合切点表达式，也会自动执行这套日志逻辑。

不需要每个接口都写一遍。

这里使用的是 `@Around`（环绕通知），它可以控制原方法执行前后的逻辑。

实际项目中还要考虑异常处理，否则原方法抛出异常时，后面的计时代码不会正常执行；通常会用 `try-finally`。

# 三、把 IoC 和 AOP 放在一起看

这两个概念很容易混淆，但它们解决的是不同问题。

|对比|IoC|AOP|
|---|---|---|
|不使用时|自己 new 对象|每个方法手动写日志|
|使用后|Spring 管理对象|Spring 自动织入公共逻辑|
|解决的问题|对象之间的依赖管理|重复的横切逻辑|
|典型注解|`@Component`、`@Autowired`|`@Aspect`、`@Around`|

## 四、最后理解一个关键问题：AOP 和 IoC 有什么联系？

你可能会发现，我们刚才的 AOP 切面上也有：

```
@Component
public class LogAspect {
}
```

为什么？

因为 Spring 需要把这个切面也管理起来。

而 Spring AOP 通常会利用动态代理，为符合条件的 Bean 创建代理对象。

例如：

```
原始 OrderController
        |
        v
Spring AOP 创建代理对象
        |
        v
用户请求进入代理对象
        |
        v
执行日志等增强逻辑
        |
        v
调用原始 Controller 方法
```

因此，Spring AOP 与 IoC 容器紧密配合。

给你留一个小问题，检查一下是否真正理解了：

假设现在有一个秒杀接口：

```
@PostMapping("/seckill")
public Result seckill() {
    // 秒杀业务
}
```

我们希望：

1. `OrderService` 不需要自己 `new`，而由 Spring 提供。
    
2. 每次调用秒杀接口，都自动进行限流检查。
    

问题 1： 第一个需求主要使用什么？

IoC

AOP

问题 2： 第二个需求主要使用什么？

IoC

AOP

提交答案