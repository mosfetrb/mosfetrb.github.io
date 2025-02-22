---
layout: page
title:  "混合密度"
author: mosfet
category: illumination
tags: 着色 照明 理论 渲染器
---
混合密度的尝试[(见)](https://raytracing.github.io/books/RayTracingTheRestOfYourLife.html)。留存一些笔记和记录，而然结论无法保证正确性。  
## 场景示例
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/4-1.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/4-2.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/4-3.png">
    </div>
  </div>
  <p>图1：场景1。对球灯混合采样</p>
</div>
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/4-4.png">
    </div>
  </div>
  <p>图2：场景2。对顶灯混合采样 2024/02/22</p>
</div>

---
## 准备
通常情况下我永远不会强调之前的知识，但这次由于太过复杂作为**例外**。如果你已经看过辐射度理论，那么跳过这段话。  

我假设您知道积分测量基本格`1*dμ`来测量域这一事实，那么通过两个积分比率`∫fxdμ`和`∫1dμ`可以得到新引入fx函数的同域平均值。  
采样出空间中特定值的概率分布称为该随机采样变量相关的概率密度函数PDF，简单来说，这是一个因子。  

我们用x、f(x)表示一个随机变量或任意随机结果函数，例如渲染结果的特定像素。  
望值是一种对概率修正乘以实际值的积分，其定义`E(f(x)) = ∫f(x)p(x)dx`通过求和衰减实际值的概率因子获得总体上的平均值。换句话说，它查询整个域。  
当密度相同时，一堆独立随机样本变量iid——通过`x1+x2+.../n`这种方式可以相当简单地获得数值期望解，而无需解析积分形式（但同样应该遵循密度函数以修正迭代值）。这就是蒙特卡洛积分方法的基本根据。  
```ruby
E(f(x)) = ∫f(x)p(x)dx           # E本身就是两项
        ~= 1/N * Σ(i=1..N)xi.   # 数值均值估计

        # 通常，将估计写成g/p的形式称为蒙特卡洛积分
E(f(x)) = ∫f(x)p(x)dx
        ~= 1/N [Σ(i=1..N)g(xi)/p(xi)]
        # 即原始的f(x)p(x) = g，并显式除以p
```
接下来，重新回顾辐射度单位并查看渲染方程。  
我们求解。在标准方程中，**不可能**存在我们提到的`px`密度，因为它是为特定BRDF的均匀采样设计的。  
```ruby
# 每个样本是vi
E(Lo(vo)) ~= Σ gvi / pvi
```

---
## 漫反射实验
渲染方程是`g/p = ρLcosθ/p`。其中的要点是：BRDF-ρ(i,o)，以及g/p中的p项，即p是采样散射的密度。  

回想一下原来的漫反射是什么。我们使用隐式方法修正分布，但随机数产生自均匀采样。  
要理解本文做什么，还是得要从方程入手。  
```cpp
// 保留我们的基本原始实现
void LambertianOriginal( in Material self, in vec3 rec_n, out vec3 rayd, out vec3 albedo) {
  rayd = rec_n + H_random_unit_vector();
  if (H_nearlly_zero(rayd)) rayd = rec_n;
  albedo *= self.albedo;
}
```

#### 推导朗伯分布密度(5) 
散射半球的概率积分为`1`，我们可以从朗伯的本身行为推测密度函数。  
由于这个概率受到角度余弦值`C * cosθ`线性影响(随着角度具有更大概率或配重)，
从球极位系积分是一个好技巧，该倍率为`C=1/pi`，以缩放到代表其每一处的基本概率。因此`PDF = cosθ / pi`。  
```ruby
∫hemisphere|pdf = 1
```

原作者将这个密度称为`sp(散射密度)`，但笔者没弄白这是什么以及属于哪个部分，似乎可能是"BRDF"的一部分并用于我们接下来用渲染方程代替。  
至少现在我们可以从一个可用的密度函数开始审视方程。  

#### 朗伯渲染方程和重要性采样(6)
这里有一个奇怪的结论，**朗伯行为的BRDF就是朗伯密度PDF**。因此当前渲染方程为`L *= (self.albedo) ρdot(n, out) / p`。注意反照率从未在方程中说明，而是通常结合在BRDF中。  
我不清楚隐式行为(即看上去类似于又一次额外分配)和BRDF有什么关系，均匀生成散射并使用BRDF似乎已经说明了让结果符合朗伯。  
目前的推测是，如果再使用隐式分配或者除以对应的采样密度(前者可以仍然保持p为更简单的均匀采样)，这两种方式的效果是一样的。而这个过程已经称为重要性采样策略，使得结果更快符合BRDF模型。  
```cpp
// 明确渲染方程
void Lambertian( in Material self, in vec3 rec_n, out vec3 rayd, out vec3 albedo) {
  rayd = rec_n + H_random_unit_vector();
  if (H_nearlly_zero(rayd)) rayd = rec_n;

  float lambertian_brdf = dot(rec_n, normalize(rayd)) / C_PI;
  float p = 1.0 / C_TWO_PI;
  albedo *= self.albedo * lambertian_brdf * dot(rec_n, rayd) / p;
}
```
实际上我得到了与以往都更亮的结果。  

相同的BRDF材料不会影响收敛结果，重要性采样(隐式或密度)仅影响收敛速度。  
结果本身只受材料的散射行为(BRDF)影响。改变材质将从根本上改变渲染，算法将收敛到不同的答案。  

存在的差异不仅仅是噪音。高大的盒子正面颜色更加均匀。如果您不确定材质的最佳采样模式是什么，那么继续假设均匀采样是相当合理的，虽然这可能会缓慢收敛，但它不会破坏您的渲染。
也就是说，先确保你的BRDF是绝对正确的，在考虑是否需要快速采样。如果使用了错误的BRDF，这种错误很难发现，因为你不知道初始就应该正确的样子。  

---
## 引导灯密度(9)
如前所述(PBR)，阴影光线可能是解决直接照明问题的一种办法。  
而然，想象一下，实际上可以通过将散射区域完全只指向灯光获得类似的效果，即令表面仅可能被来自灯光方向的光线照亮。  
在代码上，这可能很抽象，你只需要强制指定这次散射指向被采样的光线。  

如果用上文的话来说，这同样是隐式重要性采样，**但此时必须更改BRDF正确**。有这种BRDF吗？这可能吗？唯一的方法就是计算采样灯球的密度函数。  
```
抓取索引光线 -> 计算密度 -> 应用密度因子
```
唯一的问题是我们需要知道`p(v)`，但那是什么？  
```ruby
# 最好将点和角度视为微面和微立体角。dA dv，dA比dv的面积大得多，这是有道理的
# 我们通过采样dA(知道其概率)进而采样dv(的概率)，因此可以获得正确的p
P(dA) = 1 / A   # 1/A
p(dv) = ?        #

# 面积dA、dv存在几何关系
dv = dAcosθ / dot(grab-p, grab-p)  # 注意，将其视为面积，而点则只是点，提供距离
# 由于采样概率相同
  p(dv) = p(da)    # 这里可以视为点………
p(dv)dv = p(da)dA

p(dv) = dot(grab-p,grab-p)/ cosθA
```
```cpp
// 采样灯光
float tmp_area;
void grab_lightray(out vec3 origin, out vec3 ray) {
  // get a lightsource point
  vec3 light_pos = vec3(-1.25 + H_rand(0.0, 2.5), 4.99, -1.5 + H_rand(0.0, 2.0));
  // calc area
  tmp_area = 5.0;
  ray = light_pos - origin;
}
float lights_pdf( in vec3 lightray, in vec3 n) {
  float dist_square = dot(lightray, lightray);
  lightray = normalize(lightray);
  float n_cosine = dot(n, lightray);
  if (n_cosine < 0.0) return -1.0; // 拒绝与灯相背离的遮挡表面
  float p = dist_square / (dot(n, lightray) * tmp_area);
  return p;
}
```
我们已经知道如何将散射线定向至灯源，从而忽略其他方向，并因为在下一次击中灯具结束。  
这种定向到直接照明灯具的材料似乎可以视作一种新BRDF材料，因为其散射很特别。  
先尝试一下覆盖漫射表面，对于其他材料BRDF，暂时先不说明有什么用。  
```cpp
// if (material.mid == M_LAMBERTIAN) {
  // Lambertian(material, normal, ray_path, albedo);

  grab_lightray(p, ray_path);
  float lights_p = lights_pdf(ray_path, normal);
  if (lights_p < 0.0) return vec3(0.0);
  albedo *= material.albedo / lights_p;
// }
```

---
## 混合模式(10)
```cpp
if (rand() < 0.5) {
  grab_lightray(p, ray_path);
  float lights_p = lights_pdf(ray_path, normal);
  if (lights_p < 0.0) return vec3(0.0);
  albedo *= material.albedo / lights_p;
} else {
  Lambertian(material, normal, ray_path, albedo);
}
```
最后我们介绍混合多种重要性采样的机制以合并结果。  
①最简单的方式是让光线随机选择一个PDF采样。例如，对半可能是个不错的选择。  
但这种方式会在迭代时出现大量明暗线，因为单次结果混乱，尽管不影响结果。  

②另一种方式是**单次**采样**已混合的PDF**，从而获得更为接近结果的平缓度。  
回忆一下，朗伯的重要性模式是隐式强制实现的，在其**渲染方程形式**实际所使用的密度实际上仍然为均匀因子。**我们复制出来作为参考。**  
目的是混合这两个PDF。  

概念上理解有点困难，总之，笔者暂时不使用这个。  
```cpp
```

---
#### 对比阴影光线(11)
本文说明的`混合密度(mixture density)`是对更传统的**阴影光线**的替代方法。除了灯光之外，还可以对窗户或门下明亮的裂缝或任何您认为可能明亮或重要的东西进行采样。但大多数专业路径追踪器中看到阴影光线也是很正常的。  
对于粗略结果来说，阴影光线往往比混合密度更便宜，并且在实时中变得越来越普遍。  

#### 尾章(12)
玻璃物体很难很好地渲染，除了灯光以外可能还需要另一些进行重要性采样。  

12.3.  
略暂不实现。  