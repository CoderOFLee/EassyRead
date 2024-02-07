## Contribution
- 提出一种图简化的代码表征，以解决节点的远程依赖问题

- 提出一种图神经网络增强学习的方法

- 开发了一个系统AMPLE，其在3个数据集上拥有最优性能
```
https:// github.com/AMPLE001/AMPLE
```

## Framework
AMPLE包含两个大阶段，第一个是代码表征的图简化，第二个是特征提取的增强GNN学习

Source Code -> 

Graph Simplification ( Graph Generating -> TGS -> VGS ) -> 

( word2vec -> ) Enhanced Graph Representation Learning ( Edge-Aware GCN module -> Kernel-scaled representation module ) ->

( Binary Classification （2-Layer FC -> Softmax） )

## Graph Simlification
### Graph Generating
文中使用 joern 工具用作 AST、 PDG、 DFG 的初始图生成，然后同 Devign 同样处理，得到一个基于 AST 的复合图

### Type-base Graph Simplification
- 定义：根据复合图中的节点类型进行节点删除。根据解析原则$\ ^1$和手工检查生成的P-C简化规则进行相邻节点删除

- P-C简化规则，即根据父（Parent）节点的类型和子（Child）节点的类型进行子节点的删除，父节点继承子节点的边，因为子节点的会在父节点和后续节点中被反映出来

|Rule|P-Type|C-Type|
|--|--|--|
|1|Expression Statement|Assignment Expression<br>Unary Expression<br>Call Expression<br>Post Inc Dec Operation Expression
|2|Indentifier Declare Expression|Indentifier Declaraion
|3|Condition|*
|4|ForInit|*
|5|Call Expression|Argument List
|6|Argument|*
|7|Callee|Identifier

### Variable-based Graph Simplification
- 定义：根据复合图中的变量类型进行不改变层级关系的叶节点合并。

- 一个形象的说法就是，根据NCS边遍历叶节点，重复出现的同一变量节点，仅保留首次出现的叶子节点以及后续叶子的边

- 此时一个叶子节点的父节点可能有多个，这样就利用了跨语句的语义信息

### Algorithm1：Graph Simplification
$
\mathbf {Input\ \ \ :} \text{ Original Code Structure Graph: } Graph_{Original}\\
\mathbf{Output:} \text{Simplified Code Structure Graph: } Graph_{Original}\\
\mathbf{Function} \text{ GS:}\\
\quad \text{// The procedure of TGS}\\
\quad Graph_{TGS} \leftarrow Graph_{Original}\\
\quad \mathbf{for}\ each\ AST\ \mathbf T\ \in\ Graph_{TGS}\ \mathbf{do}\\
\quad\quad \text{// Do breadth-first traversal}\\
\quad\quad Stack\ \leftarrow\ \empty\\
\quad\quad Pushing\ \mathbf T.root\ into\ Stack\\
\quad\quad \mathbf{while}\ Stack\ \ne\ \empty\ \mathbf{do}\\
\quad\quad\quad u \leftarrow\ Pop\ Stack \\
\quad\quad\quad Children\ \leftarrow\ \text{Get all the child nodes of }u\\
\quad\quad\quad \mathbf{for}\ v\in\ Children\ \mathbf{do}\\
\quad\quad\quad\quad \mathbf{if}\ (u.Type,\ v.Type)\ conform\ to\ rules\ Table\ 1\ \mathbf{then}\\
\quad\quad\quad\quad\quad Delete\ edit\ (u,\ v)\ and\ v\ from\ Graph_{TGS};\\
\quad\quad\quad\quad\quad Add\ edges\ between\ u\ and\ all\ child\ nodes\ of\ v;\\
\quad\quad\quad\quad\quad Push\ all\ child\ nodes\ of\ v\ into\ Stack;\\
\quad\quad\quad\quad \mathbf{else}\\
\quad\quad\quad\quad\quad Push\ v\ into\ Stack;\\
\quad\quad\quad\quad \mathbf{end\ if}\\
\quad\quad\quad\ \mathbf{end\ for}\\
\quad\quad \mathbf{end\ while}\\
\quad \mathbf{end\ for}\\
\quad \text{// The procedure of VGS}\\
\quad Graph_{GS}\ \leftarrow\ Graph_{TGS}\\
\quad leaf_nodes\ \leftarrow\ \text{get all the leaf nodes of AST in }Graph_{GS}\\
\quad same_variable\ \leftarrow\ \text{get all the node groups with the same variable in }leaf\_ nodes\\
\quad\mathbf{for}\ each\ group\ \in\ same\_ bariable\ \mathbf{do}\\
\quad\quad \text{Merging all the nodes in the } group \text{ into one variable node}\\
\quad \mathbf{end\ for}\\
\mathbf{return\ }Graph_{GS}
$

## Enhanced Graph Representation Learning
### Node->vector
使用word2vec完成，$d$ 为向量维度

### EA-GCN：Edge-Aware Graph Convolutional Network Module
- EA-GCN 首先通过对不同类型边的加权计算节点向量，然后通过多头注意力（multi-head attention mechanism）加强向量的特征提取

- 定义经过之前的阶段，现在接受一个图 $G(\mathcal{V,\ E,\ R})$ ，其中 $v_i\in\mathcal V$ 是点集，每个点的初始向量状态为 $h_i^0\in\mathbb R^{d}$ ， $(\ v_i,\ \beta,\ v_j)\in \mathcal E$ 表示图中的一条从节点 $v_i$ 出发，在节点 $v_j$ 结束的类型为 $\beta\in\mathcal R$ 的有向边

- 考虑节点 $v_i\in\mathcal V$ 的一条 $\beta$ 类型的边，在第 $l$ 层网络中，权重矩阵定义为： $W_\beta^l=a_\beta^lV^l$ ，其中 $a_\beta$ 是可学习的权重

- 权重矩阵 $W_\beta^l$ 被用来更新节点向量：
$\\ \begin{align}\widetilde{h_i^l}=\sigma \large(\ \normalsize \sum_{\beta\in\mathcal E}\sum_{j\in\mathcal{N_i^\beta}}\ \frac{1}{c_i{,\beta}}\ W_\beta^l\ h_j^{l-1}\ +\  W_0^l\ h_i^{l-1}\ \large)\end{align}\\$

- 使用多头注意力机制更好的提取边缘信息。注意分数定义为： 
$\\ \begin{align}w_{i\rightarrow j}^{k,\ l}=softmax_j(\ \frac{\widetilde{h_i^{k,\ l}}\ \cdot\ \widetilde{h_j^{k,\ l}}}{\sqrt{d_k}}\ )\end{align}$

- 多头注意力的聚合通过下列定义进行：
$\\ \begin{align}A_i^l=CONCAT_{k=1}^P\normalsize\ (\ \sum_{j\in\mathcal N_i}\ w_{i\rightarrow j}^{k,\ l}\widetilde{h_j^{k,\ l}}\ )\ \times W_h^l+h_i^{l-1}\\ h_i^l=W_2^l\ \cdot\ \sigma(W_1^l\ A_i^l)+A_i^l\end{align}$

### Kernel-scaled Representation Module
- 捕获远程节点间的语义信息。用一个大核和一个小核并行计算，分别关注远节点和近节点

- 给定经过 EA-GCN 学习后的向量矩阵 $H_i^l$ ，以及大小核的大小 $N,\ M\ (M\lt N)$ ，可计算这层的输出为：
$\\ \begin{align} K^{out}=BN(\ H_i^l * W^L,\ \mu^L\ )+BN(\ H_i^l*W^S,\ \nu^S\ )\end{align}\\$
其中 $BN$ 是批量归一化（Batch Normalize）层， $\mu^L,\ \nu^S$ 分别是大核小核的 $BN$ 层数。 $W^L\in\mathbb R^{C_{out}\times C_{in}\times N},\ W^S\in\mathbb R^{C_{out}\times C_{in}\times M}$ 分别是大小卷积核

## Bianry Classification
- 2-Layer FC + Softmax

## Experiment
- 有效性：AMPLE是SOTA

- 图简化的有效性：通过将Original、TGS、VGS、GS分别喂同样的网络模型学习，发现GS的效果最好

- 图简化程度：通过节点和边数量精简率、节点距离精简率来对比发现，TGS和VGS都有很好的效果

- EA-GCN和Kernel-scaled的有效性：通过将MAPLE、MA-GCN、Kernel-scaled进行检测，发现MAPLE效果最好

- EA-GCN的可替代性：通过将GCN、R-GCN、GGNN替换EA-GCN，发现EA-GCN效果最好

- Kernel-scaled的可替代性：通过使用Dense层替换进行实验，发现Kernel-scaled效果最好

- 文中对大小卷积核的尺寸、EA-GCN的层数、注意力机制的头数等超参数进行了多次对比试验

## Explaination
- GS保证了数据的精简，缩短了语义距离，网络能够更有效更快的学习到特征

- EA-GCN抽取多种异构图的复合图信息

- Kernel-scaled关注了全局信息

- 多头注意力机制确保MAPLE对于漏洞语句的权重更高

## Limitation
- 数据集不够多，使用了3个常用数据集（FFMPeg+Qemu、Reveal、Fan）

- 局限与C/C++