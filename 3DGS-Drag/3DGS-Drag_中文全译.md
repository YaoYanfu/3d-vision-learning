# 3DGS-Drag：通过拖动高斯基元实现直观的基于点的三维编辑

**作者：** Jiahua Dong、Yu-Xiong Wang  
**单位：** 伊利诺伊大学厄巴纳-香槟分校  
**邮箱：** {jiahuad2, yxw}@illinois.edu  
**原文页眉：** 发表于 ICLR 2025 的会议论文  
**原文预印本标识：** arXiv:2601.07963v1 [cs.CV]，2026 年 1 月 12 日  
**代码：** [3DGS-Drag 项目仓库](https://github.com/Dongjiahua/3DGS-Drag)

> 译文说明：依据用户提供的 PDF 翻译，保留章节、引用、公式编号、图表说明和附录。图像本身请对照原 PDF，图中标签在相应图注附近译出。参考文献保留原始书目信息，并附中文题名。文中的“我们”均指原论文作者；公式和实验数据按原文保留。

**图 1：我们提出的 3DGS-Drag 框架能够实现高质量的三维拖动编辑。** 用户只需输入三维控制点（圆形）和目标点（三角形）。我们的方法能够精确地将控制点移动到目标点，同时保留整体内容与细节。

图中标签：用户编辑；拖动结果渲染图。

## 摘要

随着生成模型的进步，三维内容创作的变革潜力正逐步得到释放。近年来，能够产生几何变化的直观拖动编辑在二维编辑领域引起了广泛关注，但对于三维场景而言，这仍然是一项挑战。本文提出 3DGS-Drag：一种基于点的三维编辑框架，可对真实三维场景进行高效、直观的拖动操作。我们的方法连接了基于变形与基于二维编辑的两类三维编辑方法，解决了它们在几何相关内容编辑方面的局限。我们利用了两项关键创新：利用三维高斯泼溅实现一致几何修改的**变形引导**，以及用于内容修正和视觉质量提升的**扩散引导**。此外，**渐进式编辑策略**进一步支持大幅度的三维拖动编辑。我们的方法支持多种编辑，包括运动变化、形状调整、修补和内容扩展。实验结果表明，3DGS-Drag 在多种场景中均有效，并在几何相关的三维内容编辑方面达到当前最先进的性能。值得一提的是，该方法编辑效率较高，在单张 RTX 4090 GPU 上只需 10 至 20 分钟。代码已公开于 [项目仓库](https://github.com/Dongjiahua/3DGS-Drag)。

## 1 引言

近年来，三维场景表示技术取得了显著进展，例如神经辐射场（NeRF）（Mildenhall et al., 2021）和三维高斯泼溅（3DGS）（Kerbl et al., 2023）。这些方法彻底改变了三维内容的采集、表示和合成方式，提供了前所未有的细节与真实感。受到这些方法的成功以及二维生成模型蓬勃发展的启发（Rombach et al., 2022），近期三维生成研究（Tang et al., 2024; Poole et al., 2023）已经能够高质量、高效率地生成三维内容。然而，对三维场景进行精确、直观的编辑仍然具有挑战性，尤其是与二维图像已经具备的成熟编辑能力相比。虽然 DragGAN（Pan et al., 2023）等二维编辑方法提供了基于点的操作方式，但将这种功能扩展到三维场景仍然面临重大的技术障碍。

具体而言，其中尚未得到充分探索的能力，是实现**伴随几何变化的直观内容编辑**。近期三维编辑的发展大致可以分为两类：基于变形的方法和基于二维编辑的方法。基于变形的方法（Huang et al., 2024; Xie et al., 2024）主要关注运动编辑，通常假设具有较强的几何先验（Xie et al., 2024），或依赖视频来学习运动模式（Huang et al., 2024）。这些方法除了需要充分的先验信息外，本身也无法直观地编辑未曾观测到的内容。对于基于二维编辑的方法，近期工作（Haque et al., 2023; Dong & Wang, 2023; Chen et al., 2024a）尝试使用二维扩散模型编辑包含不同视角图像的数据集，从而提取二维扩散模型的编辑能力（Brooks et al., 2023）。这些方法仍然局限于外观修改和小幅几何调整，因为较大幅度的二维几何编辑无法收敛为一致的三维结果。它们采用的文本引导有时也会导致错误编辑，因为扩散模型无法正确理解文本提示。**如何将变形方法的几何编辑能力与二维编辑模型的内容编辑能力结合起来，尚未得到充分研究。**

基于这些观察，我们提出 3DGS-Drag——**一种面向真实场景的直观三维拖动编辑方法**。我们扩展了 DragGAN（Pan et al., 2023）灵活的编辑形式，以三维控制点和目标点作为输入，目标是实现与几何相关的三维内容编辑。我们的核心见解是充分利用两种三维内容编辑引导，对不同视角下的编辑结果施加显式约束，使其保持一致，并朝着目标三维点进行优化。

第一种引导是**变形引导**。得益于三维高斯泼溅的显式表示（Kerbl et al., 2023），我们提出了一种简单而有效、无需先验信息的变形策略。通过这一策略，我们直接对三维高斯基元进行变形，并将其用作不同视角的引导。此外，高斯基元的变形有利于在变形后的空间附近进行优化，从而简化精细几何的生成。

第二种引导是**扩散引导**。由于我们的设定中不存在先验信息，变形后的高斯基元总会带有错误内容和伪影。我们利用扩散模型修正内容并提升视觉质量。这种引导基于我们的观察：经过微调的扩散模型可以充当三维场景中保持视角一致性的编辑器。因此，在前述变形引导的基础上，它能够取得更好的一致性。

为了支持更具挑战性的三维拖动编辑，我们进一步提出了一种**渐进式编辑策略**。具体而言，我们将拖动操作分为若干区间，逐步进行编辑，并通过一种三维重定位策略保证编辑的连续性。最终，实验结果验证了 3DGS-Drag 在多种场景与编辑任务中的有效性。我们解决了三维拖动操作中的挑战，并展示了相较于已有技术更好的多视角一致性。

我们的主要贡献可概括如下：

1. 提出了一种新的三维场景编辑框架，其特点是采用基于点的拖动编辑方式。
2. 提出了一种有效结合三维变形引导与扩散引导的方法，以进行几何相关的三维内容编辑。
3. 进一步提出渐进式拖动编辑方法，以改善编辑结果。
4. 大量评估表明，我们的方法在这一设定下取得了当前最先进的结果，其能力隐含地涵盖运动变化、形状调整、修补和内容扩展。

## 2 相关工作

### 2.1 二维图像编辑

早期图像生成方法主要随着生成对抗网络（GAN）的发展而兴起（Goodfellow et al., 2014; Karras et al., 2019）。基于其潜在表示，早期工作尝试修改潜在变量，以调整图像的某些属性或内容（Abdal et al., 2021; Endo, 2022; Härkönen et al., 2020; Leimkühler & Drettakis, 2021）。然而，由于 GAN 模型的能力有限，而且潜在编码采用隐式表示，很难实现高质量、细致的编辑。近年来，扩散模型在文本到图像任务中展现出巨大潜力（Rombach et al., 2022）。其特征图表示和大规模数据使众多图像编辑方法得以发展（Kawar et al., 2023; Ramesh et al., 2022; Meng et al., 2022; Brooks et al., 2023）。SDEdit（Meng et al., 2022）通过加噪和去噪过程保留结构信息，同时改变细节。Instruct-Pix2Pix（Brooks et al., 2023）构建了指令编辑数据集，并训练扩散模型按照指令编辑图像。与以往方法相比，Instruct-Pix2Pix 展现出更好的编辑一致性。

尽管基于文本的图像编辑能够生成高保真结果，却难以实现细粒度编辑。DragGAN（Pan et al., 2023）提出了一种基于点的交互式编辑方法：用户输入若干控制点和目标点，然后通过优化潜在变量，将控制点移动到目标位置。为提高通用性，DragDiffusion（Shi et al., 2024）将这一技术迁移到扩散模型中（Rombach et al., 2022）。随后，SDE-Drag（Nie et al., 2024）和 RegionDrag（Lu et al., 2024）进一步提升了性能。这些基于扩散的方法需要进行反演—前向生成过程，因此操作耗时较长。此外，这类二维模型无法保证三维一致性，因此不能直接应用于三维场景。

本文采用二维扩散模型进行具有三维一致性的视图修正。我们的编辑不仅能够生成符合直觉的新内容，还能去除潜在的三维伪影。

### 2.2 基于二维编辑的三维编辑

在 3DGS（Kerbl et al., 2023）出现之前，神经辐射场（NeRF）（Mildenhall et al., 2021）通常被用作连接三维表示与二维模型的桥梁。早期 NeRF 工作只能处理颜色和形状调整（Chiang et al., 2022; Huang et al., 2021; 2022; Wu et al., 2023; Bao et al., 2023; Zhang et al., 2022; Jambon et al., 2023）。SNeRF（Nguyen-Phuoc et al., 2022）提出采用图像风格化模型，取得了高质量的风格化结果。此后，NeRF-Art（Wang et al., 2023）利用 CLIP（Radford et al., 2021）将知识蒸馏到 NeRF 中。然而，由于 CLIP 并非生成模型，而且高度依赖语义，这一方法无法获得高保真结果。Instruct-NeRF2NeRF（Haque et al., 2023）提出利用 Instruct-Pix2Pix 模型迭代编辑数据集，能够依据多种指令编辑不同场景。ViCA-NeRF（Dong & Wang, 2023）提出直接编辑数据集，而无需微调 NeRF。具体而言，该方法利用深度信息实现多视角一致的编辑。DreamEditor（Zhuang et al., 2023）提出使用经过微调的 DreamBooth（Ruiz et al., 2023）辅助编辑。ConsistentDreamer（Chen et al., 2024a）进一步微调 ControlNet，以获得更精细的编辑。然而，这些方法均受到不同视角间三维一致性问题的限制，因此只能进行细微的几何变化。PDS（Koo et al., 2024）提出了一种新的蒸馏损失来改善结果，但仍存在渲染质量下降、难以实现充分几何编辑的问题。

受到 3DGS 高效率的启发，近期方法（Fang et al., 2024; Chen et al., 2024b; Chen & Wang, 2024）提出将 NeRF 编辑的成功经验迁移到三维高斯基元上。然而，这些方法主要沿用了 Instruct-NeRF2NeRF（Haque et al., 2023）的思路，只是替换了三维表示，因此存在类似的局限。一些方法（Xie et al., 2023; Shen et al., 2024; Yoo et al., 2024; Dong et al., 2024）尝试将拖动操作扩展到三维，但仅限于处理单个物体。相比之下，我们的方法利用 3DGS 的显式表示，并以真实场景为研究对象。

### 2.3 基于变形的三维编辑

三维变形是一项具有挑战性的任务，因为其目标是生成未曾观测到的运动。传统方法（Sorkine-Hornung & Alexa, 2007; Sorkine, 2005）使用特定的拉普拉斯坐标进行网格变形。近年来，研究者开始关注 NeRF 和 3DGS 等三维表示中的变形。具体而言，Xu & Harada（2022）提出构建三维控制笼，将其作为运动先验来引导变形。Yuan et al.（2022）先从 NeRF 重建网格，再转而对网格进行变形。NeuralEditor（Chen et al., 2023）要求输入稠密点云的变形，并采用类似点的 NeRF 结构实施变形。上述方法均需要较强的几何先验才能进行编辑，这在实践中获取困难，也不够方便。PhysGaussian（Xie et al., 2024）将高斯椭球视为连续体，并引入物理机制。SC-GS（Huang et al., 2024）采样控制点，将它们构建成表示结构的图，以引导运动。然而，物理模拟和连续体假设使 PhysGaussian 缺乏灵活性，而且仅适用于连续场景。SC-GS 的控制点是对稠密点的近似，因此同样依赖对物体几何的充分采集。它还需要以动态场景为输入，建立运动先验知识。

这些对先验知识的较高要求或严格假设，使上述方法不适合大型真实场景，因为这类场景往往只有部分视角信息，且布局复杂。此外，这些方法不具备创建新部分的能力。我们并不着重设计更好的变形方法，而是提出一种更简单的 3DGS 变形策略，提供粗略变形。由于二维生成模型（Rombach et al., 2022）已经具备对正常运动和内容的认知，我们借助这些知识来实现更灵活的三维编辑。

## 3 方法

### 3.1 预备知识

**三维高斯泼溅。** 三维高斯泼溅（3D Gaussian Splatting）（Kerbl et al., 2023）使用一组三维高斯基元来表示三维信息，在物体与场景重建任务中展现了有效性。每个高斯基元由中心 $\mu\in\mathbb{R}^3$、缩放因子 $s\in\mathbb{R}^3$ 和旋转四元数 $q\in\mathbb{R}^4$ 表征。该模型还包含用于体渲染的不透明度值 $\alpha\in\mathbb{R}$ 和颜色特征 $c\in\mathbb{R}^d$，其中 $d$ 表示自由度。完整参数集记为 $\Gamma$，其中 
$$
$\Gamma^i=\{\mu^i,s^i,q^i,\alpha^i,c^i\}$
$$
表示第 $i$ 个高斯基元的参数。

### 3.2 框架概述

我们的框架如图2所示。其输入为一个预训练的三维高斯泼溅模型，以及若干控制点和与之对应的目标点。具体而言，控制点记为 $p_h^{n\times3}$，目标点记为 $p_t^{n\times3}$，其中 $n$ 为控制点数量。我们的目标是在保留相似内容的同时，将控制点所在部分移动到目标位置。根据输入点的不同，这一过程可能涉及外观和几何变化，从而通过对用户友好的输入实现更具挑战性的编辑。

**图2：3DGS-Drag 概览。** 给定一个训练好的三维高斯泼溅模型及其数据集，我们使用多步编辑调度器计算第 $i$ 步的中间控制点 $p'_h(i)$ 和目标点 $p'_t(i)$。在每一步中，我们首先使用控制点和目标点对三维高斯基元进行变形。随后，渲染各个视角的图像，并通过扩散模型对其进行修正。最终修正后的图像将用于训练三维高斯基元，以提升质量。扩散模型通过 LoRA 进行微调，从而获得更一致的编辑结果。

图中标签：用户输入；三维控制点 $p_h$；三维目标点 $p_t$；多步编辑调度器（第 $u$ 步）；控制点重定位与目标点调度；$p'_h(u),p'_t(u)$；图像缓冲区；更新；LoRA 微调；变形；三维高斯基元；渲染；扩散；变形引导；扩散引导；数据集；训练三维高斯基元。

二维拖动编辑技术（Pan et al., 2023; Shi et al., 2024; Lu et al., 2024）的思路是优化或操作二维图像的反演特征，与之不同，我们使用**基于变形的几何引导**和**基于扩散的外观引导**进行三维编辑。对于单步拖动操作，我们首先根据给定的控制点和目标点对三维高斯基元进行变形（第3.3节）。这种变形采用复制粘贴的方式，以获得更大的编辑灵活性。由于拖动操作存在点稀疏和长距离的挑战，变形后的高斯基元所产生的渲染结果视觉质量较差，且内容不正确。因此，我们提出对渲染图像进行扩散引导的图像修正（第3.4节），以高效地纠正内容并去除伪影。为了处理变化幅度更大的编辑，我们提出多步编辑调度器，逐步编辑场景（第3.5节）。由于整个过程被分为若干阶段，用户在获得满意结果后，可以在任意中间步骤停止。

### 3.3 用于几何修改的变形引导

我们的目标是通过对三维场景进行变形来提供几何引导，因此采用 3DGS，以利用其显式表示和高效率。在我们的任务中，变形面临两个挑战：（1）给定稀疏控制点和长距离拖动目标，如何在不对标准 3DGS 进行结构修改的情况下，对三维高斯基元进行近似变形；（2）如何避免退化为直接变形，从而使移动、延伸等编辑具有更大的灵活性。我们提出的解决方法如下。通过这一方法，即使只有有限的点信息，我们也能对 3DGS 实现可靠的变形。

**拖动变形。** 3DGS 的显式表示支持高效的三维变形与调整。然而，仅给定控制点和目标点，无法精确计算真实的变形函数。因此，我们对其进行近似，以提供粗略的几何引导。对于第 $i$ 个控制点 $p_h^i$，我们将三维空间中与它距离不超过某一阈值 $\tau$ 的高斯基元 $P_h^i$ 分配给该点。这些高斯基元被视为将由该控制点引导并进行变形的基元。将 $\{P_h^i\mid i=1,2,\ldots,n\}$ 的并集记为 $P_h=\bigcup_{i=1}^{n}P_h^i$。

首先，我们计算每个控制点的平移和旋转。对于平移，直接计算为：$\Delta p_h^i=p_t^i-p_h^i$。对于旋转，其作用并不是进一步改变控制点的位置，而是表示潜在的朝向变化。由于三维高斯基元也由旋转 $q$ 参数化，这一参数对于引导高斯基元变形至关重要。然而，我们的控制点只是坐标，不包含朝向信息。为了近似旋转，我们计算该控制点与距离最近的前 $K$ 个（$K=2$）控制点 $\{p_h^k\mid k\in N_h^i\}$ 之间的相对旋转，其中 $N_h^i$ 为最近的前 $K$ 个控制点的索引。考虑到点的稀疏性，我们使用线性权重。具体而言，权重计算如下：

$$
w_h^{ik}=1-\frac{\left\|p_h^i-p_h^k\right\|_2^2}{\sum_{j\in N_h^i}\left\|p_h^i-p_h^j\right\|_2^2}.
\tag{1}
$$

随后，我们计算 $p_h^i$ 与 $p_h^k$ 之间的相对旋转四元数 $\Delta q_h^{ik}$（详见附录D），点对 $(p_h^i,p_t^i)$ 的四元数 $\Delta q_h^i$ 计算为 $\Delta q_h^i=\sum_{k\in N_i}w_h^{ik}\Delta q_h^{ik}$。

计算每个控制点的平移和旋转四元数后，我们便可以通过插值得到整个三维高斯基元集合的变形。具体而言，对于每个高斯基元 $\Gamma^i\in P_h$，其变形通过对邻近的前 $K$ 个（$K=2$）控制点 $\{p_h^j\mid j\in N_i\}$ 的变换进行插值得到，其中 $N_i$ 为最近的前 $K$ 个控制点的索引。变形后的中心 $\mu_d^i$ 和旋转四元数 $q_d^i$ 为：

$$
w^{ik}=1-\frac{\left\|\mu^i-p_h^k\right\|_2^2}{\sum_{j\in N_i}\left\|\mu^i-p_h^j\right\|_2^2},
\tag{2}
$$

$$
\mu_d^i=\mu^i+\sum_{k\in N_i}w^{ik}\Delta p_h^k,
\tag{3}
$$

$$
q_d^i=\sum_{k\in N_i}(w^{ik}\Delta q_h^k)\otimes q^i,
\tag{4}
$$

其中，$\mu^i$ 和 $q^i$ 分别为原始中心和旋转四元数，$\otimes$ 为四元数乘法。当只有一个控制点时，不对四元数施加任何变化。我们不会直接将原有高斯基元更新为变形后的高斯基元，因为这会限制变形，并且不适用于“把他的袖子变长”这类任务。受 SDE-Drag（Nie et al., 2024）的启发，我们采用复制粘贴的方式放置变形后的高斯基元，同时保留原有高斯基元。为了给优化提供更大的灵活性，我们将原有高斯基元 $P_h$ 的不透明度调整为较小值，让二维更新决定是保留还是移除这些高斯基元。

**局部编辑掩码。** 由于拖动操作主要针对整个场景的一部分，因此需要局部编辑来保留背景信息。遵循 Gaussian Editor（Chen et al., 2024b），我们为 $P_h$ 中的高斯基元分配掩码 $M$，将它们视为可修改的高斯基元。与 Gaussian Editor 不同，我们的工作**同时构建三维和二维局部编辑掩码**，以处理更复杂的场景和几何编辑。对于三维掩码，在通过变形生成新的高斯基元或进行致密化时，我们从原有高斯基元继承掩码。掩码之外的高斯基元在优化过程中保持不变。对于二维掩码，我们为每个视角渲染掩码，并通过一个阈值将其取整为 $(0,1)$，得到掩码 $\{m^v\}$，其中 $v$ 表示第 $v$ 个视角。需要注意，掩码在变形之后渲染，因此原始区域和目标区域都会被覆盖。我们进一步对掩码进行膨胀，以改变邻近区域的上下文。

### 3.4 用于外观修正的扩散引导

直接对高斯基元进行变形往往会产生明显的伪影，而且无法生成语义正确的内容。受近期三维编辑成果（Haque et al., 2023）的启发，我们通过更新数据集来编辑三维场景。然而，将二维拖动的概念融入三维环境并非易事。以往的二维拖动方法通常需要耗时的正向和反向过程（Shi et al., 2024; Nie et al., 2024）。此外，在训练过程中，不同视角之间不一致的二维编辑会使最终结果偏离预期，并充满伪影。为解决这些问题，我们提出采用无需反演的二维图像编辑，依托变形后三维内容的一致渲染结果，实现**更强的三维一致性、更高的效率和质量**。如图3所示，我们的方法能够生成多视角一致的二维编辑。具体而言，给定变形后的三维高斯基元所渲染的图像，我们引入**图像到图像视图修正**，获得修正后的二维编辑结果。为克服伴随几何变化的数据集编辑所面临的挑战，我们采用**退火式数据集编辑**来更新数据集。

**图3：多视角一致的二维编辑。** 以变形后的渲染结果为输入，微调后的扩散模型能够进行多视角一致的编辑，修复伪影与不正确的部分（鞋子）。

图中标签：变形后的三维高斯基元；一致的二维编辑；编辑后的三维渲染。

**图像到图像视图修正。** 尽管变形后的高斯基元能够提供更好的三维一致性，但它无法受益于基于潜变量的拖动方法（Pan et al., 2023）。这是因为三维一致性通过新渲染的图像来保证。相比之下，基于潜变量的方法高度依赖于对同一张图像的特征图进行操作。受常见图像编辑方法（Meng et al., 2022）的启发，我们先添加噪声，再通过 DreamBooth（Ruiz et al., 2023）模型去噪。通过将图像转化到类似草图的程度并对其去噪，扩散模型能够在一定程度上理解并补全变形部分。

为了减轻扩散模型随机性的影响，我们针对每个场景使用 LoRA（Hu et al., 2022）对 DreamBooth 模型进行微调。我们发现，微调后的扩散模型成为一个多视角一致的编辑器。图3中的实验结果表明，即使不进行反演过程，扩散模型也能成功理解变形后的图像，并生成内容正确的图像。然而，这种修正仍无法通过一次更新完全收敛，因此需要下述更好的数据集编辑策略。

**退火式数据集编辑。** 迭代式数据集编辑已成为三维外观编辑中的常用方法（Haque et al., 2023）。其思想是逐步改变三维外观，并进一步使用渲染结果引导一致的二维编辑。然而，这一策略在涉及几何变化的编辑中效果不佳，因为几何不一致时更难收敛。此外，长时间的迭代更新还会累积严重的模糊（Haque et al., 2023）。为解决这一问题，我们提出仅对数据集进行有限的 $A$ 次更新，并在每次更新时，对图像到图像视图修正的强度（Meng et al., 2022）进行退火。退火函数如下：

$$
S(a)=S_{\mathrm{init}}-\frac{a-1}{A}(S_{\mathrm{init}}-S_{\mathrm{final}}),\quad a=1,2,3,\ldots,A,
\tag{5}
$$

其中，$S_{\mathrm{init}}$ 和 $S_{\mathrm{final}}$ 分别为初始强度和最终强度，$S(a)$ 表示第 $a$ 次更新的强度。需要注意，较低的强度意味着扩散从更靠后的时间步开始，从而进行更精细的细节修正。我们的策略以由粗到细的方式进行编辑。每次更新都会更新所有视角，以防止误差累积。

**损失函数。** 给定三维高斯基元的渲染图像 $I_r^v$，以对应的编辑后图像 $I_e^v$ 作为编辑区域的真值，以原始图像 $I_o^v$ 作为背景真值，并给定视角 $v$ 的掩码 $m^v$，我们用于训练三维高斯基元的损失函数定义为：

$$
\mathcal{L}=\sum_{v=1}^{V}(\lambda_1\mathcal{L}_1(I_r^v,I_o^v)+\lambda_{\mathrm{ssim}}\mathcal{L}_{\mathrm{ssim}}(I_r^v,I_o^v))\odot(1-m^v)+\lambda_{\mathrm{lpips}}\mathcal{L}_{\mathrm{lpips}}(I_r^v,I_e^v)\odot m^v),
\tag{6}
$$

其中，$\mathcal{L}_1$ 和 $\mathcal{L}_{\mathrm{ssim}}$ 用于确保局部编辑。$\mathcal{L}_{\mathrm{lpips}}$ 为 LPIPS（Zhang et al., 2018）损失函数，用于修正编辑区域。$\lambda_1$、$\lambda_{\mathrm{ssim}}$ 和 $\lambda_{\mathrm{lpips}}$ 分别是各项损失的权重系数。

### 3.5 从单步拖动编辑到多步拖动编辑

前文介绍了使用我们方法进行单步拖动编辑的过程。由于长距离拖动操作通常需要多于一个步骤来避免结果损坏，我们提出多步编辑调度器来解决这类问题。具体而言，我们将拖动操作分为 $T$ 个阶段，并设置渐进目标点 $\{p'_t(u)\mid u=1,2,\ldots,T\}$。在每个阶段中，我们朝相应的目标点进行拖动：

$$
p'_t(u)=p_h+\frac{u}{T}(p_t-p_h),
\tag{7}
$$

然而，在训练三维高斯基元时，实际的控制点位置通常会发生变化。我们提出在每个阶段结束时重新定位控制点，使下一阶段的变形更加精确。此外，我们进一步进行利用历史信息的扩散微调，以提升大幅度编辑的能力。

**图4：中间拖动步骤及所跟踪的掩码。** 我们的方法朝目标点进行渐进式编辑。通过跟踪被拖动的高斯基元，实现大幅度编辑。

图中标签：用户编辑；步骤1；步骤2；步骤3；拖动步骤。

**控制点重定位。** 控制点重定位在每个阶段的训练过程结束后进行。为了跟踪控制点，我们使用与每个控制点关联的高斯基元。具体而言，对于控制点 $p_h^i$，我们使用高斯基元 $P_h^i$ 的平均位置变化来更新它。如图4所示，被拖动的部分能够被成功重定位。需要注意，所分配的高斯基元 $P_h^i$ 会在变形期间更新为新变形的高斯基元，并在训练的致密化过程中从父基元继承。局部掩码通过与该掩码取并集来更新。

**利用历史信息的扩散微调。** 对于长距离拖动操作，由于扩散模型是在原始图像上微调的，编辑后的二维图像可能偏离其数据域，导致结果退化回原始图像。我们构建一个图像缓冲区来微调扩散模型。在每个阶段，都会使用图像缓冲区对扩散模型进行微调。最初，缓冲区中仅包含原始图像；在后续阶段中，新编辑的结果会被加入缓冲区。

**图 5：不同场景中的定性结果。** 我们的方法能够处理复杂场景，并生成细节丰富的结果。通过简单的拖动输入，3DGS-Drag 能够识别三维场景上下文，并执行移动物体、补全背景、调整外观、修改物体形状和调整运动等编辑。橙色边界框标出了被修改的区域。

图内文字：用户编辑（User Edits）；拖动结果的渲染图（Rendered Drag Results）。

## 4 实验

### 4.1 实现细节

**用户输入。** 用户输入为一个或多个控制点及其对应的目标点。这些输入点位于三维空间中。用户可以指定控制点的球形作用范围半径，以调整编辑范围。我们利用分配给控制点的高斯基元渲染出掩码，并应用该掩码自动进行局部编辑。我们对掩码进行膨胀，以便修改必要的周边内容。

**拖动编辑。** 预训练的三维高斯基元使用原始 3D Gaussian Splatting 方法（Kerbl et al., 2023）进行训练。编辑时，我们默认选择 50 个视角，以实现高效编辑。具体而言，我们选择能够看到控制点所对应高斯基元较大区域的视角，这由每个视角的局部编辑掩码确定。我们使用 LoRA（Hu et al., 2022）微调 DreamBooth 模型（Ruiz et al., 2023）。最初，在选定视角上进行微调，批量大小为 4，训练 200 次迭代。在每个拖动步骤之后，我们使用相应区间内更新后的图像缓冲区，继续微调扩散模型 50 次迭代。每次都将新编辑的图像加入队列。损失权重 $\lambda_1$、$\lambda_{\mathrm{ssim}}$ 和 $\lambda_{\mathrm{lpips}}$ 分别设为 8、2 和 1。注意，$\lambda_1$ 和 $\lambda_{\mathrm{ssim}}$ 是常规设置的 10 倍，以确保背景保持不变。

**数据集。** 我们的实验包括对八个场景的编辑，使用了 Instruct-NeRF2NeRF（Haque et al., 2023）、PDS（Koo et al., 2024）、Mip-NeRF360（Barron et al., 2022）和 Tank and Temple（Knapitsch et al., 2017）公开的数据集。

### 4.2 定性评估

**不同场景中的编辑结果。** 图 1 和图 5 展示了从不同视角观察到的编辑结果。由于控制点和目标点位于三维空间中，我们将它们绘制到二维图像上以便说明。每次拖动由一个红色箭头表示，箭头起点为控制点，终点为目标点。对于图 1 中的站立人物场景，抬起一只手是一项极具挑战的任务，因为只能观察到手臂的一部分，而手臂下方的区域是未知的。我们的方法展示了生成新姿势并修复手臂下方裤子纹理的能力。我们还能够改变腿部运动和延长袖子。处理更复杂的场景时，例如图 1 中的竹子场景，3DGS-Drag 能够理解植物的纹理，并将其延展得更高或更宽。我们也可以轻松修改部分背景，例如墙壁。当拖动操作用于移动足球时，我们能够将该物体与背景分离，并在其原始位置补全纹理，而不是留下空白区域。总之，无论是正面视角场景还是 360 度场景，我们的拖动操作都能够理解移动物体、延展物体等不同操作，展示出识别三维场景上下文的能力。

**图 6：基线比较。** 与基线方法相比，3DGS-Drag 能够正确修改不同部位，实现高质量、细粒度的编辑，同时在效率方面也具有优势。具体而言，Instruct-NeRF2NeRF（Haque et al., 2023）和 PDS（Koo et al., 2024）无法正确完成编辑。仅使用变形会产生不完整的编辑结果，而 SDE-Drag（Nie et al., 2024）有时无法产生变化。

图内各列文字：用户编辑（User Edit）；变形（Deformation）；Instruct-NeRF2NeRF（1.5 小时）；PDS（10 小时）；SDE-Drag（1 小时）；3DGS-Drag（本文方法，10–20 分钟）。

图内文本提示：

| 编辑行 | Instruct-NeRF2NeRF 的提示 | PDS 的提示 |
| --- | --- | --- |
| 腿部编辑 | “张开他的双腿”（“Spread his legs”） | “……张开双腿”（“... spreading his legs”） |
| 竹子编辑 | “让这盆竹子更宽”（“Make the pot of bamboo wider”） | “……变得更宽”（“... becoming wider”） |
| 发际线编辑 | “降低他的发际线”（“Make his hairline lower”） | “……有较低的发际线”（“... with low hairline”） |

**基线比较。** 由于目前没有可直接比较的、针对真实场景中直观三维拖动操作的工作，我们对具有代表性的基线方法进行了扩展和改造，使其适用于这一任务。结果见图 6。具体比较如下：

- **Instruct-NeRF2NeRF（Haque et al., 2023）：** 对于这一基线，我们手动为拖动操作编写文本描述，然后使用 Instruct-NeRF2NeRF 编辑场景。该模型未能对“人物”（person）场景进行编辑。对于更复杂的“花园”（garden）场景，Instruct-NeRF2NeRF 只是让渲染结果变得模糊。这表明其进行几何修改的能力不足。
- **变形：** 由于我们的输入设定与以往方法不同，我们使用自己的变形方法来代表先前基于变形的方法。值得注意的是，几何结构虽然发生了移动，但由此产生了大量错误内容和伪影。
- **PDS：** PDS（Koo et al., 2024）声称能够改变几何结构，但该方法在这三种编辑场景中均表现不佳。此外，与其他方法相比，PDS 容易生成噪声较多且模糊的编辑结果。
- **SDE-Drag：** 一种替代方案是直接在每个视角上使用二维拖动方法。这里我们选择 SDE-Drag（Nie et al., 2024）进行比较。然而，这种策略无法得到一致的编辑，因此产生了有缺陷的编辑结果或编辑失败的情况。

与这些基线相比，我们的方法取得了显著更好的编辑结果，具有更丰富的细节和正确的内容。特别是，对于“降低他的发际线”（“lower his hairline”）这一文本提示，Instruct-NeRF2NeRF 和 PDS 都误解了文本，反而抬高了发际线，这进一步凸显了直观三维编辑的重要性。

**图 7：局部掩码和拖动步数的消融研究。** 不使用局部掩码时，场景会变得模糊，导致编辑失败。使用过少的步骤难以实现大幅度编辑。增加步骤数会略微改善表现。

图内文字：用户编辑（User Edit）；不使用局部编辑（W/O Local Editing）；1 步拖动（1-Step Drag）；5 步拖动（5-Step Drag）；20 步拖动（20-Step Drag）。

**消融研究。** 与仅使用变形的方法进行比较，验证了扩散引导的有效性（图 6）。这里，我们进一步在图 7 中对局部掩码和多步策略进行消融分析。（1）不使用局部编辑时，整个场景会变得模糊，编辑也会失败。这是由优化问题造成的：不一致的编辑会在三维高斯基元中产生大块漂浮伪影。（2）对于拖动步数，我们比较了 $[1,5,20]$ 这三种不同设置，发现增加或减少步数带来了不同的观察结果。仅使用一步时，变形后的高斯基元无法为扩散模型提供足够的引导，导致编辑结果破损。因此，当我们进行更大幅度的编辑时，单步拖动编辑通常会遇到困难。使用更多步骤（20 步）时，编辑质量会略有提升。这说明，即使进行更多次更新，3DGS-Drag 也具有稳健性。不过，由于增加步骤数会降低执行速度，因此最好选择合适的步数。

### 4.3 定量评估

由于缺少真实参考结果，通常很难对三维编辑结果进行定量评估。这里，我们采用两种指标进行评估：用户偏好和 GPT 评分，结果如图 8 所示。对于用户偏好，我们开展了一项有 19 名参与者的用户研究，并收集了他们对每项编辑的偏好。对于 GPT 评分，由于具备视觉能力的 GPT 已被证明是一种与人类判断相符的评估器（Wu et al., 2024），我们使用 gpt4-o 对每项编辑进行五级评分。具体而言，我们衡量每种方法的两个方面：（1）内容是否得到正确编辑；（2）渲染图像质量。我们的方法在所有这些指标上均取得了最佳结果。

**图 8：定量评估。** 我们对编辑结果同时进行了用户研究和 GPT 评估。与 Instruct-NeRF2NeRF（Haque et al., 2023）和 PDS（Koo et al., 2024）相比，3DGS-Drag 的表现显著更好。

图内标题与图例：用户偏好（User Preference）；GPT 评估（GPT Evaluation）；编辑有效性（Editing Effectiveness）；渲染质量（Rendering Quality）；3DGS-Drag（本文方法，Ours）。图中数值如下：

| 方法 | 用户偏好 | GPT 评估：编辑有效性 | GPT 评估：渲染质量 |
| --- | ---: | ---: | ---: |
| 3DGS-Drag（本文方法） | 77.90% | 3.67 | 3.47 |
| Instruct-NeRF2NeRF | 13.68% | 1.8 | 2.13 |
| PDS | 8.42% | 1.53 | 1.5 |

用户偏好图的纵轴刻度为 0.00%、10.00%、20.00%、30.00%、40.00%、50.00%、60.00%、70.00%、80.00%、90.00%；GPT 评估图的纵轴刻度为 0、0.5、1、1.5、2、2.5、3、3.5、4。

### 4.4 讨论

**局限性。** 与先前基于扩散的三维编辑方法（Chen et al., 2024b; Haque et al., 2023）类似，我们的方法依赖扩散模型提供准确的引导。因此，当目标物体在视野中太小，或者场景规模较大且复杂时，我们的方法可能无法取得理想结果。我们也无法处理幅度过大的拖动操作。在这种情况下，物体可能被移到可见性受限的区域，对大多数视角而言，它已经处于视野之外。

**运行时间。** 使用 50 个视角进行编辑时，我们的方法需要 15 分钟。具体而言，初始扩散模型微调约需 2 分钟，其余编辑过程需要 13 分钟。相比之下，Instruct-NeRF2NeRF（Haque et al., 2023）需要 1 小时。运行时间是在单张 RTX 4090 GPU 上测得的。

## 5 结论

本文介绍了 3DGS-Drag，一种面向三维场景的直观拖动编辑方法。以往工作（Haque et al., 2023; Dong & Wang, 2023; Wang et al., 2023）主要关注外观，而我们解决的是与几何变化相关的内容编辑问题。实验证明，我们的方法能够在不同场景中实现细节丰富的编辑。这一优势主要来自我们的两项关键贡献：以复制粘贴方式进行的高斯基元变形，以及扩散修正。我们展示了该方法能够实现以往难以完成的编辑，为探索三维编辑的新可能性铺平了道路。

## 致谢

本研究获得了以下项目和机构的部分支持：美国国家科学基金会（NSF）项目 2106825、美国国家食品与农业研究所（NIFA）项目 2020-67021-32799、丰田研究院、IBM–伊利诺伊探索加速研究院、亚马逊–伊利诺伊交互式对话体验人工智能中心、Snap Inc.，以及通过伊利诺伊大学医疗工程系统中心和 OSF 基金会提供的 Jump ARCHES 捐赠基金。本研究使用的计算资源包括：通过“先进网络基础设施协调生态系统：服务与支持”（Advanced Cyberinfrastructure Coordination Ecosystem: Services & Support，ACCESS）计划的 CIS220014 和 CIS230012 资源分配项目获得的 NCSA Delta 和 DeltaAI 超级计算机；以及通过“国家人工智能研究资源”（National Artificial Intelligence Research Resource，NAIRR）试点计划获得的 TACC Frontera 超级计算机、亚马逊云服务（Amazon Web Services，AWS）和 OpenAI API。

## 可复现性声明

我们的代码已发布于 [3DGS-Drag 项目仓库](https://github.com/Dongjiahua/3DGS-Drag)。关于实现细节，我们在第 3.3 节介绍了数学细节，在第 4.1 节介绍了训练细节。第 3 节完整介绍了框架架构。正如第 4.1 节所述，我们使用的所有数据集均可公开获取。

## 参考文献

以下保留作者、英文题名、发表来源和年份，并在每条后附中文题名。

1. Rameen Abdal, Peihao Zhu, Niloy J Mitra, and Peter Wonka. Styleflow: Attribute-conditioned exploration of stylegan-generated images using conditional continuous normalizing flows. *ACM Transactions on Graphics*, 2021.  
   中文题名：StyleFlow：利用条件连续归一化流，对 StyleGAN 生成图像进行属性条件探索。

2. Chong Bao, Yinda Zhang, Bangbang Yang, Tianxing Fan, Zesong Yang, Hujun Bao, Guofeng Zhang, and Zhaopeng Cui. SINE: Semantic-driven image-based NeRF editing with prior-guided editing field. In *CVPR*, 2023.  
   中文题名：SINE：利用先验引导编辑场实现语义驱动的基于图像的 NeRF 编辑。

3. Jonathan T. Barron, Ben Mildenhall, Dor Verbin, Pratul P. Srinivasan, and Peter Hedman. Mip-nerf 360: Unbounded anti-aliased neural radiance fields. In *CVPR*, 2022.  
   中文题名：Mip-NeRF 360：无边界抗锯齿神经辐射场。

4. Tim Brooks, Aleksander Holynski, and Alexei A Efros. Instructpix2pix: Learning to follow image editing instructions. In *CVPR*, 2023.  
   中文题名：InstructPix2Pix：学习遵循图像编辑指令。

5. Jun-Kun Chen and Yu-Xiong Wang. Proedit: Simple progression is all you need for high-quality 3d scene editing. *NeurIPS*, 2024.  
   中文题名：ProEdit：仅需简单的渐进过程即可实现高质量三维场景编辑。

6. Jun-Kun Chen, Jipeng Lyu, and Yu-Xiong Wang. Neuraleditor: Editing neural radiance fields via manipulating point clouds. In *CVPR*, 2023.  
   中文题名：NeuralEditor：通过操作点云编辑神经辐射场。

7. Jun-Kun Chen, Samuel Rota Bulò, Norman Müller, Lorenzo Porzi, Peter Kontschieder, and Yu-Xiong Wang. Consistdreamer: 3d-consistent 2d diffusion for high-fidelity scene editing. In *CVPR*, 2024a.  
   中文题名：ConsistDreamer：用于高保真场景编辑的三维一致二维扩散。

8. Yiwen Chen, Zilong Chen, Chi Zhang, Feng Wang, Xiaofeng Yang, Yikai Wang, Zhongang Cai, Lei Yang, Huaping Liu, and Guosheng Lin. Gaussianeditor: Swift and controllable 3d editing with gaussian splatting. In *CVPR*, 2024b.  
   中文题名：GaussianEditor：利用高斯泼溅实现快速、可控的三维编辑。

9. Pei-Ze Chiang, Meng-Shiun Tsai, Hung-Yu Tseng, Wei-Sheng Lai, and Wei-Chen Chiu. Stylizing 3D scene via implicit representation and hypernetwork. In *WACV*, 2022.  
   中文题名：通过隐式表示与超网络实现三维场景风格化。

10. Jiahua Dong and Yu-Xiong Wang. Vica-nerf: View-consistency-aware 3d editing of neural radiance fields. In *NeurIPS*, 2023.  
    中文题名：ViCA-NeRF：感知视角一致性的神经辐射场三维编辑。

11. Shaocong Dong, Lihe Ding, Zhanpeng Huang, Zibin Wang, Tianfan Xue, and Dan Xu. Interactive3d: Create what you want by interactive 3d generation. In *CVPR*, pp. 4999–5008, 2024.  
    中文题名：Interactive3D：通过交互式三维生成创造你想要的内容。

12. Yuki Endo. User-controllable latent transformer for stylegan image layout editing. In *Computer Graphics Forum*, 2022.  
    中文题名：用于 StyleGAN 图像布局编辑的用户可控潜在 Transformer。

13. Jiemin Fang, Junjie Wang, Xiaopeng Zhang, Lingxi Xie, and Qi Tian. Gaussianeditor: Editing 3d gaussians delicately with text instructions. In *CVPR*, 2024.  
    中文题名：GaussianEditor：利用文本指令精细编辑三维高斯基元。

14. Ian Goodfellow, Jean Pouget-Abadie, Mehdi Mirza, Bing Xu, David Warde-Farley, Sherjil Ozair, Aaron Courville, and Yoshua Bengio. Generative adversarial nets. In *NeurIPS*, 2014.  
    中文题名：生成对抗网络。

15. Ayaan Haque, Matthew Tancik, Alexei A Efros, Aleksander Holynski, and Angjoo Kanazawa. Instruct-nerf2nerf: Editing 3d scenes with instructions. In *ICCV*, 2023.  
    中文题名：Instruct-NeRF2NeRF：根据指令编辑三维场景。

16. Erik Härkönen, Aaron Hertzmann, Jaakko Lehtinen, and Sylvain Paris. Ganspace: Discovering interpretable gan controls. In *NeurIPS*, 2020.  
    中文题名：GANSpace：发现可解释的 GAN 控制方式。

17. Edward J Hu, Yelong Shen, Phillip Wallis, Zeyuan Allen-Zhu, Yuanzhi Li, Shean Wang, Lu Wang, and Weizhu Chen. Lora: Low-rank adaptation of large language models. In *ICLR*, 2022.  
    中文题名：LoRA：大语言模型的低秩适配。

18. Hsin-Ping Huang, Hung-Yu Tseng, Saurabh Saini, Maneesh Singh, and Ming-Hsuan Yang. Learning to stylize novel views. In *ICCV*, 2021.  
    中文题名：学习对新视角进行风格化。

19. Yi-Hua Huang, Yue He, Yu-Jie Yuan, Yu-Kun Lai, and Lin Gao. StylizedNeRF: Consistent 3D scene stylization as stylized NeRF via 2D-3D mutual learning. In *CVPR*, 2022.  
    中文题名：StylizedNeRF：通过二维与三维相互学习，以风格化 NeRF 实现一致的三维场景风格化。

20. Yi-Hua Huang, Yang-Tian Sun, Ziyi Yang, Xiaoyang Lyu, Yan-Pei Cao, and Xiaojuan Qi. Sc-gs: Sparse-controlled gaussian splatting for editable dynamic scenes. In *CVPR*, 2024.  
    中文题名：SC-GS：用于可编辑动态场景的稀疏控制高斯泼溅。

21. Clément Jambon, Bernhard Kerbl, Georgios Kopanas, Stavros Diolatzis, Thomas Leimkühler, and George Drettakis. NeRFshop: Interactive editing of neural radiance fields. *Proceedings of the ACM on Computer Graphics and Interactive Techniques*, 2023.  
    中文题名：NeRFshop：神经辐射场的交互式编辑。

22. Tero Karras, Samuli Laine, and Timo Aila. A style-based generator architecture for generative adversarial networks. In *CVPR*, 2019.  
    中文题名：一种面向生成对抗网络的基于风格的生成器架构。

23. Bahjat Kawar, Shiran Zada, Oran Lang, Omer Tov, Huiwen Chang, Tali Dekel, Inbar Mosseri, and Michal Irani. Imagic: Text-based real image editing with diffusion models. In *CVPR*, 2023.  
    中文题名：Imagic：利用扩散模型进行基于文本的真实图像编辑。

24. Bernhard Kerbl, Georgios Kopanas, Thomas Leimkühler, and George Drettakis. 3d gaussian splatting for real-time radiance field rendering. *ACM Transactions on Graphics*, 2023.  
    中文题名：用于实时辐射场渲染的三维高斯泼溅。

25. Arno Knapitsch, Jaesik Park, Qian-Yi Zhou, and Vladlen Koltun. Tanks and temples: Benchmarking large-scale scene reconstruction. *ACM Transactions on Graphics*, 2017.  
    中文题名：Tanks and Temples：大规模场景重建基准评测。

26. Juil Koo, Chanho Park, and Minhyuk Sung. Posterior distillation sampling. In *CVPR*, 2024.  
    中文题名：后验蒸馏采样。

27. Thomas Leimkühler and George Drettakis. Freestylegan: Free-view editable portrait rendering with the camera manifold. In *SIGGRAPH Asia*, 2021.  
    中文题名：FreeStyleGAN：利用相机流形实现自由视角的可编辑人像渲染。

28. Jingyi Lu, Xinghui Li, and Kai Han. Regiondrag: Fast region-based image editing with diffusion models. In *ECCV*, 2024.  
    中文题名：RegionDrag：利用扩散模型实现基于区域的快速图像编辑。

29. Chenlin Meng, Yutong He, Yang Song, Jiaming Song, Jiajun Wu, Jun-Yan Zhu, and Stefano Ermon. SDEdit: Guided image synthesis and editing with stochastic differential equations. In *ICLR*, 2022.  
    中文题名：SDEdit：利用随机微分方程进行引导式图像合成与编辑。

30. Ben Mildenhall, Pratul P Srinivasan, Matthew Tancik, Jonathan T Barron, Ravi Ramamoorthi, and Ren Ng. Nerf: Representing scenes as neural radiance fields for view synthesis. *Communications of the ACM*, 2021.  
    中文题名：NeRF：将场景表示为神经辐射场以进行视图合成。

31. Thu Nguyen-Phuoc, Feng Liu, and Lei Xiao. SNeRF: Stylized neural implicit representations for 3D scenes. In *WACV*, 2022.  
    中文题名：SNeRF：三维场景的风格化神经隐式表示。

32. Shen Nie, Hanzhong Allan Guo, Cheng Lu, Yuhao Zhou, Chenyu Zheng, and Chongxuan Li. The blessing of randomness: Sde beats ode in general diffusion-based image editing. In *ICLR*, 2024.  
    中文题名：随机性的益处：在通用扩散式图像编辑中，SDE 优于 ODE。

33. Xingang Pan, Ayush Tewari, Thomas Leimkühler, Lingjie Liu, Abhimitra Meka, and Christian Theobalt. Drag your gan: Interactive point-based manipulation on the generative image manifold. In *SIGGRAPH*, 2023.  
    中文题名：拖动你的 GAN：在生成图像流形上进行基于点的交互式操作。

34. Ben Poole, Ajay Jain, Jonathan T Barron, and Ben Mildenhall. Dreamfusion: Text-to-3d using 2d diffusion. In *ICLR*, 2023.  
    中文题名：DreamFusion：利用二维扩散实现文本到三维生成。

35. Alec Radford, Jong Wook Kim, Chris Hallacy, Aditya Ramesh, Gabriel Goh, Sandhini Agarwal, Girish Sastry, Amanda Askell, Pamela Mishkin, Jack Clark, Gretchen Krueger, and Ilya Sutskever. Learning transferable visual models from natural language supervision. In *ICML*, 2021.  
    中文题名：从自然语言监督中学习可迁移的视觉模型。

36. Aditya Ramesh, Prafulla Dhariwal, Alex Nichol, Casey Chu, and Mark Chen. Hierarchical text-conditional image generation with CLIP latents. In *arXiv preprint arXiv:2204.06125*, 2022.  
    中文题名：利用 CLIP 潜在表示进行分层文本条件图像生成。

37. Robin Rombach, Andreas Blattmann, Dominik Lorenz, Patrick Esser, and Björn Ommer. High-resolution image synthesis with latent diffusion models. In *CVPR*, 2022.  
    中文题名：利用潜在扩散模型进行高分辨率图像合成。

38. Nataniel Ruiz, Yuanzhen Li, Varun Jampani, Yael Pritch, Michael Rubinstein, and Kfir Aberman. DreamBooth: Fine tuning text-to-image diffusion models for subject-driven generation. In *CVPR*, 2023.  
    中文题名：DreamBooth：微调文本到图像扩散模型以实现主体驱动的生成。

39. Sitian Shen, Jing Xu, Yuheng Yuan, Xingyi Yang, Qiuhong Shen, and Xinchao Wang. Draggaussian: Enabling drag-style manipulation on 3d gaussian representation. In *CVPR*, 2024.  
    中文题名：DragGaussian：在三维高斯表示上实现拖动式操作。

40. Yujun Shi, Chuhui Xue, Jiachun Pan, Wenqing Zhang, Vincent YF Tan, and Song Bai. Dragdiffusion: Harnessing diffusion models for interactive point-based image editing. In *CVPR*, 2024.  
    中文题名：DragDiffusion：利用扩散模型实现基于点的交互式图像编辑。

41. Olga Sorkine. Laplacian mesh processing. *Eurographics (State of the Art Reports)*, 2005.  
    中文题名：拉普拉斯网格处理。

42. Olga Sorkine-Hornung and Marc Alexa. As-rigid-as-possible surface modeling. In *Eurographics Symposium on Geometry Processing*, 2007.  
    中文题名：尽可能刚性的曲面建模。

43. Jiaxiang Tang, Jiawei Ren, Hang Zhou, Ziwei Liu, and Gang Zeng. Dreamgaussian: Generative gaussian splatting for efficient 3d content creation. In *ICLR*, 2024.  
    中文题名：DreamGaussian：用于高效三维内容创作的生成式高斯泼溅。

44. Can Wang, Ruixiang Jiang, Menglei Chai, Mingming He, Dongdong Chen, and Jing Liao. Nerf-art: Text-driven neural radiance fields stylization. *IEEE Transactions on Visualization and Computer Graphics*, 2023.  
    中文题名：NeRF-Art：文本驱动的神经辐射场风格化。

45. Qiling Wu, Jianchao Tan, and Kun Xu. PaletteNeRF: Palette-based color editing for NeRFs. In *CVPR*, 2023.  
    中文题名：PaletteNeRF：基于调色板的 NeRF 颜色编辑。

46. Tong Wu, Guandao Yang, Zhibing Li, Kai Zhang, Ziwei Liu, Leonidas Guibas, Dahua Lin, and Gordon Wetzstein. Gpt-4v (ision) is a human-aligned evaluator for text-to-3d generation. In *CVPR*, 2024.  
    中文题名：GPT-4V(ision) 是与人类判断一致的文本到三维生成评估器。

47. Tianhao Xie, Eugene Belilovsky, Sudhir Mudur, and Tiberiu Popa. Dragd3d: Vertex-based editing for realistic mesh deformations using 2d diffusion priors. In *arXiv preprint arXiv:2310.04561*, 2023.  
    中文题名：DragD3D：利用二维扩散先验进行基于顶点的编辑，以实现逼真的网格变形。

48. Tianyi Xie, Zeshun Zong, Yuxing Qiu, Xuan Li, Yutao Feng, Yin Yang, and Chenfanfu Jiang. Physgaussian: Physics-integrated 3d gaussians for generative dynamics. In *CVPR*, pp. 4389–4398, 2024.  
    中文题名：PhysGaussian：面向生成式动力学的物理融合三维高斯基元。

49. Tianhan Xu and Tatsuya Harada. Deforming radiance fields with cages. In *ECCV*, 2022.  
    中文题名：利用控制笼对辐射场进行变形。

50. Seungwoo Yoo, Kunho Kim, Vladimir G Kim, and Minhyuk Sung. As-plausible-as-possible: Plausibility-aware mesh deformation using 2d diffusion priors. In *CVPR*, pp. 4315–4324, 2024.  
    中文题名：尽可能合理：利用二维扩散先验实现感知合理性的网格变形。

51. Yu-Jie Yuan, Yang-Tian Sun, Yu-Kun Lai, Yuewen Ma, Rongfei Jia, and Lin Gao. Nerf-editing: geometry editing of neural radiance fields. In *CVPR*, 2022.  
    中文题名：NeRF-Editing：神经辐射场的几何编辑。

52. Kai Zhang, Nick Kolkin, Sai Bi, Fujun Luan, Zexiang Xu, Eli Shechtman, and Noah Snavely. ARF: Artistic radiance fields. In *ECCV*, 2022.  
    中文题名：ARF：艺术化辐射场。

53. Richard Zhang, Phillip Isola, Alexei A Efros, Eli Shechtman, and Oliver Wang. The unreasonable effectiveness of deep features as a perceptual metric. In *CVPR*, 2018.  
    中文题名：深度特征作为感知度量的惊人有效性。

54. Jingyu Zhuang, Chen Wang, Liang Lin, Lingjie Liu, and Guanbin Li. Dreameditor: Text-driven 3d scene editing with neural fields. In *SIGGRAPH Asia*, 2023.  
    中文题名：DreamEditor：利用神经场进行文本驱动的三维场景编辑。

## A 演示视频

补充材料中包含一段演示视频，介绍了我们的框架并展示了编辑结果。

## B 补充实验

### B.1 物体大幅移动的定性结果

我们进一步开展了物体移动实验，具体包括常规大小物体和大型物体的大幅移动。如图 9 所示，我们的方法成功处理了长距离移动，例如重新放置花盆。此外，对于卡车和桌子等非常大的物体，我们的方法能够有效地将其沿指定方向移动，同时尽量减少物体移走区域和放置区域中的伪影。这些结果展示了我们方法的泛化能力。

图中标签：用户编辑；拖动结果的渲染图。

**图 9：更大幅度移动和更大物体的补充定性结果。** 我们的方法成功实现了移动花盆等更长距离的移动，以及移动桌子等大型物体的移动。

### B.2 数据集编辑策略的消融实验

为验证我们的数据集编辑策略的重要性，我们将退火式数据集编辑与迭代式数据集编辑（Haque et al., 2023）进行了比较。如图 10 所示，每次只编辑一帧时，由于其他尚未编辑的视角施加了不一致的约束，无法改变几何结构，导致结果退化回原始场景。相比之下，我们的方法能够成功完成编辑。

图中标签：用户编辑；退火式数据集编辑（我们的方法）；Instruct-NeRF2NeRF 数据集编辑。

**图 10：数据集编辑策略的消融实验。** Instruct-NeRF2NeRF（Haque et al., 2023）的迭代式数据集编辑会导致结果退化。相比之下，我们的退火式数据集编辑能够保留几何变化。

### B.3 局部编辑的定量消融实验

为进一步证明局部编辑的有效性，我们对“延长袖子”这一编辑进行了定量评估。具体而言，我们在未编辑的像素处，计算编辑结果的渲染图与原始渲染图之间的相似度。如表 1 所示，我们的局部编辑策略展现出有效保留未编辑区域和背景的强大能力。

| 方法 | SSIM ↑ | PSNR ↑ | LPIPS ↓ |
| --- | ---: | ---: | ---: |
| 局部编辑 | **0.995** | **43.43** | **0.004** |
| 非局部编辑 | 0.901 | 24.44 | 0.158 |

**表 1：局部编辑的定量消融实验。** 我们的局部编辑策略展现出有效保留未编辑区域和背景的强大能力。

### B.4 扩大规模的用户研究

为提高用户研究结论的普适性，我们将参与者人数从 19 人增加到 99 人。如图 11 所示，我们的方法优于已有基线方法这一核心结论保持不变。

图中标签：用户偏好；3DGS-Drag（我们的方法）；Instruct-NeRF2NeRF；PDS。纵轴刻度：0.0%、10.0%、20.0%、30.0%、40.0%、50.0%、60.0%、70.0%、80.0%。

| 方法 | 用户偏好 |
| --- | ---: |
| 3DGS-Drag（我们的方法） | 70.3% |
| Instruct-NeRF2NeRF | 17.0% |
| PDS | 12.7% |

**图 11：扩大至 99 名参与者的用户研究。** 与基线方法相比，我们的方法仍然获得了明显更高的用户偏好。

### B.5 与二维拖动方法的比较

我们对二维拖动编辑进行了定性比较，以验证我们方法的有效性。具体而言，我们重点考察变形引导下的扩散编辑质量，并将结果与已有的二维拖动方法 DragDiffusion（Shi et al., 2024）和 SDE-Drag（Nie et al., 2024）进行了比较。如图 12 所示，DragDiffusion 往往会产生更多伪影和无关纹理，而我们的结果更加干净，也更符合编辑要求。对于 SDE-Drag（Nie et al., 2024），它未能正确移动腿部，而是在该位置生成了一个物体。结果表明，我们的方法通过直接在图像层面进行操作，能够获得更好的一致性。

图中标签：用户编辑；DragDiffusion；SDE-Drag；我们的方法。

**图 12：二维拖动结果的比较。** 与近期的二维拖动方法相比，我们的方法不需要耗时的反演与前向过程，并且能够产生更一致的结果。相比之下，基线方法 DragDiffusion 生成的腿部和地板包含噪声。SDE-Drag 成功保留了背景，但在手中插入了物体，也未能正确移动腿部。

图中标签：用户编辑；拖动结果的渲染图。

**图 13：生成未见侧面的局限性。** 将具有未见侧面的背景物体拖到前景时，从其他视角观察到的结果会出现错误。

## C 对局限性的深入讨论

我们的方法会遇到两种特定的失败情况：生成物体未见的一面，以及将物体拖到边界区域之外。这里，我们给出定性结果，以进一步说明这些局限性。

### C.1 生成未见侧面

如图 13 所示，将木制支架移动到前景时，未曾见过的部分（例如物体背面）会被错误渲染，并出现明显伪影。这是因为没有三维高斯基元来表示未见的背面。因此，变形后的结果包含毫无意义的图案，而扩散过程无法将其纠正。

### C.2 将物体拖到边界之外

如图 14 所示，当物体在边界处只有一部分可见时，我们很难消除其中的伪影。这是因为根据局部推断整体存在歧义，因此扩散模型纠正边界部分的能力较弱。此外，由于能够观察到该物体的视角更为稀疏，优化三维高斯基元也会面临进一步的挑战。

**图 14：将物体拖到边界之外的局限性。** 向未见区域或边界区域拖动时，细化与优化会变得困难。

### C.3 定量评估中的潜在偏差

虽然我们同时采用了人类偏好得分和 GPT 评估给出的自动化指标，但仍可能存在潜在偏差，例如由参与者选择过程，或所使用 GPT 模型的特定版本及训练数据引起的偏差。这一挑战在缺乏真实值的生成建模中很常见。未来的工作可以通过开展规模更大、参与者群体更多样化的人类评估研究，以及制定更全面的评估方案而受益，从而更好地衡量三维编辑的几何准确性和视觉保真度。

## D 补充变形细节

相对旋转 $\Delta q_h^{ik}$ 的计算过程简述如下。给定控制点 $p_h^i$ 和 $p_h^k$，以及与它们分别对应的目标点 $p_t^i$ 和 $p_t^k$，我们首先计算单位向量：

$$
v_h^{ik}=\frac{p_h^k-p_h^i}{\left\|p_h^k-p_h^i\right\|}
\quad\text{和}\quad
v_t^{ik}=\frac{p_t^k-p_t^i}{\left\|p_t^k-p_t^i\right\|}.
\tag{8}
$$

接下来，我们计算这两个单位向量的叉积和点积：

$$
\mathbf r=v_h^{ik}\times v_t^{ik},
\quad s=v_h^{ik}\cdot v_t^{ik}.
\tag{9}
$$

然后，我们将点积和叉积组合起来，构造四元数 $\Delta q_h^{ik}$：

$$
\Delta q_h^{ik}=[s,r_x,r_y,r_z].
\tag{10}
$$

最后，我们对四元数进行标准化和归一化，以确保其长度为 1。

## E 社会影响与未来工作

**未来工作。** 在未来工作中，我们计划扩展目前的渐进式编辑能力，用于生成三维动画。由于 3DGS-Drag 能够逐步移动或修改物体，因此有可能生成长时程轨迹和人体运动。此外，我们将重点提高模型的可扩展性，以适应包含动态物体和阴影效果的更大场景。

**潜在社会影响。** 3DGS-Drag 的潜在社会影响涉及多个方面。作为一种精细化编辑模型，我们的 3DGS-Drag 能够便捷地操控三维场景，并为增强现实（AR）应用提供有力支持。此外，随着三维高斯基元的快速发展和广泛应用，我们的方法能够无缝融入这一生态系统。其界面对用户友好，只需选择控制点和目标点，因此即使是未受过训练的用户也能够使用我们的模型。
