# 从 DFS 到 BFS：经典算法在业务中的应用

## 背景

最近着手重构了项目中一个领域对象，该对象是基于使用手册，对字节数组解码得到，解码的结果中仅有参数值信息，参数名需要对手册建模来组合得到。重构后的对象为树形结构，由于前端展示，后端的业务操作，都需要一个能够唯一定位参数的字符串
`route`，最初使用 `DFS` + `回溯`的方案来逐节点遍历、拼接 `route`，但后来情况发生了变化。

# 变更

最开始的领域对象，其手册中节点没有重复，因此基本的`DFS` + `回溯`已经能够满足业务需求，但现在的情况是同层会出现多个相同的节点，基于
`DFS` 的方案无法关注到同层的上下文，因此会产生大量重复的 `route`，这是无法接受的。

## 数据结构

为表一般性，本文并不展示原始业务代码，仅保留关键字段和算法核心内容。

### 节点数据结构

```java

@Getter
@Setter
public class BusinessNode {
    /**
     * 参数名
     */
    private String name;

    /**
     * 参数值
     */
    private String value;

    /**
     * 单个参数全路径
     */
    private String route;

    private List<BusinessNode> children = new ArrayList<>();
}

```

### 领域对象

同理，不展示非算法核心的字段、算法。

```java

@Setter
@Getter
public class BusinessTree {
    private BusinessNode body;
}
```

# DFS + 回溯算法

基础算法中，采用最符合直觉的回溯方案，可直接生成每一个节点的 `route`

```java
    /**
 * 生成每个节点的 Route 数据
 */
public void generateRouteDFS() {
    List<String> path = new ArrayList<>();
    dfs(body, path);
}

private void dfs(BusinessNode body, List<String> path) {
    if (body == null) {
        return;
    }
    String route = String.join(" / ", path);
    body.setRoute(route);
    path.add(body.getName());
    for (BusinessNode child : body.getChildren()) {
        dfs(child, path);
    }
    path.removeLast();
}
```

### 算法局限性

DFS为深度优先，因此遍历树时，完全无法得知其他分支的上下文，缺乏横向视野

```
Root
├── A
│   └── X
└── A
    └── X
```

其处理流程为

```
Root -> A(左) -> X(左) -> 回溯 -> A(右) -> X(右)
```

处理第二个 `A` 时，`path` 已被清空，回溯成了 `Root`，最终得到的 `route` 为

```
Root
├── A
|  (Root)
│   └── X
|       (Root / A)
└── A
  (Root)
    └── X
       (Root / A)
```

代码虽不存在错误，但遍历时的副作用并不适配当前需求。

# BFS

那么是否存在既可以遍历树，又具备层间视野的算法？显然是存在的，也是本文的写作目的。

```java
    /**
 * 生成每个节点的 Route 数据
 */
public void generateRouteBFS() {
    bfs(body);
}

public static void bfs(BusinessNode root) {
    if (root == null) {
        return;
    }

    Queue<BusinessNode> queue = new LinkedList<>();
    queue.offer(root);

    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<BusinessNode> currentLevel = new ArrayList<>();

        // 收集当前层所有节点
        for (int i = 0; i < levelSize; i++) {
            currentLevel.add(queue.poll());
        }

        // 统计原始 name 的出现次数（用于生成后缀）
        Map<String, Integer> nameCount = new HashMap<>();
        for (BusinessNode node : currentLevel) {
            String key = node.getRoute() + "_" + node.getName();
            nameCount.put(key, nameCount.getOrDefault(key, 0) + 1);
        }

        // 再次遍历，对重复 name 进行重命名
        // 记录当前 name 已重命名到第几个
        Map<String, Integer> nameIndex = new HashMap<>();
        for (BusinessNode node : currentLevel) {
            String key = node.getRoute() + "_" + node.getName();
            int total = nameCount.get(key);

            if (total > 1) {
                int index = nameIndex.getOrDefault(key, 0);
                if (index != 0) {
                    // 第一次出现：保持原名
                    // 第二次及以上：重命名为 originalName_index
                    node.setName(node.getName() + "_" + index);
                }
                nameIndex.put(key, index + 1);
            }

            // 将子节点加入队列，用于下一层处理
            if (node.getChildren() != null) {
                String route = (node.getRoute() == null || node.getRoute().isEmpty())
                        ? "Body" : node.getRoute() + " / " + node.getName();
                for (BusinessNode child : node.getChildren()) {
                    child.setRoute(route);
                    queue.offer(child);
                }
            }
        }
    }
}
```

在 BFS 的同层处理中，可以一次性“看见”所有节点，因此天然具备“层级对比”和“局部唯一性校验”的上下文。
也就是说：

- DFS 是“从上到下”，每次只聚焦一条路径；
- BFS 是“按层推进”，每层形成一个完整的视野；
- 对“路径 + 节点名必须唯一”这类规则，只有 BFS 能提供完整上下文。

需要注意的是，在获取下一层的 Node 时，需要同步生成 `route` ，否则就会出现同层且父节点 `route` 不同，但当前层节点依然会被视为重复的场景，也就是

```text
String route = (node.getRoute() == null || node.getRoute().isEmpty()) ? "Body" : node.getRoute() + " / " + node.getName();
for (BusinessNode child : node.getChildren()) {
    child.setRoute(route);
    queue.offer(child);
}
```

在入队时，实时生成子节点的 `route`。

# 总结

DFS 与 BFS 虽然都是针对树的遍历，但选择哪个需要综合考虑其副作用和特性。












