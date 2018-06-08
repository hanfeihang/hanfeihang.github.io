---
layout: post
title: 'SpringBoot类型安全的配置属性'
date: 2017-10-25
author: Feihang Han
tags: SpringBoot
---

# Type-safe Configuration Properties

Using the`@Value("${property}")`annotation to inject configuration properties can sometimes be cumbersome, especially if you are working with multiple properties or your data is hierarchical in nature. Spring Boot provides an alternative method of working with properties that allows strongly typed beans to govern and validate the configuration of your application.

```java
package com.example;

import java.net.InetAddress;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties("foo")
public class FooProperties {

    private boolean enabled;
    private InetAddress remoteAddress;
    private final Security security = new Security();
    public boolean isEnabled() { ... }
    public void setEnabled(boolean enabled) { ... }
    public InetAddress getRemoteAddress() { ... }    
    public void setRemoteAddress(InetAddress remoteAddress) { ... }
    public Security getSecurity() { ... }

    public static class Security {
        private String username;
        private String password;
        private List <String> roles = new ArrayList<(Collections.singleton("USER"));
        public String getUsername() { ... }
        public void setU sername(String username) { ... }
        public String getPassword() { ... }
        public void setPassword(String password) { ... }
        public List<String> getRoles() { ... }
        public void setRoles(List<String> roles) { ... }
    }
}
```

The POJO above defines the following properties:

* `foo.enabled`,`false`by default
* `foo.remote-address`, with a type that can be coerced from`String`
* `foo.security.username`, with a nested "security" whose name is determined by the name of the property. In particular the return type is not used at all there and could have been`SecurityProperties`
* `foo.security.password`
* `foo.security.roles`, with a collection of`String`


> Getters and setters are usually mandatory, since binding is via standard Java Beans property descriptors, just like in Spring MVC. There are cases where a setter may be omitted:
- Maps, as long as they are initialized, need a getter but not necessarily a setter since they can be mutated by the binder.
- Collections and arrays can be either accessed via an index \(typically with YAML\) or using a single comma-separated value \(properties\). In the latter case, a setter is mandatory. We recommend to always add a setter for such types. If you initialize a collection, make sure it is not immutable \(as in the example above\)
- If nested POJO properties are initialized \(like the`Security`field in the example above\), a setter is not required. If you want the binder to create the instance on-the-fly using its default constructor, you will need a setter.

> Some people use Project Lombok to add getters and setters automatically. Make sure that Lombok doesn’t generate any particular constructor for such type as it will be used automatically by the container to instantiate the object. 

> See also the [differences between`@Value`and`@ConfigurationProperties`](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config-vs-value).

You also need to list the properties classes to register in the`@EnableConfigurationProperties`annotation:

```java
@Configuration
@EnableConfigurationProperties(FooProperties.class)
public class MyConfiguration {
}
```

> When`@ConfigurationProperties`bean is registered that way, the bean will have a conventional name:`<prefix>-<fqn>`, where`<prefix>`is the environment key prefix specified in the`@ConfigurationProperties`annotation and`<fqn>`the fully qualified name of the bean. If the annotation does not provide any prefix, only the fully qualified name of the bean is used.The bean name in the example above will be`foo-com.example.FooProperties`.

Even if the configuration above will create a regular bean for`FooProperties`, we recommend that`@ConfigurationProperties`only deal with the environment and in particular does not inject other beans from the context. Having said that, The`@EnableConfigurationProperties`annotation is\_also\_automatically applied to your project so that any\_existing\_bean annotated with`@ConfigurationProperties`will be configured from the`Environment`. You could shortcut`MyConfiguration`above by making sure`FooProperties`is already a bean:

```java
@Component
@ConfigurationProperties(prefix="foo")
public class FooProperties {

// ... see above

}
```

This style of configuration works particularly well with the`SpringApplication`external YAML configuration:

```yaml
# application.yml

foo:
    remote-address: 192.168.1.1
    security:
        username: foo
        roles:
          - USER
          - ADMIN

# additional configuration as required
```

To work with`@ConfigurationProperties`beans you can just inject them in the same way as any other bean.

```java
@Service
public class MyService {

    private final FooProperties properties;

    @Autowired
    public MyService(FooProperties properties) {
        this.properties = properties;
    }

     //...

    @PostConstruct
    public void openConnection() {
        Server server = new Server(this.properties.getRemoteAddress());
        // ...
    }

}
```

> Using`@ConfigurationProperties`also allows you to generate meta-data files that can be used by IDEs to offer auto-completion for your own keys, see the[Appendix B,Configuration meta-data](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#configuration-metadata)appendix for details. |

# 24.7.1 Third-party configuration

As well as using`@ConfigurationProperties`to annotate a class, you can also use it on public`@Bean`methods. This can be particularly useful when you want to bind properties to third-party components that are outside of your control.

To configure a bean from the`Environment`properties, add`@ConfigurationProperties`to its bean registration:

```java
@ConfigurationProperties(prefix = "bar")
@Bean
public
 BarComponent barComponent() {
    ...
}
```

Any property defined with the`bar`prefix will be mapped onto that`BarComponent`bean in a similar manner as the`FooProperties`example above.

# 24.7.2 Relaxed binding

Spring Boot uses some relaxed rules for binding`Environment`properties to`@ConfigurationProperties`beans, so there doesn’t need to be an exact match between the`Environment`property name and the bean property name. Common examples where this is useful include dashed separated \(e.g.`context-path`binds to`contextPath`\), and capitalized \(e.g.`PORT`binds to`port`\) environment properties.

For example, given the following`@ConfigurationProperties`class:

```java
@ConfigurationProperties(prefix="person")
public class OwnerProperties {

    private String firstName;

    public String getFirstName() {
        return this.firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

}
```

The following properties names can all be used:

**Table 24.1. relaxed binding**

| Property | Note |
| :--- | :--- |
| `person.firstName` | Standard camel case syntax. |
| `person.first-name` | Dashed notation, recommended for use in`.properties`and`.yml`files. |
| `person.first_name` | Underscore notation, alternative format for use in`.properties`and`.yml`files. |
| `PERSON_FIRST_NAME` | Upper case format. Recommended when using a system environment variables. |

# 24.7.3 Properties conversion

Spring will attempt to coerce the external application properties to the right type when it binds to the`@ConfigurationProperties`beans. If you need custom type conversion you can provide a`ConversionService`bean \(with bean id`conversionService`\) or custom property editors \(via a`CustomEditorConfigurer`bean\) or custom`Converters`\(with bean definitions annotated as`@ConfigurationPropertiesBinding`\).

> As this bean is requested very early during the application lifecycle, make sure to limit the dependencies that your`ConversionService`is using. Typically, any dependency that you require may not be fully initialized at creation time. You may want to rename your custom`ConversionService`if it’s not required for configuration keys coercion and only rely on custom converters qualified with`@ConfigurationPropertiesBinding`. |

# 24.7.4 @ConfigurationProperties Validation

Spring Boot will attempt to validate`@ConfigurationProperties`classes whenever they are annotated with Spring’s`@Validated`annotation. You can use JSR-303`javax.validation`constraint annotations directly on your configuration class. Simply ensure that a compliant JSR-303 implementation is on your classpath, then add constraint annotations to your fields:

```java
@ConfigurationProperties(prefix="foo")
@Validated
public class FooProperties {

    @NotNull
    private InetAddress remoteAddress;

    // ... getters and setters

}
```

In order to validate values of nested properties, you must annotate the associated field as`@Valid`to trigger its validation. For example, building upon the above`FooProperties`example:

```java
@ConfigurationProperties(prefix="connection")
@Validated
public class FooProperties {

    @NotNull
    private InetAddress remoteAddress;

    @Valid
    private final Security security = new Security();

    // ... getters and setters

    public static class Security {

        @NotEmpty
        public String username;

        // ... getters and setters

    }

}
```

You can also add a custom Spring`Validator`by creating a bean definition called`configurationPropertiesValidator`. The`@Bean`method should be declared`static`. The configuration properties validator is created very early in the application’s lifecycle and declaring the`@Bean`method as static allows the bean to be created without having to instantiate the`@Configuration`class. This avoids any problems that may be caused by early instantiation. There is a[property validation sample](https://github.com/spring-projects/spring-boot/tree/v1.5.8.RELEASE/spring-boot-samples/spring-boot-sample-property-validation)so you can see how to set things up.

> The`spring-boot-actuator`module includes an endpoint that exposes all`@ConfigurationProperties`beans. Simply point your web browser to`/configprops`or use the equivalent JMX endpoint. See the[_Production ready features_](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#production-ready-endpoints). section for details. |

# 24.7.5 @ConfigurationProperties vs. @Value

`@Value`is a core container feature and it does not provide the same features as type-safe Configuration Properties. The table below summarizes the features that are supported by`@ConfigurationProperties`and`@Value`:

| Feature | `@ConfigurationProperties` | `@Value` |
| :--- | :--- | :--- |
| [Relaxed binding](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config-relaxed-binding) | Yes | No |
| [Meta-data support](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#configuration-metadata) | Yes | No |
| `SpEL`evaluation | No | Yes |

If you define a set of configuration keys for your own components, we recommend you to group them in a POJO annotated with`@ConfigurationProperties`. Please also be aware that since`@Value`does not support relaxed binding, it isn’t a great candidate if you need to provide the value using environment variables.

Finally, while you can write a`SpEL`expression in`@Value`, such expressions are not processed from[Application property files](https://docs.spring.io/spring-boot/docs/current/reference/htmlsingle/#boot-features-external-config-application-property-files).

