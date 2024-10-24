---
layout: page
title:  "从隐式方程近似距离场"
author: mosfet
tags: 隐式建模 符号距离函数 隐式方程
---
像球体方程`x^2 + y^2 + z^2 = C`这样的隐式方程描述一种图形。它广泛用于隐式建模中。  
**术语差异：**  
满足隐式方程等于某个常数`f: [v]-> C`的域集`v`一般使用层集(level set)或同值线(isoline)等术语，为了避免歧义，本文使用别名**值环**的概念。  

隐式方程在各个值环上都具体地定义了图形，其偏移的绘图很可能是类似前者的。图形不一定是关闭的，例如抛物面(面不是关闭的)。单个值环是2D曲线边缘或3D曲面，而然要注意。我们一般必须基于间隔才能有效绘制它们，连续值环会导致新的面积或体积的图形。扩展时，如`C->C+-1`，结果不一定是均匀的，圆可以变成均匀的环，球体可以变成多层均匀扩展的球壳。边界周围的细带是否均匀取决于该方程本身的特点，可能是它的梯度引起的！  
复杂性的简化是将隐式方程视为黑盒。其一它提供定义和构建图形的行为。第二是通过方程测试当前位置是否在图形上(或者环的区域上)。  
## 隐式方程的问题
一个容易产生的误解是将当前值环(评估值)视为到零环的距离。这通常是不可能的，请不要这样做。这超出了隐式方程黑盒本身的功能。  
事实上这仅在"密度"均匀的形状上才有可能适用，例如圆环，的确任何地方的值环都等于到零环的距离。大多数情况下这些环是扭曲的，可能极端地聚拢或分散。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/2-1.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/2-2.png">
    </div>
  </div>
  <p>图1：一个简单的隐式方程，由2.0 * p.x * sin(p.x) * cos(p.y) - 1.0给出</p>
  <p>图2：原隐式方程x^4 - y^4 - 0.01尝试直接以值环评估距离，这给出了错误的结果(可与它的SDF进行比较)</p>
</div>
可以想象当我们到达三维中，体积也不太均匀，尽管这些"地壳"层来自相同范围的值环。你几乎总是不能将值环视为到零环的距离。这种均匀度的差别会对渲染的可见性方案产生正确性影响。正是这一原因，隐式方程并不能直接用于光线行进等算法(但光线追踪可以)，因为光线行进严格要求对所有对象使用距离。  

有一种近似方法可以根据值环以正确方式评估距离。  
如果确实能够估算距离，就不用大张旗鼓地来为每种曲面单独设置和派生SDF(符号距离函数)了，因为只需要一些额外转接工作。  
用不精确的语言来说，此结果才是零环的正确图像，而不是通过扭曲后的面积和体积。  

## 从隐式方程创建近似SDF
令`d = f / length(▽f)`。如果解析梯度很困难(派生`[df/dx, df/dy]`)，请使用标准数值方法进行评估。
```cpp
/*===============SDF*/
float sdCircle(vec2 p, float r) {
  return length(p) - r;
}
float f( in vec2 p) {
  return pow(p.x, 4.0) - pow(p.y, 4.0) - 0.01;
}
vec2 f_gradient( in vec2 p) {
  const vec2 diff = vec2(0.001, 0.0);
  return vec2(
    f(p + diff.xy) - f(p - diff.xy),
    f(p + diff.yx) - f(p - diff.yx)
  );
}
float sdModel( in vec2 p) {
  return f(p) / length(f_gradient(p));
}
float sdf( in vec2 p) {
  return sdModel(p);
  return f(p);
}
```

<iframe src="https://editor.p5js.org/mosfet-archive/full/tUyIMA5xw" width="710" height="750"></iframe>