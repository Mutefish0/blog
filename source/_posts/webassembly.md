---
title: 利用WebAssembly实现50倍性能提升
date: 2020-05-17 14:55:09
tags:
---

<style>
  .align-center {
    text-align: center;
  }
</style>

上一篇博客写的是使用WebGL渲染3D模型，当时遇到一个问题没解决。那就是模型解析速度太慢。对于有40万个顶点，2M大小的 Wavefront .obj 文件，需要30多秒完成解析。关键是，40万个顶点还算是中小型的模型，所以这个速度不能接受。解决浏览器计算的性能问题，首先想到的就是最近非常流行的WebAssembly技术，最终使用WebAssembly重构了模型加载器，解析时间从30多秒缩短到0.6秒，实现了50倍的性能提升。

<img src="/images/wasm/js_cost.png">
<p class="align-center">
<i>Javascript版本</i>
</p>

<img src="/images/wasm/wasm_cost.png">
<p class="align-center">
<i>WebAssembly版本</i>
</p>

# 如何实现的？
![](/images/wasm/overview.png)

简单来说就是：利用`Emscripten`这个工具将C语言实现的模型加载库编译成二进制的WebAssembly，最后浏览器直接加载运行。

下面将分别介绍`WebAssembly`和`Emscripten`，最后介绍一些比较关键的实现细节。


# WebAssembly 介绍

可以简单总结如下：
1. WebAssembly（缩写为Wasm）是一个二进制指令格式
2. wasm在一个非常简单高效的基于栈的虚拟机中运行
3. 目前有各大主流浏览器、Nodejs、wapm.io（Rust编写的Wasm虚拟机命令行工具）
4. 目前Wasm生态已有成熟工具可以将高级语言如 C / C++ / C# /Rust / Go / Python 编译成Wasm，并在浏览器或者Nodejs中运行

有了Wasm后，使用其他语言写的项目，可以直接编译并在浏览器运行。以下有几个例子可以看看：

[在浏览器运行的openssl](https://wapm.io/package/openssl#shell)
[用Rust编写的Python解释器，在浏览器运行](https://wapm.io/package/rustpython#shell)

Wasm的文本格式为Wat（WebAssembly Text Format)，详细的规范可以在[这里](https://webassembly.github.io/spec/core/text/index.html)查阅。我们可以使用wabt（WebAssembly Binary Toolkit）这个工具将wat文件转换成二进制wasm文件。

下面通过几个小例子的实践，来了解一些基本的概念和语法


## 安装Wabt

访问 [Github](https://github.com/WebAssembly/wabt) 安装文档进行安装，安装完成后，可以使用`wat2wasm`将wat转换成wasm，`wasm2wat`将wasm转为wat

## 在浏览器的实践
准备网页环境，并实现一个可以加载wasm模块的函数

新建`index.html`文件：

```html
<html>
  <body>
    <script src="/index.js"></script>
  </body>
</html>
```

新建`index.js`文件：

```js
// 从`url`加载二进制wasm代码，并执行，然后导出wasm定义的模块
function loadWasmModules(url, imports) {
  return fetch(url)
  .then((resp) => resp.arrayBuffer())
  .then((buffer) => WebAssembly.instantiate(buffer, imports))
  .then(({ instance }) => instance.exports);
}
```

### 从Javascript执行Wasm函数
新建`add.wat`

```wat
(module
  (func (param i32) (param i32) (result i32)
    get_local 0  ;; push 索引为0的参数
    get_local 1  ;; push 索引为1的参数
    i32.add ;; pop两个操作数执行相加，并push执行得到的结果
    ;; 栈里留下一个操作数，与result匹配
    ;; 如果没有声明result，则函数执行完后，栈必须是空的
  )
  (export "add" (func 0))
)
```

将`add.wat`转换成`add.wasm`
`$ wat2wasm add.wat -o add.wasm`

在`index.js`中添加
```js
loadWasmModules('/add.wasm', {}).then(({ add }) => {
  console.log(add(1, 2));  // 3
});
```

得到了正确的结果`3`

解释一下上面的wasm程序：
- 在wasm程序中，用两个分号`;;`表示注释
- 一个wasm程序就是一个`module`，用一对括号包裹起来`(module)`，`module`后面可以声明多个模块，分别用一对括号包裹起来：`(module (module1) (module2) (module3))`
- `func`关键字定义一个函数模块
- `export`关键字表示导出一个模块，并且对其命名


`func`定义一个函数，用`param`声明参数类型，`result`声明结果类型。`i32`表示32位正数类型，`f64`表示64位浮点数类型。在参数声明和结果声明之后，可以定义一系列的指令，用换行隔开。

wasm中的函数是一个栈机器，`get_local`指令按照索引往栈里push一个数据，`i32.add`指令从栈里面pop出两个数据，并且将相加的结果push进栈中，函数执行完后，栈要么为空（没有申明result），要么留下一个数据（声明了result）。
`add`的执行过程如下
![](/images/wasm/stack.gif)


### 从Wasm中执行Javascript函数
在wasm中可以声明`import`模块，来从Javascrip导入模块

新建`calljs.wat`文件
```wat
(module
  (import "imports" "logFromJs" (func $logFromJs (param i32)))
  (func $test
    i32.const 12
    call $logFromJs
  )
  (start $test)
)
```

修改`index.js`
```js
function logFromJs(x) {
  console.log(`from js: ${x}!`);
}

loadWasmModules('/calljs.wasm', { imports: { logFromJs } }).then(() => {
  // from js: 12!
});
```

加载wasm模块后，控制台会打印：from js: 12!

解释一下程序：
- 可以使用`$name`的方式来给索引取一个别名，后续不用再手写索引，直接可以用别名替代
- 使用`import `从外部导入模块（这里是函数），并且声明类型
- `i32.const 12` 表示往栈中push一个值：12
- `call $logFromJs` 表示执行`$logFromJs`这个函数，从堆中pop一个i32类型的值，执行后无返回值，栈为空
- `start` 表示指定一个开始函数，即这个函数会被直接执行

### 共享内存

Wasm最关键的特性之一就是共享内存，即Wasm程序可以和外部程序（这里指Javascript）共享一段内存。一般有2中方式：
1. Wasm程序声明一段内存，并且导出
2. 外部程序声明一段内存，然后导入到Wasm程序

### 从Wasm导出内存
新建`exportmem.wat`
```wat
(module
  (memory $mem 1) ;; 申请一页内存(64k)
  (func $start
    i32.const 0 
    i32.const 42
    i32.store    ;; 往偏移0字节的内存位置，存储32位整数：42
    i32.const 4
    i32.const 31
    i32.store    ;; 往偏移4字节的内存位置，存储32位整数：32
  )
  (start $start)
  (export "mem" (memory $mem)) ;; 导出内存
)
```
修改`index.js`
```js
loadWasmModules('/exportmem.wasm', {}).then(({ mem }) => {
  const view = new Int32Array(mem.buffer);
  console.log(view[0]); // 42
  console.log(view[1]); // 31
});
```

加载wasm模块后，控制台会打印：42 31

解释一下程序：
- `memory`用来申请一块指定大小的连续内存，单位为页，一页等于64kb
- `i32.store`会从栈中push出两个值，分别是偏移量，以及需要存储的32位整数

### 导入外部内存到Wasm

修改`index.js`
```js
const mem = new WebAssembly.Memory({ initial: 1 });
loadWasmModules('/importmem.wasm', { imports: { mem } }).then(() => {
  const view = new Int32Array(mem.buffer);
  console.log(view[0]);
  console.log(view[1]);
});
```

新建`importmem.wat`
```wat
(module
  (import "imports" "mem" (memory $mem 1))
  (func $start 
    i32.const 0  
    i32.const 42
    i32.store
    i32.const 4
    i32.const 31
    i32.store
  )
  (start $start)
)
```

加载wasm模块后，控制台会打印：42 31

# Emscript 介绍

`Emscript`是一个工具集，可以将C / C++ 代码编译成WebAssembly

## 安装 Emscript

按照官网[这个链接](https://emscripten.org/docs/getting_started/downloads.html)的步骤进行安装
安装完毕后将可以使用`emcc`这个命令

## Hello world
使用C语言编程输出`Hello world`，然后编译成Wasm在浏览器中运行

新建文件`helloworld.c`
```c
#include <emscripten.h>
#include <stdio.h>

int main() { 
  printf("Hello world\n"); 
}
```

使用`emcc`编译
`$ emcc helloworld.c -s ENVIRONMENT="web" -o helloworld.js`
然后会生成`helloworld.js`和`helloworld.wasm`这两个文件

修改`index.html`
```html
<html>
  <body>
    <script src="/helloworld.js"></script>
  </body>
</html>
```

刷新页面，控制台会打印出`Hello world`

## 导出C函数
我们用C写一个`add`方法，并在浏览器中调用

新建`cadd.c`
```c
#include <emscripten.h>

EMSCRIPTEN_KEEPALIVE
int add(int x, int y) { 
  return x + y; 
}
```

编译
`$ emcc cadd.c -s ENVIRONMENT="web" -s EXPORTED_FUNCTIONS="['_add']" -o cadd.js`

修改`index.html`
```html
<html>
  <body>
    <script src="/cadd.js"></script>
    <script>
      console.log(Module._add(1, 2));
    </script>
  </body>
</html>
```

刷新页面，控制打印出：3

# 利用Emscript重构模型加载器

首先从Github上搜索标星比较多的`Wavefront .obj`加载器项目，最后选定`tinyobjloader-c`这个项目，性能也不错

## 修改源代码

首先folk了一份，因为原本的数据结构并不能满足项目需求，需要对源码进行了一些修改，对数据进行组装，修改了`tinyobj_loader_c.h`这个文件的部分代码：
```c
typedef struct {
  unsigned int num_vertices;
  unsigned int num_normals;
  unsigned int num_texcoords;
  unsigned int num_faces;
  unsigned int num_face_num_verts;
  unsigned int num_material_stops;

  int pad0;
  int pad1;

  float* combined_vertices;
  float* vertices;
  float* normals;
  float* texcoords;
  tinyobj_vertex_index_t* faces;
  int* face_num_verts;
  material_stop* material_stops;
  char material_lib[MAX_MATERIAL_NAME_LEN];
} tinyobj_attrib_t;

...

/* Construct composite vertex */
for (unsigned int i = 0; i < num_f; i++) {
  tinyobj_vertex_index_t* idx = &attrib->faces[i];
  memcpy(attrib->combined_vertices + i * 8,
          attrib->vertices + idx->v_idx * 3, 12);
  memcpy(attrib->combined_vertices + i * 8 + 3,
          attrib->texcoords + idx->vt_idx * 2, 8);
  memcpy(attrib->combined_vertices + i * 8 + 5,
          attrib->normals + idx->vn_idx * 3, 12);
}
```

如图所示
```
                                                        
               vertex                                   
 +-----+-----+-----+-----+-----+-----+                  
 |     |     |     |     |     |     |                  
 | x1  | y1  | z1  | x2  | y2  | z2  |                  
 |     |     |     |     |     |     |                  
 +-----+-----+-----+-----+-----+-----+                  
              texture coords                            
 +-----+-----+-----+-----+                              
 |     |     |     |     |                              
 | x1  | y1  | x2  | y2  |                              
 |     |     |     |     |                              
 +-----+-----+-----+-----+                              
              normals                                   
 +-----+-----+-----+-----+-----+-----+                  
 |     |     |     |     |     |     |                  
 | x1  | y1  | z1  | x2  | y2  | z2  |                  
 |     |     |     |     |     |     |                  
 +-----+-----+-----+-----+-----+-----+                  
                                     
                  |                                     
                  |                                     
                  v                                     
              combined vertex                           
+-----+-----+-----+-----+-----+-----+-----+-----+       
|     |     |     |     |     |     |     |     |       
| x1  | y1  | z1  | x1  | x2  | x1  | y1  | z1  | ......
|     |     |     |     |     |     |     |     |       
+-----+-----+-----+-----+-----+-----+-----+-----+       
|     vertex      |  texcoord |     normal      |       
|                 |           |                 |       
                                                    
```

## 导出函数并进行封装
需要将用到的函数进行封装，提供相应数据的内存地址映射，这样Javascript可以直接从共享内存取到对应的数据wasm程序计算好的数据

新建`tinyobj_loader.c`
```c
#include <emscripten.h>

#include "tinyobj_loader_c.h"

#define TINYOBJ_LOADER_C_IMPLEMENTATION

EMSCRIPTEN_KEEPALIVE
int tinyobj_parse_obj(tinyobj_attrib_t* attrib, const char* buf, size_t len);

EMSCRIPTEN_KEEPALIVE
void tinyobj_attrib_free(tinyobj_attrib_t* attrib);

EMSCRIPTEN_KEEPALIVE
int get_attr_size() { return sizeof(tinyobj_attrib_t); }

EMSCRIPTEN_KEEPALIVE
unsigned long* get_vertices_slice(tinyobj_attrib_t* attrib) {
  unsigned long arr[2] = {};
  arr[0] = attrib->vertices;
  arr[1] = arr[0] + attrib->num_vertices * 3 * sizeof(float);
  return arr;
}

EMSCRIPTEN_KEEPALIVE
unsigned long* get_texcoords_slice(tinyobj_attrib_t* attrib) {
  unsigned long arr[2] = {};
  arr[0] = attrib->texcoords;
  arr[1] = arr[0] + attrib->num_texcoords * 2 * sizeof(float);
  return arr;
}

EMSCRIPTEN_KEEPALIVE
unsigned long* get_normals_slice(tinyobj_attrib_t* attrib) {
  unsigned long arr[2] = {};
  arr[0] = attrib->normals;
  arr[1] = arr[0] + attrib->num_normals * 3 * sizeof(float);
  return arr;
}

EMSCRIPTEN_KEEPALIVE
char* get_material_lib(tinyobj_attrib_t* attrib) {
  return attrib->material_lib;
}

EMSCRIPTEN_KEEPALIVE
unsigned long* get_material_stops_slice(tinyobj_attrib_t* attrib) {
  unsigned long arr[2] = {};
  arr[0] = attrib->material_stops;
  arr[1] = arr[0] + attrib->num_material_stops * (80 + sizeof(unsigned long));
  return arr;
}

EMSCRIPTEN_KEEPALIVE
unsigned long* get_combined_vertices_slice(tinyobj_attrib_t* attrib) {
  unsigned int num_faces = attrib->num_faces;
  unsigned long arr[2] = {};
  arr[0] = attrib->combined_vertices;
  arr[1] = arr[0] + 32 * num_faces;
  return arr;
}
```

## 编译
使用如下命令进行编译，配置128MB的共享内存
```shell
$ emcc tinyobj_loader.c -s EXTRA_EXPORTED_RUNTIME_METHODS="['cwrap', 'stringToUTF8', 'UTF8ToString']" -s EXPORTED_FUNCTIONS="['_get_attr_size', '_tinyobj_attrib_free', '_tinyobj_parse_obj', '_get_material_lib', '_get_material_stops_slice', '_get_combined_vertices_slice']" -s MODULARIZE=1 -s ENVIRONMENT=web -s INITIAL_MEMORY=134217728 -o wamod.js
```


## Javscript封装
Javascript根据地址映射，取到对应函数的计算结果的buffer数据

新建`objloader.js`
```js
import WAModule from './wamod';

const wamodule = WAModule();

const getAttrSize = wamodule.cwrap('get_attr_size', 'number', []);
const tinyobjAttribFree = wamodule.cwrap('tinyobj_attrib_free', null, ['number']);
const tinyobjParseObj = wamodule.cwrap('tinyobj_parse_obj', null, ['number', 'number', 'number']);
const getCombinedVerticesSlice = wamodule.cwrap('get_combined_vertices_slice', 'number', [
  'number',
]);
const getMaterialStopsSlice = wamodule.cwrap('get_material_stops_slice', 'number', ['number']);
const getMaterialLib = wamodule.cwrap('get_material_lib', 'string', ['number']);

const initializedPromise = new Promise((resolve) => {
  wamodule['onRuntimeInitialized'] = function() {
    resolve();
  };
});

function parse(src) {
  const srcLen = src.length;
  const attrSize = getAttrSize();
  const attrBuf = wamodule._malloc(attrSize);

  const srcBuf = wamodule._malloc(srcLen + 1);
  wamodule.stringToUTF8(src, srcBuf, srcLen);

  tinyobjParseObj(attrBuf, srcBuf, srcLen);

  wamodule._free(srcBuf);
  tinyobjAttribFree(attrBuf);

  return attrBuf;
}

function free(attrBuf) {
  wamodule._free(attrBuf);
}

function getCombinedVerticesBufferSlice(attrBuf) {
  const p = getCombinedVerticesSlice(attrBuf);
  const view = new DataView(wamodule.HEAP8.buffer);
  return [view.getUint32(p, true), view.getUint32(p + 4, true)];
}

function getMaterialStops(attrBuf) {
  const p = getMaterialStopsSlice(attrBuf);
  const view = new DataView(wamodule.HEAP8.buffer);
  const slice = [view.getUint32(p, true), view.getUint32(p + 4, true)];

  const total = (slice[1] - slice[0]) / 84;
  const result = [];
  for (let i = 0; i < total; i++) {
    result.push({
      index: view.getUint32(slice[0] + i * 84, true),
      material: wamodule.UTF8ToString(slice[0] + i * 84 + 4, 80),
    });
  }
  return result;
}

function parseOBJ(src) {
  const ptr = parse(src);
  const combinedVerticesSlice = getCombinedVerticesBufferSlice(ptr);

  const mtlstops = getMaterialStops(ptr);
  const mtllibs = [getMaterialLib(ptr)];

  const vertices = new Float32Array(
    wamodule.HEAP8.buffer.slice(combinedVerticesSlice[0], combinedVerticesSlice[1])
  );

  free(ptr);

  return { vertices, mtllibs, mtlstops };
}

export default function getObjLoader() {
  return initializedPromise.then(() => ({
    parseOBJ,
  }));
}
```

## 使用
```js
import getObjLoader from './objloader';
getObjLoader().then((loader) => {
  const {vertices, mtllibs, mtlstops} = loader.parseOBJ(text);
});
```
`vertices`是整合好的`Float32Array`类型数据，可以直接通过WebGL输送给GPU进行渲染

最后对一个2M模型文件进行解析，实现了50倍的性能提升。

## 源码
[这里](https://github.com/Mutefish0/webgl-advanced/tree/master/src/lib)可以查看对应的源码
