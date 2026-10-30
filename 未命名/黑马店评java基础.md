`try`、`catch`、`finally`、代理对象、异步线程，其实分属不同的知识点。

我建议按下面这个顺序学习：

1. `try`、`catch`、`finally`：代码出错了怎么办？
2. `return` 和异常：方法什么时候结束？
3. 线程与异步：为什么代码可以同时执行？
4. 代理对象：Spring 为什么要在业务代码外面再包一层？
5. 事务与代理：为什么 `@Transactional` 有时会失效？
6. 把这些知识放回你的秒杀项目。
# 一、try、catch、finally：Java 怎么处理异常？

先想一个生活中的场景。

你去银行取钱：

1. 输入取款金额。
2. 银行检查账户。
3. 如果账户正常，就取钱。
4. 如果系统出现故障，就提示失败。
5. 无论成功还是失败，都需要结束本次操作。

Java 处理异常时，也有类似的机制。

## 1. try：尝试执行代码

例如：

```
public static void main(String[] args) {
    try {
        int a = 10;
        int b = 2;
        int result = a / b;

        System.out.println(result);
    }
}
```

这段代码的意思是：尝试执行 `try` 大括号中的代码。

不过，上面这段代码单独写出来是不完整的，因为 Java 要求这里还需要有 `catch` 或 `finally`。

完整写法如下：

```
public static void main(String[] args) {
    try {
        int a = 10;
        int b = 2;
        int result = a / b;

        System.out.println(result);
    } catch (Exception e) {
        System.out.println("程序出现异常");
    }
}
```

执行结果：

```
5
```

因为 `10 / 2 = 5`，没有发生异常，所以 `catch` 不会执行。

**记住：`try` 不是专门用来处理错误的，而是用来包住可能发生异常的代码。**

## 2. catch：发生异常后怎么办？

把除数改成 0：

```
public static void main(String[] args) {
    try {
        int a = 10;
        int b = 0;

        int result = a / b;

        System.out.println(result);
    } catch (Exception e) {
        System.out.println("计算失败");
    }
}
```

执行结果：

```
计算失败
```

为什么没有输出结果？

因为：

```
int result = a / b;
```

执行 `10 / 0` 时发生了 `ArithmeticException` 异常。

Java 会立即停止执行当前 `try` 中剩余的语句，寻找能够处理这个异常的 `catch`。

所以：

```
System.out.println(result);
```

不会执行。

执行过程如下：

```
进入 try
   |
   v
a = 10
   |
   v
b = 0
   |
   v
10 / 0
   |
   v
发生异常
   |
   v
跳过 try 中剩余代码
   |
   v
进入 catch
   |
   v
输出"计算失败"
```

注意：如果异常发生时，当前 `try` 后面还有其他代码，这些代码通常也不会继续执行。

### catch 里面的 `Exception e` 是什么？

你可能经常看到：

```
catch (Exception e) {
    System.out.println("程序异常");
}
```

先拆成两部分理解。

- `Exception`：表示要捕获的异常类型。
- `e`：接收异常对象的变量，可以通过它查看异常信息。

例如：

```
catch (Exception e) {
    e.printStackTrace();
}
```

`e.printStackTrace()` 可以打印异常的类型、信息和调用堆栈，帮助你定位代码出错的位置。

不过，实际开发中不建议所有异常都简单地捕获后忽略。应该根据业务决定是重试、返回错误、回滚事务，还是继续向外抛出异常。

---

## 3. finally：无论正常还是异常，都要执行的代码

继续看例子：

```
public static void main(String[] args) {
    try {
        int result = 10 / 0;
        System.out.println(result);
    } catch (Exception e) {
        System.out.println("发生异常");
    } finally {
        System.out.println("执行清理操作");
    }
}
```

执行结果：

```
发生异常
执行清理操作
```

即使 `try` 发生了异常，`finally` 仍然执行。

再看一个没有异常的情况：

```
try {
    System.out.println("开始执行");
} catch (Exception e) {
    System.out.println("发生异常");
} finally {
    System.out.println("执行清理操作");
}
```

执行结果：

```
开始执行
执行清理操作
```

所以：

|代码块|作用|
|---|---|
|`try`|尝试执行代码|
|`catch`|处理符合类型的异常|
|`finally`|执行清理等收尾操作|

### finally 最重要的用途是什么？

**释放资源、释放锁、关闭连接等。**

这就和你的黑马点评项目联系起来了。

你之前的代码是：

```
boolean isLock = lock.tryLock(1200);

if (!isLock) {
    return Result.fail("不允许重复下单");
}

try {
    IVoucherOrderService proxy =
            (IVoucherOrderService) AopContext.currentProxy();

    return proxy.createVoucherOrder(voucherId);
} finally {
    lock.unlock();
}
```

假设用户成功获取了锁。

接下来有两种情况：

**情况 A：创建订单成功**

```
获取锁
  ↓
创建订单成功
  ↓
return 返回结果
  ↓
执行 finally
  ↓
释放锁
```

**情况 B：创建订单时发生异常**

```
获取锁
  ↓
创建订单
  ↓
发生异常
  ↓
执行 finally
  ↓
释放锁
  ↓
异常继续向外传播
```

这就是为什么你经常会在加锁代码中看到 `try-finally`。

有一点需要注意：`finally` 能保证程序正常执行流程中的清理动作，但不能保证在进程被强制终止、机器断电等情况下仍然执行。

另外，`lock.unlock()` 自己也可能失败，所以生产代码还要确保解锁操作与锁的持有者相匹配。

---

# 二、return：为什么执行了 return，finally 还会运行？

这是一个非常容易混淆的地方。

看下面的代码：

```
public static int test() {
    try {
        System.out.println("执行 try");
        return 1;
    } finally {
        System.out.println("执行 finally");
    }
}
```

执行结果：

```
执行 try
执行 finally
```

方法最终返回：

```
1
```

为什么？

因为 `return` 不仅涉及返回结果，还涉及结束当前方法的执行流程。

但是，在正常情况下，Java 会先执行 `finally` 中的代码，再完成返回。

因此，下面这段代码：

```
try {
    return proxy.createVoucherOrder(voucherId);
} finally {
    lock.unlock();
}
```

并不是先返回结果、再释放锁。

而是：

1. 调用 `proxy.createVoucherOrder(voucherId)`。
2. 获取这个调用的返回结果。
3. 执行 `finally` 中的 `lock.unlock()`。
4. 完成当前方法的返回。

对于 Spring 事务代理来说，通常还会在代理方法返回给调用方之前完成事务提交或回滚。

所以在你的项目中，正常情况下会形成这样的顺序：

```
获取锁
  ↓
调用 Spring 代理
  ↓
执行创建订单业务
  ↓
事务提交或回滚
  ↓
代理调用返回
  ↓
执行 finally
  ↓
释放锁
```

这也解释了为什么你之前看到的注释说：

> 先获取锁，再开启事务，事务结束后，才会释放锁。

在当前这种调用结构下，这个理解基本正确。

**补充：不要在 `finally` 中随意写 `return`。** 因为它可能覆盖 `try` 或 `catch` 中原本要返回的结果，也可能掩盖异常。

---

# 三、Java 的线程：什么是同步执行和异步执行？

接下来讲线程。

这部分对于你学习 RocketMQ、秒杀系统和 Agent 开发都很重要。

## 1. 先理解什么是线程

假设你在奶茶店点了一杯奶茶。

店员收到订单后，要完成三个任务：

1. 煮茶。
2. 加配料。
3. 封杯。

我们把它简化成：

```
任务 1：煮茶
任务 2：加配料
任务 3：封杯
```

如果三个任务必须依次执行：

```
煮茶 → 加配料 → 封杯
```

这是一种顺序执行的流程。

Java 线程可以理解为程序执行任务的一条执行路径。

例如：

```
public static void main(String[] args) {
    taskA();
    taskB();
}

public static void taskA() {
    System.out.println("执行任务 A");
}

public static void taskB() {
    System.out.println("执行任务 B");
}
```

执行过程是：

```
main 线程
   |
   v
执行 taskA()
   |
   v
执行 taskB()
```

这里是顺序调用，`taskA()` 正常返回之后，才会执行 `taskB()`。

## 2. 什么是异步？

假设奶茶店接到订单后，店员先把订单交给后厨制作，然后自己继续接待下一位顾客。

```
店员：接收订单
       |
       v
后厨：开始制作奶茶
       |
       +----------------+
                        |
店员：继续接待顾客       |
                        |
后厨：制作完成 <--------+
```

这就是异步处理的直观思想：

**发起一个任务后，不必一直等待它完成，可以先继续做其他事情。**

在 Java 中，可以使用线程或线程池来执行异步任务。

例如：

```
public static void main(String[] args) {
    Thread thread = new Thread(() -> {
        System.out.println("子线程正在执行任务");
    });

    thread.start();

    System.out.println("主线程继续执行");
}
```

这里：

```
thread.start();
```

表示启动一个新线程。

而：

```
System.out.println("主线程继续执行");
```

会由原来的主线程执行。

一种可能的输出是：

```
主线程继续执行
子线程正在执行任务
```

也可能先输出子线程的内容。

因为两个线程的执行顺序由调度情况决定，我们不能仅凭代码的位置断定哪个先输出。

### 一个非常重要的区别

```
thread.start();
```

与：

```
thread.run();
```

不一样。

- `start()`：启动新线程，由新线程执行任务。
- `run()`：普通的方法调用，不会因为调用了它就自动创建新线程。

这是 Java 多线程的基础。

---

## 3. 异步和多线程是一回事吗？

不完全是一回事。

- **多线程**：多个线程参与任务执行。
- **异步**：调用方发起任务后，不必等待任务完成才继续执行。

多线程是实现异步的一种常见手段，但异步不一定意味着创建一个新线程。例如，基于事件循环的异步系统也可以不为每个任务创建独立线程。

你现在先掌握最常见的情况就行：Java 通过线程池执行异步任务。

---

# 四、为什么需要线程池？

如果每次处理任务都创建一个新线程，会有什么问题？

例如：

```
for (int i = 0; i < 10000; i++) {
    new Thread(() -> {
        // 执行任务
    }).start();
}
```

这可能创建大量线程。

线程本身需要占用内存，线程之间切换也有开销。如果线程数量过多，系统可能出现性能问题。

因此，Java 通常会使用线程池。

线程池可以理解为：**提前准备或按需创建一批可复用的工作线程，让任务由这些线程执行。**

例如：

```
ExecutorService pool = Executors.newFixedThreadPool(3);

pool.submit(() -> {
    System.out.println("任务 A");
});

pool.submit(() -> {
    System.out.println("任务 B");
});

pool.submit(() -> {
    System.out.println("任务 C");
});

pool.shutdown();
```

这里创建了一个固定大小为 3 的线程池。

你可以把它想象成：

```
                线程池
           /      |      \
          v       v       v
       线程 1   线程 2   线程 3
          |       |       |
        任务 A   任务 B   任务 C
```

如果有更多任务到达，而三个线程都在忙，那么多出来的任务需要等待执行，具体取决于线程池的队列和拒绝策略。

在实际后端开发中，还需要认真配置线程池的大小、队列容量、拒绝策略等参数，不能简单地认为线程越多越好。

---

# 五、结合你的黑马点评：为什么要异步发放优惠券？

你之前做的黑马点评项目中，使用了 RocketMQ 处理秒杀优惠券发放。

我们对比两种方式。

## 1. 同步处理

假设用户抢到优惠券后，系统必须立即完成订单创建，才能返回结果。

```
用户请求
   |
   v
Java 服务
   |
   v
检查库存
   |
   v
扣减库存
   |
   v
创建订单
   |
   v
返回结果给用户
```

假设每一步的耗时分别为：

|操作|耗时|
|---|---|
|检查库存|10ms|
|扣减库存|20ms|
|创建订单|80ms|
|总耗时|110ms|

这是一个简化的假设。

在这个例子中，用户需要等待这些操作完成，才能收到最终结果。

## 2. 异步处理

现在改成通过 RocketMQ 异步创建订单。

```
用户请求
   |
   v
Java 服务
   |
   v
检查库存
   |
   v
扣减库存
   |
   v
发送消息到 RocketMQ
   |
   v
返回受理结果给用户


RocketMQ
   |
   v
消费者接收消息
   |
   v
创建优惠券订单
```

这样，订单创建任务由消费者处理，生产者不必一直等待消费者完成整个流程。

但要注意：**异步不是让任务凭空消失，而是把任务交给其他执行者处理。**

因此，你还需要考虑：

- 消息有没有成功发送。
- 消费者有没有成功处理。
- 消费失败后如何重试。
- 重复消费时如何避免重复发放。
- 库存已经扣减但订单没有创建时如何处理。

这些就是你之前学习的消息重试、幂等和异常补偿问题。

另外，异步方案的收益取决于具体实现。如果生产者在返回之前还要同步等待消息确认，仍然会有一定等待时间。

---

# 六、代理对象：为什么 Spring 要在业务对象外面包一层？

现在进入你最困惑的知识点。

先看一个普通 Java 对象。

```
public class UserService {

    public void saveUser() {
        System.out.println("保存用户");
    }
}
```

使用它很简单：

```
UserService service = new UserService();

service.saveUser();
```

执行过程：

```
调用 saveUser()
      |
      v
执行保存用户的业务代码
```

但是，实际项目中的业务方法通常还需要完成一些额外工作，例如：

- 开启事务。
- 检查权限。
- 记录日志。
- 统计执行时间。

如果每个业务方法都手动写一遍这些代码，会非常麻烦。

Spring AOP 和代理机制可以帮助我们把这些通用逻辑与业务逻辑分开。

## 1. 先看一个简单的代理思想

假设你的业务方法是：

```
public void saveUser() {
    System.out.println("保存用户");
}
```

现在希望每次保存用户之前，都记录一条日志。

可以把调用流程理解成：

```
调用方
   |
   v
代理对象
   |
   +---- 记录日志
   |
   v
真实业务对象
   |
   v
执行 saveUser()
```

代理对象可以在调用真实业务方法前后执行额外逻辑。

例如：

```
代理对象开始工作
       |
       v
打印"开始保存"
       |
       v
调用真实业务方法
       |
       v
打印"保存结束"
```

这就是代理的核心思想。

不过，**代理对象并不是 Spring 才有的概念**。Java 本身就支持代理机制，Spring 在此基础上实现了 AOP、声明式事务等功能。

## 2. Spring 事务为什么需要代理？

回到你的代码：

```
@Transactional
public Result createVoucherOrder(Long voucherId) {
    // 扣减库存
    // 创建订单
    return Result.ok();
}
```

你希望 Spring 自动完成：

```
开始事务
   |
   v
执行创建订单
   |
   +---- 成功：提交事务
   |
   +---- 异常：按规则回滚事务
```

Spring 可以通过代理对象，在业务方法执行前后加入事务管理逻辑。

因此，外部调用通常是：

```
调用方
   |
   v
Spring 代理对象
   |
   v
事务管理逻辑
   |
   v
真实业务方法
```

这就是为什么你之前的代码要写：

```
IVoucherOrderService proxy =
        (IVoucherOrderService) AopContext.currentProxy();

return proxy.createVoucherOrder(voucherId);
```

它的意图是重新通过当前 Spring 代理调用创建订单方法，让事务拦截器有机会生效。

但需要满足一些前提：

1. `createVoucherOrder()` 确实配置了有效的事务。
2. 当前对象由 Spring 管理。
3. 调用通过适当的代理方式完成。
4. `AopContext.currentProxy()` 所需的代理暴露配置已经开启。

否则，这种写法可能出现异常，或者事务无法按预期生效。

---

# 七、事务和代理对象有什么关系？

我们用一个简单的例子理解。

假设你的订单系统需要执行两条 SQL：

```
UPDATE tb_seckill_voucher
SET stock = stock - 1
WHERE voucher_id = 1001
  AND stock > 0;
```

然后：

```
INSERT INTO tb_voucher_order
(user_id, voucher_id)
VALUES (1001, 1001);
```

第一条 SQL 扣减库存，第二条 SQL 创建订单。

如果两条操作属于同一个数据库事务，并且第二条 SQL 失败，那么事务可以回滚第一条 SQL。

理想的执行过程是：

```
开启事务
   |
   v
扣减库存
   |
   v
创建订单
   |
   v
全部成功？
   |
   +---- 是：提交事务
   |
   +---- 否：回滚事务
```

但要注意：**Spring 的数据库事务通常只能直接管理同一个事务管理器所控制的数据库资源，不会自动把 Redis、RocketMQ 和 MySQL 变成一个统一事务。**

例如：

```
Redis 扣减库存
       |
       v
MySQL 创建订单
```

如果 Redis 操作成功，而 MySQL 操作失败，MySQL 的事务回滚不会自动把 Redis 的库存恢复。

这也是为什么秒杀系统需要考虑异常补偿。

你可以先记住：

- 事务：保证一组受事务管理的数据库操作按照规则提交或回滚。
- 代理：让 Spring 有机会在方法调用前后加入事务管理逻辑。
- 异常补偿：处理普通数据库事务无法自动覆盖的跨系统状态问题。

---

# 八、把这些知识放到同一张图里

现在把你学过的概念全部串起来。

假设用户抢购秒杀券：

```
                 用户请求
                    |
                    v
          seckillVoucher()
                    |
                    v
               获取分布式锁
                    |
                    v
              进入 try 代码
                    |
                    v
          获取 Spring 代理对象
                    |
                    v
       proxy.createVoucherOrder()
                    |
                    v
            Spring 事务代理
                    |
                    v
              开启数据库事务
                    |
                    v
           扣减库存 + 创建订单
                    |
           +--------+--------+
           |                 |
           v                 v
         成功               异常
           |                 |
           v                 v
         提交事务           回滚事务
           |                 |
           +--------+--------+
                    |
                    v
               调用返回
                    |
                    v
             finally 执行
                    |
                    v
                 释放锁
```

这张图有一个前提：库存扣减和订单创建属于同一个受 Spring 事务管理的数据库事务。若其中涉及 Redis 操作，Redis 不会自动随 MySQL 事务一起回滚。

---

# 九、你现在应该掌握到什么程度？

考虑到你正在准备后端和 Agent 开发岗位，我建议你先掌握以下内容。

|知识点|现阶段需要掌握什么|
|---|---|
|`try`|可能发生异常的代码放在哪里|
|`catch`|发生指定类型的异常后如何处理|
|`finally`|为什么释放锁、关闭资源通常放在这里|
|`return`|方法什么时候返回，为什么 `finally` 仍然会执行|
|线程|Java 程序中的执行路径是什么|
|异步|为什么任务可以交给其他执行者处理|
|线程池|为什么需要复用线程，而不是无限创建线程|
|代理对象|为什么 Spring 能在业务方法外面增加事务等逻辑|
|`@Transactional`|为什么事务与代理调用方式有关|
|消息重试|为什么消息处理失败后可能再次执行|
|幂等|为什么同一业务执行多次不能产生重复结果|

暂时不用一口气深入所有底层实现。

尤其是线程池、AOP 和事务，你可以先理解运行流程，再通过代码实验逐步加深。

## 十、给你三个小问题，检查一下是否理解

你可以直接回复选项，我来帮你分析。

**问题 1：**

```
try {
    System.out.println("A");
    return;
} finally {
    System.out.println("B");
}
```

最终会输出什么？

A. 只输出 A  
B. 只输出 B  
C. 先输出 A，再输出 B  
D. 什么都不输出

**问题 2：**

两个 Spring Boot 实例都运行下面的代码：

```
synchronized (lock) {
    // 扣减库存
}
```

这两个实例中的 `synchronized` 能直接互相协调吗？
B. 不能，因为它们是两个独立的 Java 进程

**问题 3：**

为什么下面的调用可能导致事务不生效？

```
this.createVoucherOrder(voucherId);
```

A. 因为 `this` 调用太慢  
B. 因为同类内部调用可能绕过 Spring 事务代理  
C. 因为 Java 不允许一个方法调用另一个方法
b
你先回答这三个问题。如果这三个问题都能理解，你就已经掌握了这几个知识点之间最重要的联系。