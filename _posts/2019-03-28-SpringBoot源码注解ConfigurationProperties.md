---
layout: post
categories: [SpringBoot]
description: none
keywords: SpringBoot
---
# SpringBoot源码注解@ConfigurationProperties
@ConfigurationProperties 注解主要用于将配置文件properties或 yml 中的属性转换到Bean中的属性。

## ConfigurationProperties概述
@ConfigurationProperties 注解主要用于将配置文件properties或 yml 中的属性转换到Bean中的属性。

比如像下面这样
```
properties.name=test
```

```java
@Getter
@Setter
@ConfigurationProperties(prefix = "properties")
public class TestConfigurationProperties {
    private String name;

}
```
但是这样我们是在Spring容器中是获取不到TestConfigurationProperties 这个bean的,要获取到有两种方式,

- 在TestConfigurationProperties 上添加注解@Component
- 使用@EnableConfigurationProperties 注解

## @ConfigurationProperties 源码
@ConfigurationProperties 注解源码很简单
```
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ConfigurationProperties {

	/**
	 * The prefix of the properties that are valid to bind to this object. Synonym for
	 * {@link #prefix()}. A valid prefix is defined by one or more words separated with
	 * dots (e.g. {@code "acme.system.feature"}).
	 * @return the prefix of the properties to bind
	 */
	@AliasFor("prefix")
	String value() default "";

	/**
	 * The prefix of the properties that are valid to bind to this object. Synonym for
	 * {@link #value()}. A valid prefix is defined by one or more words separated with
	 * dots (e.g. {@code "acme.system.feature"}).
	 * @return the prefix of the properties to bind
	 */
	@AliasFor("value")
	String prefix() default "";

	/**
	 * Flag to indicate that when binding to this object invalid fields should be ignored.
	 * Invalid means invalid according to the binder that is used, and usually this means
	 * fields of the wrong type (or that cannot be coerced into the correct type).
	 * @return the flag value (default false)
	 */
	boolean ignoreInvalidFields() default false;

	/**
	 * Flag to indicate that when binding to this object unknown fields should be ignored.
	 * An unknown field could be a sign of a mistake in the Properties.
	 * @return the flag value (default true)
	 */
	boolean ignoreUnknownFields() default true;

}

```
没有导入任何配置类，让`@ConfigurationProperties` 起作用的主要是类`ConfigurationPropertiesBindingPostProcessor`

核心方法主要是 bind方法和属性ConfigurationPropertiesBinder
```
// 属性绑定器
private ConfigurationPropertiesBinder binder;

private void bind(ConfigurationPropertiesBean bean) {
  	// 如果这个 bean 为空，或者已经处理过，则直接返回
		if (bean == null || hasBoundValueObject(bean.getName())) {
			return;
		}
  // 对 @ConstructorBinding 的校验，如果使用该注解但是没有找到合适的构造器，那么在这里抛出异常
		Assert.state(bean.getBindMethod() == BindMethod.JAVA_BEAN, "Cannot bind @ConfigurationProperties for bean '"
				+ bean.getName() + "'. Ensure that @ConstructorBinding has not been applied to regular bean");
		try {
      // 通过 Binder 将指定 prefix 前缀的属性值设置到这个 Bean 中
			this.binder.bind(bean);
		}
		catch (Exception ex) {
			throw new ConfigurationPropertiesBindException(bean, ex);
		}
	}
```





















