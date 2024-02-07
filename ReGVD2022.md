## Contribution
- 提出了一个跨PL的漏洞检测系统ReGVD

## Problem Definition
- 设数据集中的一条数据为 $\{(c_i,\ y_i)\ |\ c_i \in \mathcal C,\ y_i \in \mathcal Y\},\ i \in \{1,\ 2,\ ...,\  n\}$ ，其中 $\mathcal C$ 表示函数实例集合， $\mathcal Y=\{0,\ 1\}^n$ 表示该函数是否易受攻击（1是，0否） 

- 每一个函数实例 $c_i$ 经过图嵌入层，得到一个图 $g_i\{V,X,A\} \in \mathcal G$ ，其中 $V$ 是点集， $m$ 是点集中的元素个数， $X= \mathbb R^{m\times d}$ 是每个点的特征矩阵， $d$ 是每个特征向量的维度， $A=\{0,\ 1\}^{ m\times m}$ 是点的邻接矩阵

    e.g. 邻接矩阵 $A$ 中某个元素 $e_{s,t}^{p}$ 值为 $1$ ，表示存在一条类别为 $p$ 的有向边，从点 $v_s$ 连接到点 $v_t$

- 模型目标是学习一个映射函数 $f：\mathcal G \to \mathcal Y$ ，可以通过如下最小化损失函数学习得到： $\min \sum_{i=1}^{n}\mathcal L(f(g_i(V,\ X,\ A)\ ,\ c_i\ |\ y_i))\ +\ \lambda\|(\theta)\|_2^2$

## Graph Contruction
### Overview
用别人的方法，将代码视作平面token序列，再转为两类图（unique-token-focused、index-focused），然后取消原方法种的自循环边
```
Lianzhe Huang, Dehong Ma, Sujian Li, Xiaodong Zhang, and Houfeng Wang. 2019.
Text Level Graph Neural Network for Text Classification. In EMNLP-IJCNLP.

Yufeng Zhang, Xueli Yu, Zeyu Cui, Shu Wu, Zhongzhen Wen, and Liang Wang.
2020. Every Document Owns Its Structure: Inductive Text Classification via
Graph Neural Networks. In ACL. 334–339.
```

### Unique-Token-Focused-Constructin
将函数中唯一令牌作为一个节点，根据给定窗口大小的滑动窗口（sliding window），确定一个邻接矩阵：$\mathbf A_{v,u}=
\begin{cases}
1\ \ \ token\ v,u\ appear\ together\ in\ sliding\ window,\ v\ne u; \\
0\ \ \ otherwise.
\end{cases}$

### Index-Focused-Contruction
将所有token都作为一个节点，不作去重操作，根据给定窗口大小的滑动窗口（sliding window），确定一个邻接矩阵：$\mathbf A_{i,j}=
\begin{cases}
1\ \ \ token\ i,j\ appear\ together\ in\ sliding\ window,\ i\ne j; \\
0\ \ \ otherwise.
\end{cases}$

### Node-Feature-Initialize
用预训练PL模型（如CodeBERT）的token嵌入层来对每个节点进行编码

## Feature Extracting
### Overview
代码嵌入 -> 残差GCN/残差GGNN -> Readout层（求和池化和最大池化融合） -> FC+Sofmax分类

### GNN
给定一个图 $g(V,X,A)$ ， 定义GNN: $\mathbf H^{k+1}=GNN(\mathbf A,\ \mathbf H^k),\ \mathbf H^0=X$

### GCN
$\overline h_v^{k+1}=\phi\ (\ \sum_{u \in N_v}\ a_{v,u}\overline W^k\overline h_u^k\ ),\ \forall \ v \in \mathcal V$

其中 $\phi$ 是非线性激活函数（如 $\mathcal Relu$）， $W$ 是权重矩阵， $a_{v,u}$ 是拉普拉斯再归一化邻接矩阵（Laplacian re-normalized adjacency matrix）中的值， $D^{-\frac{1}{2}}AD^{-\frac{1}{2}}$ 是拉普拉斯再归一化邻接矩阵，矩阵 $D$ 是原邻接矩阵 $A$ 的对角节点度矩阵（diagonal node degree matrix）

### GGNN
$a_{v}^{k+1}=\sum_{u\in N_v}a_v^k\overline h_u^k$

$z_v^{k+1}=\sigma(\ W^za_v^{k+1},\ U^zh_v^k\ )$

$r_v^{k+1}=\sigma(\ W^ra_v^{k+1},\ U^rh_v^k\ )$

$\widetilde h_v^{k+1}=\phi(\ W^oa_v^{k+1},\ U^o(\ r_v^{k+1}\odot h_v^k\ )\ )$

$h_v^{k+1}=(\ 1-z_v^{k+1}\ )\ \odot h_v^{k}\ + z_v^{k+1}\ \odot \widetilde h_v^{k+1}$

### Residual Connection
$H^{k+1}=H^{k}+GNN(\ A,\ H^k\ )$

### Readout
根据其他论文研究成果，发现和池化（sum pooling）对与图分类具有更好的性能，所以ReGVD使用和池化。同时最大池化（max pooling）能够提取更多图中信息，所以ReGVD将两种池化层进行融合

$e_v=\sigma (\ w^\intercal h^K_v +b\ )\ \odot\phi(\ Wh_v^K+\overline b\ )$

$e_g=MIX(\ \sum_{v\in\mathcal V}e_v,\ \mathcal MAXPOOL\{e_v\}_{v\in \mathcal V}\ )$

其中 $e_v$ 是节点 $v$ 的最终向量， $e_g$ 是函数的最终向量表示， $\sigma (\ w^\intercal h^K_v +b\ )$ 是软注意力机制（soft attention machanism）， $MIX$ 函数是集合 ${SUM,\ CONCAT,\ MUL}$ 中任意一个

$\small SUM:\ \normalsize e_g=\sum_{v\in\mathcal V}e_v+\mathcal MAXPOOL\{e_v\}_{v\in \mathcal V}$

$\small MUL:\ \normalsize e_g=\sum_{v\in\mathcal V}e_v\odot\mathcal MAXPOOL\{e_v\}_{v\in \mathcal V}$

$\small CONCAT:\ \normalsize e_g=\large[\ \normalsize\sum_{v\in\mathcal V}e_v\ \|\ \mathcal MAXPOOL\{e_v\}_{v\in \mathcal V}\ \large ] \normalsize$

### Classfication
$\widehat{y_g}=sofmax(\ W_1e_g+b_1\ )$

## Result
SOTA