## Contribution
- 一个全新思维：代码->图像；将PDG看作社会网络（social network），通过三个角度的中心性分析得到三组数据，每一组数据重组成一个矩阵后，重叠三组数据，将其组成一个三通道的图像。

- 根据上述思想构建了一个面向源代码的漏洞检测系统：VulCNN
```
https://github.com/CGCL-codes/VulCNN
```

- 一个全新的数据集

## Motivation
- 直觉：一个代码行，与其他语句的关系越多、越密切，这条代码就能为函数提供更多的语义信息，也可以说这条代码对于这个函数更重要

- PDG中的节点度可以保留重要性信息：观察下面函数的PDG，可以发现其最易受攻击的代码行（6）跟其他代码行的边（数据流、控制流）更多，也可以说，这个节点的度（入度和出度之和）更高.下面是一个例子

**Source Code**
```
1:  void bad()
2:  {
3:   char * data;
4:   char dataBuffer[100];
5:   data = dataBuffer;
6:   data = badSource(data);
7:   {
8:   char dest[50] = "";
9:   strncat(dest, data, strlen(data));
10:  dest[50-1] = '\0'; 
11:  printLine(data);
12:  }
13: }
```
**PDG**
```
                        +----------+
                        |     3    |
                        +----------+
                              |
                             \|/
                        +----------+
                        |     4    |
                        +----------+     
                              |----------+
                             \|/         |databuf
                        +----------+/    |
                        |     5    |-----+   
                        +----------+\
                              |----------+
                             \|/         |data
                        +----------+/    |
                +-------|     6    |-----+
                |       +----------+\   
                |            |  |
                |data        |  +---------+
                |        data|           \|/
                |            |      +----------+
                |            |      |     8    |
                |            |      +----------+
                |            |          |   |
                |           \|/         |   |dest
                |       +----------+/   |   |
                |       |     9    |----+---+
                |       +----------+\       |
                |            |              |
                |           \|/             |
                |      \+----------+/       |
                |       |    10    |--------+
                |      /+----------+\
                |            |
                |           \|/
                |      \+----------+
                +-------|    11    |
                       /+----------+
```
**Adjacency Matrix**
||1(line3)|2(line4)|3(line5)|4(line6)|5(line8)|6(line9)|7(line10)|8(line11)|out-degree|
|--|--|--|--|--|--|--|--|--|--
|1(line3)|0|1|0|0|0|0|0|0|1
|2(line4)|0|0|2|0|0|0|0|0|2
|3(line5)|0|0|0|2|0|0|0|0|2
|4(line6)|0|0|0|0|1|1|0|1|3
|5(line8)|0|0|0|0|0|2|1|0|3
|6(line9)|0|0|0|0|0|0|1|0|1
|7(line10)|0|0|0|0|0|0|0|1|1
|8(line11)|0|0|0|0|0|0|0|0|0|
|int-degree|0|1|2|2|1|3|2|2

**Data Combine**
|line|out-degree|in-degree|degree|
|--|--|--|--|
|char* data;|0|1|1
|char dataBuf[100];|1|2|3
|data=dataBuf;|2|2|4
|data=badSource(data);|2|3|5
|char dest[50]="";|1|3|4
|strncat(dest, data, strlen(data));|3|1|4
|dest[50-1] = '\0'; |2|1|3
|printLine(data);|2|0|2

- PDG看作是社会网络（social network），每个节点都可以看作是社会网络中的一个个体，这个个体与其他个体的交流越多，这个个体就越重要，其蕴含的语义信息就更多

## Overview
抽取PDG -> 节点向量化 -> 图像生成 -> DL分类

## Step1：Extracting Graph
- 去除非ASCII字符和注释

- 重命名用户自定义函数和变量

- 使用joern工具，生成函数的PDG

## Step2：Encoding nodes
- 使用sent2vec模型进行向量化

## Step3：Image Generating
### Centrality Analyse
- **度中心性（Degree Centrality）**：一个节点 $i$ 的度跟节点数 $N-1$ 的比值成为节点 $i$ 的度中心性 $x_i$ ，写作 $\large x_i= \frac{deg(i)}{N-1}$

- **Katz中心性（Katz Centrality）**：一个节点的一级邻接点中心性和自身的按比例加权，写作 $\large x_i=\alpha \sum_j A_{ij}x_j+\beta$ 其中 $\alpha,\beta$ 都是常数，并且 $\large \alpha \lt \frac{1}{\lambda_{max}},\ \lambda$ 是邻接矩阵 $A$ 的特征值

- **接近中心性（Closeness Centrality）**：一个节点与其他节点的接近程度，是该节点到所有节点最短距离的平均值，写作 $\large x_i=\frac{N-1}{\sum_{i\neq j}d(i,j)}$

### Image Generating
- 计算PDG里面每个节点vector的三个中心性

- 将vector同相应的中心性相乘，得到三组vector

- 按照代码行顺序将每组节点vector保序排列成矩阵，得到三个矩阵

- 三个矩阵分别对应三个中心性、对应图像的三个通道

### Algorithm1：Convertin Program to Image
$\small \text{Input: } \mathbf{F} : \text{Source code of a function;}\\
\text{Output: } \mathbf{I} : \text{An image.}\\
1: nF \leftarrow \text{CodeNormalization(F)}\\
2: PDG \leftarrow \text{GraphExtraction}(nF)\\
3: V \leftarrow \text{ObtainNodes}(PDG)\\
4: \text{for each } v \in V \text{ do}\\
5: \qquad\text{ vector} \leftarrow \text{SentenceEmbedding}(v)\\
6: \qquad\text{ degreeCentrality} \leftarrow \text{DegreeCentralityAnalysis}(\text{vector}, PDG)\\
7: \qquad\text{ degreeChannel.add}(\text{degreeCentrality} * \text{vector})\\
8: \qquad\text{ katzCentrality} \leftarrow \text{KatzCentralityAnalysis}(\text{vector}, PDG)\\
9: \qquad\text{ katzChannel.add}(\text{katzCentrality} * \text{vector})\\
10:\qquad\text{closenessCentrality} \leftarrow \text{ClosenessCentralityAnalysis}(\text{vector}, PDG)\\
11: \qquad\text{closenessChannel.add}(\text{closenessCentrality} * \text{vector})\\
12: \text{end for}\\
13: \mathbf{I} \leftarrow \text{ImageGeneration}(\text{degreeChannel}, \text{katzChannel}, \text{closenessChannel})\\
14: \text{return } I$

## Step4：Classification
### CNN-Input Length
- 长的删除尾部，短的尾部补0

- 文章根据数据集中的函数行比例，通过实验，选择了100作为输入长度

## Results
- TPR基本同Devign齐平
- TNR和A值都最优
- 运行时间没有tokenNN短，但是tokenNN基本不可用

## Future Work
- 优化PDG生成
- 组合各种社会网络中心性和深度学习网络，以找出最优组合
- 误报率没有优化，使用定向模糊尝试解决

# 总结
使用PDG，通过中心性分析和word2vec得到向量，放入CNN进行特征提取和分类