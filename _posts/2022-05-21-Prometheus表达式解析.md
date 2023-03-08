---
layout: post
categories: [Prometheus,PromQL]
description: none
keywords: Prometheus
---
# 
Java解析PromQL并修改添加Label

最近做的项目中用到了Prometheus做预警服务，其中Prometheus使用promql语言来查询。项目中用户通过UI或者自己手动输入PromQL时候是缺少一些系统参数的，所以需要在用户输入完成以后同步到Prometheus时候将这部分缺失的信息给添加回去，这里就需要修改用户写的PromQL了。

实现思路是通过Antlr4来解析PromQL并修改。

https://github.com/antlr/grammars-v4
上述url是antlr官方提供的各个语言的语法定义文件，其中就包含我需要PromQL，将上述代码中的promql包中的两个g4文件拷贝到自己项目中，我对拷贝的PromQLLexer.g4文件中的最后的空格处做了处理改成如下内容否则重写以后会丢失原语句中的空格。

WS: [\r\t\n ]+ -> channel(HIDDEN);
新建maven项目，此处我就叫promql-parser。

pom部分内容如下：

    <properties>
        <antlr4.version>4.9.3</antlr4.version>
    </properties>
 
    <dependencies>
        <dependency>
            <groupId>org.antlr</groupId>
            <artifactId>antlr4-runtime</artifactId>
            <version>${antlr4.version}</version>
        </dependency>
    </dependencies>
 
 
    <build>
        <plugins>
            <plugin>
                <groupId>org.antlr</groupId>
                <artifactId>antlr4-maven-plugin</artifactId>
                <version>${antlr4.version}</version>
                <executions>
                    <execution>
                        <phase>generate-sources</phase>
                        <goals>
                            <goal>antlr4</goal>
                        </goals>
                    </execution>
                </executions>
                <configuration>
                    <visitor>true</visitor>
                    <sourceDirectory>src/main/antlr4</sourceDirectory>
                    <outputDirectory>src/main/generate</outputDirectory>
                </configuration>
            </plugin>
        </plugins>
    </build>

将g4文件放置到项目的src/main/antlr4/packages中，这样后续生成的代码也自动会生成到src/main/generate/packages中，packages替换为自己的java包名，如下图：



现在开始编写解析PromQL并修改PromQL的代码：

public class ParserUtil {

    public static String addLabels(String promQL, Map<String, String> labels) {
        PromQLLexer lexer = new PromQLLexer(CharStreams.fromString(promQL));
        CommonTokenStream tokenStream = new CommonTokenStream(lexer);
        final TokenStreamRewriter rewriter = new TokenStreamRewriter(tokenStream);
        PromQLParser parser = new PromQLParser(tokenStream);
        ParseTreeWalker.DEFAULT.walk(
                new LabelMatcherListener(rewriter, labels), parser.expression());
        return rewriter.getText();
    }
 
    private static class LabelMatcherListener extends PromQLParserBaseListener {
        private final TokenStreamRewriter rewriter;
        private final Map<String, String> labels;
 
        public LabelMatcherListener(TokenStreamRewriter rewriter, Map<String, String> labels) {
            this.rewriter = rewriter;
            this.labels = new LinkedHashMap<>(labels);
        }
 
        @Override
        public void enterInstantSelector(PromQLParser.InstantSelectorContext ctx) {
            if (labels.isEmpty()) {
                return;
            }
            PromQLParser.LabelMatcherListContext matcherListContext = ctx.labelMatcherList();
            if (null == matcherListContext) {
                if (null == ctx.LEFT_BRACE()) {
                    rewriter.insertAfter(
                            ctx.METRIC_NAME().getSymbol(),
                            labels.entrySet().stream()
                                    .map(e -> String.format("%s=\"%s\"", e.getKey(), e.getValue()))
                                    .collect(Collectors.joining(",", "{", "}")));
                } else {
                    rewriter.insertAfter(
                            ctx.LEFT_BRACE().getSymbol(),
                            labels.entrySet().stream()
                                    .map(e -> String.format("%s=\"%s\"", e.getKey(), e.getValue()))
                                    .collect(Collectors.joining(",")));
                }
            } else {
                for (PromQLParser.LabelMatcherContext matcherContext :
                        matcherListContext.labelMatcher()) {
                    String labelName = matcherContext.labelName().getText();
                    if (labels.containsKey(labelName)) {
                        rewriter.replace(
                                matcherContext.STRING().getSymbol(),
                                String.format("\"%s\"", labels.get(labelName)));
                        labels.remove(labelName);
                    }
                }
                if (!labels.isEmpty()) {
                    rewriter.insertBefore(
                            ctx.RIGHT_BRACE().getSymbol(),
                            labels.entrySet().stream()
                                    .map(e -> String.format("%s=\"%s\"", e.getKey(), e.getValue()))
                                    .collect(Collectors.joining(",", ",", "")));
                }
            }
        }
    }
}
上述代码对应的单元测试代码：

public class ParserUtilTest {

    @Test
    public void test1() {
        Assertions.assertEquals(
                "hello{a=\"b\"}",
                ParserUtil.addLabels("hello", Collections.singletonMap("a", "b")));
    }
 
    @Test
    public void test1_1() {
        HashMap<String, String> map = new LinkedHashMap<>();
        map.put("a", "b");
        map.put("c", "d");
        Assertions.assertEquals("hello{a=\"b\",c=\"d\"}", ParserUtil.addLabels("hello", map));
    }
 
    @Test
    public void test2() {
        Assertions.assertEquals(
                "hello{a=\"b\"}",
                ParserUtil.addLabels("hello{}", Collections.singletonMap("a", "b")));
    }
 
    @Test
    public void test2_1() {
        HashMap<String, String> map = new LinkedHashMap<>();
        map.put("a", "b");
        map.put("c", "d");
        Assertions.assertEquals("hello{a=\"b\",c=\"d\"}", ParserUtil.addLabels("hello{}", map));
    }
 
    @Test
    public void test3() {
        Assertions.assertEquals(
                "hello{a=\"b\",c=\"d\"}",
                ParserUtil.addLabels("hello{a=\"b\"}", Collections.singletonMap("c", "d")));
    }
 
    @Test
    public void test3_1() {
        HashMap<String, String> map = new LinkedHashMap<>();
        map.put("a", "b1");
        map.put("c", "d");
        Assertions.assertEquals(
                "hello{a=\"b1\",c=\"d\"}", ParserUtil.addLabels("hello{a=\"b\"}", map));
    }
}

## 阿里云TimeStream系列--TimeStream promQL实现原理
TimeStream可以支持使用promQL查询存储ES TimeStream的时序数据。

关于promQL功能的使用可以参见使用文档：https://help.aliyun.com/document_detail/436523.html

这篇文章主要说明下promQL实现的关键原理。

首先TimeStream使用antlr4j来解析promQL的语法，具体解析功能在：https://github.com/antlr/grammars-v4/tree/master/promql

通过antlr4j的解析，TimeStream可以拿到一个promQL的解析树。然后TimeStream去定义每个算子的功能实现，比如遇到+算子，那么TimeStream就将算子两边的结果进行相加。如果遇到可以下推的算子，TimeStream就将算子下推到ES中，promQL大致的执行流程如下：




