# 三维重建方法3D Gaussian Splatting详解
## Gaussian的基础知识
### 一维和三维的Gasussian
![一维和三维的Gasussian](https://github.com/user-attachments/assets/fe9accb5-e7b0-4476-b0c5-48de4f761f9d)

### v的概率密度函数
![v的概率密度函数](https://github.com/user-attachments/assets/b8f228d7-5652-44be-8ee0-fa86c8f80174)


### x的概率密度函数
![x的概率密度函数](https://github.com/user-attachments/assets/2d9c58e7-c5d6-430b-ba30-e126d329e727)
![x的概率密度函数](https://github.com/user-attachments/assets/c7a60086-69c2-4448-b480-4fafb95c6a56)

### 把变换矩阵A转换为协方差矩阵：
![把变换矩阵A转换为协方差矩阵](https://github.com/user-attachments/assets/5fe42ead-70c2-450b-af67-f5a76c2e2d22)

## 3D 高斯泼溅的整体流程
### 框架图
<img width="1000" alt="整体框架" src="https://github.com/user-attachments/assets/b65e3826-5bde-4860-b0c5-0fdad162c8a2">

### 流程简介
Gaussian Splatting的输入是一组静态场景的图像，以及由SfM校准的相应摄像机，通过colmap产生一个稀疏点云作为衍生物。从初始的SfM点云出发，以每一个点为中心生成3D高斯分布，**由位置（均值）、协方差矩阵和不透明度定义**。然后利用相机参数把投影点投影到图像平面上（也就是splatting）。接下来从splatting的痕迹中进行**快速光栅化**，得到渲染的图像。将渲染的图像和Ground Trueth图像求**loss**。最后沿着蓝色的箭头进行反向传播，还有**自适应的密度控制模块**，最终传到每一个点上的梯度，决定是否需要克隆或者分裂。梯度同时会传播到Projection（协方差）中，**更新其位置、协方差矩阵、球谐函数、不透明度**等参数，完成一次迭代。
## 流程详解
### 关于“椭球”（即高斯/高斯球）的理解及形成：
1.**“椭球”形象引入**：

就像给 3 个顶点能表达任意一个 3D 三角形，继而很好的构建三维模型（ Mesh 的思想），研究者自然希望构建三维模型的基础元素能覆盖足够多样的几何体，而椭球就是一种很好的基础元素。
但是，只使用一个个实椭球来构建模型，是远远不够的。这时，我们就想寻找一种方式，来将椭球这种元素变得“强大”一点——能够构建出多样的表示。
那么，如果将椭球的边界变得不清晰，成为一个渐变的，可以融合的呢？如同真实世界中的“黑滴效应”：当你将两个物体慢慢相互靠近直到重叠时，这两个物体的影子之间的颜色会慢慢变深，形成黑滴状：
![黑滴状](https://github.com/user-attachments/assets/cd390012-05f1-433b-bd94-1559262a0ec0)

我们可以看到，如果这样的话，椭球就不再是那个有清晰边界的椭球了。继而，我们可以构建出更多样的表达。由此，我们便会想到概率分布，再由椭球的形状，我们又会想到同样是椭球状的三维高斯分布。

2.**3D高斯椭球集的创建——位置与形状**

<img width="1000" alt="位置与形状" src="https://github.com/user-attachments/assets/5d048a61-46d6-4eb6-87f6-64e467a97d4f">

![构建步骤](https://github.com/user-attachments/assets/e3bd1eb0-a528-4e7f-beb7-796285bebee0)


我们要清楚的是3D高斯依旧是一个一个的点，每一个点都是一个3D高斯分布。一维的高斯是一个钟形曲线，二维的高斯是一个像小山包的形状，而三维高斯就像是一个椭球。而3D高斯只要做到一件事情，优化每一个点的参数。

每一个3D高斯的点都存储了以下信息：

**Position（位置信息）**：这是一个椭球的中心，也是高斯分布的均值，用u表示。

**Covariance（协方差矩阵：Σ=RSS⊤R⊤）**：次变量可以控制椭球的大小，形状和方向，用Σ表示。

**Opacity（不透明度）**：用于控制不透明度，用于Splatting时的渲染，用α表示。

**Spherical harmonics（球谐函数）**：球谐函数，你和视角相关的外观，用c表示。

3.**3D高斯椭球集的创建——颜色和不透明度**

**颜色信息**:点云颜色(r,g,b)--使用球谐函数来表示，使得点云在不同角度呈现不同颜色，并有利于提高迭代效率(代码中采用4阶)。

**不透明度信息**:点云不透明度，密度优化α。

**球谐函数**：
![球谐](https://github.com/user-attachments/assets/e6cc415c-2d12-46f8-84ae-85e3231132e5)
![球](https://github.com/user-attachments/assets/1b7e8d51-6a9a-4c76-ad28-086aef57df05)

球谐函数理解过程较为复杂，讲解详见b站《中恩实验室》视频：https://www.bilibili.com/video/BV1aw411E7bA?vd_source=25c0dee0c9a65590c654aa7f42c9d718
### splatting过程：
1.形象解释：在这一步中，三维高斯椭球体被投影到二维图像空间椭圆形，以进行渲染。
Splatting 的中文翻译为：**抛雪球法**。很形象，我们可以想象一下，把一个雪球（高斯球）扔到一个玻璃盘子上，雪球散开以后，在撞击中心的雪量（对图像的贡献）最大，而随着离撞击中心距离的增加，雪量（贡献） 减少。脑补一下盘子上的图像，其实我们可以自然而然的想到二维高斯分布的密度函数。
![抛雪球](https://github.com/user-attachments/assets/8a92fdc7-65be-4f93-b636-2bbaeb07c84b)

2.原理：从3D协方差矩阵Σ到2d协方差矩阵Σ′：Σ′=Jk​MΣMTJkT
![渲染原理](https://github.com/user-attachments/assets/70fb46aa-e1de-461d-a625-c8683ccaabef)

具体推导过程可以参考文章：(https://zhuanlan.zhihu.com/p/666465701)

3.渲染方式：Tiled-based Rasterizer（快速可微光栅化）
* 确定视锥体
确定相机位姿进一步确定视锥体，有利于剔除视锥体以外的部分，防止出现浪费的计算。
* 3D->2D(splatting) 

![可微快速光栅化](https://github.com/user-attachments/assets/612261b4-796e-4393-96c6-33e4a36db05b)
  - 该过程将3D空间内的3D椭球投影到2D空间中进行渲染，该过程主要对表示椭球的协方差矩阵乘上变换矩阵加以实现。类比到现实生活中，就好像将雪球用力掷向墙壁，啪的一声绽开，在墙上留下了一个中心密集，向四周逐渐变浅变稀疏的雪块,具体步骤为：
![渲染步骤](https://github.com/user-attachments/assets/d3dcf02d-b813-4dd1-9ed9-e703c93ba1fd)

* 渲染过程的特点一：分块
  - 在处理图像时，为了减少对每个像素进行高斯运算的计算成本，论文采用了一种不同的方法:它不
是在像素级别进行精确计算，而是将精度降低到了更宏观的图块级别。这个过程首先涉及将整个图
像划分成多个不重看的小块，这些在原始的研究论文中被形象地称为“砖块”
。按照原始论文的建议，每个砖块包括 16 \times 16 个像素。
  - 接下来，会进一步识别出哪些砖块与特定的高斯投影相交。考虑到一个高斯投影可能会覆盖多个砖
块，一种有效的处理方式是复制这个东西，并为每个复制出的高斯分配一个唯一的标识符，即与之
相交的 Tie 的 ID。这样，每个砖块都与一个或多个高斯相关联，这些高斯标识了该砖块在图像中
的位置和重要性。通过这种方式，3DGS 能有效地降低计算复杂度，同时还保持了图像处理的效率
和准确性。
![分块](https://github.com/user-attachments/assets/e6e49200-79c8-45ea-8c5e-2195feece055)

  - 概括来说就是，为了减少对每个像素进行高斯运算的计算成本，降低计算精度，把图像分成多给不重叠的小块，被称为“砖块”，每个砖块包括16个像素，然后，会进一步识别出哪些砖块与特定的高斯投影相交。一个高斯投影可能会覆盖多个砖块，处理方式就是为每个复制出的高斯分配一个唯一的标识符，即与之相交的 Tile 的 ID。每个砖块都与一个或多个高斯相关联，这些高斯标识了该砖块在图像中的位置和重要性和准确性。
* 渲染过程的特点二：排序
![排序](https://github.com/user-attachments/assets/4dead03f-ea49-46c3-afbf-f9295ed083a8)
  
* 渲染公式：
![渲染公式解析](https://github.com/user-attachments/assets/78e77f7b-07ab-48e8-b812-c13e46e0cd26)
  
  详细解析可参考：【3D Gaussian Splatting原理速通(三)--迭代参数与渲染】https://www.bilibili.com/video/BV1h5411y7ds?vd_source=25c0dee0c9a65590c654aa7f42c9d718
## loss公式及作用（损失函数）
![loss](https://github.com/user-attachments/assets/abcf78ab-a1fc-445a-bf48-d102ae694ce1)

* 其中λ是一个权重因子，一般取0.2，loss公式主要是用于渲染的图像和Ground Trueth图像求loss，然后进行一个梯度回传，更新高斯球的位置、协方差矩阵、球谐函数、不透明度等参数
### 参数更新 
![参数更新](https://github.com/user-attachments/assets/4ee3973f-508e-4c7a-98a7-ff871aa4e2b9)

### 自适应密度控制
![自适应密度控制2](https://github.com/user-attachments/assets/c9b5eb8c-d8ea-4e88-a6c0-aeb9ab8dbaae)
* 初始化:3DGS 建议从 SfM 产生的稀疏点云初始化或随机初始化高斯，可以直接调用 COLMAP
库来完成这一步，然后进行点的密集化和剪枝以控制3D高斯的密度。当由于某种原因无法获得点
云时，可以使用随机初始化来代替，但可能会降低最终的重建质量。

* 点密集化:在点密集化阶段，3DGS自适应地增加高斯的密度，以更好地捕捉场景的细节。该过程
特别关注缺失几何特征或高斯过于分散的区域。密集化在一定数量的迭代后执行，比如100个选
代，针对在视图空间中具有较大位置梯度(即超过特定阈值)的高斯。其包括在未充分重建的区域
克隆小高斯或在过度重建的区域分裂大高斯。对于克降，创建高斯的复制体并朝着位置梯度移动。
对于分裂，用两个较小的高斯替换一个大高斯，按照特定因子减小它们的尺度。这一步旨在于 3D
空间中寻求高斯的最佳分布和表示，增强重建的整体质量。

* 点的剪枝:点的剪枝阶段移除冗余或影响较小的高斯，可以在某种程度上看作是一种正则化过程。
一般消除几乎是透明的高斯(α低于指定阈值)和在世界空间或视图空间中过大的高斯。此外，为
防止输入相机附近的高斯密度不合理地增加，这些高斯会在固定次数的迭代后，将 α 设置为接近0
的值。该步骤在保证高斯的精度和有效性的情况下，能节约计算资源。

* 概括的说，点密集化和点的剪枝是两种优化方式，点密集化基于梯度变化来判断，计算其协方差，方差很小-->小高斯克隆高斯来适应 Under-Reconstruction；方差很大-->大高斯分割成两个高斯 0ver-Reconstruction；点的剪枝是根据高斯球的不透明度α，其中不透明度低于设置的阈值或者离相机近的一些点会进行删除。总的就是为了更好地拟合形状。
### 伪代码流程
总结训练过程每个迭代主要执行以下的操作：
每 1000 次迭代，增加球谐系数的阶数。
随机选择一个相机视角。
渲染图像，获取视点空间点、能见度过滤器和半径等信息。
计算损失（L1 损失和 DSSIM 损失的加权和），进行反向传播。
通过无梯度的上下文进行后续操作：
根据迭代次数进行点云密度操作（densification）：
更新最大半径信息。
根据条件进行点云密度增加和修剪。
进行优化器的参数更新。
在整个训练过程中，这些步骤循环执行，逐渐优化模型参数，进行损失计算和反向传播，同时根据条件进行点云密度操作和保存检查点，以逐步提升模型性能。
![伪代码](https://github.com/user-attachments/assets/a473b898-422b-4f5d-a1c4-5f1b04e4b5cc)

### 部分背景知识
![背景知识1](https://github.com/user-attachments/assets/736bccf8-e4d4-4c8c-a7e2-6675b662555f)
![背景知识2](https://github.com/user-attachments/assets/d77cd0be-6495-47f6-a5f7-a3d5c9c5705f)
### 入门视频推荐

* B站up主中恩实验室系列视频：【3D Gaussian Splatting原理速通(一)--三维高斯概念-哔哩哔哩】 https://b23.tv/vOBnBbX
* 开源论文讲解：【【论文讲解】用点云结合3D高斯构建辐射场，成为快速训练、实时渲染的新SOTA！-哔哩哔哩】 https://b23.tv/CqGTUeJ 

### 参考文章及视频

* 【3D Gaussian Splatting原理速通(一)--三维高斯概念-哔哩哔哩】 https://b23.tv/vOBnBbX
* https://zhuanlan.zhihu.com/p/680669616
* https://zhuanlan.zhihu.com/p/678877999
* https://www.bilibili.com/video/BV1uV4y1Y7cA/?share_source=copy_web&vd_source=bd133237b66721b62ed05d453aa32bac
* https://blog.csdn.net/m0_63843473/article/details/136283934
* http://t.csdnimg.cn/FMX1r

---
- 戴海锋（2024.8.30）
