# Deep TextSpotter: An End-to-End Trainable Scene Text Localization and Recognition Framework

tags: Deep TextSpotter, OCR, YOLOv2, STN, CTC

## 前言

Deep TextSpotter的创新点并不多，基本上遵循了传统OCR或者物体检测的两步走的流程（图1），即先进行场景文字检测，再进行文字识别。在这个算法中，检测模块基于[YOLOv2](https://senliuy.gitbooks.io/advanced-deep-learning/content/chapter1/yolo9000-better-faster-stronger.html)，识别模块基于[STN](https://senliuy.gitbooks.io/advanced-deep-learning/content/chapter1/spatial-transform-networks.html)，损失函数则使用了精度的[CTC](https://senliuy.gitbooks.io/advanced-deep-learning/content/di-er-zhang-ff1a-xu-lie-mo-xing/connectionist-temporal-classification-labelling-unsegmented-sequence-data-with-recurrent-neural-networks.html)。这几个算法在当时都是state-of-the-art的，因此其效果达到了最优也不难理解。这三个知识点已分别在本书的第四章，第五章和第二章进行了解析，算法细节可参考具体内容或者阅读论文。这里不在对上面三个算法的细节再做重复，只会对Deep TextSpotter的流程做一下梳理和解释。

Deep TextSpotter的一个创新点是将NMS放到了识别之后，使用识别置信度替代了传统的检测置信度。

#### 图1： Deep TextSpotter算法流程

![](../../.gitbook/assets/DeepTextSpotter_1.png)

## 1. Deep TextSpotter解析

### 1.1 全卷积网络

**为什么使用YOLOv2**：在YOLOv2的文章中我们讲过，YOLOv2使用了高分辨率的迁移学习提高了网络对高分辨率图像的检测效果，这个能力在端到端的文字检测及识别中非常重要。因为过分的降采样将造成文本区域的识别问题。

**Feature Map的尺寸**：网络的框架也采样去YOLOv2中在$$3\times3$$卷积中插入$$1\times1$$卷积进行非线性化的结构。对于一张尺寸为$$W\times H$$的输入图像，在网络中会通过5个Max Pooling进行降采样，得到尺寸为$$\frac{W}{32} \times \frac{H}{32}$$的的Feature Map。在Deep TextSpotter中，每隔20个Epoch会更换一次输入图像的尺寸，尺寸的变化范围是$$\{352,416,480,544,608\}$$

**全卷积**：Deep TextSpotter使用了Global Average Pooling代替全连接实现非线性化，从而使网络成为全卷积网络，原因已多次提及：保留特征向量的位置信息。

### 1.2 候选区域提取

**输出向量**：Deep TextSpotter的检测部分预测了6个值，它们分别是坐标$$r_x$$，$$r_y$$，尺寸$$r_w$$，$$r_h$$，检测置信度$$r_p$$以及比YOLOv2增加的一个旋转角度$$r_\theta$$。其中角度使用了弧度值，即 $$\theta \in (-\frac{\pi}{2}, \frac{\pi}{2})$$。其它几个预测值则采用了YOLOv2中使用的预测相对值。

**锚点聚类**：Deep TextSpotter将锚点聚了14类，锚点形状见图2。

#### 图2：Deep TextSpotter聚类得到的锚点

![](../../.gitbook/assets/DeepTextSpotter_2.png)

**样本采样**：匹配正负锚点时使用了YOLOv1中介绍的bipartite策略，即只有和Ground Truth的IOU最大的锚点为正样本，其余均为负样本。

**NMS**：Deep TextSpotter的一个创新点在于并没有这检测完之后就使用NMS，考虑到的一个问题是只覆盖部分文字区域的检测框的置信度有可能高于检测到完整文字区域的置信度要高。在这里，只使用阈值$$\theta=0.1$$过滤掉部分置信度非常低的样本。

### 1.3 双线性插值

经过YOLOv2得到的检测框的尺寸，角度，比例等都是不同的，为了产生长度固定的特征向量。Faster R-CNN等方法采用的是ROI Pooling，Deep TextSpotter则是使用STN的策略，STN不仅能产生长度固定的特征向量，还能学到图像的仿射变换矩阵，是非常适用于OCR领域的。

Deep TextSpotter产生的是长宽比不变，宽度固定位$$H'=32$$的Feature Map，即对于一个检测到的区域$$U\in R^{w\times h \times C}$$，其得到的Feature Map的为$$V \in R^{\frac{wH'}{h}\times H' \times C}$$。

Feature Map中位置$$(x',y')$$处的值为:

$$
V_{x',y'}^c = \sum^w_{x=1}\sum^h_{y=1}\mathbf{U}_{x,y}^c \kappa(x - \mathcal{T}_x(x')) \kappa(y - \mathcal{T}_y(y'))
$$

这个过程便是双线性插值，不理解的参考我在STN的解释。

其中$$(x,y)$$为检测框中的一点，\(x',y'\)为输出Feature Map上的一点，范围是$$x'\in[0, \frac{wH'}{h}-1], y\in[0,H'-1]$$。$$\mathcal{T}(\cdot)$$为位置转移函数，应该是仿射变换矩阵。$$\kappa(v)=max(0,1-|v|)$$为双线性插值函数。

在STN中我们讲过其双线性插值是可导的，因此到目前为止该过程是端到端的。

### 1.4 文本识别

Deep TextSpotter使用的是基于字符序列识别方式，骨干网络使用的是基于图3的全卷积网络。网络支持宽的变长输入，但是高是固定的。图3中的Recurrent Convolution猜测是使用的结构，论文中没有给出注释。

损失函数使用的是CTC，参考第二章[CTC](https://senliuy.gitbooks.io/advanced-deep-learning/content/di-er-zhang-ff1a-xu-lie-mo-xing/connectionist-temporal-classification-labelling-unsegmented-sequence-data-with-recurrent-neural-networks.html)一节，此处不再废话。

#### 图3：Deep TextSpotter识别部分的全卷积网络

![](../../.gitbook/assets/DeepTextSpotter.png)

### 1.5 NMS

在检测中搁置的NMS将在识别完之后在使用，NMS中使用的置信度是识别的置信度，阈值给的是$$0.5$$。

## 总结

由于使用了STN连接检测和识别，Deep TextSpotter是一个真正的端到端模型，所以在训练的过程中，只需要针对分类的loss进行训练。

Deep TextSpotter，算法最核心的部件是STN，但是并没有cite该论文，识别中的RCNN也没cite，进而导致了理解上的困难，这毛病可不好。

