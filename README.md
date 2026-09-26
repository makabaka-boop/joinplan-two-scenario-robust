# joinplan

无数据库依赖的 JSON 连接顺序规划器（纯 Python 标准库）。给定 2～9 张表、
表间谓词及其有理选择率，枚举所有合法二叉连接树，返回总代价最小的树。

## 模型

- 允许任意二叉连接树，但每次合并的两侧之间必须至少有一条谓词。
- 子树估计行数 = 子树内所有基表行数之积 × 子树内部全部谓词选择率之积
  （精确有理数运算）。
- 总代价 = 所有非叶节点估计行数之和。
- 并列时：每个内部节点的左右子树按树串字节序排列，取全括号树串字典序
  最小者。
- 可选第二情形：`alternative_selectivities` 与 `predicates` 按下标一一对应；
  启用后只允许 2～6 张表，连接合法性仍只由原谓词图决定。每棵树分别用精确
  有理数计算两种数据分布下的节点行数与总代价，目标顺序为：
  1. 最小化 `max(原总代价, 第二情形总代价)`；
  2. 最小化两种总代价之和；
  3. 取规范树串字典序最小者。
- 谓词图不连通时，返回各连通分量（各自给出最优子计划）。

## 输入（stdin 或文件参数）

```json
{
  "tables": [{"name": "A", "rows": 100}, {"name": "B", "rows": 200}],
  "predicates": [{"left": "A", "right": "B", "selectivity": "1/10"}]
}
```

- `tables`：2～9 张，表名为唯一可打印 ASCII（不含括号），`rows` 为正整数。
- `predicates`：可省略；`left`/`right` 必须是不同的已声明表；同一表对
  可有多条谓词。`selectivity` 为 `[0,1]` 内的有理数，写作 `"p/q"`
  （也接受整数 `0`/`1`；未约分的分数会自动约分）。
- `alternative_selectivities`：可选第二情形选择率数组，长度必须等于谓词数，
  元素采用同样的分数字面量；提供后表数量限制为 2～6。它不改变连接谓词图，
  只改变第二套估计行数与总代价。无谓词时可提供空数组。

## 输出

连通：

```json
{"status": "ok", "cost": "27000/1", "tree_string": "((AB)C)", "tree": {...}}
```

启用第二情形时，顶层还会返回 `"alternative_cost": "p/q"`；每个连接节点同时
返回原情形 `"rows"` 和第二情形 `"alternative_rows"`，该节点首次生效的每条谓词
还会带 `"alternative_selectivity"`。

不连通：

```json
{"status": "disconnected", "components": [{"tables": [...], "cost": "p/q",
  "tree_string": "...", "tree": {...}}, ...]}
```

启用第二情形时，每个 component 也会有自己的 `"alternative_cost"`；断连分量仍
分别规划，不生成跨分量连接。

输入非法时输出 `{"status": "error", "error": "..."}` 并以退出码 2 结束。

树节点：叶子为 `{"type": "table", "name": ..., "rows": <int>}`；连接节点为
`{"type": "join", "rows": "p/q", "tables": [...], "predicates": [...],
"children": [left, right]}`，其中 `predicates` 是在该次合并首次生效的谓词
（按输入顺序），`children` 按子树串字节序排列。所有有理数（`cost`、连接
节点 `rows`）以约分后的 `"p/q"` 字符串表示。启用第二情形时，连接节点额外
包含 `"alternative_rows"`，谓词额外包含 `"alternative_selectivity"`；未提供
第二数组时输入输出字段与单情形完全兼容。

## 运行

本地：

```sh
python3 joinplan.py < examples/chain3.json
python3 joinplan.py examples/bushy4.json
python3 joinplan.py examples/dual4.json
```

Compose（`joinplan` 服务）：

```sh
docker compose build joinplan
docker compose run -T joinplan < examples/chain3.json
```

## 测试

pytest 对小图枚举所有合法二叉树（不做子集剪枝），与规划器对拍总代价和
并列裁决，并逐节点校验估计行数、首次生效谓词、子节点顺序与合并合法性。
第二情形测试独立枚举全部合法树，对拍
`(最大总代价, 总代价之和, 规范树串)` 目标顺序、两情形并列结果及双份节点
估计：

```sh
python3 -m pytest -q                      # 本地
docker compose run --rm joinplan-tests    # 或经 Compose
```
