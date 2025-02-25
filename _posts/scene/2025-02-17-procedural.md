---
layout: page
title:  "更真实的环境"
author: mosfet
category: scene
tags: 场景
---
本文不包含特定场景，而是讨论创建尽可能有意思、或真实的场景的大量实用技术。  
在图形学中，常见的一个词是**程序化(procedural)**，旨在用特定算法生成目标。  
程序化技术用于各个方面，不仅是特定的场景效果，而且也是应用于动画的一种核心方法。  

尽管各种效果本质上存在尽可能地接近物理的方案，但实际上粗略获取相似和正确的结果也很容易，在某些场景中的一种考量是通过更好的算法让计算量大幅减少。  
例如，本文首先说明最重要的天空盒应该如何制作，如果使用图片作为背景，那么就根本不需要任何复杂的计算。  
因此，第一种方式，①使用**图像**在这里一笔带过，典型用法是纹理映射立方体或球形图像。  

接下来的方法，本质上取决于目的和渲染方式是什么。  
我们使用一个非常普通的渲染器作为示例，使用SDF寻找表面，并启用路径积分。以提供体积渲染的可能性。  

## 大气 —— 从物理角度思考
<div class="x la bdl2 pdl2 sk bg-gunmetal05 tx-antiqueRuby1">
  <p>
    站在地球表面一点，看向头顶和水平线天空的颜色分别是蓝色和白色吗? 
  </p>
  <p>
    假设同一位置能看见比水平面更低一点方向的天空（不被脚下的球面大地遮挡），会是什么颜色?
  </p>
  <pre>
    天空穹顶通常是蓝色，太阳光穿过大气层时，短波长的蓝光更容易被散射
水平线方向的天空通常接近白色或浅蓝色。光线在接近地平线时穿过更厚的大气层，更多的蓝光被散射掉，剩余的光线混合后呈现白色或浅蓝色

当太阳位于斜向时，天空的颜色会进一步变化，水平线可能带有黄色、橙色或红色的色调。
光线需要穿过更长距离的大气层。更多的蓝光被散射掉，剩余的光线中波长较长的红光和黄光占比增加。
在接近日出或日落时，这种现象尤为明显。蓝光被大量散射，红光和黄光占主导，天空会呈现暖色调（如橙色或红色）。

如果大气中有较多颗粒物（如灰尘、雾霾等），天空的颜色可能会更偏向灰白色或浅黄色。


时间      头顶天空颜色  水平线天空颜色
日出/日落 深蓝色或紫色  橙色、红色或粉红色
上午/下午 明亮的蓝色    浅蓝色或白色
正午     明亮的蓝色    浅蓝色或白色
夜晚     深蓝色或黑色  深蓝色或黑色
  </pre>
</div>
创建最真实的天空的最佳方式可能是参考一些物理理论。本质上图形学(——模拟)是对物理和数学程序化的最佳表达(甚至哲学)。  
但是，读者或者笔者可能不是精通任何其中之一的人，如果没有充足的概念并倒置整个过程会损失意义吗？目前，我认为这不是必要前提。递归求解是一种良好的心态。  
另一个问题可能是精度，即你有多少精力去更好完成这一点。要知道，CPU处理一条指令是由N倍的硅原子完成的，从数量上来看，用计算机模拟真正的现实永远是不可能的。  
典型情况下，插值即可完成任务，实际上这也基本高度符合现实。只是不能随着任何时间、太阳的角度变化。这种技术当然称为程序化的，为特定事实产生了相当急切简化解决方案，这个方法可以扩展到随时间更新大气的颜色，并产生天体的圆盘使其产生特定轨迹运动、附带有噪声生成的云层等效果。实际上，笔者在大多数情况下都使用了它们。而然，它们本质上不是真正的对象，理论上无法直接参与场景照明，最后必然出现单独着色的尴尬情况。  
为了使云可以参与照明，一种方式是参考并近似正确结果，本文对正确结果的定义来自于**体积渲染(volumetric rendering)**。而然，即使是体积渲染，它也可能结合另一些常见技术。体积数据仍然可以使用相同的3D噪声。  

展开想象，甚至可以为地球和太阳实际建模，这些巨型模型需要开始公转和自转，然后从太阳获取一切照明。  
对于这个问题，笔者姑且不认为完全基于物理是可行的，如前所述，"简化"最极端的例子是线性插值的经验，尽管很简单但却很实用。  

最后我要强调一点思考技巧，图形程序员应该意识到——从观察者的角度思考场景。思考观察者会看见什么，然后再采取行动，因为这就是最终目的。例如，由于我们处于辐射球系中，即使大气不存在，我们可以令未命中对象的光线返回一个简单颜色，以达成一种大气存在的假象，这就是为什么这种程序化方式很有用且流行的原因之一。  
## 基本天空盒
基本的程序性背景总是比图像更好。最普通的情况下，使用插值天空颜色，使用基本噪声生成假云。  
使用2D噪声可能会遇到一些纹理映射问题，因此最好直接使用3D版本。  
首先寻找3D版本的噪声和FBM。提示一个技巧——AI，不过，理解1D、2D版本是前提，另外用AI分析错误也很有效，特别是一些奇怪的命中交叉问题。  

低空的云层通常最好做一些遮蔽处理，它们不应该出现在靠近水平线的位置上。  
云层非常缺少适应性，这个系统中本身所含的要素，即大气和云、圆盘等根本不能利用任何着色模型来进行着色。本质上它们是"画上去"的。  
```cpp
vec3 sky( in vec3 v) {
  float t = mod(FIX_DAYTIME ? DAYTIME : (u_time + 12.0) * 2.9, 24.0);
  float height = normalize(v).y;

  // 1 插值背景(Y-axis)
  vec3 dawnSunsetTop = vec3(0.1, 0.1, 0.3); // 深蓝色或紫色
  vec3 dawnSunsetHorizon = vec3(1.0, 0.5, 0.3); // 橙色、红色或粉红色
  vec3 dayTop = vec3(0.4, 0.7, 1.0); // 明亮的蓝色
  vec3 dayHorizon = vec3(0.7, 0.9, 1.0); // 浅蓝色或白色
  vec3 nightTop = vec3(0.0, 0.0, 0.0); // 深蓝色或黑色
  vec3 nightHorizon = vec3(0.0, 0.0, 0.01); // 深蓝色或黑色
  vec3 topColor;
  vec3 horizonColor;
  if (t >= 9.0 && t < 15.0) { // Day - Morning ~ Afternoon
    topColor = dayTop;
    horizonColor = dayHorizon;
  } else if (t >= 21.0 && t < 24.0) { // Night - Midnight
    topColor = nightTop;
    horizonColor = nightHorizon;
  } else if (t >= 0.0 && t < 6.0) { // Night ~ Dawn
    topColor = mix(nightTop, dawnSunsetTop, t / 6.0);
    horizonColor = mix(nightHorizon, dawnSunsetHorizon, t / 6.0);
  } else if (t >= 6.0 && t < 9.0) { // Dawn ~ Morning
    topColor = mix(dawnSunsetTop, dayTop, (t - 6.0) / 3.0);
    horizonColor = mix(dawnSunsetHorizon, dayHorizon, (t - 6.0) / 3.0);
  } else if (t >= 15.0 && t < 18.0) { // Afternoon ~ Sunset
    topColor = mix(dayTop, dawnSunsetTop, (t - 15.0) / 3.0);
    horizonColor = mix(dayHorizon, dawnSunsetHorizon, (t - 15.0) / 3.0);
  } else if (t >= 18.0 && t < 21.0) { // Sunset ~ Night
    topColor = mix(dawnSunsetTop, nightTop, (t - 18.0) / 3.0);
    horizonColor = mix(dawnSunsetHorizon, nightHorizon, (t - 18.0) / 3.0);
  }
  vec3 sky = mix(horizonColor, topColor, height);

  // 2 圆盘
  sun_loc = normalize(vec3(sin((t + 12.0) / 3.8), cos((t + 12.0) / 3.8), 0.0));
  float dist = distance(v, sun_loc);
  vec3 sun_disk = vec3(0.0);
  float radius = 0.05;
  float core_mask = 1.0 - step(radius * 0.5, dist);
  sun_disk += vec3(0.9, 0.3, 0.0) * core_mask;
  // 1 / 1+r2
  float ring_mask = 1.0 / (1.0 + pow((dist - 0.3 * radius) * 32.0, 2.0));
  sun_disk += mix(ring_mask * vec3(0.9, 0.3, 0.0), ring_mask * vec3(1.0), dist);
  // 圆盘距离因子
  float sun_lambertian = max(0.0, dot(sun_loc, v));
  sun_lambertian = 0.5 - 0.5 * cos(C_PI * sun_lambertian);
  float star = getStars(v);

  return sky * ENV_LIGHTSOURCE + sun_disk + vec3(star) * (1.0 - sun_lambertian);
}
```

## 使用体积云
我们特地将处理云滞后。在处理近似方式前，必须先看看体积云是什么样的。事实上，这些对象在任何时间点都具有相当不错的结果。  
将FBM噪声作为**参与介质**渲染可以获得具有正确光照的云。在场景中放置一个体积盒子就可以解决问题。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/6-1.png">
    </div>
    <div class="x la item4-lg item12 pd0">
       <img src="/assets/i/6-2.png">
    </div>
    <div class="x la item4-lg item12 pd0">
       <img src="/assets/i/6-3.png">
    </div>
  </div>
  <p>图1：使用体积云</p>
</div>

## 近似更简单的云
如前所述，它们通常很难正确着色。注意使用多个密度，如击中`y=1.3`和`y=1.0`即可，这几乎总是比单个密度更好，云层更有层次感，显著减少高密度的聚集或者扁平感。  
当您直接加入使用这些密度的白色值很容易使得天空过亮。而晚上或者傍晚则很难、甚至无法近似出正确的颜色。请观察之前的结果，傍晚只有边缘会变成橘色，夜晚的云层仍然相对保持白色。  

我只在白天使用这些。  
```cpp
// 3 云
// fbm(p + fbm(p))
// float cloud_density = fbm(p + fbm(p, 0.5, 3.0, 0.25), 0.5, 3.0, 0.25);
float t1 = 1.0 / v.y;
float t2 = 1.2 / v.y;
vec3 p1 = t1 * v * 5.0 + wind_dir * u_z * 0.01 + u_time * wind_dir;
vec3 p2 = t2 * v * 5.0 + wind_dir * u_z * 0.01 + u_time * wind_dir;
float dense1 = fbm(p1 + fbm(p1, 0.5, 2.0, 0.5), 0.5, 2.0, 0.5);
float dense2 = fbm(p2 + fbm(p2, 0.5, 2.0, 0.5), 0.5, 2.0, 0.5);

float horizon_factor = smoothstep(0.1, 0.3, v.y);
float view_factor = (v.y + 1.0) / 2.0;

dense1 = smoothstep(0.4, 0.6, 0.5 * (dense1 + dense2) * horizon_factor * view_factor);
vec3 bg = sky * ENV_LIGHTSOURCE + sun_disk + vec3(star) * (1.0 - sun_lambertian);
return mix(bg, vec3(1.0), dense1);
```

<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/6-4.png">
    </div>
  </div>
  <p>图2：近似结果</p>
</div>