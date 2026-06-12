# TuGraph 图算法实验报告：基于中心性分析的数字货币反洗钱风险识别

## 1. 问题设计与实际金融意义

### 1.1 选择的图算法

本次实验选择图的中心性分析方法。中心性算法用于衡量图中节点的重要程度，在金融交易网络中可以用于识别资金汇聚节点、资金分发节点和潜在风险枢纽。

在本实验的数据集中，每一个 `transaction` 节点表示一笔比特币交易，每一条 `transfer` 边表示交易之间的有向关联关系。若某个交易节点连接了大量其他交易，则说明该节点在交易网络中处于较重要的位置，可能承担资金汇聚、资金分发或中转的作用。

### 1.2 设计的问题

本实验设计的问题为：

```text
在比特币交易网络中，如何利用中心性分析识别具有反洗钱关注价值的高风险交易枢纽，并进一步分析其资金汇聚和风险扩散关系？
```

该问题比单纯查询两笔交易之间是否存在路径更贴近实际金融风控场景。在数字货币反洗钱分析中，监管机构和金融机构不仅关注某一笔交易是否异常，还关注异常交易是否处于网络中心位置、是否接收大量来源交易、是否继续向外传播风险。

### 1.3 实际金融意义

在反洗钱和数字货币风险控制中，高中心性的交易节点具有较强分析价值：

| 分析对象 | 金融含义 |
|---|---|
| 入度较高的风险交易 | 可能是资金汇聚点，多个来源交易集中流向同一笔风险交易 |
| 出度较高的风险交易 | 可能是资金分发点，风险资金继续扩散到多个后续交易 |
| 与 `class=1` 交易距离较近的节点 | 可能存在风险传导，需要进一步审查 |
| 连接大量 `unknown` 交易的风险节点 | 可能隐藏未识别风险，适合纳入重点监测 |

因此，本实验不只是完成图算法调用，而是尝试模拟一个具体金融问题：利用 TuGraph 从交易网络中发现值得重点关注的风险交易节点，并解释这些节点在反洗钱分析中的意义。

---

## 2. 实验原理

### 2.1 数据建模

本实验继续使用 Transactions Dataset 数据集，将比特币交易数据建模为一张有向图：

| 类型 | 名称 | 含义 |
|---|---|---|
| 点类型 | `transaction` | 表示一笔比特币交易 |
| 边类型 | `transfer` | 表示两笔交易之间的有向关联关系 |

点数据来自：

```text
elliptic_txs_classes.csv
```

主要字段如下：

| 字段 | 含义 |
|---|---|
| `txId` | 交易编号 |
| `class` | 交易类别 |

边数据来自：

```text
elliptic_txs_edgelist.csv
```

主要字段如下：

| 字段 | 含义 |
|---|---|
| `txId1` | 起点交易编号 |
| `txId2` | 终点交易编号 |

边的方向为：

```text
transaction(txId1) -> transaction(txId2)
```

### 2.2 中心性分析原理

中心性分析用于衡量节点在图网络中的重要程度。由于本实验数据集中边没有额外权重，因此可以使用节点的连接数量作为中心性分析的基础。

本实验重点关注两个指标：

| 指标 | 含义 | 金融解释 |
|---|---|---|
| 入度中心性 | 指向某节点的边数量 | 反映该交易接收多少来源交易，适合识别资金汇聚点 |
| 出度中心性 | 某节点指向其他节点的边数量 | 反映该交易流向多少后续交易，适合识别资金分发点 |

对于 `class=1` 的风险交易，如果其入度较高，说明大量交易流向该风险节点，可能存在资金归集或洗钱链条中的汇聚行为；如果其出度较高，则说明风险交易继续向多个后续交易扩散，可能形成风险传播链。

---

## 3. TuGraph 具体操作过程

### 3.1 使用阿里云 TuGraph 登录

本实验使用阿里云环境中的 TuGraph 平台。打开 TuGraph 登录页面后，按照页面中给出的连接地址、账号和密码登录。

登录信息如下：

```text
连接方式：bolt
连接地址：8.140.39.221:7687
账号：admin
密码：以实验平台登录页显示为准
```

由于报告需要上传到 GitHub，密码不建议直接写在公开 README 中。如果提交截图中包含密码，建议在公开仓库中对密码进行遮挡。

登录界面如下：

<p align="center">
  <img src="images1/1.png" alt="阿里云 TuGraph 登录界面" width="850">
</p>
<p align="center"><b>图 1 阿里云 TuGraph 登录界面</b></p>

### 3.2 进入实验图项目

登录成功后，进入已经导入 Transactions Dataset 的图项目。本实验使用的图模型如下：

| 类型 | 名称 | 说明 |
|---|---|---|
| 点 | `transaction` | 比特币交易节点 |
| 边 | `transfer` | 交易之间的有向关系 |

图模型中，`transfer` 边从一个 `transaction` 节点指向另一个 `transaction` 节点，表示交易网络中的有向连接关系。

图模型界面如下：

<p align="center">
  <img src="images2/02_graph_schema.png" alt="图模型" width="850">
</p>
<p align="center"><b>图 2 transaction 与 transfer 图模型</b></p>

### 3.3 查询风险交易的入度中心性

首先查询 `class=1` 风险交易中入度最高的节点。入度越高，表示越多交易指向该风险交易，在金融意义上可能代表资金汇聚点。

Cypher 代码如下：

```cypher
MATCH (src:transaction)-[:transfer]->(hub:transaction)
WHERE hub.class = '1'
RETURN hub.txId AS risk_tx,
       hub.class AS risk_class,
       count(src) AS in_degree
ORDER BY in_degree DESC
LIMIT 10;
```

如果 `class` 字段导入为数值类型，则条件改为：

```cypher
WHERE hub.class = 1
```

运行结果中，交易 `84460750` 是一个值得重点关注的风险节点。根据本地数据统计，该节点属于 `class=1`，入度为 68，说明有大量交易流向该风险交易，具有明显的资金汇聚特征。

查询结果截图如下：

<p align="center">
  <img src="images2/03_in_degree_result.png" alt="风险交易入度中心性查询结果" width="850">
</p>
<p align="center"><b>图 3 风险交易入度中心性查询结果</b></p>

### 3.4 查询风险交易的出度中心性

随后查询 `class=1` 风险交易中出度较高的节点。出度越高，表示该风险交易继续指向更多后续交易，在金融意义上可能代表资金分发或风险扩散。

Cypher 代码如下：

```cypher
MATCH (hub:transaction)-[:transfer]->(dst:transaction)
WHERE hub.class = '1'
RETURN hub.txId AS risk_tx,
       hub.class AS risk_class,
       count(dst) AS out_degree
ORDER BY out_degree DESC
LIMIT 10;
```

运行结果中，可以重点观察出度较高的 `class=1` 交易。例如交易 `97011545` 连接了多个后续交易，适合继续分析其风险扩散路径。

查询结果截图如下：

<p align="center">
  <img src="images2/04_out_degree_result.png" alt="风险交易出度中心性查询结果" width="850">
</p>
<p align="center"><b>图 4 风险交易出度中心性查询结果</b></p>

### 3.5 分析资金汇聚节点 `84460750`

为了进一步解释中心性结果，选择入度最高的风险节点 `84460750` 进行展开分析。首先查询流向该风险节点的交易来源。

Cypher 代码如下：

```cypher
MATCH (src:transaction)-[:transfer]->(hub:transaction {txId:'84460750'})
RETURN src.txId AS source_tx,
       src.class AS source_class,
       hub.txId AS risk_hub,
       hub.class AS hub_class
LIMIT 30;
```

如果 `txId` 为数值类型，则写为：

```cypher
MATCH (src:transaction)-[:transfer]->(hub:transaction {txId:84460750})
RETURN src.txId AS source_tx,
       src.class AS source_class,
       hub.txId AS risk_hub,
       hub.class AS hub_class
LIMIT 30;
```

该查询用于观察哪些交易流向风险节点。若来源交易中包含大量 `unknown` 类别，说明许多未识别交易与风险节点发生连接，在金融风控中需要进一步关注。

查询结果截图如下：

<p align="center">
  <img src="images2/05_hub_in_sources.png" alt="资金汇聚来源分析" width="850">
</p>
<p align="center"><b>图 5 风险交易 84460750 的资金汇聚来源</b></p>

继续查询该风险节点的后续流向：

```cypher
MATCH (hub:transaction {txId:'84460750'})-[:transfer]->(dst:transaction)
RETURN hub.txId AS risk_hub,
       hub.class AS hub_class,
       dst.txId AS next_tx,
       dst.class AS next_class;
```

该查询用于观察风险节点是否继续向外扩散。根据本地数据，`84460750` 指向交易 `87722775`，其类别为 `unknown`，说明风险交易可能进一步连接到尚未识别类别的交易节点。

查询结果截图如下：

<p align="center">
  <img src="images2/06_hub_out_target.png" alt="风险节点后续流向" width="850">
</p>
<p align="center"><b>图 6 风险交易 84460750 的后续流向</b></p>

### 3.6 分析风险扩散节点 `97011545`

为了进一步体现交易网络中的风险扩散关系，选择另一个 `class=1` 节点 `97011545` 进行两跳路径分析。

Cypher 代码如下：

```cypher
MATCH (risk:transaction {txId:'97011545'})-[:transfer]->(mid:transaction)-[:transfer]->(target:transaction)
RETURN risk.txId AS risk_tx,
       risk.class AS risk_class,
       mid.txId AS mid_tx,
       mid.class AS mid_class,
       target.txId AS target_tx,
       target.class AS target_class
LIMIT 20;
```

该查询不是单纯为了验证路径是否存在，而是用于分析风险交易向后续交易传播的具体链条。根据本地数据，可以观察到如下路径：

```text
97011545 -> 172562 -> 97774107
97011545 -> 172562 -> 98034161
97011545 -> 97731384 -> 97731146
```

其中，`97011545` 为 `class=1` 风险交易，后续节点中既包含 `class=2`，也包含 `unknown` 和其他 `class=1` 节点。这说明风险交易可能通过多跳关系连接到不同类别的交易，具有进一步风险排查价值。

查询结果截图如下：

<p align="center">
  <img src="images2/07_risk_diffusion_paths.png" alt="风险扩散路径分析" width="850">
</p>
<p align="center"><b>图 7 风险交易 97011545 的两跳扩散路径</b></p>

---

## 4. 代码汇总

### 4.1 风险交易入度中心性

```cypher
MATCH (src:transaction)-[:transfer]->(hub:transaction)
WHERE hub.class = '1'
RETURN hub.txId AS risk_tx,
       hub.class AS risk_class,
       count(src) AS in_degree
ORDER BY in_degree DESC
LIMIT 10;
```

### 4.2 风险交易出度中心性

```cypher
MATCH (hub:transaction)-[:transfer]->(dst:transaction)
WHERE hub.class = '1'
RETURN hub.txId AS risk_tx,
       hub.class AS risk_class,
       count(dst) AS out_degree
ORDER BY out_degree DESC
LIMIT 10;
```

### 4.3 资金汇聚来源分析

```cypher
MATCH (src:transaction)-[:transfer]->(hub:transaction {txId:'84460750'})
RETURN src.txId AS source_tx,
       src.class AS source_class,
       hub.txId AS risk_hub,
       hub.class AS hub_class
LIMIT 30;
```

### 4.4 风险节点后续流向

```cypher
MATCH (hub:transaction {txId:'84460750'})-[:transfer]->(dst:transaction)
RETURN hub.txId AS risk_hub,
       hub.class AS hub_class,
       dst.txId AS next_tx,
       dst.class AS next_class;
```

### 4.5 风险扩散路径分析

```cypher
MATCH (risk:transaction {txId:'97011545'})-[:transfer]->(mid:transaction)-[:transfer]->(target:transaction)
RETURN risk.txId AS risk_tx,
       risk.class AS risk_class,
       mid.txId AS mid_tx,
       mid.class AS mid_class,
       target.txId AS target_tx,
       target.class AS target_class
LIMIT 20;
```

---

## 5. 结果含义分析

通过中心性查询可以发现，交易 `84460750` 是一个典型的高入度风险交易节点。该节点属于 `class=1`，并且有大量交易流向它。从反洗钱角度看，这类节点可能对应资金归集、风险资金集中或洗钱链条中的中转环节。

进一步观察该节点的来源交易可以发现，部分来源交易属于 `unknown` 类型。这说明虽然这些来源交易尚未被明确标记为风险交易，但它们与已知风险节点存在直接连接关系，因此在实际风控中应被纳入进一步监测范围。

交易 `97011545` 的分析展示了另一类风险场景：风险交易不仅可能接收资金，还可能继续连接到后续交易。通过两跳路径可以观察到风险节点与 `class=2`、`unknown` 和其他 `class=1` 节点之间的连续连接关系。这说明风险并不只停留在单个节点，而可能沿交易网络继续扩散。

因此，本实验说明图数据库和图算法能够帮助金融机构从交易网络整体结构出发识别风险，而不是只依赖单笔交易的孤立属性。中心性分析可以用于筛选重点交易节点，路径分析可以进一步解释风险传播过程，两者结合有助于提高数字货币反洗钱分析的效率。

---

## 6. 学习和使用 TuGraph 平台的感受

通过本次实验，我进一步理解了 TuGraph 在金融交易网络分析中的应用价值。相比普通表格数据，图数据库能够更加自然地表达交易之间的连接关系，并通过 Cypher 查询快速找到网络中的重要节点和关联路径。

在实验过程中，我体会到图建模的准确性非常重要。只有正确建立 `transaction` 点和 `transfer` 边，并正确映射边的起点和终点，后续中心性分析和路径分析才有意义。尤其是在反洗钱场景中，交易关系本身往往比单笔交易属性更重要，图数据库能够很好地支持这种关系型分析。

TuGraph 的可视化界面也便于观察交易网络结构。通过查询结果和图形展示，可以直观看到风险交易的资金汇聚和扩散情况，这对理解区块链交易追踪、异常交易识别和金融风控都有帮助。

---

## 7. GitHub 提交说明

提交 GitHub 时，建议包含以下内容：

```text
README_作业2.md
images1/
images2/
```

其中，`images1/1.png` 用于展示阿里云 TuGraph 登录界面，`images2` 文件夹用于存放本实验的运行截图。

截图文件名建议与本文中的引用保持一致：

| 文件名 | 内容 |
|---|---|
| `images1/1.png` | 阿里云 TuGraph 登录界面 |
| `images2/02_graph_schema.png` | 图模型截图 |
| `images2/03_in_degree_result.png` | 风险交易入度中心性结果 |
| `images2/04_out_degree_result.png` | 风险交易出度中心性结果 |
| `images2/05_hub_in_sources.png` | 资金汇聚来源分析 |
| `images2/06_hub_out_target.png` | 风险节点后续流向 |
| `images2/07_risk_diffusion_paths.png` | 风险扩散路径分析 |

上传完成后，将 GitHub 仓库链接提交到学习通即可。
