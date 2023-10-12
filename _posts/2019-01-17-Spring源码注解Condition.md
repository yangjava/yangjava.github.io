---
layout: post
categories: [Spring]
description: none
keywords: Spring
---
# Spring源码注解Condition
Conditional注解表示仅当所有指定条件都匹配时，该组件才会被注册。


## Conditional注解三种使用方式
- 作为类级别的注解使用：作用于任何直接或间接被@Component注解的类上，除此之外还包括@Configuration注解的类
- 作为方法级别的注解使用：作用于任何被@Bean注解的方法上
- 作为元注解使用：目的是组成自定义注释

如果一个@Configuration类标注了@Conditional注解，则与该类关联的所有@Bean方法、@Import注解和@ComponentScan注解都将受指定Condition约束。

## @Conditional注解
标注在类或方法上，当所有指定Condition都匹配时，才允许向spring容器注册组件。

注解定义：
```java
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Conditional {

	/**
	 * 当所有指定Condition都匹配时，才允许向spring容器注册组件。
	 *
	 * 这个泛型的意思是接受所有Condition接口的实现类。
	 */
	Class<? extends Condition>[] value();
}

```
该注解从spring4.0开始提供，一般作用于类或@bean方法上，其中value属性是Condition[]类型，表示所有Condition同时满足时该类或者方法才会生效。

下面我们看看Condition的定义。

## Condition接口
```java
@FunctionalInterface
public interface Condition {
	/**判断是否满足条件*/
	boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```
Condition是一个函数式接口，里面提供了一个抽象方法，返回值为boolean类型且有两个入参，分别是ConditionContext和AnnotatedTypeMetadata。该接口作为判断条件是否满足的顶级抽象，那判断所需的信息就应该由这两个入参提供，那从这两个类能获取到哪些信息呢？

## ConditionContext接口
```
/**Condition类使用的一些上下文信息*/
public interface ConditionContext {

	BeanDefinitionRegistry getRegistry();

	@Nullable
	ConfigurableListableBeanFactory getBeanFactory();

	Environment getEnvironment();

	ResourceLoader getResourceLoader();

	@Nullable
	ClassLoader getClassLoader();
}

```
通过ConditionContext，我们可以获取到如下信息：

- 借助getRegistry()返回的BeanDefinitionRegistry可以检查bean定义
- 借助getBeanFactory()返回的ConfigurableListableBeanFactory检查bean是否存在，甚至探查bean的属性
- 借助getEnvironment()返回的Environment检查环境变量是否存在以及它的值是什么
- 借助getResourceLoader()返回的ResourceLoader所加载的资源
- 借助getClassLoader()返回的ClassLoader加载并检查类是否存在

## AnnotatedTypeMetadata接口
```
public interface AnnotatedTypeMetadata {

	/**@since 5.2*/
	MergedAnnotations getAnnotations();

	default boolean isAnnotated(String annotationName) {
		return getAnnotations().isPresent(annotationName);
	}

	@Nullable
	default Map<String, Object> getAnnotationAttributes(String annotationName) {
		return getAnnotationAttributes(annotationName, false);
	}

	@Nullable
	default Map<String, Object> getAnnotationAttributes(String annotationName,
			boolean classValuesAsString) {

		MergedAnnotation<Annotation> annotation = getAnnotations().get(annotationName,
				null, MergedAnnotationSelectors.firstDirectlyDeclared());
		if (!annotation.isPresent()) {
			return null;
		}
		return annotation.asAnnotationAttributes(Adapt.values(classValuesAsString, true));
	}

	@Nullable
	default MultiValueMap<String, Object> getAllAnnotationAttributes(String annotationName) {
		return getAllAnnotationAttributes(annotationName, false);
	}

	@Nullable
	default MultiValueMap<String, Object> getAllAnnotationAttributes(
			String annotationName, boolean classValuesAsString) {

		Adapt[] adaptations = Adapt.values(classValuesAsString, true);
		return getAnnotations().stream(annotationName)
				.filter(MergedAnnotationPredicates.unique(MergedAnnotation::getMetaTypes))
				.map(MergedAnnotation::withNonMergedAttributes)
				.collect(MergedAnnotationCollectors.toMultiValueMap(map ->
						map.isEmpty() ? null : map, adaptations));
	}

}

```
通过AnnotatedTypeMetadata,我们可以获取到如下信息：

- getAnnotations() 方法可以根据基础元素的直接注释返回注释详细信息
- isAnnotated(String annotationName) 方法确定基础元素是否具有已定义的给定类型的注释或元注释
- getAnnotationAttributes(String annotationName) 方法检索给定类型的注释的属性（如果有）（即，如果在基础元素上定义为直接注释或元注释），则还应考虑组合注释上的属性覆盖。
- getAnnotationAttributes(String annotationName,boolean classValuesAsString)方法检索给定类型的注释的属性（如果有）（即，如果在基础元素上定义为直接注释或元注释），则还应考虑组合注释上的属性覆盖。
- getAllAnnotationAttributes(String annotationName)方法检索给定类型的所有注释的所有属性（如果有）（即，如果在基础元素上定义为直接注释或元注释）。请注意，这个变体并没有采取属性覆盖进去。
- getAllAnnotationAttributes(String annotationName, boolean classValuesAsString)方法检索给定类型的所有注释的所有属性（如果有）（即，如果在基础元素上定义为直接注释或元注释）。请注意，这个变体并没有采取属性覆盖进去。
