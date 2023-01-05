---
title: 浅谈 Spring 解析自定义命名空间
tags: [Spring, Namespace]
date: 2023-1-5 22:18
categories: 笔记本
---
最近复习 Spring 的 IoC 容器时有些好奇 XML 配置文件及其各个标签的解析方式，毕竟分别使用 `bean` 标签将 Class 注入 IoC 容器还是最开始学习 Spring 时才使用过的方式，日常工作中基本都是使用注解来完成自动注入，因此对于 XML 文件内的一些细节倒是知之甚少。恰逢假期闲来无事，在经过一番搜索与学习之后算是有了些许收获，故此记录一篇，以便日后查阅。

## 流程分析
### 前置知识
在说明配置文件的解析之前，我们得先了解一下 Spring Bean 实例化的基本流程，毕竟配置文件是为了让 IoC 容器管理 Bean 对象而存在的。实例化流程中的细节还是非常多的，光是 Bean 的生命周期就足够我们喝上一壶的了，因此这里只介绍一下大体流程，重点只关注 XML 解析的部分。

对于配置文件的方式，Spring 提供了一个 `BeanDefinitionReader` 接口，不同的配置文件类型会由不同的实现类去解析，如 XML 配置文件会由 `XMLBeanDefinitionReader` 来解析，而 Groovy 配置文件则会由 `GroovyBeanDefinitionReader` 来解析。如果是使用注解的方式，则有一个专门的类—— `AnnotatedBeanDefinitionReader` 来解析。不过无论是使用配置文件（XML、Properties、Groovy...）还是注解的方式，都是殊途同归，它们的区别也只有入口不同，最终各自的 `BeanDefinitionReader` 都会将配置信息中的元数据转储为 BeanDefinition，继而存放至 BeanDefinitionMap 中。Spring 的上下文会在合适的时机遍历 Map 中所有的 BeanDefinition 并通过反射为其创建实例对象，最后将实例化完成后的 Bean 存放进 IoC 容器中，接受 Spring 的管理。这里我们需要关注的便是从 `XMLBeanDefinitionReader` 解析 XML 配置文件开始，直到 XML 中的元数据被转储为 BeanDefinition 并存放至 BeanDefinitionMap 中的过程。

### 详细代码
接下来，我们就根据上面的流程跟踪一下 Spring 的源代码，了解其中的一些实现细节。

因为我们使用的是 XML 配置的方式，因此首先找到入口类 `ClassPathXmlApplicationContext`，在其构造方法中有一个 `refresh()` 方法，这便是 Spring 容器的入口方法。
```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    super(parent);
    this.setConfigLocations(configLocations);
    if (refresh) {
        this.refresh();
    }

}
```
接着在 `refresh()` 方法中又执行了 `obtainFreshBeanFactory()` 方法，之后又经过 `refreshBeanFactory()` -> `loadBeanDefinitions()` -> `doLoadBeanDefinitions()` -> `registerBeanDefinitions()` -> `doRegisterBeanDefinitions()` 的流程后来到了 `parseBeanDefinitions()` 方法。
```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();

           for(int i = 0; i < nl.getLength(); ++i) {
               Node node = nl.item(i);
               if (node instanceof Element) {
                   Element ele = (Element)node;
                   if (delegate.isDefaultNamespace(ele)) {
                       this.parseDefaultElement(ele, delegate);
                   } else {
                       delegate.parseCustomElement(ele);
                   }
               }
           }
       } else {
           delegate.parseCustomElement(root);
       }

}
```
在这个方法中，XML 中的标签会被分为两类加以解析，即代码中的 `DefaultNamespace` 和 `CustomElement`。前者是对 `http://www.springframework.org/schema/beans` 命名空间下四个默认标签—— `import`、`alias`、`bean` 和 `beans` 的解析，后者便是对自定义命名空间下标签的解析。
```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, "import")) {
        this.importBeanDefinitionResource(ele);
    } else if (delegate.nodeNameEquals(ele, "alias")) {
        this.processAliasRegistration(ele);
    } else if (delegate.nodeNameEquals(ele, "bean")) {
        this.processBeanDefinition(ele, delegate);
    } else if (delegate.nodeNameEquals(ele, "beans")) {
        this.doRegisterBeanDefinitions(ele);
    }

}
```
```java
public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
    String namespaceUri = this.getNamespaceURI(ele);
    if (namespaceUri == null) {
        return null;
    } else {
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            this.error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        } else {
            return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
        }
    }
}
```
可以看到自定义命名空间的解析过程分为两步，先是通过 `namespaceUri` 获取对应的 `NamespaceHandler`，再调用对应 Handler 的 `parse()` 方法进行解析。我们先来看看通过 `namespaceUri` 是如何获取到 `NamespaceHandler` 的。
```java
public DefaultNamespaceHandlerResolver() {
    this((ClassLoader)null, "META-INF/spring.handlers");
}

public DefaultNamespaceHandlerResolver(@Nullable ClassLoader classLoader) {
    this(classLoader, "META-INF/spring.handlers");
}

public DefaultNamespaceHandlerResolver(@Nullable ClassLoader classLoader, String handlerMappingsLocation) {
    this.logger = LogFactory.getLog(this.getClass());
    Assert.notNull(handlerMappingsLocation, "Handler mappings location must not be null");
    this.classLoader = classLoader != null ? classLoader : ClassUtils.getDefaultClassLoader();
    this.handlerMappingsLocation = handlerMappingsLocation;
}

public NamespaceHandler resolve(String namespaceUri) {
    Map<String, Object> handlerMappings = this.getHandlerMappings();
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    if (handlerOrClassName == null) {
        return null;
    } else if (handlerOrClassName instanceof NamespaceHandler) {
        return (NamespaceHandler)handlerOrClassName;
    } else {
        String className = (String)handlerOrClassName;

        try {
            Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
            if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri + "] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
            } else {
                NamespaceHandler namespaceHandler = (NamespaceHandler)BeanUtils.instantiateClass(handlerClass);
                namespaceHandler.init();
                handlerMappings.put(namespaceUri, namespaceHandler);
                return namespaceHandler;
            }
        } catch (ClassNotFoundException var7) {
            throw new FatalBeanException("Could not find NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var7);
        } catch (LinkageError var8) {
            throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" + className + "] for namespace [" + namespaceUri + "]", var8);
        }
    }
}

private Map<String, Object> getHandlerMappings() {
    Map<String, Object> handlerMappings = this.handlerMappings;
    if (handlerMappings == null) {
        synchronized(this) {
            handlerMappings = this.handlerMappings;
            if (handlerMappings == null) {
                if (this.logger.isTraceEnabled()) {
                    this.logger.trace("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
                }

                try {
                    Properties mappings = PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
                    if (this.logger.isTraceEnabled()) {
                        this.logger.trace("Loaded NamespaceHandler mappings: " + mappings);
                    }

                    handlerMappings = new ConcurrentHashMap(mappings.size());
                    CollectionUtils.mergePropertiesIntoMap(mappings, (Map)handlerMappings);
                    this.handlerMappings = (Map)handlerMappings;
                } catch (IOException var5) {
                    throw new IllegalStateException("Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", var5);
                }
            }
        }
    }

    return (Map)handlerMappings;
}
```
在 `DefaultNamespaceHandlerResolver` 类中维护了一个名为 `handlerMappings` 的 Map，其内容是根据一个 Properties 配置文件生成的，这个文件的路径已经在该类的构造方法中固定下来了，即 `classpath:META-INF/spring.handlers`。可见 XML 中的 `namespaceUri` 只是一个 key，最终 Spring 会用这个 key 去 spring.handlers 文件中寻找对应的 value，从而拿到 `namespaceUri` 对应的解析器。我们可以看一下 spring-context.jar 对应目录下的 spring.handlers，差不多长这样:
```properties
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```
另外，你应该还可以在同级目录下看到一个 spring.schemas 文件，这里面定义了 XSD 文件逻辑地址与物理地址的对应关系。前者便是我们在 XML 文件的 `schemaLocation` 部分填写的链接，而后者则是前者所对应的文件在本地存在的位置。为了避免 Spring 每次解析 XML 都需要去逻辑地址标注的网络上寻找 XSD 文件，一般都推荐在 Properties 文件中定义好两者的映射关系，详见 [Spring 官方文档](https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#core.appendix.xsd-custom-registration-spring-schemas)的解释。如果你也对 XML 知之不详，可能会好奇这个文件的作用，它是 XML Schema 的定义文件，用于描述 XML 的结构，它规定了你的 XML 文件中可以出现哪些元素以及它们的格式是怎样的。详细的说明可以查看 [W3Schools 的教程](https://www.w3schools.com/xml/schema_intro.asp)。它不仅在解析的时候需要用到，IDE 一般也会通过该文件来完成编写 XML 文件时的 Auto Suggestion & Completion。
```properties
http\://www.springframework.org/schema/context/spring-context-2.5.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.0.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.1.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-3.2.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.0.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.1.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.2.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context-4.3.xsd=org/springframework/context/config/spring-context.xsd
http\://www.springframework.org/schema/context/spring-context.xsd=org/springframework/context/config/spring-context.xsd
```
说回上文，拿到 `NamespaceHandler` 后再来看它的 `parse()` 方法，其实现类为 `NamespaceHandlerSupport`:
```java
@Nullable
public BeanDefinition parse(Element element, ParserContext parserContext) {
    BeanDefinitionParser parser = this.findParserForElement(element, parserContext);
    return parser != null ? parser.parse(element, parserContext) : null;
}
```
其实就是通过当前命名空间下 XML 标签的名字找到对应的 `BeanDefinitionParser`，再调用对应的 `parse()` 方法。比如我们常见的 `<context:component-scan>`，它属于 `context` 命名空间，根据 spring.handlers 文件中的配置信息可以得知其对应的 `NamespaceHandler` 为 `ContextNamespaceHandler`，我们不妨看看这个 Handler 内是怎么写的:
```java
public class ContextNamespaceHandler extends NamespaceHandlerSupport {
    public ContextNamespaceHandler() {
    }

    public void init() {
        this.registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
        this.registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
        this.registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
        this.registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
        this.registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
        this.registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
        this.registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
        this.registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
    }
}
```
可以看到在 `ContextNamespaceHandler` 的 `init()` 方法中注册了标签名与 `BeanDefinitionParser` 的映射关系，因此 `NamespaceHandlerSupport` 才能根据不同的标签名调用不同的 `parse()` 方法。还是上面的例子，我们看看 `ComponentScanBeanDefinitionParser` 的 `parse()` 方法是怎么写的:
```java
public BeanDefinition parse(Element element, ParserContext parserContext) {
    String basePackage = element.getAttribute("base-package");
    basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
    String[] basePackages = StringUtils.tokenizeToStringArray(basePackage, ",; \t\n");
    ClassPathBeanDefinitionScanner scanner = this.configureScanner(parserContext, element);
    Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
    this.registerComponents(parserContext.getReaderContext(), beanDefinitions, element);
    return null;
}
```
内容并不复杂，只是使用 `ClassPathBeanDefinitionScanner` 扫描 XML 中配置的 `base-package` 路径，最后将该路径下的所有类转储为 BeanDefinition 并注册进 BeanDefinitionMap 中，Spring 会根据 Map 中的 Bean 信息在后续的流程中逐个实例化这些 Bean。

看到这里，XML 配置文件的解析部分就已经结束了，其实主要就是依靠 `NamespaceHandler` 及其各个子标签的 `BeanDefinitionParser` 来完成的，中间用到了两个之前没听说过的配置文件，感觉又多了点~~有用的~~知识!

## DIY 一个简单的自定义标签
有了上面的理论基础，我们便可以尝试着实现一个自定义命名空间下的标签了。比如我想实现这样一个标签，它的作用是在每个 Bean 注入到 Spring 容器前打印当前 Bean 的属性信息。功能很简单，实现起来也不难，大概就是以下几步:
1. 确定自定义命名空间、XML 标签及 XSD 文件的名称
2. 编写 XSD 文件
3. 编写 spring.handlers 及 spring.schemas 文件
4. 实现自定义的 NamespaceHandler
5. 实现自定义的 BeanDefinitionParser
6. 实现自定义的 BeanPostProcessor

其中前五步是通用操作，最后一步是为了完成本例的需求而创建的。实现了 `BeanPostProcessor` 接口的类可以选择实现 `postProcessBeforeInitialization()` 或者 `postProcessAfterInitialization()` 方法，它们会在每个 Bean 的生命周期中被调用。在本例中这两个方法都可以实现需求，这里我选择在 `postProcessAfterInitialization()` 方法中打印当前类的信息。

### 确定自定义命名空间、XML 标签及 XSD 文件的名称
这里就以我的域名为例:
* XML Namespace: `https://akaneym.com/schema/akane`
* XML Tag: `<akane:logBeanInfo/>`
* XSD File: `https://akaneym.com/schema/akane/beans.xsd`

### 编写 XSD 文件
```XML
<!-- beans.xsd (inside package classpath:com/akaneym/config) -->

<?xml version="1.0" encoding="UTF-8"?>
<xsd:schema xmlns="https://akaneym.com/schema/akane"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            targetNamespace="https://akaneym.com/schema/akane">

    <xsd:element name="logBeanInfo"></xsd:element>

</xsd:schema>
```
### 编写 spring.handlers 及 spring.schemas 文件
这里只需要注意一点，因为 Properties 文件中 `:` 也算分隔符，因此需要转义一下。

#### META-INF/spring.handlers
```properties
https\://akaneym.com/schema/akane=com.akaneym.handler.AkaneNamespaceHandler
```

#### META-INF/spring.schemas
```properties
https\://akaneym.com/schema/akane/beans.xsd=com/akaneym/config/beans.xsd
```

### 实现自定义的 NamespaceHandler
AkaneNamespaceHandler
```java
package com.akaneym.handler;

import com.akaneym.parser.LogBeanInfoBeanDefinitionParser;
import org.springframework.beans.factory.xml.NamespaceHandlerSupport;

/**
 * @author shiro
 * @since 2023-01-02 10:32 PM
 */
public class AkaneNamespaceHandler extends NamespaceHandlerSupport {
    @Override
    public void init() {
        this.registerBeanDefinitionParser("logBeanInfo", new LogBeanInfoBeanDefinitionParser());
    }
}
```

### 实现自定义的 BeanDefinitionParser
LogBeanInfoBeanDefinitionParser
```java
package com.akaneym.parser;

import com.akaneym.processor.LogBeanInfoPostProcessor;
import org.springframework.beans.factory.config.BeanDefinition;
import org.springframework.beans.factory.support.RootBeanDefinition;
import org.springframework.beans.factory.xml.BeanDefinitionParser;
import org.springframework.beans.factory.xml.ParserContext;
import org.w3c.dom.Element;

/**
 * @author shiro
 * @since 2023-01-02 10:37 PM
 */
public class LogBeanInfoBeanDefinitionParser implements BeanDefinitionParser {
    @Override
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        BeanDefinition beanDefinition = new RootBeanDefinition();
        beanDefinition.setBeanClassName(LogBeanInfoPostProcessor.class.getName());
        parserContext.getRegistry().registerBeanDefinition("LogBeanInfoPostProcessor", beanDefinition);
        return beanDefinition;
    }
}
```

### 实现自定义的 BeanPostProcessor
LogBeanInfoPostProcessor
```java
package com.akaneym.processor;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;

/**
 * @author shiro
 * @since 2023-01-02 10:42 PM
 */
public class LogBeanInfoPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("LogBeanInfo -> " + bean);
        return bean;
    }
}
```

### Show Time!
先创建两个测试用的实体类:

#### User
```java
package com.akaneym.bean;

/**
 * @author shiro
 * @since 11/1/2022 11:17 PM
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {
    private String userName;
    private Address address;
}
```

#### Address
```java
package com.akaneym.bean;

/**
 * @author shiro
 * @since 11/1/2022 11:18 PM
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Address {
    private String street;
    private Integer number;
}
```

然后在 applicationContext.xml 中将上面两个类注入 Spring 容器中并开启我们的自定义注解，别忘了添加 `xmlns` 和 `schemaLocation`:

#### applicationContext.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:akane="https://akaneym.com/schema/akane"
       xsi:schemaLocation="
            http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
            https://akaneym.com/schema/akane https://akaneym.com/schema/akane/beans.xsd">

    <bean class="com.akaneym.bean.Address" id="address">
        <constructor-arg name="street" value="下北泽"/>
        <constructor-arg name="number" value="1024"/>
    </bean>

    <bean class="com.akaneym.bean.User" id="user">
        <constructor-arg name="userName" value="Guitar Hero"/>
        <constructor-arg name="address" ref="address"/>
    </bean>

    <akane:logBeanInfo/>
</beans>
```

最后编写一个测试方法，因为我们的需求是 Bean 进入容器之前打印 Bean 信息，因此只需要获取到 IoC 容器就可以看到效果了:

#### JUnit Test
```java
@Test
public void beanConfigurationTest() {
    ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
}
```

#### Output
```shell
LogBeanInfo -> Address{street='下北泽', number=1024}
LogBeanInfo -> User{userName='Guitar Hero', address=Address{street='下北泽', number=1024}}
```

Nice！打完，收工！