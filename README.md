# Huawei-Code-Craft-2020
[toc]

## 写在前面

2020华为软件精英挑战赛京津冀东北赛区“花样划水”队，初赛第4，复赛第2，决赛第21。
本来因为决赛成绩不理想，就不打算开源了，如今比赛已经过去将近一个月，最近有人跟我说想看看当时比赛的代码，找代码的时候发现对代码已经没啥印象了。
于是决定趁着对代码还有点印象，做一个整理并开源，算是对我的第一次比赛经历做一个总结和复盘，同时也算是一个留念
[比赛链接](https://competition.huaweicloud.com/codecraft2020)

[2020华为软件精英挑战赛·京津东北赛区(初赛&复赛主页)](https://competition.huaweicloud.com/information/1000036574/introduction)

[2020华为软件精英挑战赛·全国总决赛](https://competition.huaweicloud.com/information/1000041223/introduction?track=113)

## [初赛](https://competition.huaweicloud.com/information/1000036574/introduction)

[初赛赛题下载](https://developer.huaweicloud.com/hero/thread-49130-1-1.html)

初赛的大致规则是：给定一组转账（即给定一个有向图），找出所有节点数小于8，大于2的环并输出，输出的环要求按照字典序最小的节点位于第一个位置，并以每个环的第一个节点的字典序对环进行排序。

初赛的时间周期比较长，我报名的时候比赛已经进行到一半了，所幸最后还来得及。

### 第一代思路

第一代的思路是先将图做预处理，对节点进行升序排序，并对每个节点的出边做升序排序。然后将每个点作为源点进行搜索（字典序小的先搜索，字典序大的后搜索），搜索过程中只访问大于源点id的节点，这样做的目的是既避免了重复输出环，又使得输出天然按照题目所要求的顺序。具体的搜索方法为沿着源点的反向展开3层，记录访问的点以及路径，再正向做深度为4的深度优先遍历，当访问到记录过的节点时进行输出。然而遗憾的是，这个方法虽然在思路上简单，但实现起来极复杂，并且我在实现的时候使用了不少unordered_map，因此最终速度极慢，但思路中还是有可取之处，我最终采取的反向标记三层的方法也算是脱胎于这个方法。

### 第二代思路

第二代的思路是找到所有环并把节点数小于8，大于2的环输出。于是通过stackoverflow上的[Finding all cycles in a directed graph](https://stackoverflow.com/questions/546655/finding-all-cycles-in-a-directed-graph)提问，找到了这篇1975年DONALD B. JOHNSON的有向图找环的论文[FINDING ALL THE ELEMENTARY CIRCUITS OF A DIRECTED GRAPH](https://www.cs.tufts.edu/comp/150GA/homeworks/hw1/Johnson 75.PDF)，在确定其为state-of-the-art之后用c++实现了出来并用在了这次的比赛里，结果速度实在太慢，比以深度为7对有向图做深度优先搜索的朴素方法还要慢，原因是输入的图中有很多深度超多7的路径，访问这些路径消耗的时间太多，而JOHNSON的找环中的剪枝方法的正确性是只有在遍历所有路径的前提下才能保证的，因此并不贴合这次比赛的数据和需求。

### 最终版思路

在比赛进行到最后几天的时候，我的队友在网上找到了DDD大佬对这次比赛的[分析](https://github.com/justarandomstring/2020-Huawei-Code-Craft)，看完之后受益匪浅，结合大佬的baseline以及之前我自己的第一代的思路，于是诞生了最终版思路：

先对源点反向做深度为3（如果将源点看作第1层的话一共是4层）的广度优先遍历，标记所有节点并用一个数组V保存所有标记过的节点id。然后再做深度为6（源点算作第1层的话一共是7层）的深度优先遍历。在遍历过程中，当深度超过3之后，只访问被标记过的节点，当深度达到6时可以根据源点是否存在从该点出发的入边来判断是否存在经过该点的，长度为7的环，从而避免了第7层遍历展开。

### 优化思路

这次初赛的算法思路比较简单，到最后其实大家的方法都大同小异，主要的时间差距在多线程，对数据结构的设计，输入和输出的优化，以及对内存、和数据访问的优化。

#### 多线程

初赛时我采用了任务和线程绑定的方式，这样写真的很简单，不过需要对任务分配上格外注意，直接对节点进行n等分，并交给n和线程来处理是不行的，因为以k为源点出发时，只访问节点id大于k的节点，因此源点id越小，以改节点为源点搜索时所要访问的节点数就越多，且不同节点的访问规模差距是巨大的（每个节点的规模大致为$ (节点的平均出边数\times\frac{节点总数-当前节点重编码后id}{节点总数})^7$），其中$\frac{节点总数 - 当前节点重编码后id}{节点总数}$的值表示比当前节点id大的节点占总结点数的比例。所以任务分配时要采取id小的部分节点数少，id大的部分节点数多的原则，我最终采取的是四线程（1/64，3/64，3/16，3/4）的分配方式。（因为时间关系，初赛的多线程没来的及优化，复赛采用的是更合理的任务与线程分离，任务数远大于线程数的方式，避免了人工分配任务）

#### 数据结构设计

因为原始id最多只会有20万个，为了后续处理方便，先对id按其原字典顺序进行重编码（因为初赛的线上数据较为简单，可以直接采用基数排序），读入数据之后最后先对图采用全局二维数组的方式存储。并为每个线程不同长度的答案都分配了存储的数组。

```c++
int _G[280000][50]; //G[u][v]
int _in_G[280000][50]; //G[v][u]
struct pthread_info
{
	int _step2d[280000];
	bool _visit[280000];
	char ans3[ANS3];
	char ans4[ANS4];
	char ans5[ANS5];
	char ans6[ANS6];
	char ans7[ANS7];

}pthread_info[PTHREAD_NUM];
```

#### 输入和输出的优化

为了速度上的考虑，输入和输入要避免使用scanf和printf（更不要使用cin和cout），因为其中包含了比较复杂的正则匹配，我们的比赛中都是冗余的，我们需要的只是简单的将带逗号和换行符的字符串输入转为数字，以及将数字转为用逗号和换行符分隔的字符串，所以应该以字符串的形式读入和输出并自己转格式。

字符串的转出会耗费相当长的时间，（因为数据和输出的特点。输出时存在大量重复数字，采用字典的方式将数字转为字符串是更好的选择，这个方法在复赛时采用了，初赛因为时间关系没来得及用）。在字符串和数字的转换中，存在大量的乘法、除法和模运算，及其耗时，因此我将输入的字符转看作16进制数读入，转出时也当作16进制数读出，这样都进来的数字虽然和真实的并不相同，但不影响程序的正确性（需要注意的是，因为原始的id为32位无符号整数，当作16进制读入后如果使用unsigned int型存储可能会溢出，所以必须采用64位整数存储。不过经过测试证明线上数据的原始id并不大，使用unsigned int没有报错，所以在程序中并没有使用64位存储。

另外，因为初赛时数字转字符串没有使用字典的方式，计算耗时较长，因此我将这部分计算放到多线程之中。

#### 对内存、和数据访问的优化

1. 因为在保证结果完全正确的前提下，唯一的评价指标是时间，因此对于可以确定大小范围的数据，统一采用栈空间而不是堆空间进行存储；
2. 为提高cache命中率，全局变量采用大数组在前，零散的对象在后的排布方式，且大数组占用的内存尽量按照cache页大小对齐。

## [复赛](https://competition.huaweicloud.com/information/1000036574/introduction)

### 规则描述

[复赛赛题下载](https://developer.huaweicloud.com/hero/forum.php?mod=viewthread&tid=53085)

复赛的赛题要求与初赛类似，主要改动为：

增加了对转账金额的约束

> 转账金额浮动区间约束为[0.2, 3] ：循环转账的前后路径的转账金额浮动，不能小于0.2，不能大于3。如账户A转给账户B 转X元，账户B转给账户C转Y元， 那么进行循环检测时，要满足0.2 <= Y/X <= 3的条件。

扩大了输入和输出数据的规模

> 转账记录最多为200万条
>
> 说明：数据集经过处理，会保证满足条件的循环转账个数小于2000万

以及取消了初赛中对节点平均出度的限制，同时线上数据也变得比初赛更丰富。

并为了防止线上数据泄露，改为A、B榜的形式。5月6日9:00-5月15日18:00可以提交到A榜，5月16日下午2：00~4：00进行B榜提交，2：00的时候对题目要求做小幅度修改（将找包含3 ~ 7个节点的环改为了3 ~ 8个节点的环，并把输入金额从32无符号整数改为增加到小数点后两位）。

个人感觉复赛的赛题要求虽然与初赛类似，但不论是从赛事流程上，还是从线上数据的特征上都比初赛正式了，很多，初赛更像是一个初筛，复赛才是正式的比赛。

同时复赛对最后几小时正式赛上进行了需求的修改，也是在一定程度上考察了代码的可拓展性和参赛人员对代码的临场修改和维护能力。

### 优化思路

初赛之后看到了一些大佬的开源代码，发现大家的代码写的都比较具象，有很多大佬为了避免分支判断，把重复的代码根据可以确定的预测做了展开，比如把7层的递归调用展开成了7个函数，减少了很多不必要的判断，以及把多线程中的每个子线程都分开写了。这样做虽然降低了代码的可拓展性和可读性但确实提升了速度。更加贴合本次比赛的评价标准。所以在看完大佬的代码之后，心理虽然还是有点抗拒，但我还是把复赛的代码写得更具体了。所付出的代价就是在B榜的最后两个小时里改动有点累>__<||。

#### 算法

找环算法上在初赛的基础上做了一定改动，先反向深度优先遍历3层（若将源点看作1层的话总共有4层），将前2层的路径用一个链式前向星的图进行记录，，并对第二层的节点进行标记（不妨假设标记为红色），将第三层做一个标记（不妨假设标记为黑色）。随后在正向搜索过程中，当访问到红色节点时可以根据之前记录的路径直接输出环，当深度达到3（若将源点看作1层则为第4层）时，只拓展黑色节点，且当深度达到4时即可停止拓展。

对于金额的约束方面采用在记录和拓展的过程中进行金额判断，而不是在找到环后一起做判断。这样做的目的是在一定程度上进行剪枝。

#### 多线程

复赛时我的多线程采用了任务和线程分离的模式，类似线程池模型。这样就避免了人为划分任务片大小的麻烦，直接把任务等分为k（k = 256）个，n（n=4）个线程互斥的从任务列表中取出当前未完成的id最小的任务，一直到所有的任务都被取出。前文中已经将id进行等分的话，越靠前的任务的耗时越长，因此将任务进行远大于线程数的等分之后，所有线程都可以几乎同时完成结束任务。

#### 数据结构设计

在对图的存储上，因为数据规模变得更大更复杂，且分为AB榜，所以不能像初赛那样简单的用固定长度的二维数组 进行存储，而是采用类似前向星的方式，因为总的边的数量即是输入的行数，因此数组长度可以确定。

```c++
node __G_new[INPUT_SIZE] = {};//出边表
node __in_G_new[INPUT_SIZE] = {};//入边表
int __G_point[NODE_SIZE + 32] = {};//每个节点在出边表中的起始位置
int __in_G_point[NODE_SIZE + 32] = {};//每个节点在入边表中的起始位置
```

输入部分因为任务片的输出长度都不固定，且总的任务数非常多，为了保证输出时无需额外排序，无法再像初赛时那样采用栈空间存储，而是采用堆空间存储数字，再使用字符串字典讲数字转为最终答案。

### 复赛正式赛

正式赛要求在两个小时以内，将找包含3 ~ 7个节点的环改为了3 ~ 8个节点的环，并把输入金额从32无符号整数改为增加到小数点后两位。这里有一个坑，就是原本在金额计算上有出现乘法和除法，应该通过交换分子分母的方式全部转为整型的乘法来进行计算和比较，而不能用浮点型做乘除，否则会因为精度丢失导致结果不完全正确。正式赛将金额增加小数点后两位后，应该把其看作乘以100的整型后做处理，（如11.11看作1111，22.2看作2220，33看作3300），否做依然会有精度丢失问题。有些大佬在正式赛时因为数据增加了小数部分后直接改为用浮点型存储数据，导致结果不正确，在两个小时内没有定位出问题，直接导致正式赛没有成绩。

## [总决赛](https://competition.huaweicloud.com/information/1000041223/introduction?track=113)

[决赛赛题下载](https://developer.huaweicloud.com/hero/forum.php?mod=viewthread&tid=55849)

总决赛的题目改为计算top100的介数中心性。正式赛仍然分AB榜，但取消了复赛时导致许多大佬翻车的对题目需求进行修改，改为B榜使用A榜的数据并额外增加一个数据特征不同的数据集，且增加了线上环境的cpu数量numa数量。

赛题中把介数中心性成为位置关键中心性，且给出具体定义，一开始的时候并不知道介数中心性这个概念，且根据初赛和复赛的经验想当然的认为这是这次比赛自己提出的东西，并不能找到线程的算法。一直等到决赛正式赛开始前两天，才在队友的提醒下了解了这个概念，用c++复现了网上计算state-of-the-art的方法后没有来得及做其他优化了。

## 总结

这是本人进入计算机专业以来，参加的第一个比赛，得知比赛的契机也比较偶然。这次比赛收获不少，感触良多。

1. 通过网络和比赛的官方群获取信息非常重要，编写代码只是比赛的一部分，而远非全部，有些信息是否获得会很大程度上影响最终排名
2. 对于这种基本上以速度作为唯一的评价指标的比赛里，代码的可拓展性和可读性并没有那么重要，一切还是以速度快为目的
3. 团队协作很重要，初赛和复赛的时候需求还较为简单，大部分事情其实一个人就可以完成，但是决赛的工作量较大，一个人做的话会明显的力不从心，而且整个比赛的持续时间比较长，分工合作会更加轻松。
4. 在规则允许的范围内尽可能的确定线上数据集的数据特征，线上数据和线下数据可能有较大差别，确定数据特征可以直接决定优化的方向





