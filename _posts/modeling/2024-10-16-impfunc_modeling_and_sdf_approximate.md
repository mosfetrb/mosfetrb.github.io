---
layout: page
title:  "从隐式方程近似距离场"
author: mosfet
tags: 隐式建模 符号距离函数 隐式方程
---
## 隐式方程的问题
隐式方程是描述图形的方程，无论是曲面(仅具有一层边界的东西)，还是构成体积或面积的混合图形。在这个角度上不妨说它就是一种模型。  
像球体方程`x^2 + y^2 + z^2 = C^2`这样的基本隐式方程提供一个非常"薄"的球体的外层曲面。  

对隐式方程最好的理解就是不去理解它。不要怀疑，请将它视为一个黑盒。唯一提供的两个功能是它本身构建图形的行为、第二，通过方程测试当前位置是否在图形上。这是一个真值，仅回答是和不是。  
正如前文所述，即使在已经正确估值的图形上，它也有可能给出区域而不是纯粹的边界。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/2-1.png">
    </div>
  </div>
  <p>图1：满足隐式方程的图形，但这不是边界图形</p>
</div>
如果我们开始认真思考这有什么问题，考虑一下这里存在的某种未知陷阱，这种差别会对渲染的可见性方案产生正确性影响。首先，这类图像不是那么均匀，那么对于三维而言，这意味着导致体积也不是均匀的，这些"地壳"层具有相同的测试值。我只能粗略解释这些问题，试想一下处于地壳旁边，一半具有厚度，一半则非常细，如果正好我们的位置满足较小的测试值，并且使用该差值来移动，你会找到完美的表面交点还是这中间的某个地方？答案可能取决于一开始我们处于的相对位置是否就是边界。好吧，总之种种原因测试结果不是表面距离的最佳表示。因此，隐式方程不能直接用于光线行进等算法(但光线追踪可以)，因为它严格要求对所有对象使用距离。有一种近似或估算方法能获得这种距离，总之不要尝试使用测试结果来估计距离，而是只是用它的黑盒功能。  

如果确实能够估算距离，就不用大费心思来为每种曲面单独设置SDF(符号距离函数)了，因为隐式方程可以像平常一样使用。  
注意，这个距离是指给出图形集的外边界的距离，即一种被期望成"薄"曲面的东西。  

## 从隐式方程创建近似SDF
这篇文章给出了近似方式。即`d = f / length(▽f)`。如果解析梯度很困难(寻找`[df/dx, df/dy]`)，请使用标准数值方法进行评估。
```txt
https://iquilezles.org/articles/distance/
```
```cpp
/*===============SDF*/
float sdCircle(vec2 p, float r) {
  return length(p) - r;
}
float f( in vec2 p) {
  float r = sqrt(p.x * p.x + p.y * p.y);
  float a = atan(p.y, p.x);
  return r - 1.0 + sin(3.0 * a + 2.0 * r * r) / 2.0;
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
}
```

<iframe src="https://editor.p5js.org/mosfet-archive/full/tUyIMA5xw" width="750" height="750"></iframe>