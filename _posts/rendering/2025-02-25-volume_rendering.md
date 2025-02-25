---
layout: page
title:  "参与介质"
author: mosfet
category: rendering
tags: 渲染 参与介质
---
#### 薄雾
`体积(volume)`是世界中的一片密度空间，术语上这种类薄雾质(smoke/fog/mist)通常称为体积介质或`参与介质(participating media)`。其照明称为体积渲染问题。另一种有关特性称为`子表面散射(subsurface scattering)`，即将一块稠密薄雾放入另一物体内部。  

大多数情况下，我们只处理**表面**的可见性和着色，实际上在两个介质之间不会计算任何东西，因此体积从本质上来说不同。一种技巧是将这种分散性等效为更容易接受的无数随机表面，在穿越体积时，有时视为可撞击表面，有时被认为是真空的，直到撞击另一处。  

理论上(不说明)，散射的可能性是穿越距离和密度共同决定的。我们先限定恒定密度下的这一简单模型，达到散射条件的情况近似于光线穿过一个随机距离，例如，可以选择`rd`作为单位。接下来我们必须仔细计算该方向上体积内的剩余距离(指定边界框)，特别是仔细考虑命中体积但未进入体积，或者位于体积两种不同交叉情况。在体积内时，一旦我们比较这两个长度，随机距离未能穿越体积时，就发生散射。这时可以使用密度因子和修正长度。最好选择更合适的交叉函数，只能获取最近交点在这种模型下将很难使用。  
```cpp
vec3 volume_sky( in vec3 dir) {
  vec3 albedo = vec3(1.0);

  vec3 ro = vec3(0.0);
  vec3 rd = dir;
  const float const_dense = 2.0;

  for (int i = 1; i <= 10; i++) {
    vec3 p = ro;
    float view_thickness_factor = noise(dir.xz * 5.0);
    // return vec3(view_thickness_factor);
    vec2 k = iBox2(ro - vec3(0.0, 15.0, 0.0), rd, vec3(100.0, 5.0 + view_thickness_factor, 100.0));
    // 从外面命中体积，要么直接进入体积采样，要么不散射并尝试继续前进
    if (k.x > 0.0 && k.y > 0.0) {
      p = ro + rd * (k.x + C_EPS);
      ro = p;
    } else if (k.x < 0.0 && k.y > 0.0) {
      // return vec3(1.0, 0.0, 0.0);
      // 处于内部，前方只命中一个交点
      float vol_dist = 0.0;
      vol_dist = k.y;
      // 使用密度因子，密度越大，那么减少此距离以增加可能性
      vec3 phase = p * 0.3 + wind_dir * u_time;
      float dense_factor = max(0.01, fbm(phase + fbm(phase, 0.5, 2.0, 0.5), 0.5, 2.0, 0.5) - 0.5);
      float view_factor = 15.0 / (length(ro) + 1.0);
      dense_factor *= view_factor;
      float hit_dist = rand() * (1.0 / dense_factor);
      if (hit_dist < vol_dist) {
        p = ro + rd * hit_dist;
        rd = H_random_unit_vector();
        ro = p;
        albedo *= (1.0 - dense_factor) * vol_dist / hit_dist;
        // albedo *= (1.0 - dense_factor) * hit_dist / vol_dist;
        // 当前散射失败并穿过了体积，命中背景
      } else {
        albedo *= sky(rd);
        break;
      }
    } else {
      albedo *= sky(rd);
      break;
    }
  }
  return albedo;
}
```

<div class="x gr txac">
  <div class="x la flex mg0">
    <div class="x la item4-lg item12 pd0">
      <img src="/assets/i/7-1.png">
    </div>
    <div class="x la item4-lg item12 pd0">
       <img src="/assets/i/7-2.png">
    </div>
  </div>
  <p>图1：体积云</p>
</div>