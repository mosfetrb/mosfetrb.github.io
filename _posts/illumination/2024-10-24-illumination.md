---
layout: page
title:  "渲染器实施"
author: mosfet
category: illumination
tags: 着色 照明
---
有一系列渲染或着色、照明技术，从什么方面考虑选择它们？  
除了倾诉全局光照算法之外，传统着色模型也可以通过精心设计的调整为场景提供可信的照明！  
实践中，还有一些额外的小技巧可以作为某类效果的代替。  

---
## 粗略全局照明
对大型景观进行户外照明时，间接照明的贡献只是适度且可预测的。因此代替方案是优先选择。  
场景可以由3或4个定向灯、一些阴影、一些环境光遮蔽和一个雾层组成。  

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
  <p>图1：多种代替技巧混合渲染</p>
</div>

---
## 路径追踪
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item5-lg item12 pd0">
      <img src="/assets/i/3-2.png">
    </div>
    <div class="x la item5-lg item12 pd0">
      <img src="/assets/i/3-3.png">
    </div>
  </div>
  <p>图2：Cornell盒(混合采样 2024/10/31)</p>
</div>

---
## 发射阴影光线进行路径追踪

实际上最近我正在考虑另一种传统上的路径追踪模型。目前，我分叉了几个典型的路径追踪器(标准交叉方式)，一个是具有混合密度等最高级理论的版本，仅用于探索、渲染特定实验场景；另一个在方案上稍微宽松并努力选择最简单的方式进行渲染，因此适用于通用场景。  
而然，它们从不使用**阴影射线**。从结果上讲，对光的密度采样即使存在被遮挡的情况，它也只是在下一次散射中暂时未直接命中灯具。  

当我们使用阴影光线时，就必须小心这一点，因为它将造成破坏。   
使用阴影光线的渲染在计算BRDF前，每次对场景灯光进行采样并评估照明量，特别是，这检查遮挡并产生一个因子，也就是说，交叉的计算量将翻倍。光亮因子可选择朗伯着色模型。实际上可以完美地提供其参数，表面颜色来自于BRDF，光强度、灯光序号从采样函数中返回。结果可获得强烈的阴影效果，就像我开头提到的，存在的间接照明可能被遮蔽。  
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/3-4.png">
    </div>
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/3-5.png">
    </div>
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/3-6.png">
    </div>
  </div>
  <p>图3：发送阴影获得直接照明</p>
</div>

```cpp
vec3 direct_transport( in vec3 ray_origin, in vec3 ray_path) {
  vec3 albedo = vec3(1.0);
  vec3 d_col = vec3(0.0);
  bool first_miss = false;
  for (i = 1; i <= 15; i++) {
    float k = scene(ray_origin, ray_path, id, normal);

    if (k < INF) {
      // ...
   
      // cast
      if (material.mid != M_LIGHT) {
        vec3 shadow_ray;
        int lid;
        random_sampling_lights(p, shadow_ray, lid);
        d_col += direct_lighting(p, shadow_ray, lid, normal);
      } else {
        d_col += material.albedo;
      }

      // BRDF...
      ray_origin = p;
    } else {
      if (i == 1) first_miss = true;
      break;
    }
  }
  if (first_miss) { // 对结果增加背景
    return sky(ray_path);
  }
  return d_col / float(i);
}
```

通常情况下，击中物体的离光背面并进行投射阴影会导致其被自身遮挡，这当然是正确的。对于必须透光的玻璃，不仔细考虑这个照明因子很容易产生错误的结果。首先，无法断定命中玻璃的光背面并穿过时是否应该被取消遮挡判定，如果是这样，哪怕先移动到正面并额外计算一次投射也是不行的，考虑一系列玻璃珠的排列。  
另外，侧对光源时，其直接贡献被降至0，同样容易使间接照明丢失。从技术上来说，第一个问题可以通过在投射阴影时忽略拷贝场景中的电介质解决，而然，这使得它的软阴影成为了不可能，这将与其他材质产生冲突；第二个问题则很难解决，因为其直接贡献确实为0，我们想象加强电介质的对光源的反应是有意义的，这必须修改其因子使用的方式，至少不能是乘法。避免使用乘法因子并单独只是求和可以同时避免这些问题，并且正面会更亮。  

混合两个值的直觉很难。  
尽管采样天空存在一定灯光角度，但你不应该计算其朗伯因子，这将导致地图边界发暗。  
实际上，只要对其他灯光进行采样，就会自然影响到地平线，从而对背景产生分割，一个简单的方法是按照概率降低实际看到的天空亮度。  
```cpp
if (i == 1) {
  d_col = sky(ray_path);
}
return d_col / float(i) * albedo;
```