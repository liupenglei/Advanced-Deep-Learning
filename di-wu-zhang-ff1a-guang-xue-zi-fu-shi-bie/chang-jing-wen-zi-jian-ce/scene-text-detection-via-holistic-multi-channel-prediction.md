# Scene Text Detection via Holistic, Multi-Channel Prediction

## 前言

本文是在边缘检测经典算法[HED](https://senliuy.gitbooks.io/advanced-deep-learning/content/qi-ta-ying-yong/holistically-nested-edge-detection.html)之上的扩展，在这篇论文中我们讲过HED算法可以无缝转移到实例分割场景中。而这篇论文正是将场景文字检测任务转换成语义分割任务来实现HED用于文字检测的。图1是HED在身份证上进行边缘检测得到的掩码图，从图1中我们可以看出HED在文字检测场景中也是有一定效果的。

#### 图1：HED在身份证上得到的掩码图

![](../../.gitbook/assets/HMCP_1.jpeg)

HED之所以能用于场景文字检测一个重要的原因是文字区域具有很强的边缘特征。

论文的题目为Holistic，Multi-Channel Prediction（HMCP），其中Holistic表示算法基于HED，Multi-Channel表示该算法使用多个Channel的标签训练模型。也就是为了提升HED用于文字检测的精度，这篇文章做的改进是将模型任务由单任务模型变成是由文本行分割，字符分割和字符间连接角度构成的多任务系统。由于HMCP采用的是语义分割的形式，所以其检测框可以扩展到多边形或者是带旋转角度的四边形，这也更符合具有严重仿射变换的真实场景。

## 1. HMCP详解

HMCP的流程如图2：\(a\)是输入图像，\(b\)是预测的三个mask，分别是文本行掩码，字符掩码和字符间连接角度掩码，\(c\)便是根据\(b\)的三个掩码得到的检测框。

#### 图2：HMCP流程

![](../../.gitbook/assets/HMCP_2.png)

那么问题来了：  
1. 如何构建三个channel掩码的标签值；  
2. 如何根据预测的掩码构建文本行。

### 1.1 HMCP的标签值

在文本检测数据集中，常见的标签类型有QUAD和RBOX两种形式。其中QUAD是指标签值包含四个点$$\mathbf{G}=\{(x_i,y_i)|i\in\{1,2,3,4\}\}$$，由这四个点构成的不规则四边形（Quadrangle）便是文本区域。QBOX则是由一个规则矩形$$\mathbf{R}$$和其旋转角度$$\theta$$构成，即$$\mathbf{G} = \{\mathbf{R}, \theta \}$$。QUAD和RBOX是可以相互转换的。

HMCP的数据的Ground Truth分别包含基于文本行和基于字符的标签构成\(QUAD或RBOX\)，如图3.\(b\)和图3.\(c\)。数据集中只有基于文本行的Ground Truth，其基于单词的Ground Truth是通过SWT得到的。\(d\)是基于文本行Ground Truth得到的二进制掩码图，文本区域的值为1，非文本区域的值为0。\(e\)是基于单词的二进制掩码图，掩码也是0或者1。由于字符间的距离往往比较小，为了能区分不同字符之间的掩码，正样本掩码的尺寸被压缩到了Ground Truth的一半。\(f\)是字符间连接角度掩码，其值为$$[-\pi/2, \pi/2]$$，然后再通过归一化映射到$$[0,1]$$之间。角度的值是由RBOX形式的Ground Truth得到的值。

#### 图3：HMCP的Ground Truth以及三种Mask

![](../../.gitbook/assets/HMCP_3.png)

### 1.2 HMCP的骨干网络

HMCP的骨干网络继承自HED，如图4所示。HMCP的主干网络使用的是VGG-16，在每个block降采样之前通过反卷积得到和输入图像大小相同的Feature Map，最后通过fuse层将5个side branch的Feature Map拼接起来并得到预测值。HMCP和HED的不同之处是HMCP的输出节点有三个任务。

#### 图4：HMCP的骨干网络

![](../../.gitbook/assets/HMCP_4.png)

### 1.3 HMCP的损失函数

### 1.3.1 训练

设HMCP的训练集为$$S=\{(X_n,y_n),n=1,...,N\}$$，其中$$N$$是样本的数量。标签$$Y_n$$由三个掩码图构成，即$$Y_n=\{R_n,C_n,\Theta_n\}$$，其中$$R_n=\{r_j^{(n)}\in\{0,1\},j=1,...,|R_n|\}$$表示文本区域的二进制掩码图，$$C_n=\{c_j^{(n)}\in\{0,1\},j=1,...,|C_n|\}$$是字符的二进制掩码图，$$\Theta_n=\{\theta_j^{(n)}\in\{0,1\},j=1,...,|\Theta_n|\}$$是相邻字符的连接角度。注意只有当$$r_j^{(n)}=1$$时$$\theta_j^{(n)}$$才有效。

与HED不同的是HMCP的损失函数没有使用side branch，即损失函数仅由fuse层构成：

$$
\mathcal{L} = \mathcal{L}_{\text{fuse}}(\mathbf{W}, \mathbf{w}, Y, \hat{Y})
$$

其中$$\mathbf{W}$$为VGG-16部分的参数，$$\mathbf{w}$$为fuse层部分的参数。$$\hat{Y}=\{\hat{R}, \hat{C}, \hat{\Theta}\}$$是预测值：

$$
\hat{Y} = \text{CNN}(X,\mathbf{W},\mathbf{w})
$$

$$\mathcal{L}_{\text{fuse}}(\mathbf{W}, \mathbf{w}, Y, \hat{Y})$$由三个子任务构成:

$$
\mathcal{L}_{\text{fuse}}(\mathbf{W}, \mathbf{w}, Y,\hat{Y}) = \lambda_1\Delta_r(\mathbf{W}, \mathbf{w},R,\hat{R}) + 
\lambda_2\Delta_c(\mathbf{W}, \mathbf{w},C,\hat{C}) +
\lambda_3\Delta_o(\mathbf{W}, \mathbf{w},\Theta,\hat{\Theta},R)
$$

其中$$\lambda_1 + \lambda_2 + \lambda_3 = 1$$。$$\Delta_r(\mathbf{W}, \mathbf{w})$$表示基于文本掩码的损失值，$$\Delta_c(\mathbf{W}, \mathbf{w})$$是基于字符掩码的损失值，两个均是使用HED采用过的类别平衡交叉熵损失函数：

$$
\Delta_r(\mathbf{W}, \mathbf{w},R,\hat{R}) = -\beta_R\sum_{j=1}^{|R|}R_j\text{log}Pr(\hat{R}_j=1;\mathbf{W}, \mathbf{w}) + (1-\beta_R) \sum_{j=1}^{|R|}(1-R_j) \text{log}Pr(\hat{R}_j=0;\mathbf{W}, \mathbf{w})
$$

上式中的$$\beta$$为平衡因子$$\beta_R=\frac{|R_-|}{|R|}$$，$$|R_-|$$为文本区域Ground Truth中负样本个数，$$|R|$$为所有样本的个数。

基于字符掩码的损失值与$$\Delta_r(\mathbf{W}, \mathbf{w},R,\hat{R})$$类似：

$$
\Delta_c(\mathbf{W}, \mathbf{w},C,\hat{C}) = -\beta_C \sum_{j=1}^{|C|}C_j\text{log}Pr(\hat{C}_j=1;\mathbf{W}, \mathbf{w}) + (1-\beta_C) \sum_{j=1}^{|C|}(1-C_j) \text{log}Pr(\hat{C}_j=0;\mathbf{W}, \mathbf{w})
$$

$$\Delta_o(\mathbf{W}, \mathbf{w},\Theta,\hat{\Theta},R)$$定义为：

$$
\Delta_o(\mathbf{W}, \mathbf{w},\Theta,\hat{\Theta},R)=\sum_{j=1}^{|R|}R_j(\text{sin}(\pi|\hat{\Theta}_j - \Theta_j|))
$$

### 1.4 检测

### 1.4.1 预测掩码

HMCP的预测的三个Map均是由fuse层得到，因为作者发现side branch的引入反而会伤害模型的性能。

### 1.4.2 检测框生成

HMCP的检测过程如图5：给定输入图像\(a\)得到\(b\)，\(c\)，\(d\)三组掩码。通过自适应阈值，我们可以得到\(e\)以及\(f\)的分别基于文本区域和基于字符的检测框，需要注意的是我们在制作字符掩码的时候掩码区域被压缩了一半，所以在这里我们需要将它们还原回来。

#### 图5：HMCP的检测框生成流程

![](../../.gitbook/assets/HMCP_5.png)

对于一个文本区域，假设其中有$$m$$个字符区域：$$U = \{u_i,i=1,...,m\}$$，通过德劳内三角化（Delaunary Triangulation）得到的三角形$$T$$我们可以得到一个由相邻字符间连接构成的图$$G=\{U,E\}$$，如图\(g\)。

德劳内三角化能够有效的去除字符区域之间不必要的链接，维基百科给的德劳内三角化的定义是指德劳内三角化是一种三角剖分$$DT(P)$$，使得在P中没有点严格处于$$DT(P)$$中任意一个三角形外接圆的内部。德劳内三角化最大化了此三角剖分中三角形的最小角，换句话，此算法尽量避免出现“极瘦”的三角形，如图6。

#### 图6：德劳内三角化

![](../../.gitbook/assets/HMCP_6.png)

在图$$G=\{U,E\}$$中，$$U$$表示图的顶点表示字符的位置。$$E$$表示图的边表示两个字符之间的相似度，边的权值$$w$$的计算方式为：

$$
w = \left\{
\begin{array}{}
s(i,j) & \text{if}\quad e\in T\\
0 & \text{otherwise}
\end{array}
\right.
$$

$$s(i,j)$$有空间相似性$$a(i,j)$$和角度相似性$$o(i,j)$$计算得到：

$$
s(i,j)=\frac{2a(i,j)o(i,j)}{a(i,j)+o(i,j)}
$$

空间相似性定义为

$$
a(i,j) = \text{exp}(-\frac{d^2(i,j)}{2D^2})
$$

其中$$d(i,j)$$是$$(u_i,u_j)$$之间的欧氏距离，$$D$$是整个德劳内三角形的边长均值。

角度相似性$$o(i,j)$$定义为：

$$
o(i,j)=cos(\Lambda(\phi(i,j)-\psi(i,j)))
$$

其中$$\phi(i,j)$$表示$$(u_i,u_j)$$形成的直线与水平方向的夹角，$$\psi(i,j)$$即两个节点之间的区域的所有像素点的夹角的平均值，$$\Lambda$$表示两个角度的夹角。

综上可以看出，$$s(i,j)$$和$$d(i,j)$$成反比，也就是两个字符的距离越近，这两个字符的相似度越高；$$s(i,j)$$和$$\phi(i,j)$$也是成反比，即两个字符的角度越小（越接近水平文本行），两个字符的相似度越高。

得到带权值的无向图之后，使用Kruskal等方法生成最大生成树$$M$$。由图到树的生成过程是一个剪枝的过程，如若在树的基础上再进行剪枝，此时树便会分裂成由若干棵个树组成的森林。在最大生成树的基础上剪枝$$K-1$$次会生成$$K$$棵树，$$K \geq 1$$。如图5.\(a\)所示，文本区域由两行文本构成，显然我们需要在$$M$$的基础上进行一次剪枝才能生成两棵树，从而通过两棵树分别确定一个文本区域。

那么，给定一张图片，我们如何其文本行的个数（树的个数）$$K$$呢？论文给出的策略是最大化$$S_{vm}$$，$$S_{vm}$$的计算方式为：

$$
S_{vm} = \sum_{i=1}^{K}\frac{\lambda_{i1}}{\lambda_{i2}}
$$

其中$$\lambda_{i1}$$和$$\lambda_{i2}$$是协方差矩阵$$C_i$$最大和第二大的两个特征值。而$$C_i$$是由一个文本区域（论文中叫做cluster）的字符中心点坐标构成的矩阵。

在一些样本中，会存在如图7所示的弧形文本区域，而传统的方法是基于一条直线上的文本行设计的。为了解决这个问题，HMCP引入了阈值$$\tau$$，权值大于$$\tau$$的边既不会被删除，也不会被选中。

#### 图7：HMCP用于弧形文字区域检测

![](../../.gitbook/assets/HMCP.png)

其实关于上面这两段不是很理解工作原理，带后续补充。

## 总结

这篇论文巧妙的将实例分割用于场景文字检测领域，其三个掩码图的多任务模型的设计非常漂亮。最后通过三个掩码图生成检测框的算法技术性非常高，是一篇非常值得学习的论文。

