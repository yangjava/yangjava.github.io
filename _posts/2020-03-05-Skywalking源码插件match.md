---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
# Skywalking源码插件match

## 目标类匹配
ClassMatch是一个标志接口，表示类的匹配器
```
public interface ClassMatch {
}
```

## NameMatch是通过完整类名精确匹配对应类
```
/**
 * Match the class with an explicit class name.
 * 通过完整类名精确匹配对应类
 */
public class NameMatch implements ClassMatch {
    private String className;

    private NameMatch(String className) {
        this.className = className;
    }

    public String getClassName() {
        return className;
    }

    public static NameMatch byName(String className) {
        return new NameMatch(className);
    }
}
```

## 除了NameMatch之外，还有比较常用的IndirectMatch用于间接匹配
```
public interface IndirectMatch extends ClassMatch {
    ElementMatcher.Junction buildJunction();

    // TypeDescription就是对类的描述,可以当做Class
    boolean isMatch(TypeDescription typeDescription);
}
```

## IndirectMatch的子类，比如：PrefixMatch通过前缀匹配
```
public class PrefixMatch implements IndirectMatch {
    private String[] prefixes;

    private PrefixMatch(String... prefixes) {
        if (prefixes == null || prefixes.length == 0) {
            throw new IllegalArgumentException("prefixes argument is null or empty");
        }
        this.prefixes = prefixes;
    }

    @Override
    public ElementMatcher.Junction buildJunction() {
        ElementMatcher.Junction junction = null;

        // 拼接匹配条件
        // 有多个前缀,只要一个匹配即可
        for (String prefix : prefixes) {
            if (junction == null) {
                junction = ElementMatchers.nameStartsWith(prefix);
            } else {
                junction = junction.or(ElementMatchers.nameStartsWith(prefix));
            }
        }

        return junction;
    }

    @Override
    public boolean isMatch(TypeDescription typeDescription) {
        for (final String prefix : prefixes) {
            // typeDescription.getName()就是全类名
            if (typeDescription.getName().startsWith(prefix)) {
                return true;
            }
        }
        return false;
    }

    public static PrefixMatch nameStartsWith(final String... prefixes) {
        return new PrefixMatch(prefixes);
    }
}
```

## 元素匹配
ElementMatchers






























