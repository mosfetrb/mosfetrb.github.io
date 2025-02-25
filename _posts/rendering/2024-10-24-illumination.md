---
layout: page
title:  "光照决策"
author: mosfet
category: rendering
tags: 渲染 着色/照明
---
要编写一个渲染器，可以选择多种技术。从什么方面考虑选择它们？  
除了倾诉全局光照算法之外，传统着色模型也可以通过精心设计的调整为场景提供可信的照明！  
实践中，还有一些额外的小技巧可以作为某类效果的估计。  

## 粗略近似全局照明
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
  <p>图1：代替技巧</p>
</div>

## 路径追踪
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item5-lg item12 pd0">
      <img src="/assets/i/3-2.png">
    </div>
  </div>
  <p>图2：混合采样 2025/02/24)</p>
</div>

## 发射阴影光线进行路径追踪
最近我正在考虑另一种传统方式的路径追踪。累积方式不同。截至目前，我分叉了几个典型的路径追踪器(标准交叉方式)，一个是具有混合密度等理论的优先版本，仅用于探索、渲染特定实验场景；另一个在方案上稍微宽松并努力选择最简单的方式进行渲染，并对天空等大型光源进行PDF采样，因此适用于通用场景。  
而然，它们从不使用**阴影射线**。从结果上讲，对光的密度采样即使存在被遮挡的情况，它也只是在下一次散射中暂时未直接命中灯具。  

现在我们来看阴影光线如何驱动全局光照。首先获取一条光线，然后该函数可以计算其朗伯照明贡献值。  
```cpp
void get_one_lightray( in vec3 p, out vec3 rayd, out int lid) {
  // 采样光线方向，采样ID
  // 天空不需要根据dot衰减直接光，只需要应用朗伯分布更多地采样垂直颜色
    lid = LID_01;
    rayd = normalize(light_pos - p); // 已在这里标准化shadowray
}
vec3 calc_dotdirect( in vec3 ro, in vec3 rd, in int tarlid, in vec3 n) {
  // 计算直接照明该表面的颜色
  // 当命中目标不是指定灯具时被遮挡
  float k = scene(ro, rd, id, light_normal);
  bool shadowed = (k < INF && tarlid != id) || (tarlid == LID_00 && k < INF);
  if (shadowed) return vec3(0.0);

  // 计算灯的材料获取灯光颜色
  if (tarlid == LID_00) {
    return sky(rd) * 1.0;
  } else if (tarlid == LID_01) {
    return light_material.albedo * lambertian(n, rd) * 1.0;
  }
  return vec3(0.0);
}
```
现在为了尝试只查看某种直接光照的结果，进行路径追踪，但取每一处的平均值。  
```cpp
vec3 direct_transport( in vec3 ray_origin, in vec3 ray_path) {
  vec3 d_col = vec3(0.0);
  for (i = 1; i <= 15; i++) {
    if (k >= INF) {
      if (i == 1) return sky(ray_path) && break;; // bool -> first miss -> bg
    }

    vec3 shadow_ray;
    int lid;
    get_one_lightray(p, shadow_ray, lid);
    d_col += calc_dotdirect(p, shadow_ray, lid, normal);

    // BRDF
    else if (material.mid == M_LIGHT) {
      return material.albedo;
      // Light(material, albedo);
      break;
    }
  }
  return d_col / float(i);
}
```
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
  <p>图3：发送阴影获得直接照明；1次、15次(多个光源、一个光源)</p>
</div>

混合两个值的直觉很难。`total_col`总计照明。当未命中时，中断返回总计值，如果第一次未命中，返回环境颜色。  
接下来，以朗伯着色计算常规照明，实际上求和多个光源，路径表面 * 采样光源视为一个单独光源。  
另外，灯光材料像之前一样停止就好了。  
```cpp
vec3 shadowray_transport( in vec3 ray_origin, in vec3 ray_path) {
  vec3 total_col = vec3(0.0);
  for (i = 1; i <= 5; i++) {
    float k = scene(ray_origin, ray_path, id, normal);
    if (k >= INF) {
      if (i == 1) {
        return sky(ray_path) * ENV_LIGHTSOURCE;
      } else break; // 往后中断返回total_col
    }

    // 命中发送阴影，计算光照
    // 可能出现命中灯具并采样灯具的情况
    vec3 d_col = vec3(0.0);
    vec3 shadow_ray;
    int lid;
    get_one_lightray(p, shadow_ray, lid);
    d_col = calc_dotdirect(p, shadow_ray, lid, normal);

    // BRDF
    total_col += albedo * d_col;
  }
  return total_col;
}
```
<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/3-7.png">
    </div>
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/3-8.png">
    </div>
  </div>
  <p>图4：漫射</p>
</div>

使用阴影光线这种不同的方式时，小心引入的潜在破坏。  
每个照明量检查遮挡，当然意味着交叉的计算量将翻倍。朗伯着色模型的参数可以完美地提供，表面颜色来自于BRDF，光强度、灯光序号从采样函数中返回。结果可获得强烈的阴影效果，就像我开头提到的，某些地方的间接照明可能被未发觉地遮蔽。  

通常情况下，击中物体的离光背面并进行投射阴影会导致其被自身遮挡，这当然是正确的。对于必须透光的玻璃，不仔细考虑很容易产生错误的结果。首先，无法断定命中玻璃的光背面并穿过时是否应该被取消遮挡判定，如果是这样，哪怕先移动到正面并额外计算一次投射也是不行的，考虑一系列玻璃珠的排列。  
另外，侧对光源时，其直接贡献被降至0，这也将导致电介质的显示异常。第一个问题的临时解决是可以通过在投射阴影时忽略拷贝场景中的电介质，而然，这使得它的软阴影成为了不可能，这将与其他材质产生冲突；第二个问题则很难解决，因为其直接贡献确实为0，我们想加强正常散射电介质的对光源的反应是有意义的，现在反而完全遮蔽它。  