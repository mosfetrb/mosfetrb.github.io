---
layout: page
title:  "曼德尔球"
author: mosfet
tags: 隐式建模 分形
---
## 2D分形
`分形`意味破碎，断裂。它可能是许多初学者就可以轻松创建的简单图形之一。最著名的分形是`曼德布洛集`。  
为了获得持续放大的动画，我首先考虑中心位置并将它和鼠标移动事件绑定，处于画面左侧时偏左移动，上侧向上移动，根据已缩放的比率来修正移动量。然后，渲染的平面根据基于时间线性放大的某个倍数来缩小。像平常一样，这些数据可以通过主机传送到着色器上，我通常使用完全一致的代码操作这些基本输入。  
在片着色器中，每个像素都知道它在平面中代表的顶点，因此直接开始迭代以执行某种变换操作，最终某些位置会保持在分形内，另外的则被排除到集合之外。着色可以通过它们面临临界值`a^2 + b^2 < 4`的行为确定，迭代中期跌出的位置显示灰色。  
```cpp
for (float i = 0.0; i < MAX; i++) {
  // o- 指一开始指定的位置，prev_a指上一次迭代得到的位置
  // c- 指迭代中可能更新的位置
  float prev_a = ca;
  float prev_b = cb;

  // c² + o (注：平方和加法使用复数的计算方式)
  ca = ca * ca - cb * cb + oa;
  cb = 2.0 * prev_a * cb + ob;

  float dis = ca * ca + cb * cb;
  // 越快掉出的，其迭代次数越小，如果使用比率，则越暗或全黑
  // 对于中间灰值表示在中间的边缘，对于集合内值则完全呈现白色
  if (dis > 4.0) {
    break;
  }
  n += 1.0;
}
```
下图使用了另一种技术着色，通常称为轨道捕获，旨在捕获变换中离特定兴趣点的最终累积最短距离，这可以通过不断`min`来获得。效果如细胞状。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-1.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-2.png">
    </div>
      <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-3.png">
    </div>
      <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-4.png">
    </div>
  </div>
  <p>图1：经典图形和轨道捕获</p>
</div>

<iframe src="https://editor.p5js.org/mosfet-archive/full/kkvG7wXdz" width="620" height="700"></iframe>

## 曼德尔球
曼德尔球是一种三维分形，由朱尔斯·鲁伊斯于1997年首次构建，并由丹尼尔·怀特和保罗·尼兰德于2009年使用球面坐标进一步发展。标准的三维曼德布洛特集并不存在，因为二维复数空间没有三维类似物。

其变换公式是：
```js
/*

n: level of this structure. A const. Generally take 3, 8.
v^n = r^n sin(ndeg2)cos(ndeg1), sin(ndeg2)sin(ndeg1), cos(ndeg2)
---
r = sqrt(x^2+y^2+z^2)
deg1 = atan(y/x);
deg2 = acos(z/r);
r^n * sin(n * deg2) * cos(n * deg1),
r^n * sin(n * deg2) * sin(n * deg1),
r^n * cos(n * deg2)
---
if n= 3,
(x,y,z) ->
 (3z^2-x^2-y^2)x(x^2-3y^2)  (3z^2-x^2-y^2)y(3x^2-y^2)  z(z^2-3x^2-3y^2)
 -------------------------, ------------------------, -----------------
        x^2+y^2                     x^2+y^2                    1
*/
```
## 暴力搜索
接下来使用常规光线行进以及将上述曼德尔球作为隐式建模。但这是一个非常暴力的实现，因为如果不知道某些数学，就无法以聪明的方式计算当前位置对分形的距离(尤其是3D分形)，即获取它的符号距离场SDF。幸好，在这种情况下我们仍然可以渲染不太精确的图像，而不是错误的图像，这是因为我们可以以非常小的距离进行移动，以避免错过任何表面，这确实是光线行进可见性方案的一个优势，以便所有东西都可以稍后考虑。这种策略下，距离实际上被真值化，如果认为始终徘徊，则让我们马上结束，如果有任何异样，则以极小步长前进。  
```cpp
float sdMANDELBULB( in vec3 pos) {
  float max_ite = 10.0;
  float level = 8.0;
  float ited_times = 0.0;

  vec3 z = pos;
  vec3 c = z;
  for (float i = 1.0; i <= max_ite; i++) {
    float r = sqrt(dot(z, z));
    float a1 = atan(z.y / z.x);
    float a2 = acos(z.z / r);
    float level_r = pow(r, level);

    float comp_sin2 = sin(level * a2);
    float comp_sin1 = sin(level * a1);
    float comp_cos1 = cos(level * a1);
    float comp_cos2 = cos(level * a2);

    vec3 tmp = vec3(
      level_r * comp_sin2 * comp_cos1,
      level_r * comp_sin2 * comp_sin1,
      level_r * comp_cos2
    );
    z = tmp + c;
    ited_times += 1.0;

    // 显然，计算迭代值的代码永远不应该改变
    if (dot(z, z) > 4.0) {
      break;
    }
  }

  if (ited_times >= max_ite) {
    return 0.0;
  } else {
    return 0.01;
  }
}
```
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-7.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-8.png">
    </div>
  </div>
  <p>图3：渲染结果</p>
</div>

## 通过隐式近似估值距离完成SDF
首先介绍可用材料。其中重要的两篇材料都提到了修复该隐式方程的方法是隐式近似。列表最后贴出了它们引用的文章，您也可以查看
[从隐式方程近似距离场]({% link _posts/modeling/2024-10-16-impfunc_modeling_and_sdf_approximate.md %})
```plain
https://thebookofshaders.com/   尚未更新分形，但就是下一章

https://iquilezles.org/articles/
分形与复杂动力学
Continuous iteration count        指平滑着色，因为迭代比率总是整数
The M1 bulb of the Mandelbrot set 说明集合的各个部分，不需要参考
The M2 bulb in the Mandelbrot set
Area of M1 in the Mandelbrot set
The symmetry of the Mandelbrot set
Introduction to the Mandelbrot set
渲染分形
Computing the SDF of fractals    分形距离(估值)
Mandelbulb fractal               再次提到估值
3D Julia set fractals      另一种分形
3D orbit traps             轨道
Procedural orbit traps     轨道
Bitmaps orbit traps        轨道
Geometric orbit traps      轨道
Budhabrot fractals      用纹理累积轨道逃逸值的一种算法
Popcorn images 不需要
IFS fractals               请见IFS
Lyapunov fractals 不需要
Icon images

https://iquilezles.org/articles/distance/
```
这给出近似距离等于：
```cpp
return 0.25 * log(m) * sqrt(m) / dz;
```
结果如下：  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-10.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-11.png">
    </div>
    <div class="x la item3-lg item12 pd0">
      <img src="/assets/i/1-12.png">
    </div>
  </div>
  <p>图4：使用近似距离的高效渲染</p>
</div>

<iframe src="https://editor.p5js.org/mosfet-archive/full/uf6IlIdml" width="700" height="750"></iframe>

## REFS
```plain
https://en.wikipedia.org/wiki/Fractal
https://en.wikipedia.org/wiki/Mandelbrot_set
https://en.wikipedia.org/wiki/Mandelbulb
```