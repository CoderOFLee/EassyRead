## Target
给定源程序代码，判断程序是否包含漏洞，如果有，漏洞在何处？

## Contributions
- 开启深度学习应用，确定指导原则
```
包括软件程序的表示，使深度学习适合漏洞检测，确定基于深度学习的漏洞检测的粒度，以及选择特定的神经网络进行漏洞检测。特别地，我们建议使用代码小工具来表示程序。代码小部件是一些(不一定是连续的)代码行，这些代码行在语义上彼此相关，并且可以作为深度学习的输入进行矢量化。
```
- 提出漏洞检测工具VulDeePecker
```
可识别多种漏洞类型、依赖不为特征定义的人类知识、检出率高
```
- 开放漏洞训练数据集
```
NVD+SARD。该数据集包含61,638个代码小工具，其中17,725个代码小工具易受攻击，43,913个代码小工具不易受攻击。在17725个漏洞代码中，10440个漏洞代码对应缓冲区错误漏洞(CWE-119)， 7285个漏洞代码对应资源管理错误漏洞(CWE-399)。https://github.com/CGCL-codes/VulDeePecker
```

## Guiding Principles
- Principle1：程序->中间表示->向量输入
```
使用中间表示将语义信息抽取，更适合转化向量
```
- Principle2：粒度应小于程序级和函数级
```
程序和函数内部含有多行代码，无法精准确定漏洞位置
```
- Principle3：能够处理上下文信息的神经网络更合适
```
由于一行代码可能因为上下文而出现漏洞，所以需要能够处理上下文信息的神经网络（序列化神经网络？），并且最好使用没有梯度消失（VG）问题的网络，并且中后期的代码也可能导致前序代码出现漏洞，所以最好使用双向的网络
```

## Design
### A. code gadget
- Definition：代码小部件由许多程序语句(即代码行)组成，这些语句在数据依赖关系或控制依赖关系方面在语义上相互关联
- Key Point：一个漏洞的启发式标签，可以从key-point为准展示漏洞类型
```
如一个由于函数调用或数组超限而引起的漏洞，那么漏洞的“关键点”分别为函数调用和数组。一个漏洞的关键点可能不止一个。
```
- generation：根据API/库函数调用为关键点的代码小部件的生成有几个算法
```
S. Horwitz, T. Reps, and D. Binkley, “Interprocedural slicing using
dependence graphs,” ACM Transactions on Programming Languages
and Systems (TOPLAS), vol. 12, no. 1, pp. 26–60, 1990.

S. Sinha, M. J. Harrold, and G. Rothermel, “System-dependencegraph-based slicing of programs with arbitrary interprocedural control
flow,” in Proceedings of the 1999 International Conference on Software
Engineering. IEEE, 1999, pp. 432–441.
```
### B. Training
### Step1：抽取API/库函数调用的代码片段（code slice）- 考虑了数据流
- 整体流程为：抽取API/库函数调用 -> 每个参数生成一个代码片段 -> 合并参数片段得到函数代码片段

- 函数调用分为两类：前向（forward）函数调用、后向（backword）函数调用。其中前向表示该函数参数包括外部（如命令行、程序、文件、套接字）的数据，后向则相反。
```
e.g.1：forward function -> int main(int argc, char** argv)
e.g.2：backword function -> void strcpy(char* buf, char* str)
```
- 代码片段分为两类：前向（forward）代码片段、后向（backword）代码片段。其中前向表示受考虑参数影响的代码语句，后向表示影响考虑参数的代码语句。

- 一个所考虑函数的参数，对应生成一个或多个同函数类型的程序片段。对于前向，多个片段是由于在函数被调用处或之后产生分支；对于后向，多个片段是由于函数被调用处或之前合并与该参数相关的多个片的情况。

### Step2：组装程序小部件（code gadget）和生成真实值标签
- 合并上述步骤生成的若干参数程序片段，得到小部件
```
根据用户定义（user-defined）函数为分区，将多个相关参数片段按程序进行去重归并，然后再按照程序在参数片段中的的顺序进行组装（没有出现，则随机顺序）
```

- 赋予真实值标签，0表示安全，1表示存在漏洞

### Step3：向量化程序小部件
- 去除非ASCII字符和注释字符

- 词表超限（OV）：重命名变量名为VAR1、VAR2，函数名为FUN1、FUN2

- 向量化：使用word2vec方法，根据词法分析得到的token进行词汇区分

- 向量定长：BLSTM接受的输入是定长的的，但是不同的程序小部件经过word2vec转化的向量是变长的，需要策略进行归一化
```
给定一个参数τ，将其作为BLSTM接受的向量长度
1. 小于τ：如果代码小部件是一个或多个向后（backword）切片组成，则头0占位；否则尾0占位
2. 大于τ：如果代码小部件是一个或多个向后（backword）切片组成，则去头；否则去尾
```

### Step4：训练BLSTM网络
- 将向量化后的代码小部件放入BLSTM进行训练

- 使用的网络包括多个BLSTM层，然后接dense层和softmax层

## Experiment
### Metrics
- Arguments：

| 简称 | 漏洞样本 | 样本标签 | 检出 | 检出标签 |
| -- | -- | -- | -- | -- |
| TP | True | 1 | True | 1 |
| FP | False | 0 | True | 1 |
| FN | True | 1 | False | 0 |
| TN | False | 0 | False | 0 |

- false positive rate(FPR) = $\frac{FP}{FP+TN}$

- false negtive rate(FNR) = $\frac{FN}{FN+TP}$

- true positive rate(TPR) = $\frac{TP}{TP+FN}$ = 1 - FNR

- P = $\frac{TP}{TP+FP}$

- F1 = $\frac{2 * P * TPR}{P + TPR}$

- 理想状态： 不漏报（FNR ≈ 0 && TPR ≈ 1）、不误报（FPR ≈ 0 && P ≈ 1），也就是F1 ≈ 1

### Dataset and Truth-Label
- NVD + SARD，C/C++语言的两种漏洞程序（缓冲区漏洞、资源管理漏洞）

- Code Gadget Dateset(CDG)，获得的代码小部件数据集，分为若干小的数据集

| 子集名称 | 漏洞类型 | 关键点 | 
| -- | -- | -- | 
|BE-ALL|缓冲区漏洞（CWE-119）|所有函数调用
|RM-ALL|资源管理漏洞（CWE-399）|所有函数调用|
|HY-ALL|缓冲区（CWE-119）和资源管理漏洞（CWE-399）|所有函数调用|
|BE-SEL|缓冲区漏洞（CWE-119）|专家提取库函数|
|RM-SEL|资源管理漏洞（CWE-399）|专家提取库函数|
|HY-SEL|缓冲区（CWE-119）和资源管理漏洞（CWE-399）|专家提取库函数|

- 自动化打标签
```
NVD：针对打了补丁（删除/修改）的代码行，将其设为1，人工检查标签为1的漏洞是否有效
SARD：粒度比NVD更高，直接打，只有相同代码的情况下可能误标，但是在误差范围内，无需人工检查
```

### Results
- VulDeePecker在两种漏洞类型（CWE-119、CWE-399）上表现都是当时最优，可以看出VulDeePecker可以应用到多种漏洞类型（每个类型对应一个网络）
- VulDeePecker强依赖数据量
- VulDeePecker在借助专家知识后，能够大幅提高模型效率

## Limitation
- 针对可执行文件的漏洞检测？
- 跨语言的漏洞检测？
- 将关键点更加细化的粒度？
- 引入控制流进行检测？
- $\tau$参数的使用？更详细的定长策略？
- 比BLSTM更优秀、更合适的深度学习网络？

## Related Work
### 基于模式识别的漏洞检测
- 人类专家手动生成
- 基于漏洞类型半自动生成
- 未知漏洞类型半自动生成
### 基于代码相似性的漏洞检测
- 只能检测由于代码克隆导致的漏洞
- 对于不同的漏洞类型，需要不同的相似性计算算法



# 总结
预处理使用代码小片段（code slice）组装为函数调用小部件（code gadget），然后使用word2vec向量化，放入BLSTM进行训练。数据集来自NVD和SARD。整体是一个二元分类问题。