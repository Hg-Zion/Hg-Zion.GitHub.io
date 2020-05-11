---
title: Spring——bean（一）
tags: java spring bean
---

## 1 创建 bean

### 1.1 创建一个简单的 bean

在 **Spring IOC** 容器中创建一个简单（指不包含任何其他对象的引用）的 `bean`，有 **XML** 配置和注解配置（后面会讲）两种方式。
{:.success}

举个栗子，我们以 *Person.java* 为例：

```java
public class Person {
    String name;
    int age;

    public void setName(String name) {
        this.name = name;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Person{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

对于 `Person` 的两个字段 `name` 和 `age`，我们需要为其构造相应的 `setter` 方法，以便于 **Spring** 通过反射机制为其字段赋值。

在 **XML** 文件中，我们需要声明该 `bean` 的存在：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="child" class="pojo.Person">
        <property name="name" value="Li Si"/>
        <property name="age" value="12" />
    </bean>
</beans>
```

这里用到了 `spring-beans` 模块，我们可以通过 **Maven** 在 *pom.xml* 文件中引入其坐标：

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-beans</artifactId>
    <version>5.2.5.RELEASE</version>
</dependency>
```

最后简单运行测试一番即可：

```java
public static void main(String[] args) {
    ApplicationContext context = new 
        ClassPathXmlApplicationContext("XMLApplicationContext.xml");
    Person child = (Person) context.getBean("child");
    System.out.println(child); // "Person{name='Li Si', age=12}"
}
```

### 1.2 创建一个不简单的 bean

在 **Spring IOC** 容器中声明一个简单 `bean` 是为了便于其他 `bean` 去引用，而 **Spring** 可以通过依赖注入的方式将其他组件的示例传递给其引用字段。
{:.success}

而这种传递方式有两种，基于 `setter` 的 ***DI*** 和基于构造函数的 ***DI***。
{:.conclude}

**（01）基于 `setter` 的 *DI* ：**

```java
public class Town {
    Person mayor;

    public void setMayor(Person mayor) {
        this.mayor = mayor;
    }
}
```

```xml
<bean id="sweet water" class="pojo.Town">
    <property name="mayor" ref="child"/>
</bean>
<bean id="child" class="pojo.Person">
    <property name="name" value="Li Si"/>
    <property name="age" value="12"/>
</bean>
```

**（02）基于构造函数的 *DI*：**

```java
public class Town {
    String name;
    Person mayor;

    public Town(String name, Person mayor) {
        this.name = name;
        this.mayor = mayor;
    }
}
```

```xml
<bean id="sweet water" class="pojo.Town">
    <constructor-arg index="0" value="sweet water"/>
    <constructor-arg index="1" ref="child"/>
</bean>
<bean id="child" class="pojo.Person">
    <property name="name" value="Li Si"/>
    <property name="age" value="12"/>
</bean>
```

当然，这两者是可以结合使用的，因为本质上它就是 **spring** 通过反射机制对 `setter` 方法或者构造方法进行调用。另外，由于 **spring** 有类型判断机制，我们在上述代码中的 `index` 其实是多余的，但我想写出 `index` 应该是一个良好易懂的代码习惯。

### 1.3 使用工厂方法创建 bean

**Spring** 容器可以通过调用静态工厂方法或者通过工厂实例对象的工厂方法来创建对象。
{:.success}

**（01）静态工厂方法：**

```java
public class PersonFactory {
    public static Person createPerson(String name, int age) {
        Person p = new Person();
        p.name = name;
        p.age = age;
        return p;
    }
}
```

```xml
<bean id="Yang CD" class="pojo.Town">
    <constructor-arg index="0" value="yang cd"/>
    <constructor-arg index="1" ref="man"/>
</bean>
<bean id="man" class="pojo.PersonFactory" factory-method="createPerson">
    <constructor-arg index="0" value="Zan yi"/>
    <constructor-arg index="1" value="24"/>
</bean>
```

可以看到，`man` 是以 `Person` 对象的身份存在的，**Spring** 直接调用静态工厂方法创建了它。

**（02）实例工厂方法：**

相较于静态工厂方法，仅仅多了创建工厂类对象以及引用该对象两步。

```java
public class PersonFactory {
    // 仅去掉了 static
    public Person createPerson(String name, int age) {
        Person p = new Person();
        p.name = name;
        p.age = age;
        return p;
    }
}
```

```xml
<bean id="factory" class="pojo.PersonFactory"/>
<bean id="man" factory-bean="factory" factory-method="createPerson">
    <constructor-arg index="0" value="Zan yi"/>
    <constructor-arg index="1" value="24"/>
</bean>
```



## 2 配置 bean

### 2.1 构造函数的参数匹配

通过构造函数创建的对象在配置文件中的参数匹配有三种方式，基于参数位置匹配、基于参数类型匹配以及基于名称匹配。
{:.success}

**（01）基于参数位置匹配**

我们可以用 index 特性显式指定参数位置，也可以根据编写顺序隐式指定。

```java
public class School {
    Person teacher;
    Person student;

    public School(Person teacher, Person student) {
        this.teacher = teacher;
        this.student = student;
    }
}
```

```xml
<bean id="school" class="pojo.School">
    <constructor-arg ref="child"/>
    <constructor-arg ref="man" />
</bean>
```

**（02）基于类型匹配**

```java
public class School {
    Person teacher;
    Person student;
    Town town;

    public School(Person teacher, Person student, Town town) {
        this.teacher = teacher;
        this.student = student;
        this.town = town;
    }
}
```

```xml
<bean id="school" class="pojo.School">
    <constructor-arg ref="child" type="pojo.Person"/>
    <constructor-arg ref="sweet water" type="pojo.Town"/>
    <constructor-arg ref="man" type="pojo.Person"/>
</bean>
```

**（03）基于名称匹配**

```xml
<bean id="school" class="pojo.School">
    <constructor-arg ref="child" name="student"/>
    <constructor-arg ref="sweet water" index="2"/>
    <constructor-arg ref="man" type="pojo.Person"/>
</bean>
```

在这里我使用了一个混搭的例子，"基于名称匹配"里的"名称"指的是构造函数的参数使用的名称。

### 2.2 配置不同类型的属性

我们在 Spring IOC 容器里使用的对象，其属性分为基本类型、引用类型、其他标准类型（容器类）或自定义类型。
{:.success}

**（01）基本类型**

Spring 通过内置属性编辑器 PropertyEditors 提供了将 Java 常见类型转换为字符串类型的实现，反之亦然。
{:.conclude}

```java
public class ManHasAll {
    boolean isMan;
    Boolean isPerson;
    int age;
    Integer birth;
    char first;
    String name;
    byte[] nickname;
    Date date;
    Properties prop;
    
    // 很多 setter 方法
}
```

```xml
<bean id="all" class="pojo.ManHasAll">
    <property name="man" value="true"/>
    <property name="person" value="true"/>
    <property name="age" value="88"/>
    <property name="birth" value="88"/>
    <property name="first" value="A"/>
    <property name="name" value="TheMan"/>
    <property name="nickname" value="the man"/>
    <property name="date" value="1900-01-01"/>
    <property name="prop">
        <props>
            <prop key="gender">XY</prop>
        </props>
    </property>
</bean>
```

**（02）引用类型**

```xml
<bean id="sweet water" class="pojo.Town">
    <constructor-arg index="0" value="sweet water"/>
    <constructor-arg index="1" ref="child"/>
</bean>
```

**（03）集合类型的值**

我们可以通过 `<list>` 、`<map>`、`<set>`、`<props>` 元素用于设置对应的 **Java** 集合类型属性或者构造方法参数。
{:.conclude}

```java
public class ManHasAll {
    Map<Set<Integer>,List<String>> map;
    int[] array;
    Properties properties;
    // 省略 setter 方法
}
```

```xml
<bean id="all" class="pojo.ManHasAll">
    <property name="map">
        <map>
            <entry>
                <key>
                    <set>
                        <value>100</value>
                    </set>
                </key>
                <list>
                    <value>"100"</value>
                </list>
            </entry>
        </map>
    </property>
    <property name="array">
        <list>
            <value>100</value>
        </list>
    </property>
    <property name="properties">
        <props>
            <prop key="amd">"yes!"</prop>
        </props>
    </property>
</bean>
```



### 2.3 作用域

对象在 ***Spring IOC*** 容器中有 **prototype** 和 **singleton** 两种作用域，后者是默认的。
{:.conclude}

### 2.4 p 和 c 命名空间

用 `p` 和 `c` 简化 **XML** 文件中的 `bean` 定义，用于取代元素 `<property>` 和 `<constructor-arg>` 元素。
{:.success}

```xml
<bean id="school" class="pojo.School">
    <constructor-arg ref="child" name="student"/>
    <constructor-arg ref="sweet water" index="2"/>
    <constructor-arg ref="man" type="pojo.Person"/>
</bean>
<bean id="child" class="pojo.Person">
    <property name="name" value="Li Si"/>
    <property name="age" value="12"/>
</bean>

<!-- ============修改后================== -->
<bean id="school" class="pojo.School">
    c:ref="child" name="student"
    c:ref="sweet water" index="2"
    c:ref="man" type="pojo.Person"
</bean>
<bean id="child" class="pojo.Person">
    p:name="name" value="Li Si"
    p:name="age" value="12"
</bean>
```

### 2.5 util 模式

该模式用于将属性值转换为一种特殊的 `bean`，从而可以被其他 `bean` 引用赋值。
{:.success}

**util** 模式的元素有 `<list>` 、`<map>`、`<set>`、`<constant>` 、`<property-path>`、`<properties>`。
{:.conclude}

```xml
<util:list id="listType" list-class="java.util.ArrayList">
    <value>Something right</value>
    <value>Something left</value>
</util:list>
<util:map id="mapType" map-class="java.util.HashMap">
    <entry key="Something" value="right"/>
</util:map>
<util:set id="setType" set-class="java.util.HashSet">
    <value>Something right</value>
    <value>Something left</value>
</util:set>
<util:properties id="propType" location="something.properties"/>
<util:constant static-field="java.lang.Boolean.TRUE"/>
<util:property-path id="an age" path="child.age"/>
```



