---
layout: page
title:  "照明"
author: mosfet
tags: 杂项 着色 照明
---
如何巧妙组合各种着色技术可能是一项复杂的决策。  
除了经典的全局光照算法外，传统着色模型可以通过精心设计的调整为场景提供可信的照明。  

**户外照明**  
(来自于Inigo Quilez)对大型景观进行户外照明时，间接照明的贡献只是适度且可预测的。  
基本上由3或4个定向灯、一个阴影、一些环境光遮蔽和一个雾层组成。  

请确保材料、纹理的漫射颜色在0.2左右，如果需要让物体在屏幕上更亮，让灯光更强烈。  
以太阳光作为主光。它呈黄色且非常强烈，大约1.0或1.5。这些阴影稍微柔和一些，可以将半影着色并过度饱和为某种红色或橙色。  

AO应该留给补光灯。太阳是如此小的光源，阴影已经处理了这个问题。不要用环境遮挡来调节太阳光。  
天空光作为次要光源。亮度0.2就足够了。可以是垂直到地面的方向。可以考虑AO和阴影。  
最后一个光源用于模拟间接照明，相当有限，相当于反方向反射回来的太阳光，强度为1.25。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item6-lg item12 pd0">
      <img src="/assets/i/3-1.png">
    </div>
  </div>
  <p></p>
</div>

**路径追踪**  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item6-lg item12 pd0">
      <img src="/assets/i/3-2.png">
    </div>
    <div class="x la item6-lg item12 pd0">
      <img src="/assets/i/3-3.png">
    </div>
  </div>
  <p>图2：一个周末的光线追踪复现结果</p>
</div>