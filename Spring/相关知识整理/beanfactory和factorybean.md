# beanfactory和factorybean

## beanfactory

 这个其实是所有Spring Bean的容器根接口，给Spring 的容器定义一套规范，给IOC容器提供了一套完整的规范，比如我们常用到的getBean方法等 

**定义方法：**

- getBean(String name): Spring容器中获取对应Bean对象的方法，如存在，则返回该对象
- containsBean(String name)：Spring容器中是否存在该对象
- isSingleton(String name)：通过beanName是否为单例对象
- isPrototype(String name)：判断bean对象是否为多例对象
- isTypeMatch(String name, ResolvableType typeToMatch):判断name值获取出来的bean与typeToMath是否匹配
- getType(String name)：获取Bean的Class类型
- getAliases(String name):获取name所对应的所有的别名

**主要的实现类(包括抽象类)：**

- AbstractBeanFactory：抽象Bean工厂，绝大部分的实现类，都是继承于他
- DefaultListableBeanFactory:Spring默认的工厂类
- XmlBeanFactory：前期使用XML配置用的比较多的时候用的Bean工厂
- AbstractXmlApplicationContext:抽象应用容器上下文对象
- ClassPathXmlApplicationContext:XML解析上下文对象，用户创建Bean对象我们早期写Spring的时候用的就是他

**使用方式：**

1. 使用ClassPathXmlApplicationContext读取对应的xml文件实例对应上下文对象

```java
ApplicationContext context = new ClassPathXmlApplicationContext(new String[] {"applicationContext.xml"});
BeanFactory factory = (BeanFactory) context;
```

 

## FactoryBean

 该类是SpringIOC容器是创建Bean的一种形式，这种方式创建Bean会有加成方式，融合了简单的工厂设计模式于装饰器模式

有些人就要问了，我直接使用Spring默认方式创建Bean不香么，为啥还要用FactoryBean做啥，在某些情况下，对于实例Bean对象比较复杂的情况下，使用传统方式创建bean会比较复杂，例如（使用xml配置），这样就出现了FactoryBean接口，可以让用户通过实现该接口来自定义该Bean接口的实例化过程。即包装一层，将复杂的初始化过程包装，让调用者无需关系具体实现细节。 

**方法：**

- T getObject()：返回实例
- Class<?> getObjectType();:返回该装饰对象的Bean的类型
- default boolean isSingleton():Bean是否为单例

**常用类：**

- ProxyFactoryBean :Aop代理Bean
- GsonFactoryBean:Gson

**使用：**

1. Spring XML方式

application.xml

```java
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="demo" class="cn.lonecloud.spring.example.bo.Person">
        <property name="age" value="10"/>
        <property name="name" value="xiaoMing"/>
    </bean>

    <bean id="demoFromFactory" class="cn.lonecloud.spring.example.bean.PersonFactoryBean">
        <property name="initStr" value="10,init from factory"/>
    </bean>
</beans>
```

personFactoryBean.java

```java
package cn.lonecloud.spring.example.bean;

import cn.lonecloud.spring.example.bo.Person;
import org.springframework.beans.factory.FactoryBean;

import java.util.Objects;


public class PersonFactoryBean implements FactoryBean<Person> {

    /**
     * 初始化Str
     */
    private String initStr;

    @Override
    public Person getObject() throws Exception {
        //这里我需要获取对应参数
        Objects.requireNonNull(initStr);
        String[] split = initStr.split(",");
        Person p=new Person();
        p.setAge(Integer.parseInt(split[0]));
        p.setName(split[1]);
        //这里可能需要还要有其他复杂事情需要处理
        return p;
    }

    @Override
    public Class<?> getObjectType() {
        return Person.class;
    }

    public String getInitStr() {
        return initStr;
    }

    public void setInitStr(String initStr) {
        this.initStr = initStr;
    }
}
```

main方法

```java
package cn.lonecloud.spring.example.factory;

import cn.lonecloud.spring.example.bo.Person;
import org.springframework.beans.factory.BeanFactory;
import org.springframework.beans.factory.FactoryBean;
import org.springframework.context.ApplicationContext;
import org.springframework.context.support.ClassPathXmlApplicationContext;


public class SpringBeanFactoryMain {

    public static void main(String[] args) {
        //这个是BeanFactory
        BeanFactory beanFactory = new ClassPathXmlApplicationContext("application.xml");
        //获取对应的对象化
        Object demo = beanFactory.getBean("demo");
        System.out.println(demo instanceof Person);
        System.out.println(demo);
        //获取从工厂Bean中获取对象
        Person demoFromFactory = beanFactory.getBean("demoFromFactory", Person.class);
        System.out.println(demoFromFactory);
        //获取对应的personFactory
        Object bean = beanFactory.getBean("&demoFromFactory");
        System.out.println(bean instanceof PersonFactoryBean);
        PersonFactoryBean factoryBean=(PersonFactoryBean) bean;
        System.out.println("初始化参数为："+factoryBean.getInitStr());
    }
}
```

输出

```java
true
Person{name='xiaoMing', age=10}
Person{name='init from factory', age=10}
true
初始化参数为：10,init from factory
```

## 区别

1. BeanFactory:负责生产和管理Bean的一个工厂接口，提供一个Spring Ioc容器规范,
2. FactoryBean: 一种Bean创建的一种方式，对Bean的一种扩展。对于复杂的Bean对象初始化创建使用其可封装对象的创建细节。