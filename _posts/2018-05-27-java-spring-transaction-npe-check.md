---
layout: post
title: Spring中@Transaction事务失效,空指针异常问题排查
comments: true
categories:
  - JAVA学习
---

## Spring中@Transaction事务失效,空指针异常问题排查
Spring中也是有一些一不注意就会踩到的坑。本屌原来在支付做过一段时间，也遇到过Spring中事务失效的问题。前几天写代码，又发现了一个空指针问题，闲来无事，写一写，方便后人踩坑后有个思路。

### 问题代码

本人怕被查水表，还是抽象了一下代码。

```java
@Service
public class A {

    @Resource
    private B b;

    @Resource
    private A self;

    @Monitor //monitor是一个AspectJ,是一个切面
    public void a(){
        System.out.println("进入了a");
        self.a0(); //Scene1
    }

    @Transaction
    private void a0(){
        System.out.println("进入了a0");
        b.b();  //Scene2
    }
}
```

```java
@Service
public class B {

    public void b(){
        System.out.println("进入了b");
    }
}
```

**问题1:**  当执行到`Scene1`的时候，如果直接调用`a0()`方法，而不通过self这种方式，@Transaction 注解将会失效(熟悉Spring的同学应该喜闻乐见)。

**问题2:**  当执行到`Scene2`的时候，抛出了

```java
java.lang.NullPointerException
	at xxx.test.A.a0(A.java:29)
	at xxx.test.A.a(A.java:24)
```

传说中的空指针。(原谅我的简单写,测试时脑子抽抽在公司代码里测试的)

### 分析问题

首先看问题2吧，What the fuck ？这都能抛出空指针，凭啥。我们单步调试进去看看:

![placeholder](https://raw.githubusercontent.com/CodingRookieH/blog-image/master/Step1.png)

果然，代理类的b果然没有被装配，这是怎么回事呢？

#### 初探元凶Cglib

我们来看看Spring中的```CglibAopProxy```

```java
		/**
		 * Gives a marginal performance improvement versus using reflection to
		 * invoke the target when invoking public methods.
		 */
		@Override
		protected Object invokeJoinpoint() throws Throwable {
			if (this.publicMethod) {
				return this.methodProxy.invoke(this.target, this.arguments);
			}
			else {
				return super.invokeJoinpoint();
			}
		}
```

这里的***target*** ，其实就是最终被代理的类，Spring中cglib这种动态代理方式，虽然是对被代理类的一种extands继承代理，但是不会装配代理类，因此代理类中的fields都为null也不足为奇，因为最后都会调用到被代理类取执行，而被代理类是被装配过的(不太熟的同学可以看看cglib是如何生成代理类的)。

#### 诡异的函数调用

说到这，我们只是解释了为什么代理类的field为null，但是大家肯定有疑惑了，你大爷，我为了使切面生效，都已经使用self这种自己注入自己的方式了，你告诉我没装配，那我调用```a0()```为什就调用不到***target***上边去呢？

嗯，好问题，不知道客官你是否注意到，```a0()```方法是一个```private```的呢？我们把`A$$EnhancerBySpringCGLIB$$d33fc0a3`给Dump下来，其实可以发现，内存中生成的代理类，也完全没有`a0()`方法的踪迹，Dump内存中的类的方法可以参考这个[Dump内存中的类](https://blog.csdn.net/hengyunabc/article/details/51106980)。

很正常，因为cglib这种动态代理方式是extands类然后做代理，`private`不能被extands当然不能代理了。

OK，Question2，你们肯定要问了，那都没有`a0()`方法，我怎么调用到的呢？不知大家注意到没，装配的时候，我们类的签名是**`A`**，那么JAVA基础扎实的大家应该就懂了，虽然对象是`A$$EnhancerBySpringCGLIB$$d33fc0a3`对象，但是类的签名是**`A`**，调用`private`方法的时候当然能取到**`A`**类的方法地址，进行调用。也就是说，我们用子类的对象，调用了父类的方法，而子类的内存区域中，父类占用的区域fields都为null，因此抛出了空指针。

### 再看@Transaction失效

相信看完上边解释，并且自己细细品味过的你，应该已经知道，为什么不用`self`这种方式调用`a0()`会引起`@Transaction`失效了。Spring中事务其实也是通过做了切面，去实现的，包括事务的提交，回滚，传播逻辑都是在这个切面中增强的。如果程序通过某种方式(这里就是指`@Monitor`这个切面)调用到**target**后，后续方法调用的都是未增强过的方法，自然会导致`@Transaction`失效。



本文为作者原创，转载请注明出处 。**邮箱：568718043@qq.com**