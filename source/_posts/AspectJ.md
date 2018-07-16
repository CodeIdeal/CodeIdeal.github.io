title: AOP框架AspectJ介绍及实战
date: 2018-07-16 20:00:40
---
今天给大家介绍一个AOP框架，[AspectJ](https://www.eclipse.org/aspectj/)。
## AOP简介
相信大家都很熟悉OOP(面向对象编程)，AOP即是与之相对的另一种编程思想，即面向切面编程。AOP一般是使用了预编译生成代理类，或者是运行时动态代理的技术，实现了在尽量保持原有业务代码不变的情况下增加一些业务无关的功能或逻辑。例如；日志记录，性能统计，安全控制，事务处理，异常处理等。

## 与OOP的关系
按我们OOP单一职责原则思想进行设计，一般都会将这些模块封装成一个单一的类(比如Log)。而这种封装，虽然解决了代码的重用性，但还是没能解决这些模块的代码调用的重复分散且与业务代码耦合的现状。

最原始逻辑 和 OOP封装后的逻辑::
![](https://i.ooxx.ooo/2018/07/11/1c2ff4e7570c93e931ee3d63a2b05432.png)

AOP与OOP并不是对立的，我们并不是为了代替OOP而提出AOP，AOP是OOP的延续和补充。当我们需要为分散的对象引入公共行为的时候，OOP则显得很无力，而AOP正是为解决这种问题而生的。

AOP的目标就是把这些功能集中并隔离起来，放到一个统一的地方来控制和管理。如果说，OOP如果是把问题划分到单个模块的话，那么AOP就是把涉及到众多模块的某一类问题收集起来进行统一管理。

![](http://data.christianschenk.org/aop-with-aspectj/comparison.png)

## AOP初窥
AOP在我们Android开发中并不多见，但在后端开发中却是有着重要的地位，就拿著名的Spring框架来说，其背后核心的两个点就是IoC(控制反转,依赖注入)和AOP。

我们来看个事务操作的例子。

下面这个是原生Android操作SQLite数据库时，开启事务的一般流程：
```java
public  void  batchSave(Data data) {
    MySQLiteOpenHelper dbOpenHelper = new MySQLiteOpenHelper(context);
    SQLiteDatabase db = dbOpenHelper.getWritableDatabase();
    //开启事务
    db.beginTransaction();
    try {
        //数据库操作1
        db.execSQL("SQL语句", data);
        //数据库操作2
        db.execSQL("SQL语句2",data);
        //设置事务标志为成功，当结束事务时就会提交事务
        db.setTransactionSuccessful();
    } catch（Exception e）{
        throw(e);
    } finally {
        //结束事务
        db.endTransaction();
    }
}
```

在Spring中开启数据库的事务是怎样的呢：
```java
...
@AutoWired
DataMapper mapper;
...

@Transaction
public  void  batchSave(Data data) {
    mapper.insert(data);
    mapper.update(data);
}
```
上面的例子我们看到，Spring把所有业务无关的代码全部都剔除了出去，只增加了两个注解。而这两个注解也正是Spring框架IoC和AOP核心两点的体现。必要的依赖通过了`@AutoWired`注解进行注入；事务开启、事务提交、异常处理和事务回滚等流程性的代码换成了一个 `@Transaction`注解。这里我们先不讲`@Transaction`的原理，在我们看完这篇AspectJ的介绍后你自然也就就知道Spring是怎么实现的了。

## AspectJ语法基础
在介绍AspectJ之前我们先看下他的一些概念和术语：

### Join Point
Join Point: 连接点，程序中可切入的点，例如方法调用时、读取某个变量时。

| Join Point | 说明 |
| --- | --- |
| Method call | 方法被调用 |
| Method execution | 方法执行 |
| Constructor call | 构造函数被调用 |
| Constructor execution | 构造函数执行 |
| Field get | 读取属性 |
| Field set | 写入属性 |
| Pre-initialization | 与构造函数有关，很少用到 |
| Initialization | 与构造函数有关，很少用到 |
| Static initialization | static 块初始化 |
| Handler | 异常处理 |
| Advice execution | 所有 Advice 执行 |

### PointCut
PointCut: 切点，代码注入的位置，其实就是有条件限定的特定Join Point，例如只在特定方法中注入代码。

#### 连接点PointCut
与连接点相对应的切点的定义语法如下：

| Join Point | Pointcuts Pattern |
| --- | --- |
| Method call | call(MethodPattern) |
| Method execution | execution(MethodPattern) |
| Constructor call | call(ConstructorPattern) |
| Constructor execution | execution(ConstructorPattern) |
| Field get | get(FieldPattern) |
| Field set | set(FieldPattern) |
| Pre-initialization | initialization(ConstructorPattern) |
| Initialization | preinitialization(ConstructorPattern) |
| Static initialization | staticinitialization(TypePattern) |
| Handler | handler(TypePattern) |
| Advice execution | adviceexcution() |

#### 匹配规则
上面的xxxPattern是这些java元素的匹配规则，具体的匹配规则如下：

| Pattern | 规则 |
| --- | --- |
| MethodPattern | [!] [@Annotation] [public,protected,private] [static] [final] 返回值类型 [类名.]方法名(参数类型列表) [throws 异常类型] |
| ConstructorPattern | [!] [@Annotation] [public,protected,private] [final] [类名.]new(参数类型列表) [throws 异常类型] |
| FieldPattern | [!] [@Annotation] [public,protected,private] [static] [final] 属性类型 [类名.]属性名 |
| TypePattern | [!] [@Annotation] [public,protected,private] [static] [final] [包名.]类名 |

另外Pattern涉及到的类型规则还可以使用 `!`、`*`、`..`、`+`等。
- `!` 表示取反，
- `*` 匹配除 . 外的所有字符串，单独使用时表示匹配任意单个类型，
- `..` 匹配任意字符串， 单独使用时表示匹配任意数量的任意类型，
- `+` 匹配其自身及子类。

#### 额外的匹配语法

另外，在针对连接点的切点语法之外，还有一些额外的语法来筛选连接点。

| Pointcuts syntax | 说明 |
| --- | --- |
| within(指定类型)    | 用于匹配指定类型内的方法执行 |
| this(当前类型)      | 用于匹配当前AOP代理对象类型的执行方法 |
| target(目标类型)    | 用于匹配当前目标对象类型的执行方法 |
| args(参数类型列表)      | 用于匹配当前执行的方法传入的参数为指定类型的执行方法 |
| @within(注解类型) | 用于匹配所以持有指定注解类型内的方法 |
| @target(注解类型) | 用于匹配当前目标对象类型的执行方法，其中目标对象持有指定的注解 |
| @args(参数注解类型列表)   | 用于匹配当前执行的方法传入的参数持有指定注解的执行 |
| @annotation(注解类型) | 用于匹配当前执行方法持有指定注解的方法 |
| *以上"类型"皆为 全限定名称 * |

`args()`、`this()`、`target()`、`@args()`、`@within()`、`@target()`和`@annotation()`这7个函数**除了可以指定类名外，还可以指定参数名用于将目标对象连接点上的方法入参绑定到Advice的方法中**。

### Advice
Advice: 在切入点注入的代码，一般有 before、after、around 三种类型。

| Advice | 执行时机 |
| --- | --- |
| @Before | 在执行 Join Point 之前 |
| @After | 在执行 Join Point 之后，包括正常的 return 和 throw 异常 |
| @AfterReturning | Join Point 为方法调用且正常 return 时，不指定返回类型时匹配所有类型 |
| @AfterThrowing | Join Point 为方法调用且抛出异常时，不指定异常类型时匹配所有类型 |
| @Around | 替代 Join Point 的代码，如果要执行原来代码的话，要使用 ProceedingJoinPoint.proceed() |

### 一个切面例子

Pointcuts 可以定义在由`@Aspect`注解注释的class中定义，由 org.aspectj.lang.annotation.Pointcut 注解修饰的方法声明，方法返回值除 Around 的 Advice 外只能是 void。@Pointcut 修饰的方法只能由空的方法实现而且不能有 throws 语句，方法的参数和 pointcut 中绑定的参数相对应。

eg:
```
@Aspect
public class AspectJ{
    @Before("execution(void android.view.View.OnClickListener+.onClick(..))  && args(view)")
    public void beforeOnClick(JoinPoint joinPoint, View view) {
        Log.d(TAG,"点击了View:"+view.toString());
    }
}
```
这个切面匹配了所有OnClickListener接口及其子类(也就是我们自己的实现类)的onClick方法的执行前的时刻，并在onClick方法执行前打印日志。其中args(view)并不是在匹配参数，而是在将onClick(View view)方法中的参数`view`绑定到Before Advice方法中的`view`参数上。

### 其他概念
- Aspect: 切面，由PointCut和Advice组成，它包括了横切逻辑和注入代码的定义
- Target: 被一个或多个 aspect 横切拦截操作的目标对象。
- Weaving: 织入，在编译时把 Advice 代码插入到目标对象特定连接点的过程。

## AspectJ原理简介(静态织入)
实操演示，展示反编译后的class代码，和源码的差别。

## 用AspectJ写个小框架
实操演示，从0开始写一个 权限校验申请 的小框架。


## 写在最后

给大家介绍这个AspectJ框架其实是想给大家分享一下AOP的思想，和OOP一样，它本身其实是一套方法论，也可以说成设计模式、思维方式、理论规则等等。分享给大家，也是希望大家在看待问题，解决问题的时候能有更多不同方向，不同的思维模式去思考。

参考资料：
[The AspectJ<sup>TM</sup> 5 Development Kit Developer's Notebook](https://www.eclipse.org/aspectj/doc/released/adk15notebook/index.html)
[The AspectJ<sup>TM</sup> Programming Guide](https://www.eclipse.org/aspectj/doc/released/progguide/index.html)
[Android-AspectJX](https://github.com/HujiangTechnology/gradle_plugin_android_aspectjx  )




