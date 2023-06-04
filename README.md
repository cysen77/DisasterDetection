# DisasterDetection

references https://www.kaggle.com/code/gunesevitan/nlp-with-disaster-tweets-eda-cleaning-and-bert
部分代码数据清洗的代码参照了以上链接

一、	小组成员
蔡宇生 和另外两位
二、	选题简介
1任务：文本二分类，根据一个推文描述的内容，预测它是否表示真的有灾难发生。
2数据：train.csv训练集，test.csv 测试集，训练集中含有标签，标签是0和1，分别代表帖子表述的内容表示无灾难和真的有灾难发生。训练集、测试集的规模分别是7613和3263。特征有keyword, location, text。其中text是推文的文本，location是发送推文的位置，keyword是推文中的特定关键字。我们发现location和keyword两组特征的缺失值多，并且对此任务无用。我们主要利用text特征去预测标签。
3 数据的简单可视化：
 
在训练集中，标签0和1的比例是0.46: 0.57，是相对平均的。

三、	解题步骤

我们用预训练模型bert-base-uncased，在尾端接上一层全连接层作为输出层。在此之前，我们试过bert的一系列模型，包括bert-base-cased, bert-large-uncased等，结果是bert-base-uncased是最有效率的。bert-base-uncased有110M参数，12头多注意力头，词向量长度是768，由嵌入器和12层transformer编码器组成。
1 数据预处理：
从huggingface上下载预训练模型后，我们得到了两个工具，分别是tokenizer和bert。我们用tokenizer可以一键预处理数据。在bert-base-uncased中，经过tokenizer，文本的头和尾分别被添加上特殊符号[CLS]和[SEP]，并子词切割，最终将会得到数字序列。
2文本向量化:
文本向量化将会在bert模型中实现。每个词(子词)将会被嵌入成长度为768的向量。
3模型:
把bert作为骨干，拿到[CLS]上的对应输出，接上全连接层和Softmax激活函数作为输出层，输出将会是规模为2的向量，分别对应推文与真实灾难无关和有关的概率。在向量的2个数字中，我们选出最大的一个数字的对应的标签作为预测。
4训练:
我们冻结bert骨干，而只优化输出前的几层全连接层。我们用交叉熵损失函数和Adam优化器。
 ![image](https://github.com/cysen77/DisasterDetection/assets/86369829/9d71ec62-cde7-4272-9ffb-401ee9e7475c)

	
四、	结果展示
 ![image](https://github.com/cysen77/DisasterDetection/assets/86369829/9497ab1c-fce2-4ae9-a994-db53f6ba31cd)

五、	细节
在数据上，由于课程是自然语言处理，所以我们只去关注文本text这一特征。我们的数据清洗包括把特殊字符删去、把缩合拆开、处理用户名和俗语和错别字和删除链接。例子如下。用到python库re。
清洗前	清洗后	清洗前	清洗后
£3million 	3 million	ArianaGrande	Ariana Grande
he's	he is	YEEESSSS	yes
&gt;	>	kaggle.com	删除
16yr	16 year	å¨	删除
在经过网络前，文本由从python库transforers的tokenizer得到一个数字序列，值得注意的是，每个文本的前端应该添加特殊令牌[CLS]，在末端添加[SEP]。为了使输入到网络的数据是规模统一的，应该把句子截断，或者添加令牌[PAD]，使每批的序列的长度一致。我们使每文本对应的序列为长度为128的浮点数张量。
特征target为标签，它只有0和1，分别表示推文与真实的灾难有关和推文与真实灾难无关。我们把它变成长整数张量，由于我们用的损失函数工具是torch.nn.CrossEntrophyLoss, 它要求标签是长整数张量。

在我们的两份代码中，第一份的模型是把bert作为骨干，把bert的[CLS]对应的位置的输出去经过全连接层分类器和激活函数Softmax，就得到长度为2的向量，分别对应两个标签的的概率。我们投入很多经历去找最好的分类器和优化器。我们从分类器的规模一层、二层和三层找，从Adam优化器和SGD优化器找。
以下是网格。
1层、Adam	2层、Adam	3层、Adam
1层、SGD	2层、SGD	3层、SGD

以下是各网格优化10轮的结果。
	验证F1score	时间
1层、Adam	0.81	10: 53
2层、Adam	0.85	10: 53
3层、Adam	0.74	13: 23
1层、SGD	0.71	9: 43
2层、SGD	0.72	9: 51
3层、SGD	0.72	10: 23

我们选择两个模型，1层(768 -> 2)或2层(768->256->2)的全连接层作为分类器，Adam作为优化器。学习率是1e-7，损失函数是交叉熵损失函数。

在优化中，我们运用交叉验证，去解决模型过拟合的问题。我们用以上的参数分别实例化5个模型。把数据集互斥地随机地平均地分成5份，每次轮流选4份去优化模型，剩下的1份留下去验证，重复5次。那么我们得到10个模型，求出它们的验证分数，分别按两个参数保留1个验证分数最高的模型。以下是各折的验证分数。
	1层、Adam	2层、Adam
第1折	0.84	0.84
第2折	0.82	0.83
第3折	0.82	0.85
第4折	0.82	0.84
第5折	0.82	0.85

下图记录了一次训练。在第3轮前，训练的损失及验证的损失都在平稳地下降，且验证的损失低于训练的损失。它们分别从约0.67和0.64开始，在第3轮相交在0.5。此后，验证的损失保持了相对的静止，然而，训练的损失继续下降，即使放慢了速度。同时，在整个过程中。验证分数F1score逐渐上升，从轮0.72上升到约0.85，即使速度在下降。
 
软投票。我们用这两个模型去预测测试集的标签。我们得到2组的各样本分布属于两个标签的概率。我们把这两组概率对应的取均值。然后取概率最大的标签作为最终的预测。



