title: Dagger2
date: 2018-03-15 20:02:24
---

### 0.什么是依赖注入

首先我们应该都知道什么是依赖，废话不多说直接上代码：

```java
class A {
    B b；
    C c;
}
```

如上图代码，我们知道class A 对class B 和 class C有依赖，而依赖注入则是如何把class B 和 class C引入class A的过程和方法，依赖注入听着十分高大上，其实就在我们平时写的代码中就有着许多的依赖注入的例子,你肯定不信，那我们来看看依赖注入最常见的三种常规方式吧：

* 构造方法注入

```java
class A {
	B b；
    C c;
	
	A(B b, C c) {
    	this.b = b;
      	this.c = c;
    }
}
```

* set方法注入

```java
class A {
	B b；
    C c;
	
	void setB(B b) {
    	this.b = b;
    }
  
  	void setC(C c) {
    	this.c = c;
    }
}
```

* 接口注入

```java
interface Inject{
  	inject(B b);
  	inject(C c);
}

class A implements Inject{
	B b；
    C c;
	
 	@override
	void inject(B b) {
    	this.b = b;
    }
  
  	@override
  	void inject(C c) {
    	this.c = c;
    }
}
```

看到这你肯定会想：“啥？！这也算依赖注入？”。我要十分肯定的告诉你：是的！这就是依赖注入。

而正式这种常见的依赖注入方式免不了会在class A中去new 一个依赖的B或C对象，更有甚者，B 或 C又需要依赖其他类。这样的话这样在构造依赖对象的时候就会把大量的代码和时间浪费在依赖关系上，又由于直接调用的构造方法是不可能动态改变的还会造成代码的耦合。

因此，java中有许多的依赖注入框架来解决这个问题，在java后端中应该最流行的就是Spring框架了吧，通过注解方式完成依赖的注入，在写代码的时候不关心引用对象的依赖关系，只需要拿到对象调用方法即可，是不是很方便。而同时在Android开发中也有一个这样的框架，也即是今天的主角——Dagger2.



### 1.什么是Dagger2 

引用Google文档中的一句话:

> Dagger is a fully static, compile-time dependency injection framework for both Java and Android. 

翻译过来就是:

> Dagger是用于java和Android的，完全静态的编译时依赖注入框架。

完全静态保证了可追溯性，而编译时则说明Dagger2完全弃用了使用反射的方式在运行时进行注入，而是在代码编译期，生成代码实现依赖注入，这完全避免了运行时的反射注入造成的额外性能消耗。



### 2.引入Dagger2到你的工程

> ### Android Gradle
>
> 引入Dagger2需要的依赖:
>
> ```
> dependencies {
>   compile 'com.google.dagger:dagger:2.x'
>   annotationProcessor 'com.google.dagger:dagger-compiler:2.x'
> }
> ```
>
> 如果你使用了 `dagger.android` 你还需要引入以下依赖:
>
> ```
> compile 'com.google.dagger:dagger-android:2.x'
> compile 'com.google.dagger:dagger-android-support:2.x' // if you use the support libraries
> annotationProcessor 'com.google.dagger:dagger-android-processor:2.x'
> ```
>
> 如果你是用的gradle插件版本号小于 `2.2`, 参见 [https://bitbucket.org/hvisser/android-apt](https://bitbucket.org/hvisser/android-apt).

### 3.Dagger2基本框架

先来一段代码吧，一下就是很简单的dagger2示例代码:

```java
class Person{
  	@inject
	User user;
  	
  	public static void main(String[] args){
    	DaggerPersonComponent.builder().build().inject(this);
      	user.say();
    }
}
-----------------------------------------------
@component
interface PersonComponent{
	void inject(Person person);
}
-----------------------------------------------
class User {
  	String name;
  
	@inject
  	User(){
    	this.name = "kaka";
  	}
  
  	public void say(){
      	System.out.println("my name is"+this.name);
  	}
}
```

以上就是只使用@inject 和 @component 两个注解实现的最简单的Dagger2例子。



#### 3.1 @inject 和 @component注解

@inject是Dagger2的最基本的注解，它本身包含了两种含义注入和提供注入(特殊情况下它能同时表达这两个含义):

1. 一是标注目标类中所依赖的其他类（适用于构造方法 和 成员变量），需要依赖的地方
2. 二是标注所依赖的其他类的构造函数，提供依赖的地方

在有了@inject注解后dagger2怎么知道把相应的构造方法注入到对应的引用中去呢，那就是靠@component注解了。@component注解又叫注入器是连接目标类和依赖类的桥梁。@component注解的使用一般有两种方法:

1. `void inject(目标类 obj);`Dagger2 会从目标类开始查找 @Inject 注解，自动生成依赖注入的代码，调用 inject 可完成依赖的注入。
2. `Object getObj();` 如：`User getUser();`
   Dagger2 会到 User 类中找被 @Inject 注解标注的构造器，自动生成提供 User 依赖的代码，这种方式一般为其他 Component 提供依赖。（一个 Component 可以依赖另一个 Component，后面会说）

@inject 和 @component的关系可以用下图来解释:

![](http://ww2.sinaimg.cn/large/a15b4afegy1fi325oo9l6j20k2069mx5)

@component先查找目标类中用`@Inject`标注的成员变量，然后再到相应的类中去寻找有@inject注解的构造方法，最后调用构造方法生成对象实例注入给目标类的相应成员变量。

#### 3.3 @module 和  @provider 注解

上面的例子我们看到的依赖类都是无参的构造方法，那么如果依赖类的构造方法是带参数的怎么办呢？亦或是当我们依赖的类是第三方的类库，我们没办法去改变源码在构造方法上加上@inject注解，又怎么办呢？

这时候就要用到@module 和  @provider这两个注解了。还是拿上面的Person类做例子，如果User类的构造方法是带参数情况或者User类是第三方库中的类:

```java
class User {
  	String name;
  
  	User(String name){
    	this.name = name;
  	}
  	...
}
```

那么，用上@module 和  @provider注解来提供依赖如下，同时修改@Component中对Module的依赖:

```java
@Module
public class PersonModule{
  	String name;	
  
  	PersonModule(String name){
    	this.name = name;
  	}
  
    @Provides
 	User provideUser(){
    	return User(this.name);
    }
}
-----------------------------------------------
@component(modules = PersonModule.class)
interface PersonComponent{
	void inject(Order order);
}
```

在Person类中注入User对象的时候就可以如下带参数传入了:

```java
class Person{
  	@inject
	User user;
  	
  	public static void main(String[] args){
    	DaggerPersonComponent.builder().PersonModule("kaka").build().inject(this);
      	user.say();
    }
}
```

这样User类的构造就完全由Module类接管了，不需要对User类做任何处理，即可在Person 类中进行注入。在@Component中添加modules 参数是告诉Component 注入器，可以从@module中获取依赖的实例对象。Component 就会在module中去找被 @Provide 标注的方法，这也相当于构造器的 @Inject，可以提供依赖。

另外，@Component 可以指定多个 @Module 的，并且 Component 也可以依赖其它 Component 存在。

#### 3.4 @Qualifier 和 @Named注解 

这两个都是用于区分相同的依赖(为同一个类，或继承某一个父类或者都实现了同一个接口)的，其中@Qualifier 是用来自定义限定符的，而 @Named 则是基于 String 的限定符。

当我有两个相同的依赖提供时，那么Dagger就不知道我们到底要提供哪一个依赖，因为它找到了两个。这时候我们就可以通过限定符为两个依赖分别打上标记，指定提供某个依赖。

```java&amp;amp;#39;
比如Person类中有一个Face的抽象类,而有两个子类同时提供了依赖:
class Person {
	@inject
	@Named("Left")
  	Hand leftHand;
  	
	@inject
	@Right
  	Hand rightHand;
}
-----------------------------------------------
@module
class PersonModule {
  	@provides
  	@Named("Left")
  	Face providerLeftHand(){
      	return new LeftHand();
  	}
  	
  	@provides
  	@Right
  	Face providerRightHand(){
      	return new RightHand();
  	}
}
-----------------------------------------------
@Qualifier
@Retention(RetentionPolicy.RUNTIME)
public @interface Right {}
-----------------------------------------------
abstract class Hand{
  
} 
-----------------------------------------------
class LeftHand extends Face{
  	LeftHand(){
  	}
}
-----------------------------------------------
class RightHand extendsFace{
  	RightHand(){
  	}
}
```

将限定符分别作用于对应的@Provides和 @inject两处，他们就能按限定符所对应的类关系中找到对应的依赖进行注入。

####3.5 @Component 的 dependence 和 @SubComponent

前面已经提到过Component是可以有依赖关系的，依然是Person的例子:

```java
class Person {
  	Hand rightHand;
  	Hand lefthand;
  
  	Person(Hand rightHand, Hand Lefthand){
      	this.rightHand = rightHand;
      	this.lefthand = lefthand;
  	}
}
-----------------------------------------------
@Module
class HandModule{
  	@provides
  	@Named("Left")
  	Hand providerLeftHand(){
      	return new LeftHand();
  	}
  	
  	@provides
  	@Right
  	Hand providerRightHand(){
      	return new RightHand();
  	}
}
-----------------------------------------------
@Module
class PersonModule{
  	@providers
  	Person getPerson(@Right Hand rightHand，@Named("Left") Hand leftHand){
      	return new Person(rightHand,lefthand);
  	}
}
-----------------------------------------------
@Component(modules = HandModule.class)
interface HandComponent{
  	@Named("Left")
  	Hand getLeftHand();
  
  	@Right
  	Hand getRightHand();
}
-----------------------------------------------
@Component(modules = PersonModule.class,dependencies = HandComponent.class)
interface PersonComponent{
  	Person getPerson();
}
-----------------------------------------------
@Component(dependencies = PersonComponent.class)
interface TargetComponent{
  	void inject(TargetObject object);
}
```

最后，我们就可以在Target类中，这样使用:

```java
class target{
	@inject
  	person
      
    public void onCreate(){
      	DaggerTargetComponent.builder().personComponent(
        	DaggerPersonComponent.builder().handComponent(
            	DaggerHandComponent.builder().create()
            )
        ).build().inject(this);
    }
}
```

在Component中我们不需要在TargetComponent中额外指定依赖的Module，因为component中的dependencies参数已经做了这样的事情，有了这个参数，component会自动的去依赖所它自己依赖的component的module。

如果说@Component 的 dependence是一层一层的依赖关系，那么@SubComponent就可以说是自顶向下的继承关系，我们拿Android中常用的Context来举个例子:

很多工具类都需要使用到 Application 的 Context 对象，此时就可以用一个 Component 负责提供，我们可以命名为 AppComponent。
需要用到的 context 对象的 SharePreferenceComponent，ToastComponent 就可以它作为 Subcomponent 存在了。

而且在 AppComponent 中，我们可以很清晰的看到有哪些子 Component，因为在里面我们定义了很多`XxxComponent plus(Module... modules)`

每个 ActivityComponent 也是可以作为 AppComponent 的 Subcomponent，这样可以更方便的进行依赖注入，减少重复代码。













参考链接：	https://google.github.io/dagger//users-guide.html

​			https://juejin.im/entry/587499ff128fe1006b42f7de

​			http://www.jianshu.com/p/24af4c102f62

​			http://www.jianshu.com/p/5d6b21351086

​			http://www.jianshu.com/p/269c3f70ec1e