# WebGL笔记
记录WebGL一些基础概念

- [WebGL-1.0参考卡片](https://www.khronos.org/files/webgl/webgl-reference-card-1_0.pdf)
- [khronos](https://www.khronos.org/webgl/wiki/)

## 概念

- [MDN](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGL_API)
- WebGL 应用由 JavaScript程序和着色器程序构成
- 开发者需要针对 CPU 和 GPU 进行编程，CPU 部分是 JavaScript程序，GPU 部分是着色器程序
- WebGL 渲染管线的简单演示
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151782-b1907bd3-5296-4e5f-a0c7-1b210c38c22f.webp#averageHue=%23d5d7da&height=768&id=E3YK2&originHeight=768&originWidth=1024&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=1024)
   - JavaScript 程序
处理着色器需要的`顶点坐标`、`法向量`、`颜色`、`纹理`等信息，并负责为`着色器`提供这些数据
   - 顶点着色器
接收 JavaScript 传递过来的`顶点信息`(如位置，颜色)，将顶点绘制到对应坐标。进行的是逐顶点操作
   - 图元装配阶段
将顶点装配成指定`图元类型`，类别由`gl.drawArrays()`的第一个参数决定。上图采用的是三角形图元。
   - 光栅化阶段
将三角形内部区域用空像素进行填充（将装配好的几何图形转化为片元）。
   - 片元着色器
为三角形内部的像素填充颜色信息，上图为暗红色。片元着色器进行逐片元处理过程。可以将`片元`理解为像素(图像的单位)
- 图元：WebGL 能够绘制的基本图形元素，包含三种：`点`、`线段`、`三角形`
- 片元：可以理解为像素，像素着色阶段是在片元着色器中
- `WebGLShader(着色器)` `WebGLShader`可以是一个顶点着色器（`vertex shader`）或片元着色器（`fragment shader`）创建并使用`WebGLShader`流程
   - 使用 `WebGLRenderingContext.createShader`创建一个 `WebGLShader`
   - 通过 `WebGLRenderingContext.shaderSource()` 挂接`GLSL`源代码
   - 最后调用 `WebGLRenderingContext.compileShader()`完成着色器的编译。
   - 此时 `WebGLShader` 仍不是可用的形式，他需要被添加到一个 `WebGLProgram`里
- `WebGLProgram(着色器程序)` 
   - `WebGLProgram`由两个`WebGLShaders`，分别为`顶点着色器`和`片元着色器`
   - 一个 `WebGL 应用`可以包含多个 `program`
   - 创建并使用流程

#### 一个简单的例子
```javascript
const program = gl.createProgram(); // 创建着色器程序

gl.attachShader(program, vertexShader);  //  将顶点着色器挂载在着色器程序上。

gl.attachShader(program, fragmentShader);

// 链接着色器程序
gl.linkProgram(program);

// 返回WebGLProgram的信息
if (!gl.getProgramParameter( program, gl.LINK_STATUS)) {
	// 返回参数中指定的WebGLProgram object 的信息.
	// 这些信息包括在linking过程中的错误
	// 以及 WebGLProgram objects 合法性检查的错误.
	const info = gl.getProgramInfoLog(program);
	throw 'WebGL program 不能编译. \n\n' + info;
}


// 使用着色器程序
gl.useProgram(program);

// 删除着色器程序
gl.deleteProgram(program);
```


## [WebGL API](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLTexture)

- `gl.drawingBufferWidth` 当前绘图缓冲区的实际宽度。它应当匹配与绘图上下文相关联的  元素的宽度属性。如果实现未能提供所要求的宽度，值将有所不同
- `gl.drawingBufferHeight`
- `WebGLRenderingContext` 其他相关函数
   - `gl.clearColor(red, green, blue, alpha)`
      - 用于设置清空颜色缓冲时的颜色值。
      - 指定调用 `clear()` 方法时使用的颜色值。这些值在0到1的范围间
   - `gl.clear(mask)`
   
	```javascript
	// 设置清空画布颜色为黑色。
	gl.clearColor(0.0, 0.0, 0.0, 1.0);

	// 用上一步设置的清空画布颜色清空画布。
	gl.clear(gl.COLOR_BUFFER_BIT);
	```

   - [gl.drawArrays(mode, first, count)](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/drawArrays) 从向量数组中绘制图元(按顶点绘制)
      - `mode`  图元类型。
      - `first` 从第几个点开始绘制。
      - `count` 绘制的点的数量。
	  
		```javascript
		  // 绘制图元设置为三角形
		  var primitiveType = gl.TRIANGLES;

		  // 从顶点数组的开始位置取顶点数据
		  var offset = 0;

		  // 要绘制三个点，所以执行三次顶点绘制操作。
		  var count = 3;

		  gl.drawArrays(primitiveType, offset, count);
		```

   - `gl.drawElements(mode, count, type, offset)` 从数组数据渲染图元(按照`顶点索引`进行绘制)
      - `mode`：指定绘制图元的类型，是画点，还是画线，或者是画三角形。
      - `count`：指定绘制图形的顶点个数。
      - `type`：指定索引缓冲区中的值的类型,常用的两个值
         - `gl.UNSIGNED_BYTE` 无符号8位整数值
         - `gl.UNSIGNED_SHORT` 无符号16位整数。
      - `offset`：指定索引数组中开始绘制的位置，以字节为单位。
      - 示例：绘制如图所示矩形
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151764-0648367d-7a12-4756-8925-edfa99ec0bf0.webp#averageHue=%23fbfaf9&height=430&id=d1Pdy&originHeight=430&originWidth=598&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=598)

		```javascript
		 // 存储顶点信息的数组, 包含位置和颜色信息
		 var positions = [
			 30, 30, 255, 0, 0, 1,    //V0
			 30, 300, 255, 0, 0, 1,   //V1
			 300, 300, 255, 0, 0, 1,  //V2
			 300, 30, 0, 255, 0, 1    //V3
		 ];
		 // 存储顶点索引的数组
		 var indices = [
			 0, 1, 2, //第一个三角形
			 0, 2, 3  //第二个三角形
		 ];

		 // 创建索引缓冲区
		 var indicesBuffer = gl.createBuffer();
		 gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indicesBuffer);

		 gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array(indices), gl.STATIC_DRAW);

		 gl.drawElements(gl.TRIANGLES, 6, gl.UNSIGNED_SHORT, 0);
		```

- `WebGLShader` 可以是一个顶点着色器（`vertex shader`）或片元着色器（`fragment shader`）
   - `gl.createShader(type)` 创建一个 WebGLShader 着色器对象
      - 参数为`gl.VERTEX_SHADER` 或 `gl.FRAGMENT_SHADER`两者中的一个
   - `gl.shaderSource(shader, source)` 设置着色器（顶点着色器及片元着色器）的GLSL程序代码
      - `shader` 着色器对象(顶点着色器或片元着色器)
      - `source` 包含GLSL程序代码的字符串
   - `gl.compileShader(shader)` 用于编译一个GLSL着色器，使其成为为二进制数据，然后就可以被WebGLProgram对象所使用.
- 如何往着色器中传递数据
   - `gl.getAttribLocation(program, name)`：找到着色器中的 `attribute` 变量地址
   - `gl.getUniformLocation(program, name)`: 找到着色器中的 `uniform` 变量地址
   - `gl.vertexAttrib[1234]f[v](location, ...value)`: 给 attribute 变量传递数据
```javascript
gl.vertexAttrib1f(index, v0);
gl.vertexAttrib2f(index, v0, v1); // 给 attribute 变量传递两个浮点数
gl.vertexAttrib3f(index, v0, v1, v2);
gl.vertexAttrib4f(index, v0, v1, v2, v3);
```

   - `gl.uniform[1234][fi][v](location, ...value)`: 给 uniform 变量传递数据
      - [1234]
         - `gl.uniform1f(location, v0)` 传递一个值，将被填充到`uniform`变量的第一个分量，第二、三个分量被设为0，第四个分量被设为1.0
         - `gl.uniform1f` 传入一个浮点数，对应的 uniform 变量的类型为 float
         - `gl.uniform4f` 传入四个浮点数，对应的 uniform 变量类型为 float[4]
         - `gl.uniform3fv` 传入一个三维向量，对应的 uniform 变量类型为 vec3
      - [fi][v]
         - 浮点值 Number(方法名跟"f").
         - 浮点数组 (例如 Float32Array 或 Array 的数组) 用于浮点型向量方法 (方法名跟 "fv").
         - 整型值 Number  (方法名跟"i").
         - 整型数组Int32Array 用于整型向量方法 (方法名跟 "iv").
      - eg:

		```javascript
		// eg1: 传递二维向量数组
		glsl: uniform vec2 vFrames[3]; // [2, 88, 90, 176, 178, 264]

		const vFrames = gl.getUniformLocation(program, "vFrames");
		gl.uniform2fv(vFrames, new Float32Array([2, 88, 90, 176, 178, 264]));

		// eg2: 传递二维向量
		glsl: uniform vec2 vFrames; // [2, 88, 90, 176, 178, 264]

		const vFrames = gl.getUniformLocation(program, "vFrames");
		gl.uniform2fv(vFrames, new Float32Array([2, 88]));
		```

   - `gl.uniformMatrix[234]fv(location, transpose, value)` 给 uniform 变量指定矩阵值
      - `transpose` 是否转置矩阵。必须为 false
      - `value`  `Float32Array` 型或者是 `GLfloat` 序列值
      - `gl.uniformMatrix4fv` 传入一个 4x4 的矩阵，对应的 uniform 变量类型为 mat4
- 缓冲区([WebGLBuffer](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLBuffer)), 储存诸如顶点或着色之类的数据
   - 创建
      - `gl.createBuffer()` 创建一个缓冲区对象
      - `gl.bindBuffer(target, buffer)`  将给定的WebGLBuffer绑定到target(绑定某个缓冲区对象为当前缓冲区).绑定之后，对缓冲区绑定点的的任何操作都会基于该缓冲区（即`buffer`） 进行
         - `target`可能值
            - `gl.ARRAY_BUFFER`: 包含顶点属性的`Buffer`，如顶点坐标，纹理坐标数据或顶点颜色数据。
            - `gl.ELEMENT_ARRAY_BUFFER`: 用于元素索引的`Buffer`。
         - `buffer` 要绑定的`WebGLBuffer`
         - 程序中如果有多个 `buffer` 的时候，在切换 `buffer` 进行操作时，一定要通过调用 `gl.bindBuffer` 将要操作的 `buffer` 绑定到 `gl.ARRAY_BUFFER` 上，这样才能正确地操作 `buffer`
         - `bufferData` 和 `vertexAttribPointer` 都要在 `bindBuffer`之后
   - 如何从缓冲区中取数据
      - `gl.enableVertexAttribArray(index)` 激活顶点属性
		- `index` 类型为`GLuint`的索引，指向要激活的顶点属性。可以使用
		`getAttribLocation()`来获取索引
		
		```javascript
		gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);

		aVertexPosition = gl.getAttribLocation(shaderProgram, "aVertexPosition");

		gl.enableVertexAttribArray(aVertexPosition);

		gl.vertexAttribPointer(aVertexPosition, vertexNumComponents, gl.FLOAT, false, 0, 0);

		gl.drawArrays(gl.TRIANGLES, 0, vertexCount);
		```
	- 一旦激活，以下其他方法就可以获取到属性的值了，包括`vertexAttribPointer()`，`vertexAttrib()`，和 `getVertexAttrib()`
    	 - `gl.vertexAttribPointer(index, size, type, normalized, stride, offset)` 决定了目标属性如何从缓冲区中读取数据
			 - `index/target` 允许哪个属性读取当前缓冲区的数据.
			 - `size` 一次取几个数据赋值给 `target` 指定的目标属性。即每个顶点属性包含的数据长度。如，vec2 类型，即每次接收两个数据，所以 `size` 设置为 2
			 - `type` 数据类型，一般而言都是浮点型。
			 - `normalize` 是否需要将非浮点类型数据单位化到【-1, 1】区间
			 - `stride` 步长，即`每个顶点`所包含数据的**字节**数，默认是 0 ，0 表示一个属性的数据是连续存放的;
				- 当缓冲区只为一个属性服务时，缓冲区的数据是连续存放的，因此我们可以使用默认值 0 来表示
				- 如，一个顶点包含两个分量，X 坐标和 Y 坐标，假设每个分量都是一个 `Float32` 类型，占 4 个字节，所以，stride = 2 * 4 = 8 个字节
			 - `offset` 在每个步长的数据里，目标属性需要偏移多少**字节**开始读取
         - 示例1：缓冲区只为一个属性服务
		 
		```javascript
		var size = 2;
		var type = gl.FLOAT;
		var normalize = false;
		var stride = 0;
		var offset = 0;
		gl.vertexAttribPointer(a_Position, size, type, normalize, stride, offset);

		gl.bufferData(
			gl.ARRAY_BUFFER,
			new Float32Array(positions),
			gl.DYNAMIC_DRAW
		);
		```

  	- 示例2：缓冲区为两个属性(`vec2 a_Position`、`vec4 a_Color`)服务
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151786-a14ffb03-4ca0-49f8-a997-d8e11113ac7d.webp#averageHue=%23f6eee7&height=500&id=KjkG6&originHeight=500&originWidth=1280&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=1280)
            - a_Position：坐标信息占用 2 个元素，故 size 设置为 2。 坐标信息是从第一个元素开始读取，偏移值为 0 ，所以 offset 设置为 0.
            - a_Color：由于 color 信息占用 4 个元素，所以 size 设置为 4 。 color 信息是在坐标信息之后，偏移两个元素所占的字节（2 * 4 = 8）。所以，offset 设置为 8。
            - stride：代表一个顶点信息所占用的字节数，我们的示例，一个顶点占用 6 个元素，每个元素占用 4 字节，所以，stride = 4 * 6 = 24 个字节。
```javascript
gl.vertexAttribPointer(a_Position, 2, gl.FLOAT, false, 24, 0);
gl.vertexAttribPointer(a_Color, 4, gl.FLOAT, false, 24, 8);
```

   - 写入数据
      - [gl.bufferData(target, size, srcData?, usage)](https://developer.mozilla.org/zh-CN/docs/Web/API/WebGLRenderingContext/bufferData)
         - 创建并初始化了Buffer对象的数据存储区(可理解为往当前缓冲区（即通过 `bindBuffer` 绑定的缓冲区）中写入数据)
         - `bufferData`针对的缓冲区都是上一次`bindBuffer`绑定的缓冲区
   - 绘制
      - `gl.drawArrays(...)`
      - `gl.drawElements(...)`
   - 对比
      - 不使用缓冲区
	  
		```javascript
		// 为片元着色器中的 u_Color 传递随机颜色
		gl.uniform4f(u_Color, color.r, color.g, color.b, color.a);

		// 为顶点着色器中的 a_Position 传递顶点坐标。
		gl.vertexAttrib2f(a_Position, points[i].x, points[i].y);

		// 绘制点
		gl.drawArrays(gl.POINTS, 0, 1);
		```

      - 用 gl.vertexAttrib2f 直接给 a_Position 赋值，所以每绘制一个点，都要给着色器变量赋值一次，并且绘制一次，效率比较低
      - 使用缓冲区
	  
		```javascript
		const a_Position = gl.getAttribLocation(program, "a_Position");

		// 为片元着色器中的 u_Color 传递随机颜色
		const positionBuffer = gl.createBuffer();
		// 将当前 buffer 设置为 postionBuffer，接下来对 buffer 的操作都是针对 positionBuffer 了。
		gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);

		// 设置 a_Position 变量读取 positionBuffer 缓冲区的方式。
		var size = 2;
		var type = gl.FLOAT;
		var normalize = false;
		var stride = 0;
		var offset = 0;
		//  即如何读取buffer并修改 a_Position
		gl.vertexAttribPointer(a_Position, size, type, normalize, stride, offset);

		gl.enableVertexAttribArray(a_Position); 

		gl.bufferData(
		   gl.ARRAY_BUFFER,
		   new Float32Array(positions),
		   gl.STATIC_DRAW
		);

		gl.uniform4f(u_Color, color.r, color.g, color.b, color.a);

		gl.drawArrays(gl.POINTS, 0, 1);
		```

   - 用缓冲区向着色器传递数据的方式：
      - 利用一个缓冲区传递多种数据
      - 利用多个缓冲区传递多个数据
- [WebGL types](https://developer.mozilla.org/en-US/docs/Web/API/WebGL_API/Types)
   - `gl.FLOAT` 32位浮点型

#### 利用图元(点，线段，三角形)绘制图形

- WebGL 图元
   - `gl.POINTS`: 将绘制图元类型设置成点图元
   - 三角形图元
      - 基本三角形（`gl.TRIANGLES`） 【v1, v2, v3】为一个三角形，【v4, v5, v6】 为另一个三角形
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151772-9ca9f5d2-7ac3-414e-a264-066ef5d086d5.webp#averageHue=%23d8dbdc&height=394&id=Mh65B&originHeight=394&originWidth=994&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=994)
绘制三角形的数量 = 顶点数 / 3
      - 三角带（`gl.TRIANGLE_STRIP`） 【v1, v2, v3】, 【v3, v2, v4】, 【v3, v4, v5】, 【v5, v4, v6】 共计 4 个三角形(寻找公用的那条线)
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151784-f2e106bc-3a5c-4ed9-a29c-359a8282c1d8.webp#averageHue=%23d7dadc&height=462&id=lhnU5&originHeight=462&originWidth=1010&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=1010)
绘制三角形的数量 = 顶点数 - 2
      - 三角扇（`gl.TRIANGLE_FAN`）
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151780-239cfe89-0dee-4fb3-b366-94e2fc7b6268.webp#averageHue=%23d9dbdc&height=700&id=pTtUo&originHeight=700&originWidth=1010&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=1010)
绘制三角形的数量 = 顶点数 - 2
   - 线段图元
      - `gl.LINES` 基本线段
      - `gl.LINE_STRIP` 带状线段; 用前一个顶点作为当前线段的起始端点
      - `gl.LINE_LOOP` 环状线段 环状线段除了包含 `LINE_STRIP` 的绘制特性，还有一个特点就是将线段的终点和第一个线段的起点进行连接，形成一个线段闭环
- 绘制基本平面 

	glsl部分
	```html
	  <!-- 顶点着色器源码 -->
	  <script type="shader-source" id="vertexShader">
		// 浮点数设置为中等精度
		precision mediump float;

		// 接收canvas的尺寸。
		attribute vec2 a_Screen_Size;

		// 接收 JavaScript 传递过来的点的坐标（X, Y, Z）
		attribute vec2 a_Position;

		// 接收顶点颜色
		attribute vec4 a_Color;

		// 传递颜色
		varying vec4 v_Color;

		void main(){
		  // 将 canvas 的坐标值 转换为 [-1.0, 1.0]的范围。
		  vec2 position = (a_Position / a_Screen_Size) * 2.0 - 1.0;
		  // canvas的 Y 轴坐标方向和设备坐标系的相反。
		  position = position * vec2(1.0, -1.0);
		  // 最终的顶点坐标。
		  gl_Position = vec4(position, 0.0, 1.0);
		  // 将顶点颜色传递给片元着色器
		  v_Color = a_Color;
		}
	  </script>

	  <!-- 片元着色器源码 -->
	  <script type="shader-source" id="fragmentShader">
		// 浮点数设置为中等精度
		precision mediump float;
		// 全局变量，用来接收 JavaScript传递过来的颜色。
		varying vec4 v_Color;

		void main(){
		  // 将颜色处理成 GLSL 允许的范围[0， 1]。
		  vec4 color = v_Color / vec4(255, 255, 255, 1);
		  // 点的最终颜色。
		  gl_FragColor = color;
		}
	  </script>
	```
- 绘制矩形
     - 利用基本三角形构建
         - drawArrays
		 
```javascript
  const a_Position = gl.getAttribLocation(program, 'a_Position');
  const a_Color = gl.getAttribLocation(program, 'a_Color');
  gl.enableVertexAttribArray(a_Position);
  gl.enableVertexAttribArray(a_Color);
  
  const positions = [
    30, 30, 255, 0, 0, 1,  // v0
    30, 300, 255, 0, 0, 1, // v1
    300, 300, 255, 0, 0, 1, // v2
  
    30, 30, 0, 255, 0, 1,  // v0
    300, 300, 0, 255, 0, 1,// v2
    300, 30, 0, 0, 255, 1  // v3
  ]
  
  const buffer = gl.createBuffer();
  // 绑定缓冲区为当前缓冲
  gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
  // 设置buffer获取方式
  gl.vertexAttribPointer(a_Position, 2, gl.FLOAT, false, 24, 0);
  gl.vertexAttribPointer(a_Color, 4, gl.FLOAT, false, 24, 8);
  
  gl.bufferData(
    gl.ARRAY_BUFFER,
    new Float32Array(positions),
    gl.STATIC_DRAW
  );
  
  
  //设置清空画布颜色为黑色。
  gl.clearColor(0.0, 0.0, 0.0, 1.0);
  //用上一步设置的清空画布颜色清空画布。
  gl.clear(gl.COLOR_BUFFER_BIT);
  gl.drawArrays(gl.TRIANGLES, 0, positions.length / 6);
```

- 利用索引
	```javascript
	  const positions = [
		30, 30, 255, 0, 0, 1,  // v0
		30, 300, 255, 0, 0, 1, // v1
		300, 300, 255, 0, 0, 1, // v2
		300, 30, 0, 255, 0, 1  // v3
	  ]

	  const buffer = gl.createBuffer();
	  // 绑定缓冲区为当前缓冲
	  gl.bindBuffer(gl.ARRAY_BUFFER, buffer);
	  // 设置buffer获取方式
	  gl.vertexAttribPointer(a_Position, 2, gl.FLOAT, false, 24, 0);
	  gl.vertexAttribPointer(a_Color, 4, gl.FLOAT, false, 24, 8);

	  gl.bufferData(
		gl.ARRAY_BUFFER,
		new Float32Array(positions),
		gl.STATIC_DRAW
	  );

	  const indices = [0, 1, 2, 0, 2, 3]; // 逆时针
	  const indicesBuffer = gl.createBuffer();
	  // 绑定索引缓冲区
	  gl.bindBuffer(gl.ELEMENT_ARRAY_BUFFER, indicesBuffer);
	  // 向索引缓冲区传递索引数据
	  gl.bufferData(gl.ELEMENT_ARRAY_BUFFER, new Uint16Array(indices), gl.STATIC_DRAW);

	  gl.enable(gl.CULL_FACE)

	  //设置清空画布颜色为黑色。
	  gl.clearColor(0.0, 0.0, 0.0, 1.0);
	  //用上一步设置的清空画布颜色清空画布。
	  gl.clear(gl.COLOR_BUFFER_BIT);
	  gl.drawElements(gl.TRIANGLES, indices.length, gl.UNSIGNED_SHORT, 0);
	```
- 使用三角带构建矩形
```javascript
// [0=>1=>3]  [3=>1=>2]
var positions = [
    //V0
    30, 30, 0, 255, 0, 1,
    //V1
    30, 300, 255, 255, 0, 1, 
    //V3
    300, 30, 0, 0, 255, 1,
    //V2
    300, 300, 255, 0, 0, 1, 
]
gl.drawArrays(gl.TRIANGLE_STRIP, 0, positions.length / 6);
```

- 三角扇绘制矩形
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151841-d02480c6-1ea4-4a70-91c7-c1856e3e17cc.webp#averageHue=%23fcfbfa&height=436&id=XI0gD&originHeight=436&originWidth=506&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=506)
```javascript
// V0 -> V1 -> V2  V0 -> V2 -> V3  V0 -> V3 -> V4  V0 -> V4 -> V1
var positions = [
    165, 165, 255, 255, 0, 1, //V0
    30, 30, 255, 0, 0, 1,    //V1
    30, 300, 255, 0, 0, 1,   //V2
    300, 300, 255, 0, 0, 1,  //V3
    300, 30, 0, 255, 0, 1,   //V4
    30, 30, 255, 0, 0, 1,    //V1
]
gl.drawArrays(gl.TRIANGLE_FAN, 0, positions.length / 6);
```

- 绘制圆 

![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151854-5e56c942-db6d-4457-8cd2-2401b4edbe61.webp#averageHue=%23fdfcfc&height=760&id=i0EYF&originHeight=760&originWidth=848&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=848)
```javascript
var sin = Math.sin;
var cos = Math.cos;
function createCircleVertex(x, y, radius, n) {
  var positions = [x, y, 255, 0, 0, 1];
  for (let i = 0; i <= n; i++) {
    var angle = (i * Math.PI * 2) / n;
    positions.push(
      x + radius * sin(angle),
      y + radius * cos(angle),
      255,
      0,
      0,
      1
    );
  }
  return positions;
}
var positions = createCircleVertex(100, 100, 50, 12);
```

   - 环形 

![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151796-8ca928d2-7a75-4090-bb12-101fed5a6060.webp#averageHue=%23fcf9f9&height=602&id=UrGiB&originHeight=602&originWidth=650&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=650)

#### 3D 形体 C8

- 背面剔除
   - 一个矩形其实可以由两个共线的三角形组成，即 V0, V1, V2, V3，其中 V0 -> V1 -> V2 代表三角形A，V0 -> V2 -> V3代表三角形B
![](https://cdn.nlark.com/yuque/0/2021/webp/263005/1619344151788-6f48f1ee-0c6f-4534-a980-708de9e55a0a.webp#averageHue=%23fbfaf9&height=430&id=TxSH0&originHeight=430&originWidth=598&originalType=binary&ratio=1&rotation=0&showTitle=false&size=0&status=done&style=none&title=&width=598)
> 组成三角形的顶点要按照一定的顺序绘制。默认情况下，WebGL 会认为顶点顺序为逆时针时代表正面，反之则是背面，区分正面、背面的目的在于，如果开启了背面剔除功能的话，背面是不会被绘制的。当我们绘制 3D 形体的时候，这个设置很重要

- `gl.enable(gl.CULL_FACE)` 开启背面剔除功能
- `gl.cullFace(gl.FRONT)` 剔除正面，只显示背面

---

## [类型化数组](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/TypedArray#Syntax)

- `Float32Array` 32位浮点数组
   - 对应 `gl.FLOAT`
   - `Float32Array.BYTES_PER_ELEMENT` 返回元素字节数(返回4), 即一个元素四个字节
- `Uint16Array` 16位无符号整数
   - 一个元素2个字节

---

- 法线向量
- 叉乘

---

