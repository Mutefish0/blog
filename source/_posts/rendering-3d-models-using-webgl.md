---
title: 从零实现Web端的3D模型渲染
date: 2020-04-26 19:34:24
tags:
---

<style>
  .align-center {
    text-align: center;
  }
</style>

上次写了一篇关于WebGL入门的博客，只简单实现了一些比较基础的东西，比较无趣，这次从实际应用出发，从零实现Web端的3D模型渲染。下面是得物App上的3D商品预览功能，觉得比较有趣，也是因为这个原因催生了写这篇博客的想法。

<img width="240" src="/images/webgl-3d-model/dewu.gif">
<p class="align-center">
<i>得物App的3D商品渲染</i>
</p>

# 为什么一定要用WebGL？
WebGL是一个Javascript API，用来渲染交互式的2D和3D图形。放眼其他技术，不管是DOM操作API、又或者是Canvas画布API，都是通过调用浏览器API，让浏览器来帮你进行渲染的。唯独WebGL可以让你手动操纵显卡，来决定如何进行渲染。为了能够实现逼真的3D渲染效果，这个能力至关重要。

# 为什么说WebGL能够操纵显卡？
为了解释这个问题，我们先看看GPU是如何工作的，下面这张图展示了一帧画面，从数据准备，到经过显卡程序处理，最后渲染到屏幕的过程。

![](/images/webgl-3d-model/gpu_work.png)

可以看到，一帧画面，其实是一批数据，经过显卡程序处理，最后渲染到屏幕上。可以看到，如果能够操纵数据和显卡程序，则我们操纵了显卡。WebGL正好提供了这些API。

# 着色器
实际上，所谓的“显卡渲染程序”有个正式名称：**着色器**，它是由GLSL ES（OpenGL ES Shading Language）语言编写的程序。利用着色器可以实现非常强大的功能，各种游戏特效、影视特效，其底层其实就是着色器代码。可以说着色器是显卡的灵魂。

每个渲染程序都至少需要包含两个最常用的部分：**顶点着色器（Vertex Shader）**和**片段着色器（Fragment Shader）**，还有可选的**几何着色器(Geometry Shader)**，这篇博客只涉及到了顶点着色器和片段着色器。

下图展示了顶点着色器和片段着色器的分工：
![](/images/webgl-3d-model/shaders.png)

顶点着色器负责生成**屏幕坐标**点，片段着色器则负责生成颜色值，最终形成了屏幕上的一个像素点。

所谓的“屏幕坐标”，具体指的是**标准化设备坐标（Normalize Device Coordinate）**。如下图所示，以屏幕中心为原点，从左到由为X轴，从下到上为Y轴，从屏幕中心指向你自己为Z轴。特点如下：
1. 坐标范围为[-1, 1]，超出范围则不会被渲染
2. 绘制像素点只需要用到X和Y轴，Z轴可用于**深度缓冲（Z－buffering）**

![](/images/webgl-3d-model/coords.png)


# 着色器入门
下面介绍着色器程序的基本用法，实现非常简单的图形绘制。这些示例虽然简单无趣，但是掌握了这些，等于你掌握了80%的着色器知识。实际上你能想象到的各种逼真、炫酷的效果，都是通过这些基础用法组而成的。

## Hello World
学习任何一门语言的时候，入门程序就是跑通“Hello World”程序。在计算机图形领域，入门程序是绘制一个三角形。我们输入3个顶点数据，直接输出为屏幕坐标，颜色固定为绿色。

### 准备顶点数据
```js
const data = [
  0, 0.5, 0,
  -0.5, 0, 0,
  0.5, 0, 0
]
```
如图所示
![](/images/webgl-3d-model/coords.png)

### 编写顶点着色器
```glsl
// 声明版本
#version 300 es
in vec3 a_Pos; // 取一个输入顶点数据

void main() { 
  // 直接输出
  gl_Position = vec4(a_Pos, 1.0); 
}
```

### 编写片段着色器
```glsl
#version 300 es
out mediump vec4 fragColor;

void main() {
  // 输出纯绿色，完全不透明
  fragColor = vec4(0, 1, 0, 1);
}
```

### 结果
![](/images/webgl-3d-model/preview1.png)

## 画一个正方形
显然，画一个正方形等于画两个三角形。

### 顶点数据
我们修改顶点数据即可：
![](/images/webgl-3d-model/rectangle.png)

### 结果
![](/images/webgl-3d-model/preview2.png)

### 画一个立方体
同样的道理，分别会在六个面，每个面绘制两个三角形。
![](/images/webgl-3d-model/cube.png)

### 结果
![](/images/webgl-3d-model/preview2.png)

并没有看出是一个立方体，结果还是还是一个正方形。回顾一下前面提到的，绘制屏幕坐标的时候，只用了X, Y两个轴，Z轴并没有用到的。那么如何才能绘制出3D图形呢？下一节给出答案。

# 如何绘制立体图形？
说出来可能有些失望，WebGL或者显卡程序并没有提供能够绘制3D图形的API，并且它也不是被设计用来做这个事情的。

对于如何表现立体物体，我们可以从素描中得到启示：
- 透视画法，将立体坐标映射到平面上
- 画阴影，将不同的表面区分开来 

计算机图像学中分别由这两种算法实现：
- 空间变换算法
- 光照算法

我们可以得出结论，表现立体物体，实际是靠算法实现的。稍后的内容将会一步一步编码实现这两种算法。

理论上你可以开发出一个Javascript库，实现一套完整的算法，在Canvas上控制每一个像素点的绘制，最终也能实现和WebGL一样的效果。但是，第一，不太现实，这样相当于自己实现了一个OpenGL库。第二，性能很差，这些算法里面涉及到大量的矩阵运算，而CPU不擅长做这些运算，GPU则天生就擅长做这些运算。

# 空间变换
我们通过矩阵乘法运算来实现空间变换。还是以素描打比方，我们可以实现轻松”透视画法“，不仅如此，我们还可以实现：
- 随意改变观察的视角和位置（额外发现的宝藏）
- 随意改变物体的位置、方位、大小（额外发现的宝藏）

## 透视效应
在生活中可以观察到，照片或者素描作品中，越近的东西越大，越远的东西则越小。越正对着的面越大，顶点的夹角变化越小，越侧对着的面越小，顶点的夹角变化越大。

![](/images/webgl-3d-model/projection_effect.png)

这个效应称之为**透视（Perspective）效应**。在数学上有一种运算叫**透视投影**，专门模拟这种效果，它实际上是利用了摄像机或者人眼成像的原理。

## 投影矩阵
如下图所示，我们有一个摄像机，可以通过平截头体（金字塔削掉顶部）定义它的参数。Near是**近平面**，Far是**远平面**，fov是**视场（field of view）**，aspectRatio是宽高比。平截头体有如下特点：

- 只有平截头体内的点可以看到，超出则会被裁剪
- 平截头体空间内的任意一点都会被投影到近平面
- 近平面上的点将会被成像到电脑屏幕或者摄像机屏幕上
![](/images/webgl-3d-model/projection.png)

通过这四个参数可以得到一个透视投影矩阵，将空间坐标乘以这个矩阵后可以得到屏幕坐标
```js
const f = 1.0 / Math.tan(((fov / 180) * Math.PI) / 2);
const rangeInv = 1 / (near - far);
const projectionMatrinx = [
  f / aspectRatio, 0, 0, 0,
  0, f, 0, 0,
  0, 0, (near + far) * rangeInv, -1,
  0, 0, near * far * rangeInv * 2, 0
]
```

## 观察矩阵
如图所示：通过定义观察矩阵，我们可以变换摄像机的位置和方位
![](/images/webgl-3d-model/view.png)

## 模型矩阵
如图所示：通过定义模型矩阵，我们可以变换物体的位置、方位、大小
![](/images/webgl-3d-model/model.png)

## 修改顶点着色器
```glsl
#version 300 es

in vec3 a_Pos;

uniform mat4 projection; // 接收投影矩阵
uniform mat4 model; // 接收模型矩阵
uniform mat4 view; // 接收观察矩阵

void main() {
  // 通过矩阵乘法，最终将空间物体坐标变换到屏幕坐标
  gl_Position = projection * view * model * vec4(a_Pos, 1.0); 
}
```

## 效果如下
可以看出有一定立体效果了
![](/images/webgl-3d-model/preview4.png)

# 光照
光照也是通过数学运算实现的。还是以素描打比方，我们可以轻松实现”画阴影“，不仅如此，我们还可以实现：
- 模拟环境光照（月亮等）
- 模拟反光效应
- 通过在任意足够小的局部，定义3种光照的叠加参数，从而逼真地模拟任意材质的物体

## 环境光照（Ambient）
理论上，如果没有光，我们看到的物体的颜色是完全黑的。生活中，一般总会存在一些光，比如太阳、月亮产生的光。我们模拟这样的效果，称为环境光照。
![](/images/webgl-3d-model/ambient.png)

### 修改片段着色器
```glsl
#version 300 es

out mediump vec4 fragColor;

void main() {
  mediump vec3 lightColor = vec3(1, 1, 1);
  mediump vec3 ambient = lightColor * 0.1; // 取白光强度的0.1为环境光照
  mediump vec3 object = vec3(0, 1, 0);
  fragColor = vec4(ambient * object, 1);
}
```

### 效果

![](/images/webgl-3d-model/preview5.png)


## 漫反射（Diffuse）
现实中，光对一个物体的不同表面产生的影响是不同的
* 正对着光看起来更亮
* 背对着光看起来更暗
使用漫反射可以很好地模拟这种效果

![](/images/webgl-3d-model/diffuse.png)

算法里面引入了定点的法向量。我们需要修改顶点数据，除了坐标外，我们需要加上对应的法向量。

![](/images/webgl-3d-model/norm_data.png)

### 修改顶点着色器
```glsl
#version 300 es

in vec3 a_Pos;
in vec3 a_Norm; // 接受法向量

uniform mat4 projection;
uniform mat4 model;
uniform mat4 view;

// 将经过模型变换后的坐标传给片段着色器
out highp vec3 f_Pos;
// 将经过模型变换后的法向量传给片段着色器
out highp vec3 f_Norm;

void main() {
  // 计算物体经过模型变换后的坐标
  f_Pos = vec3(model * vec4(a_Pos, 1.0));
  // 计算经过模型变换后的法向量
  f_Norm = mat3(transpose(inverse(model))) * a_Norm;
  gl_Position = projection * view * model * vec4(a_Pos, 1.0);
}
```

### 修改片段着色器
```glsl
#version 300 es

in highp vec3 f_Pos;
in highp vec3 f_Norm;

out mediump vec4 fragColor;

void main() {
  mediump vec3 lightColor = vec3(1, 1, 1);
  highp vec3 lightPos = vec3(0, 0.5, 100); // 灯的位置

  mediump vec3 ambient = lightColor * 0.1; // 取白光强度的0.1为环境光照
  mediump vec3 object = vec3(0, 1, 0);

  mediump vec3 n = normalize(f_Norm);
  mediump vec3 l = normalize(lightPos - f_Pos);
  mediump float cosTheta = max(dot(n, l), 0.0);
  mediump vec3 diffuse = cosTheta * lightColor * object;

  fragColor = vec4(ambient * object + diffuse, 1);
}
```

### 结果

如下图，立体效果已经非常明显了

![](/images/webgl-3d-model/preview6.png)


## 镜面反射（Specular）

现实中，有的物体表面会反射光，有的则不会，这和物体表面的光滑程度和反光程度有关。**镜面反射**用来模拟这种效果。

![](/images/webgl-3d-model/reflect.png)

### 修改片段着色器代码
```glsl
#version 300 es

in highp vec3 f_Pos;
in highp vec3 f_Norm;

out mediump vec4 fragColor;

void main() {
  mediump vec3 lightColor = vec3(1, 1, 1);
  highp vec3 lightPos = vec3(0, 0.5, 100);
  highp vec3 viewPos = vec3(0, -0.5, 2);

  mediump vec3 ambient = lightColor * 0.1; // 取白光强度的0.1为环境光照
  mediump vec3 object = vec3(0.2, 0.8, 0.2);

  mediump vec3 n = normalize(f_Norm);
  mediump vec3 l = normalize(lightPos - f_Pos);

  mediump float cosTheta = max(dot(n, l), 0.0);
  mediump vec3 diffuse = cosTheta * lightColor * object;

  // specular
  mediump float Ns = 500.0;
  highp vec3 v = normalize(viewPos - f_Pos);
  highp vec3 r = reflect(-l, f_Norm);
  mediump float spec = pow(max(dot(v, r), 0.0), Ns);
  mediump vec3 specular = spec * lightColor * object;

  fragColor = vec4(ambient * object + diffuse + specular, 1);
}
```

### 效果
基本上通过这三种光照参数的叠加，我们可以模拟任意的一种材质
![](/images/webgl-3d-model/preview7.gif)


# 3D模型渲染
至此我们已经解决了立体感的问题。但是到目前为止似乎还是感觉有些单调。主要有两点：
- 整个物体都表现出一种材质，现实中一个物体往往有多种材质组成
- 除了立方体还是立方体，现实中往往是各种复杂的形状，比如人体

这两个问题分别由这两种技术解决：**材质贴图（UV Mapping）**和**网格（Mesh）**

## UV Mapping
与每个顶点都使用同样一个物体颜色相反，转而使用着色器提供了采样器函数，从一个指定的图片中，为每个顶点采样一个颜色。

同样地，与每个顶点使用同样一个反射强度相反，从一个指定的灰度图片中，为每个顶点采样一个颜色作为反射强度

甚至，顶点的法向量也可以从图片中进行采样，r，g, b分量则分别代表x, y, z。
![](/images/webgl-3d-model/uv_mapping.png)

## Mesh
如下图所示，其实不管多复杂的模型，其实都是由三角形组成的（3D设计软件自动生成），只是顶点变多了而已，把前面的技术应用到每一个三角形上面，就可以表示非常逼真的模型了。
<img width="300" src="/images/webgl-3d-model/model_mesh.png"> 

## 模型加载
模型一般是一个文件，里面包含顶点信息（Mesh），材质映射等数据 。对于3D模型文件，市面上有多种文件格式。我们这次使用obj格式。
根据规范，编写了解析函数。最终解析出顶点信息和材质信息。
[源代码](https://github.com/Mutefish0/webgl-advanced/blob/master/src/model.ts#L5)

## 最终效果
![](/images/webgl-3d-model/model.gif)
完整代码在 [Github](https://github.com/Mutefish0/webgl-advanced)，欢迎Star。

