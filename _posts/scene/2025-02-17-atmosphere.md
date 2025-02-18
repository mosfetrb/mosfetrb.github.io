---
layout: page
title:  "更真实的大气"
author: mosfet
category: scene
tags: 场景
---
尽管本文的主题是对环境的建模，但这种模型的构建与一般对象所具有的曲面、仿射等属性无直接关系，换句话说，不是一般几何实体或者属于对象集，处理的方式也多种多样，因此放在这个分类里。  
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

最近我正在尝试创建尽可能真实的天空，就像独自探索一个新的主题一样，这需要理论。说到图形学无非是对物理和数学的程序化，如果没有充足的物理和数学知识(甚至哲学)并倒置整个过程会损失意义吗？目前，我认为这不是必要前提。  
另一个值得考虑的问题是，究竟要模拟到什么程度才满意？最简单情况下只需要线性插值，实际上这高度符合现实。但不随任何时间、太阳的角度变化。那么，如果越来越精确，是否需要为它们建立实体？这些巨型模型需要开始公转和自转，而观察者位于远处。（在程序上，让太阳绕地球转是没问题的，这样会简化问题 ）对于这个问题，姑且不认为完全基于物理是可行的，最好只做简化计算，这也导致了最基本的线性经验，尽管它很简单但却很实用。  
最后我要强调一点思考技巧，应该意识到——模拟可从观察者的视角考虑问题从而简化计算。可见场景，即表面或光照的结果，只是世界上某个部分中自我辐射的球系所看到的(即地面的上半球体、底部包含大气层和地面，上部分几乎真空)。这种直觉与相对论之类的理论对空间的观察有一定相似。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/g/2-1.jpg">
    </div>
    <div class="x la item4-lg item12 pd0">
       <img src="/assets/g/2-2.jpg">
    </div>
  </div>
  <p>图1：太阳和地球(deepai.org)</p>
</div>

## 程序性纹理
先说明最简单的模拟方式，即使用噪声和线性插值。使用2D噪声可能会产生纹理拉伸的问题，因此最好使用3D版本。  
下例是用AI编码的，其中已为低水平高度的云层做了删除处理。  
我们主要检查天空颜色和云是否正确(如果可以着色，那么也要检查着色方式)，因此先搁置天体的显示。  
```cpp
vec3 sky( in vec3 v, in float time) {
    // return mix(vec3(1.0), vec3(0.5, 0.7, 1.0), smoothstep(0.0, 0.7, 0.5 * (v.y + 1.0)));

    float height = normalize(v).y;
    vec3 dawnSunsetTop = vec3(0.1, 0.1, 0.3); // 深蓝色或紫色
    vec3 dawnSunsetHorizon = vec3(1.0, 0.5, 0.3); // 橙色、红色或粉红色
    vec3 dayTop = vec3(0.4, 0.7, 1.0); // 明亮的蓝色
    vec3 dayHorizon = vec3(0.7, 0.9, 1.0); // 浅蓝色或白色
    vec3 nightTop = vec3(0.0, 0.0, 0.1); // 深蓝色或黑色
    vec3 nightHorizon = vec3(0.0, 0.0, 0.1); // 深蓝色或黑色
    float t = mod(time, 24.0) / 24.0;

    // Determine the interpolated sky color based on the balanced time of day
    vec3 topColor;
    vec3 horizonColor;
    if (t < 0.125) {
        // Dawn
        topColor = mix(nightTop, dawnSunsetTop, t / 0.125);
        horizonColor = mix(nightHorizon, dawnSunsetHorizon, t / 0.125);
    } else if (t < 0.375) {
        // Morning
        topColor = mix(dawnSunsetTop, dayTop, (t - 0.125) / 0.25);
        horizonColor = mix(dawnSunsetHorizon, dayHorizon, (t - 0.125) / 0.25);
    } else if (t < 0.5) {
        // Afternoon
        topColor = dayTop;
        horizonColor = dayHorizon;
    } else if (t < 0.75) {
        // Sunset
        topColor = mix(dayTop, dawnSunsetTop, (t - 0.5) / 0.25);
        horizonColor = mix(dayHorizon, dawnSunsetHorizon, (t - 0.5) / 0.25);
    } else {
        // Night
        topColor = mix(dawnSunsetTop, nightTop, (t - 0.75) / 0.25);
        horizonColor = mix(dawnSunsetHorizon, nightHorizon, (t - 0.75) / 0.25);
    }
    // Interpolate between top and horizon colors based on height (Y-axis)
    vec3 skyColor = mix(horizonColor, topColor, height);

    // 计算云
    vec3 windDir = vec3(1.0);
    vec3 p = v * 10.0 + windDir * u_time;
    float cloudDensity = fbm(p);

    // Adjust cloud density based on view direction height
    float horizonFade = smoothstep(0.1, 0.3, v.y);
    cloudDensity *= horizonFade;
    cloudDensity = smoothstep(0.4, 0.6, cloudDensity);

    // Compute light scattering
    vec3 sunDir = vec3(0.0, 1.0, 0.0);
    float lightIntensity = max(dot(v, sunDir), 0.0);
    vec3 cloudColor = mix(vec3(0.8, 0.9, 1.0), vec3(1.0, 1.0, 1.0), lightIntensity);
    cloudColor *= cloudDensity;

    return mix(skyColor, mix(cloudColor * horizonColor, cloudColor * topColor, t), cloudDensity);
    return cloudColor;
    return (skyColor + cloudColor) * ENV_SCAT;
}
```

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
  <p>图2：使用旧渲染器 结果（左：白天 右：黑夜）</p>
</div>