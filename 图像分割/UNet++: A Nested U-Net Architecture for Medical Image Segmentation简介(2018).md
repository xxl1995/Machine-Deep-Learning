#  UNet++: A Nested U-Net Architecture for Medical Image Segmentation(2018)

> 重点阅读作者的一个分享, 介绍了他的思考
>
> https://zhuanlan.zhihu.com/p/44958351

## 结构

文章对U-Net改进的点主要是skip connection.

**作者认为skip connection 直接将U-Net中encoder的浅层特征与decoder的深层特征结合是不妥当的, 会产生semantic gap. **

整篇文章的一个假设就是, 当所结合的浅层特征与深层特征是semantically similar时, 网络的优化问题就会更简单, 因此文章对skip connection的改进就是想bridge/reduce 这个semantic gap.

理解这篇文章的关键就是看懂文中的这张图. 其中黑色部分代表的就是原始Unet结构,绿色代表添加的卷积层, 蓝色代表改进的skip connection.

![img](https://img-blog.csdn.net/20181009223130476?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzI2MDIwMjMz/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## 关于深度的思考

针对U-Net结构而言, 问题实际是这样的, 看这个图, **既然 ![X^{1,0}](http://www.zhihu.com/equation?tex=X%5E%7B1%2C0%7D) ,  ![X^{2,0}](http://www.zhihu.com/equation?tex=X%5E%7B2%2C0%7D) ,  ![X^{3,0}](http://www.zhihu.com/equation?tex=X%5E%7B3%2C0%7D) ,  ![X^{4,0}](http://www.zhihu.com/equation?tex=X%5E%7B4%2C0%7D) 所抓取的特征都很重要, 为什么我非要降到 ![X^{4,0}](http://www.zhihu.com/equation?tex=X%5E%7B4%2C0%7D) 层了才开始上采样回去呢？**

![img](https://pic2.zhimg.com/80/v2-0180b2e84ca1b581bedbe7d4ccb8d1f9_hd.jpg)

为了搞清楚是不是越深越好, 我们应该对它们做一下实验, 看看它们各自的分割表现：

![img](https://pic1.zhimg.com/80/v2-b70bb7e451954a0c88accaf5da36f2d4_hd.jpg)

先不要看后两个UNet++, 就看这个不同深度的U-Net的表现, 我们可以看出, 不是越深越好吧, 它背后的传达的信息就是, 不同层次特征的重要性对于不同的数据集是不一样的, 并不是说我设计一个4层的U-Net, 就像原论文给出的那个结构, 就一定对所有数据集的分割问题都最优.

## 使用多尺度的特征

那么接下来是关键, 我们心中的目标很明确了, **就是使用浅层和深层的特征！(因为我并不知道用哪个!)**但是总不能训练这些个U-Net吧, 未免也太多了. 好, 这里可以暂停一下, 想一想, 要你来, 你怎么来利用这些不同深度的, 各自能抓取不同层次的特征的U-Net.

![img](https://pic3.zhimg.com/80/v2-189da65b3cacae53d5f16f423981a01a_hd.jpg)

所以使用了如下的结构:

![img](https://pic4.zhimg.com/80/v2-8b76a55017c4cb60270880d9ac58b1a3_hd.jpg)

我们来看一看, 这样是不是把1～4层的U-Net全给连一起了. 我们来看它们的子集, 包含1层U-Net, 2层U-Net, 以此类推.

* 这个结构的好处就是我不管你哪个深度的特征有效, 我干脆都给你用上, 让网络自己去学习不同深度的特征的重要性.
* 第二个好处是它共享了一个特征提取器, 也就是你不需要训练一堆U-Net, 而是只训练一个encoder, 它的不同层次的特征由不同的decoder路径来还原.

这个encoder依旧可以灵活的用各种不同的backbone来代替.

可惜的是, 这个网络结构是不能被训练的, 原因在于, 不会由任何梯度会经过这个红色区域, 因为它和算loss function的地方是在反向传播时是断开的. 我停顿一下, 大家想一想是不是这么回事.

![img](https://pic1.zhimg.com/80/v2-672c0f585a2bc78098af79bd0f82f438_hd.jpg)

请问, 如果是你, `如何去解决这个问题？`

其实既然已经把结构盘成这样了, 还是很自然能想到的吧, 我这里提供有两个候选的解决方案.

- 第一个是用deep supervision, 强行加梯度是吧, 关于这个, 可以看结尾.
- 第二个解决方案是把结构改成这样子：

![img](https://pic1.zhimg.com/80/v2-9bee627e3ff09bd89ae4cb4e31231d2c_hd.jpg)

我们继续来看这个结构, 请问, `你觉得有什么问题？`

为了回答这个问题, 现在我们和上面那个结构对比一下, 不难发现这个结构强行去掉了U-Net本身自带的长连接. 取而代之的是一系列的短连接. 那么我们来看看U-Net引以为傲的长连接到底有什么优点.

![img](https://pic2.zhimg.com/80/v2-5c2ceb14f8a5409a56c5add37aa82335_hd.jpg)

我们认为, U-Net中的长连接是有必要的, 它联系了输入图像的很多信息, 有助于还原降采样所带来的信息损失, 在一定程度上, 我觉得它和残差的操作非常类似, 也就是residual操作, x+f(x). 我不知道大家是否同意这一个观点. 因此, 我的建议是最好给出一个综合长连接和短连接的方案.

![img](https://pic1.zhimg.com/80/v2-36e3f4c3342bf872fd5fcb8186f91c5c_hd.jpg)

基本上就是这样一个结构没跑了是么. 我们来对比一下一开始的U-Net, 它其实是这个结构的一个子集. 这个结构, 就是我们在MICCAI中发表的UNet++.

我们在回来看这个UNet++, 对于这个主体结构, 我们在论文中给出了一些点评, 说白了就是把原来空心的U-Net填满了, 优势是可以抓取不同层次的特征, 将它们通过特征叠加的方式整合, 不同层次的特征, 或者说不同大小的感受野, 对于大小不一的目标对象的敏感度是不同的, 比如, 感受野大的特征, 可以很容易的识别出大物体的, 但是在实际分割中, 大物体边缘信息和小物体本身是很容易被深层网络一次次的降采样和一次次升采样给弄丢的, 这个时候就可能需要感受野小的特征来帮助. 另一个解读就是如果你横着看其中一层的特征叠加过程, 就像一个去年很火的DenseNet的结构, 非常的巧合, 原先的U-Net, 横着看就很像是Residual的结构, 这个就很有意思了, UNet++对于U-Net分割效果提升可能和DenseNet对于ResNet分类效果的提升, 原因如出一辙, 因此, 在解读中我们也参考了Dense Connection的一些优势, 比方说特征的再利用等等.

这些解读都是很直观的认识, 其实在深度学习里面, 某某结构效果优于某某结构的理由, 或者你加这个操作比不加操作要好, 很多时候是有玄学的味道在里头, 也有大量的工作也在探索深度网络的可解释性. 关于UNet++的主体结构, 我不想花时间赘述了.

## 深监督与剪枝

刚才说了, 一个非常直接的解决方案就是深监督, 也就是deep supervision. 这个概念不是新的, 有很多对U-Net对改进论文中也有用到, 具体的实现操作就是在图中 ![X^{0,1}](http://www.zhihu.com/equation?tex=X%5E%7B0%2C1%7D) , ![X^{0,2}](http://www.zhihu.com/equation?tex=X%5E%7B0%2C2%7D) , ![X^{0,3}](http://www.zhihu.com/equation?tex=X%5E%7B0%2C3%7D) , ![X^{0,4}](http://www.zhihu.com/equation?tex=X%5E%7B0%2C4%7D) 后面加一个1x1的卷积核, 相当于去监督每个level, 或者每个分支的U-Net的输出.

我给大家提供三个用Deep Supervision的结构来比较, 一个就是这个, 第二个是加在UC Berkeley提出的结构上, 最后一个是加在UNet++上, `请问是你认为哪个会更棒？`这是一个开放问题, 我这里不做展开讨论, 我们论文中使用的是最后一个.

因为deep supervision具体应该套在哪个结构上面不是我想说的重点, deep supervision的优点也在很多论文中有讲解, 我这里想着重讨论的是当它配合上这样一个填满的U-Net结构时, 带来其中一个非常棒的优势. 此处强烈建议暂停, 想一想, `如果我在训练过程中在各个level的子网络中加了这种深监督, 可以带来怎样的好处呢？`

![img](https://pic2.zhimg.com/80/v2-debfd1acf4b9f2a63eea5db0fe920ef5_hd.jpg)

**两个字：剪枝. **

同时引出三个问题：

- `为什么UNet++可以被剪枝`(测试的时候)
- `如何剪枝`
- `好处在哪里`

![img](https://pic1.zhimg.com/80/v2-53e23bbec3f7667ad87a2513667b1c08_hd.jpg)

我们来看看为什么可以剪枝, 这张图特别的精彩. 关注被剪掉的这部分, 你会发现, 在测试的阶段, 由于输入的图像只会前向传播, 扔掉这部分对前面的输出完全没有影响的, 而在训练阶段, 因为既有前向, 又有反向传播, 被剪掉的部分是会帮助其他部分做权重更新的. 这两句话同样重要, 我再重复一遍, 测试时, 剪掉部分对剩余结构不做影响, 训练时, 剪掉部分对剩余部分有影响. 这意味什么？

因为在深监督的过程中, 每个子网络的输出都其实已经是图像的分割结果了, 所以如果小的子网络的输出结果已经足够好了, 我们可以随意的剪掉那些多余的部分了.

来看一下这个动图, 为了定义的方便起见, 我们把每个剪完剩下的子网络根据它们的深度命名为UNet++ L1, L2, L3, L4, 后面会简称为L1～L4. 最理想的状态是什么？当然是L1喽, 如果L1的输出结果足够好, 剪完以后的分割网络会变得非常的小.

这里我想问两个问题：

- `为什么要在测试的时候剪枝, 而不是直接拿剪完的L1, L2, L3训练？`
- `怎么去决定剪多少？`

对于为什么要在测试的时候剪枝, 而不是直接拿剪完的L1, L2, L3训练, 我们的解释其实上一页ppt上面写了, **剪掉的部分在训练时的反向传播中是有贡献的**, 如果直接拿L1, L2, L3训练, 就相当于只训练了不同深度的U-Net, 最后的结果会很差.

第二个问题, 如何去决定剪多少, 还是比较好回答的. 因为在训练模型的时候会把数据分为训练集, 验证集和测试集, 训练集上是一定拟合的很好的, 测试集是我们不能碰的, 所以我们会**根据子网络在验证集的结果来决定剪多少**. 所谓的验证集就是一开始从训练集中分出来的数据, 用来监测训练过程用的.

好, 讲完了思路, 我们来看结果.

![img](https://pic1.zhimg.com/80/v2-62f88a068a51c718b18d9bd2fb0697b4_hd.jpg)

先看看L1～L4的网络参数量, 差了好多, L1只有0.1M, 而L4有9M, 也就是理论上如果L1的结果我是满意的, 那么模型可以被剪掉的参数达到98.8%. 不过根据我们的四个数据集, L1的效果并不会那么好, 因为太浅了嘛. 但是其中有三个数据集显示L2的结果和L4已经非常接近了, **也就是说对于这三个数据集, 在测试阶段, 我们不需要用9M的网络, 用半M的网络足够了**.

回想一下一开始我提出的问题, 网络需要多深合适, 这幅图是不是就一目了然.

网络的深度和数据集的难度是有关系的, 这四个数据集当中, 第二个, 也就是息肉分割是最难的, 大家可以看到纵坐标, 它代表分割的评价指标, 越大越好, 其他都能达到挺高的, 但是唯独息肉分割只有在30左右, **对于比较难的数据集, 可以看到网络越深, 它的分割结果是在不断上升的**. 对于大多数比较简单的分割问题, 其实并不需要非常深, 非常大的网络就可以达到很不错的精度了.

横坐标代表的是在测试阶段, 单显卡12G的TITAN X (Pascal)下, 分割一万张图需要的时间. 我们可以看到不同的模型大小, 测试的时间差好多. 如果比较L2和L4的话, 就差了三倍之多.

![img](https://pic2.zhimg.com/80/v2-b17679c0f8c3b4a3ab49f11d359c7c1d_hd.jpg)

对于测试的速度, 用这一幅图会更清晰. 我们统计了用不同的模型, 1秒钟可以分割多少的图. 如果用L2来代替L4的话, 速度确实能提升三倍.

剪枝应用最多的就是在移动手机端了, 根据模型的参数量, 如果L2得到的效果和L4相近, 模型的内存可以省18倍. 还是非常可观的数目.

关于剪枝的这部分我认为是对原先的U-Net的一个很大的改观, 原来的结构过于刻板, 并且没有很好的利用不用层级的特征.

简单的总结一下, UNet++的第一个优势就是精度的提升, 这个应该它整合了不同层次的特征所带来的, 第二个是灵活的网络结构配合深监督, 让参数量巨大的深度网络在可接受的精度范围内大幅度的缩减参数量.
