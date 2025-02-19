---
layout: page
title:  "更真实的大气"
author: mosfet
category: scene
tags: 场景
---
本文探讨建模通用现实主义环境的方法，通常是程序性的，处理的方式多种多样。可能涉及对象，也可以不涉及。  

总的来说有以下几种独立方式：  
①**程序化(procedural)**：使用特定算法生成目标。在动画分支中也有此核心方法。  
②使用**图像**。例如纹理映射。典型用法是立方体或球形图像。  
③**体积渲染(volumetric rendering)**。具体怎么做取决于渲染技术。  

其中两种方式都可以尽可能地接近物理，但实际上粗略获取相似和正确的结果也很容易，即使计算量大幅减少。  
在本文中笔者使用一个自己的较旧模型的渲染器作为示例，使用SDF寻找表面，并启用路径积分。以提供体积渲染的可能性。  
<hr>

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

## 思考
最近我正在尝试创建尽可能真实的天空，用于场景的通用外环境。  
就像以往独自探索一个新的主题一样，这需要学习一些新的理论，图形学(——模拟)无非是对物理和数学的程序化(甚至哲学)，如果没有充足的概念并倒置整个过程会损失意义吗？目前，我认为这不是必要前提。你站在等待求解的递归上层但心安理得。  

另一个值得考虑的问题是——可见的水平如何？要知道，典型的CPU本身处理一条指令需要多少个硅原子构成的芯片，用计算机模拟真正的现实永远是不可能的。  
典型情况下，插值即可完成任务，实际上这也基本高度符合现实。只是不能随着任何时间、太阳的角度变化。这种技术称为程序化的，为特定事实产生了相当急切简化解决方案，这个方法可以扩展到随时间更新大气的颜色，并产生天体的圆盘使其产生特定轨迹运动、附带有噪声生成的云层等效果。实际上，笔者在大多数情况下都使用了它们。而然，它们本质上不是真正的对象，理论上无法直接参与场景照明，最后必然出现单独着色的尴尬情况。  
而然，即使是体积渲染，它也可能结合了程序化的体积数据，仍然可以使用相同的3D噪声。在这种意义上，3似乎也只是程序化的一种更逼近物理的方式。  

除此之外是对程序化的深层次扩展，即尽可能地追加物理，例如甚至可以为地球和太阳实际建模，这些巨型模型需要开始公转和自转，然后从太阳获取一切照明。  
对于这个问题，笔者姑且不认为完全基于物理是可行的，最好只做简化计算或者完成必要部分，一旦确定达到某种相当高的真实水平就简化计算。  
如前所述，"简化"最极端的例子是线性插值的经验，尽管很简单但却很实用。  

最后我要强调一点思考技巧，图形程序员应该意识到——从观察者的角度思考模拟场景。思考观察者会看见什么并采取行动，这就是最终目的。例如，由于我们处于辐射球系中，即使大气不存在，我们可以令未命中对象的光线返回一个简单颜色，以达成一种大气存在的假象，这就是为什么这种程序化方式很有用且流行的原因之一。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/g/2-1.jpg">
    </div>
    <div class="x la item4-lg item12 pd0">
       <img src="/assets/g/2-2.jpg">
    </div>
  </div>
  <p>图1：天体运行直觉(deepai.org)</p>
</div>

## 方案一
我们先说明最普通的实现方式。即使用插值天空颜色，使用基本噪声生成假云。注意本质上这类似于②，只不过是动态图像。  
使用2D噪声可能会产生纹理拉伸的问题，因此最好直接使用3D版本。  
笔者首先寻找3D版本的噪声和FBM(这种事情可以从常用的提供GLSL函数的网站找到，或者AI）。我为低空的云层做了遮蔽处理，因为经验上来看不应该出现在靠近水平线的位置上。  
在这种形式中，我们主要检查天空背景颜色和云是否正确。天体的显示暂时作为附加选项。  
这种方法尽管实用，但缺少真实性，即使是这个系统中本身所含的要素，即大气和云、圆盘等根本不能利用任何着色模型来进行着色。它们像是"画上去"的。  
```cpp
vec3 sky( in vec3 v) {
    float t = mod(FIX_DAYTIME ? DAYTIME : (u_time + 12.0) * 2.9, 24.0);
    float height = normalize(v).y;

    // === Interpolate between top and horizon colors based on height (Y-axis)
    vec3 dawnSunsetTop = vec3(0.1, 0.1, 0.3); // 深蓝色或紫色
    vec3 dawnSunsetHorizon = vec3(1.0, 0.5, 0.3); // 橙色、红色或粉红色
    vec3 dayTop = vec3(0.4, 0.7, 1.0); // 明亮的蓝色
    vec3 dayHorizon = vec3(0.7, 0.9, 1.0); // 浅蓝色或白色
    vec3 nightTop = vec3(0.0, 0.0, 0.0); // 深蓝色或黑色
    vec3 nightHorizon = vec3(0.0, 0.0, 0.1); // 深蓝色或黑色
    vec3 topColor;
    vec3 horizonColor;

    if (t >= 9.0 && t < 15.0) {
        // Day - Morning ~ Afternoon
        topColor = dayTop;
        horizonColor = dayHorizon;
    } else if (t >= 21.0 && t < 24.0) {
        // Night - Midnight
        topColor = nightTop;
        horizonColor = nightHorizon;
    } else if (t >= 0.0 && t < 6.0) {
        // Night ~ Dawn
        topColor = mix(nightTop, dawnSunsetTop, t / 6.0);
        horizonColor = mix(nightHorizon, dawnSunsetHorizon, t / 6.0);
    } else if (t >= 6.0 && t < 9.0) {
        // Dawn ~ Morning
        topColor = mix(dawnSunsetTop, dayTop, (t - 6.0) / 3.0);
        horizonColor = mix(dawnSunsetHorizon, dayHorizon, (t - 6.0) / 3.0);
    } else if (t >= 15.0 && t < 18.0) {
        // Afternoon ~ Sunset
        topColor = mix(dayTop, dawnSunsetTop, (t - 15.0) / 3.0);
        horizonColor = mix(dayHorizon, dawnSunsetHorizon, (t - 15.0) / 3.0);
    } else if (t >= 18.0 && t < 21.0) {
        // Sunset ~ Night
        topColor = mix(dawnSunsetTop, nightTop, (t - 18.0) / 3.0);
        horizonColor = mix(dawnSunsetHorizon, nightHorizon, (t - 18.0) / 3.0);
    }
    vec3 sky = mix(horizonColor, topColor, height);

    // === disk
    sun_loc = normalize(vec3(sin((t + 12.0) / 3.8), cos((t + 12.0) / 3.8), 0.0));
    float dist = distance(v, sun_loc);
    vec3 sun_disk = vec3(0.0);
    float radius = 0.05;
    float core_mask = 1.0 - step(radius * 0.5, dist);
    sun_disk += vec3(0.9, 0.3, 0.0) * core_mask;
    // 1 / 1+r2
    float ring_mask = 1.0 / (1.0 + pow((dist - 0.3 * radius) * 32.0, 2.0));
    sun_disk += mix(ring_mask * vec3(0.9, 0.3, 0.0), ring_mask * vec3(1.0), dist);

    // === clouds
    vec3 p = v * 5.0 + wind_dir * u_time * 0.1;
    // fbm(p + fbm(p))
    float cloud_density = fbm(p + fbm(p, 0.5, 2.0, 0.5), 0.5, 2.0, 0.5);
    // float cloud_density = fbm(p + fbm(p, 0.5, 3.0, 0.25), 0.5, 3.0, 0.25);
    // 云密度
    // Adjust cloud density based on view direction height
    float horizonFade = smoothstep(0.1, 0.3, v.y);
    cloud_density *= horizonFade;
    cloud_density = smoothstep(0.4, 0.6, cloud_density);

    // 模拟太阳照明云
    // 如果v和sun_loc越接近，则越亮，尽管理论上不应该影响太多，因为天空也散射云
    // 太阳位置处于背面时会完全隐藏云，因此最好加上固定值（模拟环境照明）
    // kd vec3 1.0 I 1.0
    vec3 n = vec3(0.0, 1.0, 0.0);
    float cloud_lambertian = max(0.0, dot(sun_loc, v));
    cloud_lambertian = smoothstep(0.0, 1.0, cloud_lambertian);

    float star = getStars(v);
    return sky * ENV_SACT_FACTOR + sun_disk + vec3(1.0) * cloud_density * cloud_lambertian + vec3(star) * (1.0 - cloud_lambertian) * horizonFade;
}
```

<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item5-lg item12 pd0">
      <img src="/assets/i/6-1.png">
    </div>
    <div class="x la item5-lg item12 pd0">
       <img src="/assets/i/6-3.png">
    </div>
    <div class="x la item3-lg item12 pd0">
       <img src="/assets/i/6-2.png">
    </div>
       <div class="x la item3-lg item12 pd0">
       <img src="/assets/i/6-4.png">
    </div>
  </div>
  <p>图2：程序化背景（按时间顺序）</p>
</div>