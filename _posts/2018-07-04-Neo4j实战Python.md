---
layout: post
categories: [Neo4j]
description: none
keywords: Neo4j
---
# Neo4j实战Python
python操作neo4j目前了解到有两个库，一个是py2neo，一个是neo4j。对比文档来看，前者和后者的区别，前者是面向对象形式的，没有提供多线程的支持，后者是依赖cypher语法实现的操作，提供了多线程的支持。

## Neo4j实战
安装py2neo。输入命令
```
pip install py2neo==4.3.0 -i https://pypi.douban.com/simple
```
注意：不能输入命令pip install py2neo，该命令会安装最新版本，执行Python程序时会报错“ValueError: The following settings are not supported: {‘http_port‘: 7474}”等问题，或报账号密码错误

报错原因：py2neo版本过高
```
from py2neo import Graph
 
graph = Graph('http://localhost:7474/', username='neo4j', password='******')
 
# '******'为您设置的密码
```

### 构建结点代码
```
from py2neo import Node, Relationship, Graph, NodeMatcher, RelationshipMatcher
graph = Graph('http://localhost:7474/', username='neo4j', password='04051835a')
Person2 = Node('Person', name='于一博')    
graph.create(Person2)  # 创建结点
Person3 = Node('Person', name='杨聪浩')    
graph.create(Person3)  # 创建结点
relation16 = Relationship(Person2,'室友',Person3)
graph.create(relation16)  # 创建关系
```

### 查询结点代码
```
match = nodematcher.match('Person') # 查询结点
for node in match:
    print(node)
print(list(match))
```

### 删除结点代码
```
graph.delete(Person3)  # 删除Person3结点
graph.delete(relation3)  # 删除relation3中所包含的结点及其所有关系
```

以下是一个示例代码，展示了如何使用py2neo执行Cypher查询：
```
from py2neo import Graph

# 连接到 Neo4j 数据库
graph = Graph("bolt://localhost:7687", auth=("neo4j", "password"))

# 执行 Cypher 查询
query = """
MATCH (startNode:Node {node_type:'person'})-[r:INVEST*1..4]->(endNode:Node {node_type:'company'})
WHERE endNode.id = $endNodeId AND r.x > 0.33
RETURN startNode, COLLECT(DISTINCT endNode.id) AS distinctEndIds
"""
params = {"endNodeId": "your_end_node_id"}

result = graph.run(query, parameters=params)

# 处理查询结果
for record in result:
    startNode = record['startNode']
    distinctEndIds = record['distinctEndIds']
    
    # 进行进一步的操作
    
# 关闭与数据库的连接（可选）
graph.close()
```






























