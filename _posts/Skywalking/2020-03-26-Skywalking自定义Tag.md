### Skywalking自定义Tag

#### Skywalking内置Tags

org.apache.skywalking.apm.agent.core.context.tag.Tags

| 取值 | 对应Tag      |
| ---- | ------------ |
| 1    | url          |
| 2    | status_code  |
| 3    | db.type      |
| 4    | db.instance  |
| 5    | db.statement |
| 6    | db.bind_vars |
| 7    | mq.queue     |
| 8    | mq.broker    |
| 9    | mq.topic     |
| 10   | http.method  |
| 11   | http.params  |
| 12   | x-le         |
| 13   | http.body    |
| 14   | http.headers |

我们提到了“skywalking收集链路时，使用的URL都是通配符，在链路中，无法针对某个pageId，或者其他通配符的具体的值进行查找。或许skywalking出于性能考虑，但是对于这种不定的通用大接口，的确无法用于针对性的性能分析了。”

那么在skywalking 8.2版本中引入的对tag搜索的支持，就能够解决这个问题，我们可以根据tag对链路进行一次过滤，得到一类的链路。让我们来看看如何配置的。
我们根据一个叫`biz.id`的标签，查找过滤其值为"bbbc"的链路。那么这种自定义标签`biz.id`，如何配置呢？（skywalking 默认支持`http.method`等标签的搜索）

## 配置oap端application.yml

skywalking分析端是一个java进程，使用application.yml作为配置文件，其位置位于“apache-skywalking-apm-bin/config”文件夹中。

core.default.searchableTracesTags 配置项为可搜索标签的配置项，其默认为：“${SW_SEARCHABLE_TAG_KEYS:http.method,status_code,db.type,db.instance,mq.queue,mq.topic,mq.broker}”。如果没有配置环境变量SW_SEARCHABLE_TAG_KEYS，那么其默认就支持这几个在skywalking 中有使用到的几个tag。 那么我在里面修改了配置，加上了我们用到的“biz.id”、“biz.type”。
修改配置后，重启skywalking oap端即可支持。

## 代码进行打标签

racer#activeSpan()方法将会将自身作为构造去生成Span，最终仍是同一个Span。

使用H2数据库的时候，通过tag进行查询就会失效，会查不出链路，通过debug是可以看到对应的sql并无问题，拼出了biz.id的查询条件，具体原因还未查找，通过切换存储为es6解决了问题。(猜测普通的关系型数据库不支持，需要列式存储的数据库才可以)

源码分析

SegmentAnalysisListener

```
   private void appendSearchableTags(SpanObject span) {
        HashSet<Tag> segmentTags = new HashSet<>();
        span.getTagsList().forEach(tag -> {
            if (searchableTagKeys.contains(tag.getKey())) {
                final Tag spanTag = new Tag(tag.getKey(), tag.getValue());
                if (!segmentTags.contains(spanTag)) {
                    segmentTags.add(spanTag);
                }

            }
        });
        segment.getTags().addAll(segmentTags);
    }
```