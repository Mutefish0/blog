---
title: 初探WebGL
date: 2020-02-23 11:34:25
tags:
---

## 什么是WebGL？
WebGL是一个Javascript API，用来渲染交互式的2D和3D图形。WebGL程序主要由两部分组成：第一部分是由Javascrip语言编写的控制代码，在CPU上运行；第二部分是由GLSL ES(OpenGL ES Shading Language)语言编写的着色器(Shader)代码，它直接在GPU上运行。

## WebGL Pipeline
渲染在屏幕上的每一个像素点都是经过了一系列的按顺序的步骤处理，这些步骤做成了渲染管线（Pipeline）。
![](/images/webgl-getting-started/pipeline-simplified.png)
简单解释每个步骤：
首先Javascript程序可以准备一系列「顶点」（Vertex）数据。对于每一个顶点，都会进行下面的步骤：

- 「顶点着色器」（Vertex Shader）程序拿到一个顶点，在这里可以很方便高效得对其进行一系列的坐标变换，这些计算主要都是矩阵运算。例如旋转、位移、仿射变换、投影变换等，都是在这里进行计算的。随后输出经变换后的顶点。
- 「片段着色器」（Fragment Shader）程序负责计算对应该顶点的颜色值，同时根据根据三个顶点组成的三角形，自动插值计算三角形内的颜色值。在这里可以很方便高效地处理纹理材质、阴影计算、光照计算等。
- 最后将坐标和颜色进行光栅化处理，形成每一个像素点。然后在canvas上渲染出来

## 标准设备坐标（Normalized Device Coordinate, NDC）
WebGL期望每次顶点着色器运行后，输出的所有**可见的**顶点都为标准设备坐标。也就是说，顶点的x、y、z坐标都应该在-1.0到1.0之间，超出这个范围的顶点将不可见。
![](/images/webgl-getting-started/coord.png)

下面的教程将实现在网页中渲染一个旋转的3D箱子。

## 初始化WebGL

创建一个canvas标签
```html
  <canvas id = "canv"></canvas>
```
获取canvas的WebGL上下文，并检查兼容性
```js
  var canvas = document.getElementById("canv");
  var gl;
  var glExtensions;
  if (window.WebGLRenderingContext) {
    gl = 
      canvas.getContext("webgl") || 
      canvas.getContext("experimental-webgl") || 
      canvas.getContext("moz-webgl") || 
      canvas.getContext("webkit-3d");
    if (gl) {
      gl.viewportWidth = canvas.width;
      gl.viewportHeight = canvas.height;
      glExtensions = gl.getSupportedExtensions();
    } else {
      throw new Error("WebGL功能被关闭");
      alert("WebGL功能被关闭");
    }
  } else {
    throw new Error("当前浏览器不支持WebGL");
    alert("当前浏览器不支持WebGL");
  }
```
## 画一个三角形
写一个简单的顶点着色器
```js
// 顶点着色器源代码，坐标不变
// 齐次向量w为1.0（w用于“透视除法”，即x、y、x分别处以w，这个操作会在顶点着色器最后阶段自动执行）
var vertexShaderSource = `
  attribute vec3 a_Pos;
  void main() {
    gl_Position = vec4(a_Pos.x, a_Pos.y, a_Pos.z, 1.0);
  }
`;
```
写一个简单的片段着色器
```js
// 片段着色器源代码，固定输出绿色
var fragShaderSource = `
  void main() {
    gl_FragColor = vec4(0.0, 0.5, 0.0, 1.0);
  }
`;
```
创建顶点着色器和片段着色器并编译
```js
var shader_vs = gl.createShader(gl.VERTEX_SHADER);
var shader_frag = gl.createShader(gl.FRAGMENT_SHADER);
gl.shaderSource(shader_vs, vertexShaderSource);
gl.shaderSource(shader_frag, fragShaderSource);
// 编译着色器源代码
gl.compileShader(shader_vs);
gl.compileShader(shader_frag);
// 检查编译错误
if (!gl.getShaderParameter(shader_vs, gl.COMPILE_STATUS)) {
  throw new Error("顶点着色器编译出错");
}
if (!gl.getShaderParameter(shader_frag, gl.COMPILE_STATUS)) {
  throw new Error("片段着色器编译出错");
}
```
创建一个程序，并将顶点着色器和片段着色器链接到一起
```js
var program = gl.createProgram();
gl.attachShader(program, shader_vs);
gl.attachShader(program, shader_frag);

var linkError = gl.linkProgram(program);
if (linkError) {
  throw new Error(linkError);
}
```
至此，着色器程序创建完成了。现在创建一块内存，把顶点数据存进去
```js
// 创建一块buffer内存来存储数据
var vertexbuffer = gl.createBuffer();
// 把gl.ARRAY_BUFFER绑定到vertexbuffer
gl.bindBuffer(gl.ARRAY_BUFFER, vertexbuffer);
// 把顶点数据送进buffer
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([
   -0.0, 0.5, 0.0,
  -0.5, -0.5, 0.0,
  0.5, -0.5, 0.0
]), gl.STATIC_DRAW);
```
激活上面创建好的着色器程序，并定义数据结构
```js
gl.useProgram(program);

var aPosLocation = gl.getAttribLocation(program, "a_Pos");
// 属性为“a_Pos”，分量为3，类型FLOAT（4个字节）,不归一化，间隔0个字节，偏移0字节
gl.vertexAttribPointer(aPosLocation, 3, gl.FLOAT, false, 0, 0);
gl.enableVertexAttribArray(aPosLocation);
```
最后画出三角形
```js
// 用黑色背景填充屏幕
gl.clearColor(0.0, 0.0, 0.0, 1.0);
gl.clear(gl.COLOR_BUFFER_BIT);
// 画三角形
gl.drawArrays(gl.TRIANGLES, 0, 3);
```
结果如图：
![](/images/webgl-getting-started/simple-triangle.png)

## 画一个正方形
如果按照上面的方法，画一个正方形，需要使用6个顶点（画2个三角形），这样就多存储了两个冗余的顶点数据。
为解决这个问题，WebGL支持通过按照索引来处理顶点数据，这样就只需4个顶点数据就够了
![](/image/webgl-getting-started/draw-by-index.png)
额外创建一个对应gl.ELEMENT_ARRAY_BUFFER的buffer，来存储索引
```js
var vertics = [
  -0.5, 0.5, 0.0,
  0.5, 0.5, 0.0,
  -0.5, -0.5, 0.0,
  0.5, -0.5, 0.0,
];
var indics = [
  0, 1, 2,
  2, 1, 3,
];

var vertexBuffer = gl.createBuffer();
gl.bindBuffer(gl.ARRAY_BUFFER, vertexbuffer);
gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(vertics), gl.STATIC_DRAW);

var indexBuffer = gl.createBuffer();
gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indexBuffer)
gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array(indics), gl.STATIC_DRAW);
```
使用gl.drawElements来绘制
```
gl.drawElements(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0);
```
结果如下
![](/images/webgl-getting-started/draw-a-square.png)
## 给正方形添加材质
创建材质（Texture）
```js
var texture = gl.createTexture();
gl.bindTexture(gl.TEXTURE_2D, texture);
gl.activeTexture(gl.TEXTURE0);

// 图片不重复，采取编译拉伸的方式
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
// 图片缩放采用线性过滤
gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
// 加载图片，并传送图片数据
var image = new Image();
image.onload = function () {
  gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, image);
}
image.src = "./box.png";
```
顶点着色器和片段着色器程序
```js
// 把材质坐标传给片段着色器
var vertexShaderSource = `
  attribute vec3 a_Pos;
  attribute vec2 tex_Coord;
  varying highp vec2 f_tex_Coord;
  void main() {
    f_tex_Coord = tex_Coord;
    gl_Position = vec4(a_Pos.x, a_Pos.y, a_Pos.z, 1.0);
  }
`;

// 通过材质插值得到颜色
var fragShaderSource = `
  uniform sampler2D u_Texture;
  varying highp vec2 f_tex_Coord;
  void main() {
    gl_FragColor = texture2D(u_Texture, f_tex_Coord);
  }
`;
```
在WebGL里，材质坐标(0, 0)对应图片左下角，(0, 1)对应图片左上角，(1, 0)对应图片右下角，（1，1）对应图片的右上角
![](/images/webgl-getting-started/texture-coord.png)
我们这次同时存储顶点数据和材质坐标数据
```js
var vertics = [
  // vertex          // texture coords
  -0.5, 0.5, 0.0,    0.0, 1.0,
  0.5, 0.5, 0.0,     1.0, 1.0,
  -0.5, -0.5, 0.0,   0.0, 0.0,
  0.5, -0.5, 0.0,    1.0, 0.0,
];
```
激活程序后，把材质绑定到第0个单元
```js
// 把材质绑定到第0个单元
var samplerLocation = gl.getUniformLocation(program, "u_Texture");
gl.uniform1i(samplerLocation, 0);
```
同时重新定义数据格式，这次步长为20个字节（5 * 4，一个Float32占四个字节），顶点数据的偏移为0，材质坐标的偏移位12个字节（3 * 4）
```js
var aPosLocation = gl.getAttribLocation(program, "a_Pos");
gl.vertexAttribPointer(aPosLocation, 3, gl.FLOAT, false, 4 * 5, 0);
gl.enableVertexAttribArray(aPosLocation);

var texCoordLocation = gl.getAttribLocation(program, "tex_Coord");
gl.vertexAttribPointer(texCoordLocation, 2, gl.FLOAT, false, 4 * 5, 4 * 3);
gl.enableVertexAttribArray(texCoordLocation);
```
最后画出来
```js
setInterval(() => {
  // 用黑色背景填充屏幕
  gl.clearColor(0.0, 0.0, 0.0, 1.0);
  gl.clear(gl.COLOR_BUFFER_BIT);
  // 画一个正方形
  gl.drawElements(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0);
}, 1000);
```
结果如下
![](/images/webgl-getting-started/draw-with-texture.png)

## 画一个旋转的木箱
定点数据，一共需要画6个面，每个面4个顶点
```js
var vertics = [
  // 后面
  -0.5, 0.5, -0.5,      0.0, 1.0,   
  0.5, 0.5, -0.5,       1.0, 1.0,
  -0.5, -0.5, -0.5,     0.0, 0.0,
  0.5, -0.5, -0.5,      1.0, 0.0,
  // 前面
  -0.5, 0.5, 0.5,      0.0, 1.0,
  0.5, 0.5, 0.5,       1.0, 1.0,
  -0.5, -0.5, 0.5,     0.0, 0.0,
  0.5, -0.5, 0.5,      1.0, 0.0,
  // 左面
  -0.5, -0.5, 0.5,      0.0, 1.0,
  -0.5, 0.5, 0.5,       1.0, 1.0,
  -0.5, -0.5, -0.5,     0.0, 0.0,
  -0.5, 0.5, -0.5,      1.0, 0.0,
  // 右面
  0.5, -0.5, 0.5,      0.0, 1.0,
  0.5, 0.5, 0.5,       1.0, 1.0,
  0.5, -0.5, -0.5,     0.0, 0.0,
  0.5, 0.5, -0.5,      1.0, 0.0,
  // 下面
  -0.5, -0.5, 0.5,     0.0, 1.0,
  0.5, -0.5, 0.5,      1.0, 1.0,
  -0.5, -0.5, -0.5,    0.0, 0.0,
  0.5, -0.5, -0.5,     1.0, 0.0,
  // 上面
  -0.5, 0.5, 0.5,      0.0, 1.0,
  0.5, 0.5, 0.5,       1.0, 1.0,
  -0.5, 0.5, -0.5,     0.0, 0.0,
  0.5, 0.5, -0.5,      1.0, 0.0,

];
var indics = [
  0, 1, 2,
  2, 1, 3,

  4, 5, 6,
  6, 5, 7,

  8, 9, 10,
  10, 9, 11,

  12, 13, 14,
  14, 13, 15,

  16, 17, 18,
  18, 17, 19,

  20, 21, 22,
  22, 21, 23,
];
```
修改顶点着色器代码，进行矩阵运算
```js
var vertexShaderSource = `
  attribute vec3 a_Pos;
  attribute vec2 tex_Coord;
  uniform mat4 translation;
  uniform mat4 rotation_x;
  uniform mat4 rotation_y;
  uniform mat4 projection;
  varying highp vec2 f_tex_Coord;
  void main() {
    f_tex_Coord = tex_Coord;
    gl_Position = projection * translation * rotation_y * rotation_x * vec4(a_Pos, 1.0);
  }
`;
```
开启Z缓冲，不然可以看到背面
```js
gl.enable(gl.DEPTH_TEST);
```
定义一系列uniform，可以在运行过程中，通过js实时传递给着色器
```js
var translationLocation = gl.getUniformLocation(program, "translation");
var rotationXLocation = gl.getUniformLocation(program, "rotation_x");
var rotationYLocation = gl.getUniformLocation(program, "rotation_y");
var projectionLocation = gl.getUniformLocation(program, "projection");
```
设置一系列工具函数，来进行坐标变换，这里涉及到一些线性代数和3D图形学的知识。具体可以查看
[参考资料](https://learnopengl-cn.github.io/01%20Getting%20started/08%20Coordinate%20Systems/)
代码实现：
```js
// 平移变换
function setTranslate(dx, dy, dz) {
  gl.uniformMatrix4fv(translationLocation, false, [
    1, 0, 0, 0,
    0, 1, 0, 0,
    0, 0, 1, 0,
    dx, dy, dz, 1,
  ]);
}

// 绕x轴旋转
function setRotateXDeg(rotateXDeg) {
  var rotateX = rotateXDeg / 180 * Math.PI;
  gl.uniformMatrix4fv(rotationXLocation, false, [
    1, 0, 0, 0,
    0, Math.cos(rotateX), -Math.sin(rotateX), 0,
    0, Math.sin(rotateX), Math.cos(rotateX), 0,
    0, 0, 0, 1
  ]);
}

// 绕y轴旋转
function setRotateYDeg(rotateYDeg) {
  var rotateY = rotateYDeg / 180 * Math.PI;
  gl.uniformMatrix4fv(rotationYLocation, false, [
    Math.cos(rotateY), 0, Math.sin(rotateY), 0,
    0, 1, 0, 0,
    -Math.sin(rotateY), 0, Math.cos(rotateY), 0,
    0, 0, 0, 1,
  ]);
}

// 透视变换
function setProjection(near, far, fov, aspectRatio) {
  var f = 1.0 / Math.tan(fov / 180 * Math.PI / 2);
  var rangeInv = 1 / (near - far);

  gl.uniformMatrix4fv(projectionLocation, false, [
    f / aspectRatio, 0, 0, 0,
    0, f, 0, 0,
    0, 0, (near + far) * rangeInv, -1,
    0, 0, near * far * rangeInv * 2, 0
  ]);
}
```
设置好初始参数
```js
setTranslate(0, 0, -4);
setRotateXDeg(45);
setRotateYDeg(45);
setProjection(0.1, 50.0, 45, 0.75);
```
通过requestAnimationFrame来进行实时渲染，并统计FPS
```js
// 统计FPS
var frameCount = 0;
var fpsEl = document.getElementById("fps");
setInterval(function () {
  fpsEl.innerText = "FPS: " + frameCount;
  frameCount = 0;
}, 1000);

function frameStep() {
  // 用黑色背景填充屏幕
  gl.clearColor(0.0, 0.0, 0.0, 1.0);
  gl.clear(gl.COLOR_BUFFER_BIT);
  // 画一个立方体
  gl.drawElements(gl.TRIANGLES, 36, gl.UNSIGNED_SHORT, 0);
  frameCount += 1;
  window.requestAnimationFrame(frameStep);
}

// 旋转木箱
var r = 0;
setInterval(function () {
  r += 1;
  if (r == 360) {
    r = 0;
  }
  setRotateXDeg(r);
  setRotateYDeg(r);
}, 10);

window.requestAnimationFrame(frameStep);
```
效果图：
![](/images/webgl-getting-started/cube.gif)
附上完整的[源代码](https://github.com/Mutefish0/webgl-getting-started)
