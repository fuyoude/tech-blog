# Spring FactoryBean 解析

## 1.BeanFactory 与 FactoryBean 的区别

**`FactoryBean`** 和 **`BeanFactory`** 比较容易搞混。**`BeanFacotry`** 是 spring 中比较原始的 Factory。如 **`XMLBeanFactory`** 就是一种典型的 BeanFactory。原始的 BeanFactory 无法支持 spring 的许多插件，如 AOP 功能、Web 应用等。 ApplicationContext 接口，它由 BeanFactory 接口派生而来，ApplicationContext 包含 BeanFactory 的所有功能，通常建议比 BeanFactory 优先。

BeanFactory 和 FactoryBean 的区别：

**<font color="red">BeanFactory 是接口，提供了 IOC 容器最基本的形式，给具体的 IOC 容器的实现提供了规范</font>**，FactoryBean 也是接口，为 IOC 容器中 Bean 的实现提供了更加灵活的方式，**<font color="red">FactoryBean 在 IOC 容器的基础上给 Bean 的实现加上了一个简单工厂模式和装饰模式</font>**。我们可以在 **`getObject()`** 方法中灵活配置。其实在 Spring 源码中有很多 FactoryBean 的实现类。

区别是 BeanFactory 是个 Factory，也就是 IOC 容器或对象工厂，FactoryBean 是个 Bean。在 Spring 中，所有的 Bean 都是由 BeanFactory (也就是 IOC 容器) 来进行管理的。但对 FactoryBean 而言，这个 Bean 不是简单的 Bean，而是一个能生产或者修饰对象生成的工厂 Bean，它的实现与设计模式中的工厂模式和修饰器模式类似。

## 2.FactoryBean 代码示例

### 2.1 Car 类

```java{.line-numbers}
package com.zjl.factorybean;

public class Car {
    public Car(String name) {
        this.name=name;
    }
    String name;
    public void run(){
        System.out.println(this.name+" is running");
    }
}
```

### 2.2 Person 类

```java{.line-numbers}
package com.zjl.factorybean;

public class Person {
    public Person(String name) {
        this.name=name;
    }
    public String name;
    public int age;
    
    public Car car;
    
    public void sayHello(){
        System.out.println(this.name+" say hello");
    }
} 
```

### 2.3 PersonFactory 类

```java{.line-numbers}
public class PersonFactory implements FactoryBean<Person> {

    String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public Person getObject() throws Exception {
        return new Person(name);
    }

    @Override
    public Class<Person> getObjectType() {
        return Person.class;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }

    public static void main(String[] args) {
        ClassPathXmlApplicationContext context =
                new ClassPathXmlApplicationContext("classpath*:rpc-invoke-config-server.xml");
        Person person1 = (Person) context.getBean("personFactory");
        Person person2 = (Person) context.getBean("personFactory");
        PersonFactory pf = (PersonFactory) context.getBean("&personFactory");
        System.out.println(person1);
        System.out.println(person2);
        System.out.println(pf);
        System.out.println(person1 == person2);
    }
} 
```

### 2.4 配置文件

```java{.line-numbers}
<bean id="personFactory" class="jtest.PersonFactory">
    <property name="name" value="xuweilin"/>
</bean> 
```

### 2.5 结果分析

```java{.line-numbers}
jtest.Person@3972a855
jtest.Person@3972a855
jtest.PersonFactory@62e7f11d
true 
```

从结果我们可以看到以下两点：

- 通过 **`getBean(beanName)`** 方法获取到的直接就是 Person 的实例，而不是 BeanFactory 或者 PersonFactory 的实例。但是在 beanName 的前面加上字符 &，获取到的对象就是 PersonFactory 类型的对象。
- 每次获取到的 Person 实例都是同一个，即使在 PersonFactory 的 getObject 方法中每次都会重新生成一个 Person 类型的对象。获取到的 Person 实例是单例与否是由 isSingleton 方法和配置文件中指定 bean 的 scope 是 singleton 还是 prototype 类型两种因素共同决定。

## 3.FactoryBean 源码分析

接口 FactoryBean<T> 的源代码如下：

```java{.line-numbers}
public interface FactoryBean<T> {
    T getObject() throws Exception;
    Class<?> getObjectType();
    boolean isSingleton();
}
```

实现该接口的类需要实现以上的方法，其中 getObject 就是用来生产 bean 的方法。通过 **`AbstractBeanFactory.doGetBean`** 可以知道，不论是单例还是原型的实例对象，最终都要通过 **`getObjectForBeanInstance`** 进行转换，最终得到的才是符合要求的 bean 实例。如：

```java{.line-numbers}
// Create bean instance.
// 单例的处理
if (mbd.isSingleton()) {
   sharedInstance = getSingleton(beanName, () -> {
      try {
         return createBean(beanName, mbd, args);
      }
      catch (BeansException ex) {
         destroySingleton(beanName);
         throw ex;
      }
   });
   bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

进入到 getObjectForBeanInstance，进入方法，判断如果是 bean 不是 FactoryBean 的实例且 beanName 是 & 开头，抛出错误。是 FactoryBean 的实例，且以 & 开头或者说 bean 不是 FactoryBean 类型的，则直接返回实例。

```java{.line-numbers}
protected Object getObjectForBeanInstance(
        Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

    // 进行类型检测，如果是& 作为前缀，但是传入 beanInstance 不是 FactoryBean 类型的，则抛出异常
    if (BeanFactoryUtils.isFactoryDereference(name)) {
        if (beanInstance instanceof NullBean) {
            return beanInstance;
        }
        if (!(beanInstance instanceof FactoryBean)) {
            throw new BeanIsNotAFactoryException(transformedBeanName(name), beanInstance.getClass());
        }
    }

    // 如果是非 FactoryBean 类型的 Bean 或者是 BeanName 以 & 作为前缀的前缀，则返回实例对象，无需转换
    if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
        return beanInstance;
    }

    // 需要转换，说明输入的 name 获取的 bean 是需要调用 FactoryBean 的 getObject，也就是说，
    // beanInstance 是 FactoryBean 类型，同时 name 不是以& 作为前缀
    Object object = null;
    if (mbd == null) {
        // 先到 FactoryBean 生产 bean 对象的缓存中去取，如果没有的话则用 factorybean 去生产一个，
        object = getCachedObjectForFactoryBean(beanName);
    }
    if (object == null) {
        FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
        if (mbd == null && containsBeanDefinition(beanName)) {
            mbd = getMergedLocalBeanDefinition(beanName);
        }
        boolean synthetic = (mbd != null && mbd.isSynthetic());
        // 通过 factoryBean 去生产 bean
        object = getObjectFromFactoryBean(factory, beanName, !synthetic);
    }
    return object;
}
```

首先，假如现在要获取的是 FactoryBean 生成的 Bean 实例，那么首先，会去缓存中取这个 bean，进入到 getCachedObjectForFactoryBean，是 FactoryBeanRegistrySupport 的方法。

```java{.line-numbers}
/** Cache of singleton objects created by FactoryBeans: FactoryBean name --> object */
/** 由factoryBean生成的单例bean都会缓存到factoryBeanObjectCache*/
private final Map<String, Object> factoryBeanObjectCache = new ConcurrentHashMap<>(16);

protected Object getCachedObjectForFactoryBean(String beanName) {
    Object object = this.factoryBeanObjectCache.get(beanName);
    return (object != NULL_OBJECT ? object : null);
}
```

假如获取不到，则生成对应的 FactoryBean，然后通过 FactoryBean 去生成，进入到 getObjectFromFactoryBean, 是 FactoryBeanRegistrySupport 的方法。我们可以看到，通过 FactoryBean 的对象是否是单例模式取决于 bean 定义的范围和方法 isSingleton 同时为单例才可以。

```java{.line-numbers}
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
    // 针对单例处理，必须要符合以下两个条件：
    // 1.FactoryBean 的 isSingleton 返回 true（isSingleton）
    // 2.实现 FactoryBean 接口的 bean 在配置文件中的 scope 值为 singleton（containsSingleton）
    if (factory.isSingleton() && containsSingleton(beanName)) {
        synchronized (getSingletonMutex()) {
            Object object = this.factoryBeanObjectCache.get(beanName);
            if (object == null) {
                // 通过 factory.getObject 获取
                object = doGetObjectFromFactoryBean(factory, beanName);
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
                if (alreadyThere != null) {
                    object = alreadyThere;
                }
                else {
                    if (object != null && shouldPostProcess) {
                        try {
                            object = postProcessObjectFromFactoryBean(object, beanName);
                        }
                        catch (Throwable ex) {
                            throw new BeanCreationException(beanName,
                                    "Post-processing of FactoryBean's singleton object failed", ex);
                        }
                    }
                    // 将获取到的对象放到 factoryBeanObjectCache 单例缓存 map 进行存储
                    this.factoryBeanObjectCache.put(beanName, (object != null ? object : NULL_OBJECT));
                }
            }
            return (object != NULL_OBJECT ? object : null);
        }
    }
    else {
        // 非单例的处理，直接通过 factory.getObejct 获取，然后再返回给用户，这样的话，
        // factoryBeanObjectCache 单例缓存中永远都不会有 beanName 的 bean，也就是说，前面的
        // getObjectForBeanInstance 方法中的 getCachedObjectForFactoryBean 只有在 bean 是单例的情况下才有效，
        // 不是单例的话，获取到的结果 bean 永远是 null
        Object object = doGetObjectFromFactoryBean(factory, beanName);
        if (object != null && shouldPostProcess) {
            try {
                object = postProcessObjectFromFactoryBean(object, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
            }
        }
        return object;
    }
}
```

从 getObjectFromFactoryBean 和 getObjectForBeanInstance 这两个方法中可以看出，如果 bean 不是单例的，那么它从 FactoryBean 生产 bean 对象的缓存中去取的结果永远是一个 null，这是因为在 getObjectFromFactoryBean 方法中，如果 bean 不是单例处理的话，根本不会把 bean 加到缓存中（只有单例才行），所以每次 **`context.getBean("personFactory")`** 获取到的结果都是新创建的 Person 对象。如果是对 bean 进行单例处理的话，那么在第一次获取到 Person 对象之后，就会把它放到缓存中，以后都会从缓存中取到同一份 Person 对象。

无论是单例和非单例，都会调用 **`doGetObjectFromFactoryBean`** 方法，此方法，使用 factory.getObject 生成 bean 对象。

