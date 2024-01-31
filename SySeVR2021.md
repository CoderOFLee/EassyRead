## Contribution
- 提出新框架SySeVR(Syntax-based, Semantic-based and Vector Representations)
```
SyVCs：Syntax-based Vulnerability Candidates，基于语法的漏洞候选
SeVCs：Semantic-based Vulnerability Candidates，基于语义的漏洞候选
SeVCs在SyVCs上进行扩展，将数据流和控制流纳入考虑范围
并设计了自动获取SyVCs和SeVCs的算法
```

- 开放自NVD和SARD整合的数据集
```
含有126种漏洞信息，https://github.com/SySeVR/SySeVR
```

- 允许多种深度神经网络在框架内运行，其中BGRU效果最好

- 将控制流纳入考虑，融入了更多的语义信息

## Basic Idea
来自CNN中的框图概念。对于一个程序中，易受攻击的代码只有部分行，将整个程序区分为多个在语义（数据依赖和控制依赖）上关联的片段，检查这个片段是否是易受攻击的。

所以得到如下的框架：抽取语法片段->转为语义片段->向量化->深度学习网络

## Dataset
- NVD+SARD

- 真实值标签同VulDeePecker

- Merics：除了FNR、FPR、P、F1以外，添加 $A（Accuracy）=\frac{TP+TN}{TP+FP+TN+FN}$ 评价模型预测的准确度和 $MCC=\frac{TP*TN-FP*FN}{\sqrt{(TP+FP)(TP+FN)(FP+TN)(FN+TN)}}$ 评价模型预测与真值标签匹配的程度

## Extract SyVC
### Definition
- 漏洞具有自己的语法特征，定义为 $H=\{h_k\}, 1 \leq k \leq \beta$ ，$\beta$是漏洞语法特征总数
```
如果一个程序的具有语义关联的部分的语法特征跟H集合中的某个语法特征相匹配了，则将其认为是SyVC
```

- 程序P是函数 $f_1,...,f_\eta$ 组成的集合，写作 $P=\{f_1,...,f_\eta\}$ 

- 函数 $f_i$ 是由语句 $s_{i,1},...,s_{i,m_i}$ 组成的集合 ，写作 $f_i=\{s_{i,1},...,s_{i,m_i}\}, 1\leq i \leq \eta$ 

- 语句 $s_{i,j}$ 是由令牌（token） $t_{i,j,1},...t_{i,j,\omega_{i,j}}$ 组成的集合，写作 $s_{i,j}=\{t_{i,j,1},...t_{i,j,\omega_{i,j}}\}, 1\leq i \leq \eta , 1 \leq j \leq m_i$
```
一个SyVC可能对应一个令牌，也可能对应多个令牌组合。对应到程序的AST中，就是AST的叶子节点和中间节点
```

- 代码元素（code element） $e_{i,j,z}$ 是由若干令牌组成的，写作 $e_{i,j,z}=(s_{i,j,\mu},...,s_{i,j,\nu}),1 \leq \mu \leq \nu \leq \omega_{i,j}$ 。如果一个代码元素 $e_{i,j,z}$ 跟漏洞语法特征H集合中的某个特征匹配上，则这个代码元素 $e_{i,j,z}$ 被称为SyVC（Syntax-based Vulnerability Candidates, 基于语法的漏洞候选）

### Algorithm1：Extracting SyVCs from a program

$\small\text{Input: A program } P = \{ f_1, \ldots, f_{\eta} \}; \text{ a set } H = \{ h_k | 1 \leq k \leq \beta \} \text{ of vulnerability syntax characteristics} \\
\text{Output: A set } Y \text{ of SyVCs} \\
1: Y = \{\}; \\
2: \text{for each function } f_i \in P \text{ do} \\
3: \qquad\text{Generate an abstract syntax tree } T_i \text{ for } f_i; \\
4: \qquad\text{for each code element } e_{i,j,z} \text{ in } T_i \text{ do} \\
5: \qquad\qquad\text{for each } h_k \in H \text{ do} \\
6: \qquad\qquad\qquad\text{if } e_{i,j,z} \text{ matches } h_k \text{ then} \\
7: \qquad\qquad\qquad Y = Y \cup \{ e_{i,j,z} \}; \\
8: \qquad\qquad\qquad\text{end if} \\
9: \qquad\qquad\text{end for} \\
10: \qquad\text{end for} \\
11: \text{end for} \\
12: \text{return } Y \text{, the set of SyVCs}$

### H set
使用checkmarx工具提取数据集中的漏洞语法特征，经过人工检查得到如下四个特征：
|名称（缩写）|简介|漏洞对应数量|
|--|--|--|
|Libarary/API Function Call(FC)|811|106|
|Array Usage(AU)|元素访问、地址运算|87|
|Pointer Usage(PU)|指针算数、引用、传参|103|
|Arithmetic Expression(AE)|溢出|45|

注意：一个漏洞可能对应多个语法特征

### AST
文中使用Joern工具生成函数的AST

### Match
根据H集合中的语法特征进行条件匹配：
|特征|代码|代码元素$e_{i,j,z}$|匹配条件1|匹配条件2|
|--|--|--|--|--|
|FC|memset(buf,'\0')|memset|在AST中是一个callee节点|调用的函数是FC总结的811之一|
|AU|source[99]='a'|source|在AST中是一个identifier declaration节点|声明的节点中具有'['和']'字符|
|PU|data[99]='\0'|data|在AST中是一个identifier declaration节点|声明节点中具有'*'字符|
|AE|data=databuf-8|data=databuf-8|在AST中是一个expression节点|节点包含'='字符，并且'='右侧具有多个标识符|

## Transfrom SyVCs to SeVCs
### Definition
```
SyVC是从AST中截取而来，SeVC是从PDG中截取而来
```
- CFG(Control Flow Graph)：对于一个程序 $P=\{f_1,...,f_\eta\}$ 中的函数 $f_i$ ，给定一个图 $G_i = \{V_i, E_i\}$ , 其中 $V_i=\{n_{i,1},...,n_{i,c_i}\}$ 中的每一个节点（node） $n_{i,j}$ 都表示一个语句（statement）或控制谓词（control predicate）， $E_i=\{\epsilon_{i,1},...,\epsilon_{i,d_i}\}$ 中的每一个有向边 $\epsilon_{i,j}$ 都表示两个节点间可能的控制流（control flow）

- Data Dependency（数据依赖）：考虑一个程序 $P=\{f_1,...,f_\eta\}$ ，函数 $f_i$ 的CFG $G_i = \{V_i, E_i\}$ 中，有两个节点 $n_{i,j},n_{i,\gamma}, 1 \leq j,\gamma \leq c_i, j \ne \gamma$ ，有一条从 $n_{i,\gamma}$ 到 $n_{i,j}$ 的路径，并且 $n_{i,\gamma}$ 的值经过运算合并到 $n_{i,j}$ 中，则称 $n_{i,j}$ 数据依赖（Data Dependent on）$n_{i,\gamma}$

- Control Dependency（控制依赖）：考虑一个程序 $P=\{f_1,...,f_\eta\}$ ，函数 $f_i$ 的CFG $G_i = \{V_i, E_i\}$ 中的两个节点 $n_{i,j},n_{i,\gamma}, 1 \leq j,\gamma \leq c_i, j \ne \gamma$ ，如果从节点 $n_{i,\gamma}$ 出发到程序结束的所有路径都要经过 $n_{i,j}$， 则称节点 $n_{i,j}$ 后向支配（post-dominate）节点 $n_{i,\gamma}$ 。如果存在一条路径 $p$ ，从节点 $n_{i,\gamma}$ 出发，在节点 $n_{i,j}$ 结束，那么则有（i）节点 $n_{i,j}$ 后向支配路径 $p$ 上除头尾节点的所有节点；（ii）节点 $n_{i,j}$ 控制依赖（control dependent on）节点 $n_{i,\gamma}$

- **PDG(Program Dependency Graph)**：对于一个程序 $P=\{f_1,...,f_\eta\}$ 中的函数 $f_i$ ，给定一个图 $G_{i}^{'} = \{V_i, E_{i}^{'}\}$ , 其中 $V_i=\{n_{i,1},...,n_{i,c_i}\}$ 中的每一个节点（node） $n_{i,j}$ 都表示一个语句（statement）或控制谓词（control predicate）， $E_i=\{\epsilon_{i,1}^{'},...,\epsilon_{i,d_{i}^{'}}^{'}\}$ 中的每一个有向边 $\epsilon_{i,j}^{'}$ 都表示两个节点间的控制流

```
依旧使用程序切片的思路抽取漏洞相关代码，如下定义全部都基于上述所有定义（抽取SeVC和PDG）中的程序P、函数f、PDG G、SyVC ei,j,z
```
- Forward Slice（前向切片）：$fs_{i,j,z}=\{n_{i,x_1},...,n_{i,x_{\mu_i}}\}\subseteq\ V_i,\ where\ x_p\ satisfying\ 1 \leq x_1 \leq x_p \leq \mu_i \leq c_i$ ，并且切片中的所有节点都存在于图 $G^{'}$ 中自 $e_{i,j,z}$ 出发的路径中

- Interprocedual Forward Slice（程序间前向切片）：$fs_{i,j,z}^{'}$ 是 $fs_{i,j,z}$ 跨越或不跨越PDG的版本，即 $fs_{i,j,z}^{'}$ 中的所有节点都满足（i）节点可能属于一个或多个PDG；（ii）每一个节点都可以从 $e_{i,j,z}$ 出发，通过若干个函数调用到达

- Backword Slice（后向切片）：$bs_{i,j,z}=\{n_{i,y_1},...,n_{i,y_{\nu_i}}\}\subseteq\ V_i,\ where\ y_p\ satisfying\ 1 \leq y_1 \leq y_p \leq \nu_i \leq c_i$ ，并且切片中的所有节点都存在于图 $G^{'}$ 中在 $e_{i,j,z}$ 结束的路径中

- Interprocedual Backword Slice（程序间后向切片）：$bs_{i,j,z}^{'}$ 是 $bs_{i,j,z}$ 跨越或不跨越PDG的版本，即 $bs_{i,j,z}^{'}$ 中的所有节点都满足（i）节点可能属于一个或多个PDG；（ii）每一个节点都可以通过若干个函数调用到达 $e_{i,j,z}$

- **Program Slice（程序切片）**：SyVC $e_{i,j,z}$ 对应的 $fs_{i,j,z}^{'}$ 和 $bs_{i,j,z}^{'}$ 经过保序的去重合并，得到程序切片 $ps_{i,j,z}^{'}$
```
将SyVC对应的程序切片中的节点替换为源代码之后就能够得到对应的SeVC
```

- **SeVC**：SeVC就是SyVC对应的程序切片的源代码形式。形式化描述，对于程序P和SyVC $e_{i,j,z}$ ，对应的 SeVC 定义为 $\delta_{i,j,z}=\{s_{a_1,b_1},...,s_{a_{\nu_{i,j,z}},b_{\nu_{i,j,z}}}\}$ ，其中语句 $s_{a_p,b_q}\ ,\ 1 \leq a_p,\ b_q \leq \nu_{i,j,z}$ 和SyVC $e_{i,j,z}$ 之间存在数据依赖或者控制依赖

### Algorithm2：Transforming SyVCs to SeVCs in program P

$\small\text{Input: } A \text{ program } P = \{f_1, \ldots, f_\eta\};
\text{a set } Y \text{ of SyVCs generated by Algorithm 1}\\
\text{Output: The set of SeVCs}\\
1: C \gets \{\};\\
2: \text{for each } f_i \in P \text{ do}\\
3: \quad \text{Generate a PDG } G_{0i} = (V_i, E_{0i}) \text{ for } f_i;\\
4: \text{end for}\\
5: \text{for each } e_{ijz} \in Y \text{ in } G_{0i} \text{ do}\\
6: \quad \text{Generate forward slice } f_{sijz} \text{ \& backward slice } b_{sijz} \text{ of } e_{ijz};\\
7: \quad \text{Generate interprocedural forward slice } f_{s0ijz} \\
\quad\quad\ \text{ by interconnecting }
f_{sijz} \text{ and the forward slices from the functions called by } f_i;\\
8: \quad \text{Generate interprocedural backward slice } b_{s0ijz} \\\quad\quad\ \text{ by interconnecting } b_{sijz} \text{ and the backward slices from both the functions called
by } f_i \text{ and the functions calling } f_i;\\
9: \quad \text{Generate program slice } p_{sijz} \text{ by connecting } f_{s0ijz} \text{ and } b_{s0ijz} \text{ at }
e_{ijz};\\
10:\quad \text{for each statement } s_{ij} \in f_i \text{ appearing in } p_{sijz} \text{ as a node do}\\
11: \quad \quad \delta_{ijz} \gets \delta_{ijz} \cup \{f_{sij}\}, \text{ according to the order of the appearance of } s_{ij} \text{ in } f_i;\\
12: \quad \text{end for}\\
13: \quad \text{for two statements } s_{ij} \in f_i \text{ and } s_{apbq} \in f_{ap}
(i \neq ap) \text{ appearing
in } p_{sijz} \text{ as nodes do}\\
14: \quad \quad \text{if } f_i \text{ calls } f_{ap}
\text{ then}\\
15: \quad \quad \quad \delta_{ijz} \gets \delta_{ijz} \cup \{f_{sij} ; f_{sapbq}\}, \text{ where } s_{ij} < s_{apbq};\\
16: \quad \quad \text{else}\\
17: \quad \quad \quad \delta_{ijz} \gets \delta_{ijz} \cup \{f_{sij} ; f_{sapbq}\}, \text{ where } s_{ij} > s_{apbq};\\
18: \quad \quad \text{end if}\\
19: \quad \text{end for}\\
20: C \gets C \cup \{\delta_{ijz}\};\\
21: \text{end for}\\
22: \text{return } C \text{, the set of SeVCs}\\
$

## Encoding SeVCs
1. 函数和变量名重写解决OV问题

2. 词法分析得到token，使用word2vec向量化

3. 定长 $\theta$ ：

- $\lt \theta$：尾0占位
- $\gt \theta$：使用不同策略，尽可能使SeVC在向量中间
    
    - 到SyVC的子向量长度 $\lt \frac{\theta}{2}$ ，则删除SyVC右边多余的元素
    - SyVC到末尾的子向量长度 $\lt \frac{\theta}{2}$ ，则删除SyVC左侧多余的元素
    - 否则，取SyVC左侧元素 $\lfloor \frac{\theta-1}{2} \rfloor$ 个，右侧元素 $\lceil \frac{\theta-1}{2} \rceil$ 个

### Algorithm3:
$\small\text{Input: } \text{A set Y of SyVCs generated by Algorithm 1;}\\
\qquad\ \ \ \ \text{a set C of SeVCs corresponding to Y and generated by
Algorithm 2;}\\
\qquad\ \ \ \ \text{A threshold }\theta \\
\text{Output: The set of vectors corresponding to SeVCs}\\
1: R = \{\};\\
2: \text{for each } \delta_{i,j,z} \in C \text{ (corresponding to } e_{i,j,z} \in Y \text{) do}\\
3: \qquad\text{Remove non-ASCII characters in } \delta_{i,j,z} ;\\
4: \qquad\text{Map variable names in } \delta_{i,j,z} \text{ to symbolic names;};\\
5: \qquad\text{Map function names in } \delta_{i,j,z} \text{ to symbolic names;}\\
6: \text{end for}\\
7: \text{for each } \delta_{i,j,z} \in C \text{ (corresponding to } e_{i,j,z} \in Y \text{) do}\\
8: \qquad R_{i,j,z} = \{\};\\
9: \qquad\text{Divide } \delta_{i,j,z} \text{ into a set of symbols S;}\\
10: \qquad\text{for each } \alpha \in S \text{ in order to do}\\
11: \qquad\qquad\text{Transform } \alpha \text{ to a fixed-length vector } v(\alpha);\\
12: \qquad\qquad R_{i,j,z} = R_{i,j,z} || v(\alpha), \text{where } || \text{ means concatenation};\\
13: \qquad\text{end for}\\
14: \qquad\text{if } R_{i,j,z} \text{ is shorter than } \theta \text{ then}\\
15: \qquad\qquad\text{Zerors are padded to the end of } R_{i,j,z};\\
16: \qquad\text{else if the sub-vector (of } \delta_{i,j,z} \text{) up to the position of the SyVC } \\ \qquad\qquad e_{i,j,z} \text{ is shorter than }\frac{\theta}{2} \text{ then}\\
17: \qquad\qquad\text{Delete the rightmost portion of } R_{i,j,z} \text{ to make the resulting vector of length } \theta;\\
18: \qquad\text{else if the sub-vector (of } \delta_{i,j,z} \text{) next to the the position of the SyVC }\\ \qquad\qquad e_{i,j,z}\text{ is shorter than } \frac{\theta}{2} \text{ then}\\
19: \qquad\qquad\text{Delete the leftmost portion of } R_{i,j,z} \text{ to make resulting vector of length } \theta;\\
20: \qquad\text{else}\\
21: \qquad\qquad\text{Keep the sub-vector (of }\delta_{i,j,z} \text{) immediately left to the position of the SyVC } \\\qquad\qquad\qquad \text{of length } \lfloor \frac{\theta-1}{2} \rfloor \text{, the sub-vector corresponding to the SyVC, and the sub-vector} \\\qquad\qquad\qquad \text{immediately right to the
position of the SyVC of length } \lceil\frac{\theta-1}{2}\rceil \text{ fthe resulting vector has length }\theta;\\
22: \qquad\text{end if}\\
23: \qquad R = R \cup \{R_{i,j,z}\};\\
24: \text{end for}\\
25: \text{return } R; \text{\{fthe set of vectors corresponding to SeVCs\}}\\
$

## Experiment Result
```
训练集和测试集为4：1，5-折交叉验证
```
- SySeVR-BLSTM可以检测与函数调用、数组使用、指针使用和算术表达式相关的漏洞，在检测库/API函数调用相关漏洞时，与VulDeePecker相比，FPR降低3.4%，FNR降低5.0%

- SySeVR-BGRU在若干深度神经网络中效果最好，但是FNR始终高于FPR

- 使用分布式表示(如word2vec)来捕获上下文信息对SySeVR很重要。特别是，以令牌频率为中心的表示是不够的

- 如果一个语法元素(例如，令牌)出现在易受攻击(例如:非易受攻击的sevc比非易受攻击的sevc出现的频率要高得多。脆弱的)，语法元素可能会导致误报(如。误报警);这意味着语法元素的出现频率很重要

- 一个包含更多语义信息(即控制依赖和数据依赖)的模型可以获得更高的漏洞检测能力

- 启用sysevr的BGRU比最先进的漏洞检测方法有效得多

## Limitation
- 局限在C/C++

- 漏洞语法特征只提取了4个，需要提取更多

- 可改进SeVC和SyVC的生成算法，以求得包含更多的语义信息

- 现在使用单一模型检查多类型，但是是否可以根据不同类型选择最适合的模型网络

- 我们在切片级别检测漏洞(即，语义上彼此相关的多行代码)，这可以改进为更精确地确定包含漏洞的代码行

- 数据集真是标签的自动标注

- 可解释性？


# 总结
```
SySeVR的预处理是将程序弄成AST（joern），提取SyVC（树节点），弄成PDG，提取Program Slice（图节点），转为SeVC（代码行），放入各种深度学习网络中进行检测。注意，漏洞特征此处只使用了4个
```
# 灵感
```
加一个深度学习网络，从数据集的漏洞中自动进行提取漏洞特征，做checkmarx的内容
```