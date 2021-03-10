# CCF_BDCI_News_Classification

### Abstract

​	项目源自CCF BDCI比赛，赛题为[面向数据安全治理的数据内容智能发现与分级分类](https://www.datafountain.cn/competitions/471)

​	该任务的初衷是为了有效、规范保护企业敏感数据，因此首先要对数据进行分级分类，以识别敏感数据，从而进一步围绕保护对象的全生命周期进行开放、动态的数据安全治理，解决数据开放共享与数据隐私保护的矛盾与统一。任务要求识别样本中的敏感数据，构建基于敏感数据本体的分级分类模型，判断数据所属的类别以及级别。
​	任务所使用的训练数据共 40000 篇，其中 7000 篇是已标注的文档，共 7 个类别，分别为财经、房产、家居、教育、科技、时尚、时政，每一类包含 1000 篇文档，另外还有 33000 篇未标注文档；测试数据共 20000 篇文档，测试数据包含 10 个类别，分别为财经、房产、家居、教育、科技、时尚、时政、游戏、娱乐、体育，其中游戏、娱乐、体育三类为训练集中未标注的类别。

​	**最后在比赛中排名在前100名**

### 思路

这个题目难点在于标注数据很少，甚至有三个类别压根没有标注数据，只有未标注数据。这样一般有两种思路，一种是使用zero shot学习，另一种使用远程监督学习。我们使用了第二种思路。首先在未标注数据中使用关键词+LDA抽取出一些三类完全没有标注数据的数据，然后使用bert训练预测未标注数据，进入半监督学习，迭代若干次后完成学习。

### 方法

#### 1.1 LDA

隐含狄利克雷分布简称LDA(Latent Dirichlet allocation)，是一种主题模型，它可以将文档集中每篇文档的主题按照概率分布的形式给出。同时它是一种无监督学习算法，在训练时不需要手工标注的训练集，需要的仅仅是文档集以及指定主题的数量k即可。此外LDA的另一个优点则是，对于每一个主题均可找出一些词语来描述它。lda也是一种典型的词袋模型，它假设每一篇文档都是一组词的集合，并且词与词之间没有顺序和先后关系。一篇文章可以包含多个主题，文档中的每一个词都是由其中的一个主题生成。

首先需要对文档数据进行文本分词、去停用词等一系列数据预处理工作。对于本次比赛中LDA使用Cntopic包来实现，其Topics设置为10（对应赛题的十个类别），训练epochs为20。

#### 1.2 key words关键字

收集一些新闻然后使用词频统计的方法得到每个类别的key words，然后对新的三种类别(游戏、体育、娱乐)使用两者(LDA打标签和key words打标签)的结果取交集，即LDA 认为content[i]属于游戏类并且key words 方法认为content[i]属于游戏类最终content[i]才会判定为游戏类，否则会被判定为 None类。

最后对两者取交集的类别结果，分别对其统计三个新的类的类别关键词(游戏、体育、娱乐)的出现次数并按类别数出现的次数从高到底排序。得到了类别关键词的出现次数排序结果，对于三个新类别（游戏、体育、娱乐）分别取前1000的content作为最终的判定得到的新的类别训练数据，并将其加入到7*1000的训练数据中。

#### 2 Bert和训练流程

Bert是目前NLP领域很常用的预训练模型。Bert使用大量文本进行预训练，预训练任务包括对字进行随机mask然后预测和判断任意两个句子之间顺序这两个子任务。

半监督迭代过程可能出现的问题之一是对前期学到的错误在后续的学习过程中进行了放大，所以我们搭建训练流程的时候做了一个基本假设，那就是前期模型学习后产生的结果一定会有非常大的噪声。所以我们在半监督学习迭代过程中对样本进行了采样操作，对于前一个模型对未标签数据打标签产生的新训练数据，并不是每次都全用上，我们第一轮大概用上了10000条，第二轮用上14000条，逐步加大采样率，五轮迭代后才全部加入训练。这样尽量避免噪声放大问题。

#### 3 ERNIE

对于ERNIE模型的使用与Bert类似，同样是使用预训练的ERNIE模型。

ERNIE 认为，BERT 在NLP各个应用领域取得了不凡的成绩，主要得益于其使用了大量文本语料进行了无监督预训练：(a) Masked LM，(b) Next Sent Prediction；但 BERT 仍然存在以下缺点：

(1) Masked LM 的建模过程中是以字为单位的，这使得模型很难学习到语义知识单元的完整语义表示，而这一缺点对于中文的 BERT 预训练模型与应用而说尤为明显

(2) 训练语料单一（百科类），且除 Masked LM 外只有 Next Sent Prediction 任务，较为单一。

针对 BERT 模型的不足，ERNIE 做了以下改进：

(1) ERNIE 模型通过建模海量数据中的实体概念等先验语义知识，学习完整概念的语义表示。即在 Masked LM 中通过对词和实体概念等语义单元进行 mask 来预训练模型，使得模型对语义知识单元的表示更贴近真实世界。

(2) 引入多源数据语料训练 ERNIE。包括百科类，新闻资讯类、论坛对话类数据来训练模型。尤其是论坛对话语料的引入，文章认为，“对话数据的学习是语义表示的重要途径，往往相同回复对应的 Query 语义相似”。基于该假设，ERINE 采用 DLM（Dialogue Language Model）建模 Query-Response 对话结构，将对话 Pair 对作为输入，引入 Dialogue Embedding 标识对话的角色，利用 Dialogue Response Loss 学习对话的隐式关系，通过该方法建模进一步提升模型语义表示能力。

总结来说，ERNIE 优势在于：

(1) 对实体概念知识的学习来学习真实世界的完整概念的语义表示；

(2) 对训练语料的扩展尤其是论坛对话语料的引入来增强模型的语义表示能力。

#### 4 XLNET 

Bert是一种自编码的语言模型，通过上下文的信息来预测[MASK]位置上的词。但是，Bert存在的一个缺点就是：在预训练时，假设句子中多个单词都被MASK掉，那么对于Bert来说这些MASK掉的单词是条件独立的，它们之间是没有任何关系的。但有时，这些单词之间可以能有关系甚至是互相影响的。

XLNet针对BERT的很多内容都做了改进，但我们认为XLNet针对Bert上面所提到的问题进行的改进对于我们本次的分类任务来说是有帮助的。

当然，XLNet是站在自回归的语言模型预训练的角度去看待问题。然而，自回归语言模型的一大缺点就是它只能从左向右或者从右向左地去生成，无法像Bert一样结合上下文的信息去预测当前位置的信息。所以XLNet就希望模型技能通过自回归的方式去训练语言模型，又可以让当前预测的位置看到后文的信息，最简单的思路当然就是把后文的信息拿到前面来做预测，XLNet也就是这样做的。在预训练阶段，XLNet会对输入的句子的token做随机的排列组合，在随机的排列组合里选择一部分作为模型的输入，这样模型就可以在预测某一位置的token时看到上下文的信息。

在具体的实现阶段，XLNet还使用了双流注意力机制。对于内容注意力，就是Transformer的计算过程。而对于Query注意力，相当于用它来替代BERT中[MASK]的标记。BERT希望通过[MASK]来覆盖掉预测单词的信息，而XLNet中抛弃了[MASK]部分，对于Query流来说，它直接忽略了预测位置内容的信息，只保留其位置信息。

总结来说，XLNet的优势在于：

(1)采用了新的预训练方式，在自回归语言模型的模式下引入了双向语言模型的信息。

(2)增加了预训练的数据集，相对于BERT来说，更能具有阅读理解能力。

### 比赛结果分析：

这个赛题出的比较有意思并且有一定的难度。小组从开始0.73989280的成绩，经过不断的迭代优化，上升到0.83231359的最终成绩，最终排名为第99名，虽然小组成绩最终提升了将近10%，但显然最终的结果并不理想，只达到了83.23%的F1值，而排行榜上排名第一的小组达到了94%的F1值，差距还是挺大的。由于现在还在比赛复赛阶段，进入复赛的队伍的思路及模型还未公布，所以无法与其进行充分对比分析，这里仅仅是组内对本次比赛结果比较差的分析：

（1）数据预处理不足，1000*7(共7000条)有标签数据，对于新的三类无标签数据，我们采用LDA+key words 的方法共同生成，然后再喂入Bert、ERNIE、XLNet进行迭代训练，现阶段三种模型的准确率还是蛮高的，但是比赛中的F1值却在80左右，说明我们对于无标签数据的处理还是存在一定的不足的。

（2）与1类似，未能找出类别样本的分布，比如可能测试数据与所有的训练数据的类别样本(33000+7000)分布是相同的，但是第一次迭代时，我们认为数据样本的每个类别的分布是相同的，即认为每个类别的样本条数相同，这就导致了训练样本分布不均的问题。

（3）使用模型少，小组只试了Bert、ERNIE和XLNet三种模型，未使用现阶段的图模型等其他模型。

（4）迭代采样时采样数比较大，这也是导致最终效果不是很好的原因。

（5）前期调研不足。在比赛初期未进行充分的相关调研，尤其是现阶段文本分类的最新方法了解不足，等反应过来的时候为时已晚。