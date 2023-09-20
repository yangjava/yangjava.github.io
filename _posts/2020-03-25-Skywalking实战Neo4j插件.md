---
layout: post
categories: [Skywalking]
description: none
keywords: Skywalking
---
## Skywalking实战Neo4j插件

# Neo4j插件
Spring原生的Neo4j插件有问题。重新写了一个新的插件。
```
neo4j-4.x=org.apache.skywalking.apm.plugin.neo4j.v4x.define.OgmSessionInstrumentation
```
对neo4j相关的类进行增强
```java
import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.ConstructorInterceptPoint;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.InstanceMethodsInterceptPoint;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassInstanceMethodsEnhancePluginDefine;
import org.apache.skywalking.apm.agent.core.plugin.match.ClassMatch;

import static net.bytebuddy.matcher.ElementMatchers.*;
import static org.apache.skywalking.apm.agent.core.plugin.bytebuddy.ArgumentTypeNameMatch.takesArgumentWithType;
import static org.apache.skywalking.apm.agent.core.plugin.match.NameMatch.byName;


public class OgmSessionInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private final static String ENHANCED_CLASS = "org.neo4j.ogm.session.Neo4jSession";
    private final static String QUERY = "query";

    private final static String OGM_SESSION_QUERY_INTERCEPTOR = "org.apache.skywalking.apm.plugin.neo4j.v4x.OgmSessionQueryInterceptor";

    private final static String OGM_SESSION_QUERY_CYPHER_INTERCEPTOR = "org.apache.skywalking.apm.plugin.neo4j.v4x.OgmSessionQueryCypherInterceptor";

    private final static String CONSTRUCTOR_INTERCEPTOR = "org.apache.skywalking.apm.plugin.neo4j.v4x.OgmSessionConstructorInterceptor";

    @Override
    protected ClassMatch enhanceClass() {
        return byName(ENHANCED_CLASS);
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[]{
                new ConstructorInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getConstructorMatcher() {
                        return any();
                    }

                    @Override
                    public String getConstructorInterceptor() {
                        return CONSTRUCTOR_INTERCEPTOR;
                    }
                }
        };
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return named(QUERY).and(takesArgumentWithType(0, "java.lang.Class"))
                                .and(takesArgumentWithType(1, "java.lang.String"))
                                .and(returns(Iterable.class));
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return OGM_SESSION_QUERY_INTERCEPTOR;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return false;
                    }
                },
                new InstanceMethodsInterceptPoint() {
                    @Override
                    public ElementMatcher<MethodDescription> getMethodsMatcher() {
                        return named(QUERY).and(takesArgumentWithType(0, "java.lang.String"))
                                .and(takesArgumentWithType(1, "java.util.Map"));
                    }

                    @Override
                    public String getMethodsInterceptor() {
                        return OGM_SESSION_QUERY_CYPHER_INTERCEPTOR;
                    }

                    @Override
                    public boolean isOverrideArgs() {
                        return false;
                    }
                }
        };
    }
}
```

编写拦截类
```
import org.apache.skywalking.apm.agent.core.context.ContextManager;
import org.apache.skywalking.apm.agent.core.context.tag.Tags;
import org.apache.skywalking.apm.agent.core.context.trace.AbstractSpan;
import org.apache.skywalking.apm.agent.core.context.trace.SpanLayer;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.EnhancedInstance;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstanceMethodsAroundInterceptor;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.MethodInterceptResult;
import org.apache.skywalking.apm.plugin.neo4j.v4x.util.CypherUtils;
import org.neo4j.ogm.driver.Driver;

import java.lang.reflect.Method;

import static org.apache.skywalking.apm.plugin.neo4j.v4x.Neo4jPluginConstants.DB_TYPE;


public class OgmSessionQueryCypherInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        if (ContextManager.isActive()) {
            Driver driver = (Driver) objInst.getSkyWalkingDynamicField();
            String peer = driver.getConfiguration().getURI();
            final AbstractSpan span = ContextManager.createExitSpan("Neo4j", peer);
            span.setComponent(Neo4jPluginConstants.NEO4J);
            Tags.DB_TYPE.set(span, DB_TYPE);
            String cypher = (String) allArguments[0];
            Tags.DB_STATEMENT.set(span, CypherUtils.limitBodySize(cypher));
            if (Neo4jPluginConfig.Plugin.Neo4j.TRACE_CYPHER_PARAMETERS) {
                Neo4jPluginConstants.CYPHER_PARAMETERS_TAG
                        .set(span, CypherUtils.limitParametersSize(allArguments[1].toString()));
            }

            SpanLayer.asDB(span);
        }

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        if (ContextManager.isActive()) {
            ContextManager.stopSpan();
        }
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
                                      Class<?>[] argumentsTypes, Throwable t) {
        ContextManager.activeSpan().log(t);
    }
}
```

```
import org.apache.skywalking.apm.agent.core.context.ContextManager;
import org.apache.skywalking.apm.agent.core.context.tag.Tags;
import org.apache.skywalking.apm.agent.core.context.trace.AbstractSpan;
import org.apache.skywalking.apm.agent.core.context.trace.SpanLayer;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.EnhancedInstance;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstanceMethodsAroundInterceptor;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.MethodInterceptResult;
import org.apache.skywalking.apm.plugin.neo4j.v4x.util.CypherUtils;
import org.neo4j.ogm.driver.Driver;

import java.lang.reflect.Method;

import static org.apache.skywalking.apm.plugin.neo4j.v4x.Neo4jPluginConstants.DB_TYPE;


public class OgmSessionQueryInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {
        if (ContextManager.isActive()) {
            Driver driver = (Driver) objInst.getSkyWalkingDynamicField();
            String peer = driver.getConfiguration().getURI();
            final AbstractSpan span = ContextManager.createExitSpan("Neo4j", peer);
            span.setComponent(Neo4jPluginConstants.NEO4J);
            Tags.DB_TYPE.set(span, DB_TYPE);
            String cypher = (String) allArguments[1];
            Tags.DB_STATEMENT.set(span, CypherUtils.limitBodySize(cypher));
            if (Neo4jPluginConfig.Plugin.Neo4j.TRACE_CYPHER_PARAMETERS) {
                Neo4jPluginConstants.CYPHER_PARAMETERS_TAG
                        .set(span, CypherUtils.limitParametersSize(allArguments[2].toString()));
            }

            SpanLayer.asDB(span);
        }

    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method, Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        if (ContextManager.isActive()) {
            ContextManager.stopSpan();
        }
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method, Object[] allArguments,
                                      Class<?>[] argumentsTypes, Throwable t) {
        ContextManager.activeSpan().log(t);
    }
}
```