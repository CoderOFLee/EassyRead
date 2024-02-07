## Contribution
- 提出了一种全新的代码表征模式。基于AST，使用更多语义方面的图或者关系，在树中添加边，得到一个多维度图

- 提出了一个带Conv模块的GGRN（Gated Graph Recurrent Network），在图层级进行分类

- 实现了Devign漏洞检测系统，在当时提高了SOTA。开源代码和数据集
```
https://sites.google.com/view/devign
```

## Overview
源代码 -> 图嵌入 -> GGRN学习 -> Conv分类

## Problem Formulation
- 设数据集中的一条数据为 $\{(c_i,\ y_i)\ |\ c_i \in \mathcal C,\ y_i \in \mathcal Y\},\ i \in \{1,\ 2,\ ...,\  n\}$ ，其中 $\mathcal C$ 表示函数实例集合， $\mathcal Y=\{0,\ 1\}^n$ 表示该函数是否易受攻击（1是，0否） 

- 每一个函数实例 $c_i$ 经过图嵌入层，得到一个图 $g_i\{V,X,A\} \in \mathcal G$ ，其中 $V$ 是点集， $m$ 是点集中的元素个数， $X= \mathbb R^{m\times d}$ 是每个点的特征矩阵， $d$ 是每个特征向量的维度， $A=\{0,\ 1\}^{k\times m\times m}$ 是点的邻接矩阵， $k$ 是边的种类

    e.g. 邻接矩阵 $A$ 中某个元素 $e_{s,t}^{p}$ 值为 $1$ ，表示存在一条类别为 $p$ 的有向边，从点 $v_s$ 连接到点 $v_t$

- 模型目标是学习一个映射函数 $f：\mathcal G \to \mathcal Y$ ，可以通过如下最小化损失函数学习得到： $\min \sum_{i=1}^{n}\mathcal L(f(g_i(V,\ X,\ A)\ ,\ c_i\ |\ y_i))\ +\ \lambda\omega(f)$

## Graph Embedding
其目的是为了将实例 $c_i$ 转变为图 $g_i(V,\ X,\ A)$ ，可以形式化表示为 $g_i(V,X,A)=E\ M\ B(c_i), \forall \ i \in \{1,\ 2,\ ...,\ n\}$

### 经典图嵌入
- AST：只能识别出参数不安全的漏洞

- AST+Control Flow：额外识别出两种错误，资源泄露和释放后使用问题

- AST+Control Flow+Data Flow：除了两种错误，可以识别大多数（依赖于运行时属性的竞争条件和如果没有程序预期设计的详细信息就难以建模的设计错误）

### 图嵌入层
- AST（Abstract Syntax Tree，抽象语法树）：Devign代码表征图的基础架构，但是只考虑语句的AST，跟高层次放弃语法树的边

- CFG（Control Flow Graph，控制流图）：控制流图描述程序在执行过程中可能的所有路径，以函数头为根节点，语句为最小节点进行，将语句的AST连接起来

- DFG（Data Flow Graph，数据流图）：数据流图描述程序在运行期间，变量相关的语句，同CFG层次，在语句层面将前后对同一变量有关联的语句连接起来

- NCS（Natural Code Sequence，自然代码序列）：将上述图的所有叶子节点，按照自然代码的顺序连接起来

- 综上定义实例 $c_i$ 对应的图表征 $g_i$ 。图表征 $g_i$ 是由AST/ CFG/ DFG/ NCS四个子图合并而来，四个子图共享点集 $\mathcal V=\mathcal V^{ast}$ 。每组节点包含两个域 $Code$ 和 $Type$ ，其中 $Code$ 域代表节点的代码，通过在整个项目的语料库上预训练的 word2vec 模型编码得来， $Type$ 表示节点类型，通过标签赋值。拼接两个域向量，得到节点向量 $x_\nu$

## Gated Graph Recurrent Layer
- 对于实例 $c_i$ 经过图嵌入层，得到图 $g_i(V,X,A)$ ，对于图中任意一个节点 $v_j \in \mathcal V$ ，定义其节点状态向量为 $h_{j}^{t}\in \mathbb R^{z},\ z \gt d,\ 1\leq t\leq T,\ h_j^t=[\ x_j^\intercal,\ \mathbf{0}^{z-d}]^\intercal$，其中 $z$ 是向量长度，高于特征维是为了隐藏状态的扩展， $t$ 是时间步， $T$ 是总的时间步数

- 对于 $t\leq T$ ， $p$ 类型路径的邻接矩阵$\ A_p\in\mathbb R^{m\times m}$ ，节点 $v_j$ 的邻点影响为 $a_{j,p}^{t-1}=A_p^\intercal\ (\ W_p\ [\ h_1^{t-1},\ ...,\ h_m^{t-1}\ ]+b\ )$ ，根据GRU，节点新状态应由自身和周边节点影响共同决定，那么就有 $h_j^t=G\ R\ U\large(\ \normalsize h_j^{t-1},\ A\ G\ G(\ {\{a_{j,p}^{t-1}\}}_{p=1}^k\ )\ \large)\normalsize$ ，其中 $ AGG$ 是多种邻接点的聚合方式，文中使用 $SUM$ ，理论可从 $\{\ SUM,\ MEAN,\ MAX,\ CONCAT\ \}$ 任选

- $H^T=\{h_1^T,\ ...,\ h_m^T\}$ 是最终状态

## Classfication
- 图分类的标准方法： $\widetilde y_i = Sigmoid\large (\ \normalsize MLP(H_i^T,\ x_i)\ \large)\normalsize$

- GGRN已经将图中的特征学习完毕，本层没有特征学习的任务，只需要进行分类，所以不需要连入CNN/Dense/Softmax/Sort-Pooling等网络

- 定义一个带最大池化层的一维卷积单元 $\sigma(\cdot)=MAXPOOL(\ \mathcal Relu(\ CONV(\cdot)\ )\ )$

- 设 $l$ 为卷积层数，那么有如下公式：
$\displaystyle
\begin{align}
Z_i^1=\sigma(\ H_i^T,\ x_i\ ),\ ...,\ Z_i^l=\sigma(\ Z_i^{l-1}\ )
\\
Y_i^1=\sigma(\ H_i^T\ ),\ ..., \ Y_i^l=\sigma(\ Y_i^{l-1}\ )
\\
\widetilde y_i = Sigmoid\large (\ \normalsize AVG(\ MLP(Z_i^l)\odot\ MLP(Y_i^l)\ )\ \large)\normalsize
\end{align}
$

## Experiment
### Dataset
- 根据四个开源原件的补丁文件进行人工标注（Linux Kernel，QEMU，Wireshark，FFmpeg）

### Graph Generate
- 使用joern工具得到AST和DFG，但是joern生成的DFG非常庞大

- 使用DFG-W、DFG-C、DFG-R代替，分别展示了变量最后一次写、来源、最后一次读，通过这三个子图，足够实验添加边

### Baseline
- Metrics+Xgboots
- 3-Layer BiLSTM
- 3-Layer BiLSTM + slef-attention

### Result
- Devign是SOTA

- 符合图的准确定整体高于单边图

- Devign对CWE最新漏洞的检测准确率为74左右

# 总结
代码表征使用符合图，使用GGRN进行特征学习，再使用Conv-MLP进行分类