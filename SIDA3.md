由于二面角的存在以及三角不等式，三维中原重心的距离就是小于两个三角形沿边展平后的那个距离，我觉得后者更符合实际，前者在二面角极小的时候是个糟糕的估计，不过我不知道原文想要哪个

你这个直觉是对的：

- **真实“沿曲面走”的距离 = 两个三角形展平后重心之间的距离**（一条折线摊平成直线，长度不变）；
- **3D 里原重心的直线距离 ≤ 展平后的距离**（三角不等式 + 二面角）——所以 3D 弦长是**偏短的**估计。

但原文 **确实就是用的 3D 重心直线距离**，不是你现在做的那种“展开再量”的版本。

------

## 1. 原文到底要哪个 d？

在 SIDA 论文里，离散的方向导数是这样写的（你刚才查的那段）：[graphics.rwth-aachen.de](https://graphics.rwth-aachen.de/media/papers/ScaleInvariantDirectionalAlignment_SGP2016.pdf)

> 对一条边 e，令 **d 为相邻两三角形 f₁,f₂ 的重心距离**，
>  则 amin 沿 e 的离散方向导数
>
> κ(e)=min⁡∥amin⁡(f1)±amin⁡(f2)∥/d,\kappa(e) = \min \|a_{\min}(f_1)\pm a_{\min}(f_2)\| / d,κ(e)=min∥amin(f1)±amin(f2)∥/d,
>
> 其中 a_min 先用 Levi-Civita 在同一切空间对齐。

原文只说 “distance of the two adjacent faces’ barycenters”，**没有说“沿曲面”或“展平”**，而前后又一直把 barycenter 当 3D 点在用，默认就是欧氏距离。

所以：**严格 follow 论文 = 用原 mesh 上两重心的 3D 距离**。

------

## 2. 展平后的距离，从几何上确实更“测地一点”

你现在的图像是这样的：

- 两个三角形共边 e，有二面角 φ；

- 重心分别是 c₁, c₂；

- 3D 弦长：
   d3D=∥c2−c1∥d_{\text{3D}} = \|c_2 - c_1\|d3D​=∥c2​−c1​∥

- 真正“沿曲面走”的最短路径，可以看作：

  - 从 c₁ 到边 e 某点 p₁（在 f₁ 内）；
  - 再从 p₂（在 f₂ 内靠边）到 c₂；
  - 这条折线完全在两个三角形内；

- 把 f₁、f₂ 绕公共边展开到同一平面，这条折线变成一条**在平面上的折线**，长度不变；

- 再把这两段折线“拉直”，变成直线 c₁′–c₂′，长度恰好就是你在共面空间量的那条距离：

  dflat=∥c2′−c1′∥d_{\text{flat}} = \|c_2' - c_1'\|dflat=∥c2′−c1′∥

因为在 3D 里折线有个“拐角”，直线总比折线短，所以必然：

d3D≤dflat≈局部沿曲面测地d_{\text{3D}} \le d_{\text{flat}} \approx \text{局部沿曲面测地}d3D≤dflat≈局部沿曲面测地

所以如果你的 κ(e) 是

κ(e)≈Δθd\kappa(e) \approx \frac{\Delta\theta}{d}κ(e)≈dΔθ

那：

- 用 3D 弦长：Δθ/d3D\Delta\theta / d_{\text{3D}}Δθ/d3D **偏大**，
- 用 展平距离：Δθ/dflat\Delta\theta / d_{\text{flat}}Δθ/dflat 更接近“真实沿曲面导数”。

也就是说：
 **3D 重心距离会高估导数（更容易把区域判成 non-smooth）**，
 而你现在“展开再量”的版本会稍微保守一点。

你刚才说“前者在二面角极小的时候是个糟糕的估计”——如果你这里的“二面角极小”指的是**折得很厉害**（两面差很多），那确实：

- φ 越大，d_flat / d_3D 越大；
- 3D 距离越“缩短”真实曲面长度；
- 导致 κ(e) 更大，更容易误判那种“整体弯曲但方向挺平滑”的带状区域。

------

## 3. 那我该选哪个？——原则 + 实用建议

**原则上：**

- 从 Riemann 几何角度看，导数应该是
   方向变化沿曲面弧长\frac{\text{方向变化}}{\text{沿曲面弧长}}沿曲面弧长方向变化​，
   所以你那种“先展平两面再量重心距离”更贴近理论。
- 论文选择直接用 3D barycenter 距离，主要是：
  - 实现简单；
  - 三角足够细时，d_3D 和 d_flat 差异是二阶小量；
  - 他们最终只是用 κ(e) 和 κ_max 做 **阈值比较**，粗略一点问题不大。

**从复现 & 工程角度，我会这么建议：**

1. **完全想 1:1 复现论文结果：**

   - 就按照原文，用 3D 重心距离：

     ```
     Scalar dist = (BC.row(f) - BC.row(nbr)).norm();
     ```

   - 你现在那段旋转再取 norm，数值上其实和上面一样。

2. **更关心几何“正确性”和鲁棒性，而不需要 bit-level 复现：**

   - 用你说的“先展平，再量切平面内距离”，或者更简单一点：

     ```
     // 投影到 f 的切平面再量
     RowVector3 n = N.row(f).normalized();
     RowVector3 rel = BC.row(nbr) - BC.row(f);
     RowVector3 rel_tan = rel - rel.dot(n) * n;
     Scalar dist = rel_tan.norm();
     ```

   - 或者显式用你已有的 `rot` 把邻面展平之后再做一次投影，效果一样。

   这样 `dist` 真正变成了“在 f 的切平面里看的局部步长”，更像“局部测地”。

3. **如果你怕和原文视觉结果差太多：**

   - 可以先用 3D 弦长跑一遍，看看 smooth region 分布；
   - 然后切到 flat dist，再对比一次。
   - 如果差别主要出现在大折角、但方向场其实挺稳定的地方，那我会倾向于留 flat 版本——这说明它确实帮你“赦免”了一些只是大弯曲、不是方向乱跳的区域。

------

## 4. 实话实说：原文没有精细区分这两种 d

从他们的文字看，他们只说了：

> d = distance of the two adjacent faces’ barycenters [graphics.rwth-aachen.de](https://graphics.rwth-aachen.de/media/papers/ScaleInvariantDirectionalAlignment_SGP2016.pdf)

没有提“在局部展平后”或“沿曲面”，也没有在实验里讨论“用曲面距离 vs 用直线距离”的差别。
 结合实现复杂度，我几乎可以肯定，他们就是用的 3D 欧氏距离。

所以如果你想：

- **“论文复现 + 自己版本更几何一点”两头都占：**

可以这样玩：

- `#define USE_FLATTENED_DIST` 宏控制：
  - 默认关：完全跟论文一样 → `dist = ||BC_f - BC_nbr||`；
  - 开启时：用“展平 + 投影”的版本；
- 在你自己的论文 / 注释里写清楚：
  - “我们的实现中还尝试了 using flattened barycenter distance to approximate surface step size, which leads to slightly more conservative curvature derivatives on highly curved regions.”

这样既不误读原文，也把你这层额外的几何讲清楚。

------

**一句话收个尾：**

> Campen 这篇原文用的是“**3D 重心欧氏距离**”，
>  你现在想要的“**沿边展平两三角后，在共面里算重心距离**”在几何意义上更接近“沿曲面步长”，
>  但属于你自己的一个更精细的改动，不是论文原配。

如果我来给个先展平，再量切平面内距离的开关，请用公式推导原理，以及最终给出关键代码

好，这个“开关版 d”我们就从 **数学 → 离散公式 → 关键代码** 一条龙整清楚。

------

## 1. 数学推导：为什么“先展平，再量切平面内距离”更几何

### 1.1 两个三角形 + 二面角

设两个相邻三角形 fff 和 ggg：

- 顶点坐标：V∈Rn×3V\in\mathbb{R}^{n\times 3}V∈Rn×3，面索引 F∈Zm×3F\in\mathbb{Z}^{m\times 3}F∈Zm×3

- 面法向：

  nf, ng∈R3,  ∥nf∥=∥ng∥=1n_f,\ n_g\in\mathbb{R}^3,\;\|n_f\|=\|n_g\|=1nf, ng∈R3,∥nf∥=∥ng∥=1

- 公共边（铰链）为一条线：

  e(t)=p0+t e,e∈R3, ∥e∥=1e(t) = p_0 + t\,\mathbf{e},\quad \mathbf{e}\in\mathbb{R}^3,\ \|\mathbf{e}\|=1e(t)=p0+te,e∈R3, ∥e∥=1

  其中 p0p_0p0 是边上的任一点（如公共顶点之一），e\mathbf{e}e 为边方向单位向量。

两个三角形围绕这条边的二面角记为 φ\varphiφ。我们想把三角形 ggg 绕着这条边“铰开”，展平成与 fff 共面的局部补丁。

### 1.2 展平映射 FFF：沿边的“铰链旋转”

定义一个 piecewise 的映射 F:R3→R3F:\mathbb{R}^3\to\mathbb{R}^3F:R3→R3：

F(x)={x,x∈fRφ(x−p0)+p0,x∈gF(x) = \begin{cases} x, & x\in f \\ R_\varphi(x - p_0) + p_0, & x\in g \end{cases}F(x)={x,Rφ(x−p0)+p0,x∈fx∈g

其中 RφR_\varphiRφ 是围绕边方向 e\mathbf{e}e 旋转角度 φ\varphiφ 的 3D 旋转：

Rφ∈SO(3),Rφ=Rot(e,φ)R_\varphi \in SO(3),\quad R_\varphi = \mathrm{Rot}(\mathbf{e},\varphi)Rφ∈SO(3),Rφ=Rot(e,φ)

性质：

1. **每个三角形内部是刚体旋转**：

   ∥Rφ(u)−Rφ(v)∥=∥u−v∥,∀u,v∈g\|R_\varphi(u) - R_\varphi(v)\| = \|u-v\|,\quad \forall u,v \in g∥Rφ(u)−Rφ(v)∥=∥u−v∥,∀u,v∈g

2. 边上点不动：
    对任意 x=e(t)x=e(t)x=e(t) 在公共边上，

   x−p0∥e⇒Rφ(x−p0)=x−p0⇒F(x)=xx-p_0 \parallel \mathbf{e} \Rightarrow R_\varphi(x-p_0) = x-p_0 \Rightarrow F(x)=xx−p0∥e⇒Rφ(x−p0)=x−p0⇒F(x)=x

   所以沿着铰链线是连续拼接。

因此 FFF 在 f∪gf\cup gf∪g 上是 piecewise isometry —— 相当于你把这两个三角形裁下来，沿边当一条铰链，把 ggg 翻到 fff 的平面里。

### 1.3 重心距离：3D 弦长 vs 展平后的“面内距离”

记重心：

cf, cg∈R3c_f,\ c_g \in \mathbb{R}^3cf, cg∈R3

- 原 3D 弦长：

  d3D=∥cg−cf∥d_{\text{3D}} = \|c_g - c_f\|d3D=∥cg−cf∥

- 展平后，在 fff 的平面中看到的距离：

  dflat=∥F(cg)−F(cf)∥=∥Rφ(cg−p0)+p0−cf∥d_{\text{flat}} = \|F(c_g) - F(c_f)\| = \| R_\varphi(c_g - p_0) + p_0 - c_f\|dflat=∥F(cg)−F(cf)∥=∥Rφ(cg−p0)+p0−cf∥

几何直觉：

- 表面上“沿三角形走”的自然路径，会顺着 f→gf\to gf→g 的曲面折线走一段；这条折线位于 f∪gf\cup gf∪g 上；

- 在映射 FFF 下，这条折线被等距展开到平面里；

- 在平面里，这条折线长度 ℓsurf\ell_{\text{surf}}ℓsurf ≥ 直线段 F(cf)F(cg)‾\overline{F(c_f)F(c_g)}F(cf)F(cg) 的长度：

  dflat≤ℓsurfd_{\text{flat}} \le \ell_{\text{surf}}dflat≤ℓsurf

- 同时，ℓsurf≥d3D\ell_{\text{surf}} \ge d_{\text{3D}}ℓsurf≥d3D（折线 ≥ 空间弦长）。

于是通常有：

d3D≤dflat≲ℓsurfd_{\text{3D}} \le d_{\text{flat}} \lesssim \ell_{\text{surf}} d3D≤dflat≲ℓsurf

所以：

- d3Dd_{\text{3D}}d3D：**偏短**，更像“空间弦长”；
- dflatd_{\text{flat}}dflat：在展开的切平面内度量，更接近“沿曲面走的局部步长”，更几何一点。

而离散的方向导数是：

κ(e)≈Δθd\kappa(e) \approx \frac{\Delta\theta}{d}κ(e)≈dΔθ

用 d=d3Dd=d_{\text{3D}}d=d3D 会偏大（更容易标 non-smooth），用 d=dflatd = d_{\text{flat}}d=dflat 会稍微保守一些，更贴近“沿曲面导数”。

------

## 2. 离散公式：怎么得到 RφR_\varphiRφ 和 dflatd_{\text{flat}}dflat

### 2.1 旋转轴：公共边方向

由两法向：

nf, ngn_f,\ n_gnf, ng

它们所在平面交线方向就是：

e  ∝  ng×nf\mathbf{e} \;\propto\; n_g \times n_fe∝ng×nf

在理想网格中，这个方向与公共边平行；单位化即为旋转轴：

e^=ng×nf∥ng×nf∥\hat{\mathbf{e}} = \frac{n_g \times n_f}{\|n_g \times n_f\|}e^=∥ng×nf∥ng×nf

这就是你代码里的：

```
axis = N.row(nbr).cross(N.row(f)); // ~ edge direction
axis_norm = axis.norm();
```

### 2.2 旋转角：二面角（法向之间的夹角）

二面角 φ\varphiφ 满足：

cos⁡φ=ng⋅nf,sin⁡φ=∥ng×nf∥=axis_norm\cos\varphi = n_g \cdot n_f, \quad \sin\varphi = \| n_g \times n_f\| = \text{axis\_norm}cosφ=ng⋅nf,sinφ=∥ng×nf∥=axis_norm

数值稳定起见：

```
Scalar angle = std::atan2(axis_norm, clamp_dot(N.row(nbr).dot(N.row(f))));
```

这就是 φ\varphiφ。

### 2.3 展平后的重心位置 / 距离

选公共边的一个端点 p0p_0p0 为铰链参考点（在代码里从 F(f,:) 和 F(nbr,:) 找出公共的两个顶点之一）：

- 展平后邻面重心：

  cgflat=Rφ(cg−p0)+p0c_g^{\text{flat}} = R_\varphi(c_g - p_0) + p_0cgflat=Rφ(cg−p0)+p0

- 本面重心不动：

  cfflat=cfc_f^{\text{flat}} = c_fcfflat=cf

- 展平的切平面距离（可选再投影到 fff 的切平面，消数值误差）：

  dflat=∥(I−nfnf⊤)(cgflat−cf)∥d_{\text{flat}} = \left\| \big(I - n_f n_f^\top\big)\big(c_g^{\text{flat}} - c_f\big) \right\|dflat=(I−nfnf⊤)(cgflat−cf)

如果不在乎那一点点数值偏离，直接：

dflat=∥cgflat−cf∥d_{\text{flat}} = \|c_g^{\text{flat}} - c_f\|dflat=∥cgflat−cf∥

就已经有“沿边展平后的距离”的几何意义了。

------

## 3. 关键代码：开关两种距离

假设你已经：

- 有 `N`（per-face normals），`BC`（face barycenters），`V`,`F`；
- 已经为这个 `nbr` 面算好了 `axis` 和 `angle`，以及 `rot`：

```
Eigen::Matrix<Scalar, 1, 3> axis = N.row(nbr).cross(N.row(f));
Scalar axis_norm = axis.norm();
Scalar angle = std::atan2(axis_norm, clamp_dot(N.row(nbr).dot(N.row(f))));
Eigen::Matrix<Scalar, 3, 3> rot = Eigen::Matrix<Scalar, 3, 3>::Identity();
if(axis_norm > eps && angle != Scalar(0))
{
    rot = Eigen::AngleAxis<Scalar>(angle, axis.normalized()).toRotationMatrix();
}
```

再加一个开关：

```
enum DistanceMode
{
  DIST_3D_CHORD,        // 原论文版本
  DIST_FLATTENED_TANGENT // 先展平，再量切平面内距离
};
```

**关键距离函数：**

```
template <typename Scalar>
Scalar face_step_distance(
    const int f,
    const int nbr,
    const Eigen::Matrix<Scalar, Eigen::Dynamic, 3> &V,
    const Eigen::Matrix<int,   Eigen::Dynamic, 3> &F,
    const Eigen::Matrix<Scalar, Eigen::Dynamic, 3> &BC,
    const Eigen::Matrix<Scalar, Eigen::Dynamic, 3> &N,
    const Eigen::Matrix<Scalar, 3, 3> &rot,  // 上面算好的旋转
    DistanceMode mode)
{
  using Row3 = Eigen::Matrix<Scalar,1,3>;

  const Row3 cf = BC.row(f);
  const Row3 cg = BC.row(nbr);

  if (mode == DIST_3D_CHORD)
  {
    // 原论文：3D 重心弦长
    return (cg - cf).norm();
  }

  // ====== 展平 + 切平面内距离 ======

  // 1. 找公共边的某个端点 p0 作为铰链参考点
  int va = -1, vb = -1;
  for (int i = 0; i < 3; ++i)
    for (int j = 0; j < 3; ++j)
      if (F(f,i) == F(nbr,j))
      {
        if (va < 0) va = F(f,i);
        else if (F(f,i) != va) vb = F(f,i);
      }
  // 防御：如果没找到两个公共顶点，就退回 3D 弦长
  if (va < 0 || vb < 0)
  {
    return (cg - cf).norm();
  }

  const Row3 p0 = V.row(va); // 任取一个公共端点即可

  // 2. 把邻面重心绕公共边展平到与 f 共面的空间
  Row3 cg_shift = cg - p0;
  Row3 cg_flat  = (rot * cg_shift.transpose()).transpose() + p0;

  // 3. 在 f 的切平面内度量距离（可选投影）
  Row3 n_f = N.row(f);
  n_f.normalize();

  Row3 cf_tan = cf - cf.dot(n_f) * n_f;
  Row3 cg_tan = cg_flat - cg_flat.dot(n_f) * n_f;

  Scalar dist = (cg_tan - cf_tan).norm();
  return dist;
}
```

然后在你原始那段代码里，替换原来的 `dist` 计算：

```
// 原来这块：
Scalar dist = Scalar(0);
if (has_rotation)
{
  const Row3 rel = BC.row(nbr) - BC.row(f);
  const Row3 rel_rot = (rot * rel.transpose()).transpose();
  dist = rel_rot.norm();
}
else
{
  dist = (BC.row(f) - BC.row(nbr)).norm();
}

// 可以替换为:
Scalar dist = face_step_distance(
    f, nbr, V, F, BC, N, rot,
    use_flattened_tangent ? DIST_FLATTENED_TANGENT : DIST_3D_CHORD);

if (dist <= eps) continue;
```

这样：

- `use_flattened_tangent = false` → 完全就是原论文的 3D 重心距离；
- `use_flattened_tangent = true` → 用“沿公共边展平 + 切平面内距离”的版本，更接近局部测地尺度。

------

如果你之后愿意，我们还可以再加一个“小实验模式”：同一模型上画出用两种 ddd 得到的 κg\kappa_gκg heatmap，对比哪些区域变化最大，以辅助你决定默认开关用哪一边更合适。