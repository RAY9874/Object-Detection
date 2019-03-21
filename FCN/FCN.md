# FCN

**CVPR2015的最佳论文**

全卷积网络与CNN不同，一般CNN会在卷积层的后面接全连接作为输出，但全连接层需要输入固定长度的向量，导致了输入的图片必须进行处理成为相同大小才可以经过卷积输入到FC中。在FCN中，作者使用卷积层代替全连接层，采用反卷积层对最后一个卷基层的特征图进行上采样，使它恢复到输入图像相同的尺寸，从而可以对每一个像素都产生一个预测，同时保留了原始输入图像中的空间信息，最后在上采样的特征图进行像素的分类。

## 模型结构

![strut](img/strut.jpg)

- 蓝色 ： 卷积   
- 绿色：maxpooling
- 橙色：反卷积
- 灰色：裁剪
- 黄色 ： eltwise+

***什么是eltwise***

> Concat层虽然利用到了上下文的语义信息，但仅仅是将其拼接起来，之所以能起到效果，在于它在不增加算法复杂度的情形下增加了channel数目。那有没有直接关联上下文的语义信息呢？答案是Eltwise层，被广泛使用，屡试不爽，并且我们常常拿它和Concat比较，所以我常常一起说这两个层。我们普遍认为，像这样的“encoder-decoder”的过程，有助于利用较高维度的feature map信息，有利于提高小目标的检测效果。
>
> Eltwise层有三种类型的操作：product(点乘)、sum(求和)、max(取最大值)，



可见模型最后以卷积作为结尾

在虚线下方的反卷积过程中，结合了3 部分的感受野，上采样的结果更加照顾细节

最后对拼接的3个featuremap，逐像素分类



## 反卷积

反卷积和卷积类似，都是相乘相加的运算。只不过后者是多对一，前者是一对多。而反卷积的前向和后向传播，只用颠倒卷积的前后向传播即可。所以无论优化还是后向传播算法都是没有问题。

## 如何逐像素分类

按[github上的一个人写的](https://github.com/shekkizh/FCN.tensorflow/blob/master/FCN.py)，最后一层卷积：

```python
conv_t3 = utils.conv2d_transpose_strided(fuse_2, W_t3, b_t3,
                          output_shape=deconv_shape3, stride=8)
annotation_pred = tf.argmax(conv_t3, dimension=3, name="prediction")

return tf.expand_dims(annotation_pred, dim=3), conv_t3

....很多代码....

loss = tf.reduce_mean ( 
    	(tf.nn.sparse_softmax_cross_entropy_with_logits(
        	logits=logits,
        	labels=tf.squeeze(annotation, squeeze_dims=[3]),                           name="entropy")))
```



conv_t3的shape是原图（shape[0],shape[1],shape[2],num_classes）

就是这个sparse_softmax_cross_entropy_with_logits 起到了逐像素softmax的功能

## 启发

- 想要精确预测每个像素的分割结果 ，必须经历从大到小，再从小到大的两个过程 
- 在升采样过程中，分阶段增大比一步到位效果更好 
- 在升采样的每个阶段，使用降采样对应层的特征进行辅助



## 缺点

是对各个像素进行分类，没有充分考虑像素与像素之间的关系。忽略了在通常的基于像素分类的分割方法中使用的空间规整（spatial regularization）步骤，缺乏空间一致性。