我们一起来试着复现Scale-Invariant Directional Alignment of Surface Parametrizations，我将会将讨论到的细节送进codex，首先它好像是通过计算曲率的梯度来找到可供对其的粗区域

###### ChatGPT 说：

对，你抓到的是“用曲率信息找可对齐区域”这一层意思，但**它不是用曲率标量的梯度 ∇κ，而是用主曲率方向场的“测地曲率” κ_g 来筛选区域**。这个细节很关键，直接决定你怎么实现。

我先把“粗区域检测”这一步完整梳理一遍，顺便用你之后要在 libigl 里写代码的视角来讲。

------

## 1. 它到底算的是什么：不是 ∇κ，而是 κ_g

论文先定义了两个主方向场 amin, amax 和对应主曲率 κ_min, κ_max：ScaleInvariantDirectionalAlignm…

- amin(x), amax(x)：单位切向向量场，互相正交，是主曲率方向。
- κ_max, κ_min：对应的主曲率标量场。
- γ_max：沿 amax 的曲率线（integral curves of amax）。

然后它引入**最大测地曲率 κ_g**：

κg(p)=max⁡∥v∥=1∥∇vamin⁡∥=max⁡∥v∥=1∥∇vamax⁡∥.\kappa_g(p) = \max_{\|v\|=1} \|\nabla_v a_{\min}\| = \max_{\|v\|=1} \|\nabla_v a_{\max}\|.κg(p)=∥v∥=1max∥∇vamin∥=∥v∥=1max∥∇vamax∥.

直观上：**κ_g 衡量“主方向场在切平面里弯得有多厉害”**——也就是曲率线网络偏离一堆笔直测地线的程度。对于完美圆柱，κ_g=0；表面是弯弯绕绕的“弯环”，κ_g 就会大。ScaleInvariantDirectionalAlignm…

这跟“曲率大小 κ_max 的梯度 ∇κ_max”完全不是一个东西（后者更像你在 multi-scale crease 那篇里看到的 extremality ∇k·t 的那种思路Efficient_Multi_scale_Curvature…）。

------

## 2. “粗区域”的数学定义：κ_g 与 |κ_max| 的对比

他们把“可对齐的**平滑区域 smooth region**”定义为：

> 一个最大连通区域 SSS，在其中
>
> \kappa_g(x) < |\kappa_{\max}(x)|, \quad \forall x \in S \tag{2} \]:contentReference[oaicite:3]{index=3}  

也就是：

- normal curvature = |κ_max|：表面法向沿曲率线方向弯曲的强度
- tangential/geodesic curvature = κ_g：主方向场在切平面里“扭来扭去”的强度

**只要“方向场扭弯程度 < 表面本身弯曲程度”，就认为这个地方方向是“coherent 的、可以当指导”的。**

这个比值 κ_g / |κ_max| 是无量纲的，所以有**尺度不变性**：整体放大模型不会改变这个判据。ScaleInvariantDirectionalAlignm…

这些满足 (2) 的三角形连成的连通分量，就是论文中的“粗区域候选”——smooth regions。后面再用一个“significance 角度阈值”继续筛选掉意义很小的区域。

------

## 3. 离散化：κ_g 的实际算法（这一步是你要在 libigl 里复现的核心）

离散实现部分写得很清楚，关键几行就是你之前 msearch 到的这一段：ScaleInvariantDirectionalAlignm…

> We approximate the curvature κ_g per triangle. For each triangle, we take the maximum over a directional (covariant) derivative across each of its edges… For an edge e … the discrete directional derivative of a_min across e is
>  κ(e)=min⁡∥amin⁡(f1)±amin⁡(f2)∥/d\kappa(e) = \min \| a_{\min}(f_1) \pm a_{\min}(f_2)\| / dκ(e)=min∥amin​(f1​)±amin​(f2​)∥/d …
>  … using Levi-Civita connection (rotate about n1×n2 by angle arccos(n1·n2)).

翻成“libigl 实现语言”就是：

### 3.1 预处理：每个 face 上的曲率信息

按论文 3.2 节，他们用 Alliez 2003 的 curvature tensor 算 amin, amax, κ_min, κ_max：ScaleInvariantDirectionalAlignm…

- 输入：V, F
- 得到 per-face：
  - n(f)：法向（他们还做了一次修正，让 n ⟂ amin）
  - a_min(f), a_max(f)：主方向向量（单位，3D）
  - κ_min(f), κ_max(f)：主曲率

这一块你可以先直接复用：

- 要么自己在 libigl 里接 Alliez 的 C++ 实现；
- 要么先用 libigl 自带的 curvature estimator 做个替代版，先跑通 pipeline 再替换成更准的。

### 3.2 对每条边 e 计算 “方向导数” κ(e)

对一条内部边 e，对应两侧三角形 f1, f2，它们的重心距为 d：ScaleInvariantDirectionalAlignm…

1. 把 a_min(f1), a_min(f2) 投到各自三角面的切平面上（论文说 amin/amax 在表示上可以先不强制切向，用时正交投影回去）。ScaleInvariantDirectionalAlignm…

2. **平行移动（Levi-Civita connection）**：

   - 法向分别为 n1, n2
   - 在离散里，就是把 a_min(f2) 绕轴 n1×n2 旋转一个角度 arccos(n1·n2)，把它搬到 f1 的切平面里。ScaleInvariantDirectionalAlignm…

3. 方向场是 **线场**，有 ± 号不定：

   κ(e)=min⁡(∥amin⁡(f1)−amin⁡′(f2)∥,  ∥amin⁡(f1)+amin⁡′(f2)∥)d\kappa(e) = \frac{\min\big(\|a_{\min}(f_1) - a_{\min}'(f_2)\|,\; \|a_{\min}(f_1) + a_{\min}'(f_2)\|\big)}{d}κ(e)=dmin(∥amin(f1)−amin′(f2)∥,∥amin(f1)+amin′(f2)∥)

   这里 a_min'(f2) 是平行移动之后的向量。前面的 min 就是论文里写的 “min‖a_min(f1) ± a_min(f2)‖”。ScaleInvariantDirectionalAlignm…

这其实就是**用有限差分近似 covariant derivative**：跨面“跳一步”，看方向场变化了多少，再除以距离。

### 3.3 得到每个三角形的 κ_g(f)

论文说：

> For each triangle, we take the maximum over a directional derivative across each of its edges.ScaleInvariantDirectionalAlignm…

也就是：对三角形 f 的三条边 e₁,e₂,e₃，取

κg(f)=max⁡e incident to fκ(e).\kappa_g(f) = \max_{e \text{ incident to } f} \kappa(e).κg(f)=e incident to fmaxκ(e).

### 3.4 用 κ_g(f) 和 |κ_max(f)| 做 smooth / non-smooth 分类

论文的离散版就直接实现了式 (2)：

> If κ(e) exceeds the threshold |κ_max(f)| for any one of the three adjacent edges, the triangle f is non-smooth; otherwise smooth.

写成你熟悉的条件就是：

- 对每个三角形 f：
  - 如果存在某条边 e of f，使得 κ(e) ≥ |κ_max(f)| → f 标记为 **non-smooth**；
  - 否则 → f 标记为 **smooth**。

所有 smooth faces 做连通分量 → 就是他们所谓的 smooth regions（粗区域候选）。

------

## 4. 再往后一步：用 ∡(F) 过滤掉“没意义的小区域”

上面这些 smooth regions 只是“κ_g 够小的区域”，还不一定是你想用来约束 cross field / 参数化的“粗特征区域”。接下来他们定义一个 **significance angle ∡(F)**，量的是“沿最大曲率线穿过该区域时，法向总共转了多少角度”：

∠(F)=max⁡γmax⁡  max⁡s0<s1∫s0s1κmax⁡(γmax⁡(s)) ds\angle(F) = \max_{\gamma_{\max}}\; \max_{s_0<s_1}  \int_{s_0}^{s_1} \kappa_{\max}(\gamma_{\max}(s))\,ds∠(F)=γmaxmaxs0<s1max∫s0s1κmax(γmax(s))ds

离散实现大意是：

1. 每个 smooth region 的边界顶点为起点，沿 a_max 做 streamline（face-by-face 走），直到再碰到 region 边界为止。ScaleInvariantDirectionalAlignm…
2. 对一条 streamline 穿过的一串面 f₀,…,f_n，记录相邻法向变化角
    βi=±arccos⁡(ni⋅ni+1)\beta_i = \pm \arccos(n_i \cdot n_{i+1})βi​=±arccos(ni​⋅ni+1​)，累积求和，取 max minus min → 就是这条线的 ∡(γ_max)。
3. 区域的 ∡(F) = 所有这些 ∡(γ_max) 里的最大值。ScaleInvariantDirectionalAlignm…
4. 设一个阈值（文中用 70°），小于的区域扔掉。ScaleInvariantDirectionalAlignm…

这一步才是把“粗区域 candidate”变成“真正用于对齐的 feature 区域”。