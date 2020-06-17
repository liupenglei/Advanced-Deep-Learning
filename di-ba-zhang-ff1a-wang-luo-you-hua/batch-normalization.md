# Batch Normalization

tags: Normalization

## 前言

Batch Normalization\(BN\)是深度学习中非常好用的一个算法，加入BN层的网络往往更加稳定并且BN还起到了一定的正则化的作用。在这篇文章中，我们将详细介绍BN的技术细节以及其能工作的原因。

在提出BN的文章中，作者BN能工作的原因是BN解决了普通网络的内部协变量偏移（Internel Covariate Shift, ICS）的问题，所谓ICS是指网络各层的分布不一致，网络需要适应这种不一致从而增加了学习的难度。而在中，作者通过实验验证了BN其实和ICS的关系并不大，其能工作的原因是使损失平面更加平滑，并给出了其结论的数学证明。

## 1. BN详解

### 1.1 内部协变量偏移

BN的提出是基于小批量随机梯度下降（mini-batch SGD）的，mini-batch SGD是介于one-example SGD和full-batch SGD的一个折中方案，其优点是比full-batch SGD有更小的硬件需求，比one-example SGD有更好的收敛速度和并行能力。随机梯度下降的缺点是对参数比较敏感，较大的学习率和不合适的初始化值均有可能导致训练过程中发生梯度消失或者梯度爆炸的现象的出现。BN的出现则有效的解决了这个问题。

在Sergey Ioffe的文章中，他们认为BN的主要贡献是减弱了内部协变量偏移（ICS）的问题，论文中对ICS的定义是：as the change in the distribution of network activations due to the change in network parameters during training。作者认为ICS是导致网络收敛的慢的罪魁祸首，因为模型需要学习在训练过程中会不断变化的隐层输入分布。作者提出BN的动机是企图在训练过程中将每一层的隐层节点的输入固定下来，这样就可以避免ICS的问题了。

在深度学习训练中，白化（Whiten）是加速收敛的一个小Trick，所谓白化是指将图像像素点变化到均值为0，方差为1的正态分布。我们知道在深度学习中，第$$i$$层的输出会直接作为第$$i+1$$层的输入，所以我们能不能对神经网络的每一层的输入都做一次白化呢？其实BN就是这么做的。

### 1.2 梯度饱和

我们知道sigmoid激活函数和tanh激活函数存在梯度饱和的区域，其原因是激活函数的输入值过大或者过小，其得到的激活函数的梯度值会非常接近于0，使得网络的收敛速度减慢。传统的方法是使用不存在梯度饱和区域的激活函数，例如ReLU等。BN也可以缓解梯度饱和的问题，它的策略是在调用激活函数之前将$$WX+b$$的值归一化到梯度值比较大的区域。假设激活函数为$$g$$，BN应在$$g$$之前使用：

$$
z = g(\text{BN}(Wx+b))
$$

### 1.3 BN的训练过程

如果按照传统白化的方法，整个数据集都会参与归一化的计算，但是这种过程无疑是非常耗时的。BN的归一化过程是以批量为单位的。如图1所示，假设一个批量有$$n$$个样本 $$\mathcal{B} = \{x_1, x_2, ..., x_m\}$$，每个样本有$$d$$个特征，那么这个批量的每个样本第$$k$$个特征的归一化后的值为

$$
\hat{x}^{(k)} = \frac{x^{(k)} - E[x^{(k)}]}{\sqrt{\text{Var}[x^{(k)}]}}
$$

其中$$E$$和 $$\text{Var}$$分别表示在第$$k$$个特征上这个批量中所有的样本的均值和方差。

 ![&#x56FE;1&#xFF1A;&#x795E;&#x7ECF;&#x7F51;&#x7EDC;&#x7684;BN&#x793A;&#x610F;&#x56FE;](../.gitbook/assets/BN_1.png)图1：神经网络的BN示意图

这种表示会对模型的收敛有帮助，但是也可能破坏已经学习到的特征。为了解决这个问题，BN添加了两个可以学习的变量$$\beta$$和$$\gamma$$用于控制网络能够表达直接映射，也就是能够还原BN之前学习到的特征。

$$
y^{(k)} = \gamma^{(k)} \hat{x}^{(k)} + \beta^{(k)}
$$

当$$\gamma^{(k)} = \sqrt{\text{Var}[x^{(k)}]}$$并且$$\beta^{(k)} = E[x^{(k)}]$$时，$$y^{(k)} = x^{(k)}$$，也就是说经过BN操作之后的网络容量是不小于没有BN操作的网络容量的。

综上所述，BN可以看做一个以$$\gamma$$和$$\beta$$为参数的，从$$x_{1...m}$$到$$y_{1...m}$$的一个映射，表示为

$$
BN_{\gamma,\beta}:x_{1...m}\rightarrow y_{1...m}
$$

BN的伪代码如算法1所示

![](../.gitbook/assets/BN_a1.png)

在训练时，我们需要计算BN的反向传播过程，感兴趣的同学可以自行推导，这里直接给出结论（$$l$$表示损失函数）。

$$
\frac{\partial l}{\partial \hat{x}_i} = \frac{\partial l}{\partial y_i} \cdot \gamma
$$

$$
\frac{\partial l}{\partial \sigma_\mathcal{B}^2} = \sum^m_{i=1} \frac{\partial l}{\partial \hat{x}_i} \cdot (x_i - \mu_\mathcal{B}) \cdot \frac{-1}{2} (\sigma_\mathcal{B}^2 + \epsilon)^{-\frac{3}{2}}
$$

$$
\frac{\partial l}{\partial \mu_\mathcal{B}} = (\sum^m_{i=1}\frac{\partial l}{\partial \hat{x}_i} \cdot \frac{-1}{\sqrt{\sigma_\mathcal{B}^2 + \epsilon}}) + \frac{\partial l}{\partial \sigma_\mathcal{B}^2} \cdot \frac{\sum_{i=1}^m -2(x_i - \mu_\mathcal{B})}{m}
$$

$$
\frac{\partial l}{\partial x_i} = \frac{\partial l}{\partial \hat{x}_i} \cdot \frac{1}{\sqrt{\sigma_\mathcal{B}^2 + \epsilon}}) + \frac{\partial l}{\partial \sigma_\mathcal{B}^2} \cdot \frac{2(x_i - \mu_\mathcal{B})}{m} + \frac{\partial l}{ \partial \mu_\mathcal{B}} \cdot \frac{1}{m}
$$

$$
\frac{\partial l}{\partial \gamma} = \sum^m_{i=1} \frac{\partial l}{\partial y_i} \cdot \hat{x}_i
$$

$$
\frac{\partial l}{\partial \beta} = \sum^m_{i=1} \frac{\partial l}{\partial y_i}
$$

通过上面的式子中我们可以看出BN是处处可导的，因此可以直接作为层的形式加入到神经网络中。

### 1.4 BN的测试过程

在训练的时候，我们采用SGD算法可以获得该批量中样本的均值和方差。但是在测试的时候，数据都是以单个样本的形式输入到网络中的。在计算BN层的输出的时候，我们需要获取的均值和方差是通过训练集统计得到的。具体的讲，我们会从训练集中随机取多个批量的数据集，每个批量的样本数是$$m$$，测试的时候使用的均值和方差是这些批量的均值。

$$
\text{E}(x) \leftarrow \text{E}_{\mathcal{B}}[\mu_\mathcal{B}]
$$

$$
\text{Var}(x) \leftarrow \frac{m}{m-1}\text{E}_{\mathcal{B}}[\sigma^2_\mathcal{B}]
$$

上面的过程明显非常耗时，更多的开源框架是在训练的时候，顺便就把采样到的样本的均值和方差保留了下来。在Keras中，这个变量叫做滑动平均（moving average），对应的均值叫做滑动均值（moving mean），方差叫做滑动方差（moving variance）。它们均使用`moving_average_update`进行更新。在测试的时候则使用滑动均值和滑动方差代替上面的$$\text{E}(x)$$和$$\text{Var}(x)$$。

滑动均值和滑动方差的更新如下：

$$
\text{E}_{moving}(x) = m \times \text{E}_{moving}(x) + (1-m) \times \text{E}_{sample}(x)
$$

$$
\text{Var}_{moving}(x) = m \times \text{Var}_{moving}(x) + (1-m) \times \text{Var}_{sample}(x)
$$

其中$$\text{E}_{moving}(x)$$表示滑动均值，$$\text{E}_{sample}(x)$$表示采样均值，方差定义类似。$$m$$表示遗忘因子momentum，默认值是0.99。

滑动均值和滑动方差，以及可学参数$$\beta$$，$$\gamma$$均是对输入特征的线性操作，因此可以这两个操作合并起来。

$$
y = \frac{\gamma}{\sqrt{\text{Var}[x] + \epsilon}} \cdot x + (\beta - \frac{\gamma \text{E}[x]}{\sqrt{\text{Var}[x] + \epsilon}})
$$

### 1.5 卷积网络中的BN

BN除了可以应用在MLP上，其在CNN网络中的表现也非常好，但是在RNN上的表现并不好，具体原因后面解释，这里详细介绍BN在卷积网络中的使用方法。

卷积网络和MLP的不同点是卷积网络中每个样本的隐层节点的输出是三维（宽度，高度，维度）的，而MLP是一维的，如图2所示。

 ![&#x56FE;2&#xFF1A;&#x5377;&#x79EF;&#x7F51;&#x7EDC;&#x7684;BN&#x793A;&#x610F;&#x56FE;](../.gitbook/assets/BN_2.png)图2：卷积网络的BN示意图

在图2中，假设一个批量有$$m$$个样本，Feature Map的尺寸是$$p\times q$$，通道数是$$d$$。在卷积网络的中，BN的操作是以Feature Map为单位的，因此一个BN要统计的数据个数为$$m\times p \times q$$，每个Feature Map使用一组$$\gamma$$和$$\beta$$。

## 2. BN的背后原理

### 2.1 BN与ICS无关

最近MIT的一篇文章否定了BN的背后原理是因为其减少了ICS的问题。在这篇文章中，作者通过两个实验验证了ICS和BN的关系非常小的观点。

第一个实验验证了ICS和网络性能的关系并不大，在这个实验中作者向使用了BN的网络中加入了随机噪声，目的是使这个网络的ICS更加严重。实验结果表明虽然加入了随机噪声的BN的ICS问题更加严重，但是它的性能是要优于没有使用BN的普通网络的，如图3所示。

 ![&#x56FE;3&#xFF1A;BN&#xFF0C;&#x666E;&#x901A;&#x7F51;&#x7EDC;&#x7684;&#xFF0C;&#x52A0;&#x5165;&#x566A;&#x97F3;&#x7684;BN&#x7684;ICS&#x5B9E;&#x9A8C;&#x6570;&#x636E;&#x56FE;](../.gitbook/assets/BN_3.png)图3：BN，普通网络的，加入噪音的BN的ICS实验数据图

第二个实验验证了BN并不会减小ICS，有时候甚至还能提升ICS。在这个实验中，作者对ICS定义为：

**定义**：假设$$\mathcal{L}$$是损失值，$$W_1^{(t)}, ..., W_k^{(t)}$$是在$$k$$个层中在时间$$t$$时的参数值，$$(x^{(t)}, y^{(t)})$$是在$$t$$时刻的输入特征和标签值，ICS定义为在时间$$t$$时，第$$i$$个隐层节点的两个变量的距离$$||G_{t,i}-G'_{t,i}||_2$$，其中

$$
G_{t,i} = \nabla_{w_i^{(t)}} \mathcal{L}(W_1^{(t)}, ..., W_k^{(t)};x^{(t)}, y^{(t)})
$$

$$
G'_{t,i} = \nabla_{w_i^{(t)}} \mathcal{L}(W_1^{(t+1)}, ..., W_{i-1}^{(t+1)}, W_i^{(t)}, W_{i+1}^{(t)}..., W_k^{(t)};x^{(t)}, y^{(t)})
$$

两个变量的区别在于$$W_1, ..., W_{i-1}$$是$$t$$时刻的还是$$t+1$$时刻的，其中$$G_{t,i}$$表示更新梯度时使用的参数，$$G'_{t,i}$$表示使用这批样本更新后的新的参数。在上面提到的欧氏距离中，值越接近0说明ICS越小。另外一个相似度指标是cosine夹角，值越接近于1说明ICS越小。图4的实验结果（25层的DLN）表明BN和ICS的关系并不是很大。

 ![&#x56FE;4&#xFF1A;&#x666E;&#x901A;&#x7F51;&#x7EDC;&#x548C;&#x5E26;BN&#x7684;&#x7F51;&#x7EDC;&#x5728;&#x4E24;&#x4E2A;ICS&#x6307;&#x6807;&#x4E0A;&#x7684;&#x5B9E;&#x9A8C;&#x7ED3;&#x679C;](../.gitbook/assets/BN_4.png)图4：普通网络和带BN的网络在两个ICS指标上的实验结果

### 2.2 BN与损失平面

通过上面两个实验，作者认为BN和ICS的关系不大，那么BN为什么效果好呢，作者认为BN的作用是平滑了损失平面（loss landscape），关于损失平面的介绍，参考文章，这篇文章中介绍了损失平面的概念，并指出残差网络和DenseNet均起到了平滑损失平面的作用，因此他们具有较快的收敛速度。

作者证明了BN处理之后的损失函数满足Lipschitz连续，即损失函数的梯度小于一个常量，因此网络的损失平面不会震荡的过于严重。

$$
||f(x_1) - f(x_2)|| \leq L||x_1 - x_2||
$$

而且损失函数的梯度也满足Lipschitz连续，这里叫做$$\beta$$-平滑，即斜率的斜率也不会超过一个常量。

$$
||\nabla f(x_1) - \nabla f(x_2)|| \leq \beta ||x_1 - x_2||
$$

作者认为当着两个常量的值均比较小的时候，损失平面就可以看做是平滑的。图5是加入没有跳跃连接的网络和加入跳跃连接（残差网络）的网络的损失平面的可视化，作者认为BN和残差网络对损失平面平滑的效果类似。

 ![&#x56FE;5&#xFF1A;&#x635F;&#x5931;&#x5E73;&#x9762;&#x53EF;&#x89C6;&#x5316;&#xFF08;&#x65E0;&#x8DF3;&#x8DC3;&#x8FDE;&#x63A5; vs ResNet&#xFF09;](../.gitbook/assets/BN_5.png)图5：损失平面可视化（无跳跃连接 vs ResNet）

通过上面的分析，我们知道BN收敛快的原因是由于BN产生了更光滑的损失平面。其实类似于BN的能平滑损失平面的策略均能起到加速收敛的效果，作者在论文中尝试了$$l_p$$-norm的策略（即通过取它们$$l_p$$-norm的均值的方式进行归一化）。实验结果表明它们均取得了和BN类似的效果。

### 2.3 BN的数学定理

作者对于自己的猜想，给出了5个定理，引理以及观察并在附录中给出了证明，由于本人的数学能力有限，这些证明有些看不懂，有需要的同学自行查看证明过程。

**定理4.1**: 设$$\hat{\mathcal{L}}$$为BN网络的损失函数，$$\mathcal{L}$$为普通网络的损失函数，它们满足：

$$
||\nabla_{\mathbf{y}_j} \hat{\mathcal{L}}||^2 \leq \frac{\gamma^2}{\sigma_j^2}(||\nabla_{\mathbf{y}_j} \mathcal{L}||^2 - \frac{1}{m} \langle \mathbf{1}, \nabla_{\mathbf{y}_j} \mathcal{L} \rangle^2 - \frac{1}{\sqrt{m}}\langle \nabla_{\mathbf{y}_j}\mathcal{L},\hat{\mathbf{y}_j} \rangle^2)
$$

在绝大多数场景中，$$\sigma$$作为不可控的项往往值是要大于$$\gamma$$的，因此证明了BN可以使神经网络满足Lipschitz连续；

**定理4.2**：假设$$\hat{\mathbf{g}}_j = \nabla_{\mathbf{y}_j} \mathcal{L}$$，$$\mathbf{H}_{jj} = \frac{\partial \mathcal{L}}{\partial \mathbf{y}_j \partial \mathbf{y}_j}$$是Hessian矩阵：

$$
(\nabla_{\mathbf{y}_j} \hat{\mathcal{L}})^T \frac{\partial \hat{\mathcal{L}}}{\partial \mathbf{y}_j \partial \mathbf{y}_j} (\nabla_{\mathbf{y}_j} \hat{\mathcal{L}}) \leq
 \frac{\gamma^2}{\sigma_j^2} (\hat{\mathbf{g}}_j^T \mathbf{H}_{jj} \hat{\mathbf{g}}_j - \frac{1}{m\gamma}\langle\hat{\mathbf{g}}_j\hat{\mathbf{y}}_j \rangle||\frac{\partial\hat{\mathcal{L}}}{\partial \mathbf{y}_j}||^2)
$$

同理，4.2证明了BN是神经网络的损失函数的梯度也满足Lipschitz连续；

**观察4.3**：由于BN可以还原为直接映射，所以普通神经网络的最优损失平面也一定存在于带BN的网络中。

**定理4.4**：设带有BN的网络的损失为$$\hat{\mathcal{L}}$$，与之等价的无BN的损失为$$\mathcal{L}$$，它们满足如果$$g_j = \max_{||X||\leq \lambda} ||\nabla_w \mathcal{L}||^2$$，$$\hat{g}_j = \max_{||X|| < \lambda} ||\nabla_w \hat{\mathcal{L}}||^2$$，我们可以推出：

$$
\hat{g}_j \leq \frac{\gamma^2}{\sigma^2}(g_j^2 - m \mu_{g_j}^2 - \lambda^2\langle \nabla_{\mathbf{y}_j}, \hat{\mathbf{y}_j}\rangle^2)
$$

定理4.3证明了BN可以降低损失函数梯度的上界。

**引理4.5**：假设$$W^*$$和$$\hat{W}^*$$分别是普通神经网络和带BN的神经网络的局部最优解的权值，对于任意的初始化$$W_0$$，我们有：

$$
||W_0 - \hat{W}^*||^2 \leq ||W_0 - W^*||^2 - \frac{1}{||W^*||^2}(||W^*||^2 - \langle W^*, W_0\rangle)^2
$$

引理4.5证明了BN对参数的不同初始化更加不敏感。

## 总结

BN是深度学习调参中非常好用的策略之一（另外一个可能就是Dropout），当你的模型发生梯度消失/爆炸或者损失值震荡比较严重的时候，在BN中加入网络往往能取得非常好的效果。

BN也有一些不是非常适用的场景，在遇见这些场景时要谨慎的使用BN：

* 受制于硬件限制，每个Batch的尺寸比较小，这时候谨慎使用BN；
* 在类似于RNN的动态网络中谨慎使用BN；
* 训练数据集和测试数据集方差较大的时候。

在Ioffe的论文中，他们认为BN能work的原因是因为减轻了ICS的问题，而在Santurkar的论文中则对齐进行了否定。它们结论的得出非常依赖他们自己给出的ICS的数学定义，这个定义是否不能说不对，但是感觉不够精确，BN真的和ICS没有一点关系吗？我是觉得不一定。

