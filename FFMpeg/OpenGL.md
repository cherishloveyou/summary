## OpenGL 

### BO（Buffer Object，缓冲对象）

 缓冲对象是OpenGL管理的一段内存，为了与我们CPU的内存区分开，一般称OpenGL管理的内存为：显存。

显存区域，存放顶点数据，就叫VBO，存放图像数据，就叫PBO，根据它存放的数据的不同，有不同的叫法。

- VAO顶点数组对象：Vertex Array Object，用来管理VBO
- VBO顶点缓冲对象：Vertex Buffer Object，用来缓存用户传入的顶点数据。
- EBO元素缓冲对象：Element Buffer Object，或IBO索引缓冲对象 Index Buffer Object，用来存放顶点索引数据。

### VBO和EBO

VBO（Vertex Buffer Object）是指**顶点缓冲区对象**，而 EBO（Element Buffer Object）是指**图元索引缓冲区对象**，VAO 和 EBO 实际上是对同一类 Buffer 按照用途的不同称呼。

OpenGL ES 2.0 编程中，**用于绘制的顶点数组数据首先保存在 CPU 内存**，在调用 glDrawArrays 或者 glDrawElements 等进行绘制时，需要将顶点数组数据从 CPU 内存拷贝到显存。但是很多时候我们没必要每次绘制的时候都去进行内存拷贝，如果可以在显存中缓存这些数据，可以在很大程度上降低内存拷贝带来的开销。

OpenGL ES 3.0  VBO 和 EBO 的出现就是为了解决这个问题，VBO 和 EBO 的作用是在显存中提前开辟好一块内存**，用于缓存顶点数据或者图元索引数据，从而避免每次绘制时的 CPU 与 GPU 之间的内存拷贝，可以改进渲染性能，降低内存带宽和功耗。**

**OpenGL ES 3.0 支持两类缓冲区对象：顶点数组缓冲区对象、图元索引缓冲区对象。**

```glsl
//VBO（EBO）的创建和更新：（EBO 实际上跟 VBO 一样，只是按照用途的另一种称呼）

//顶点缓冲对象（VBO）就像OpenGL中的其它对象一样，这个缓冲有一个独一无二的ID，所以我们可以使用glGenBuffers函数和一个缓冲ID生成一个VBO对象：
  
unsigned int VBO;
glGenBuffers(1, &VBO)
  
//也可以声明一个unsigned int 数组，那么创建的n个缓冲对象的ID会依次保存在数组里。
unsigned int VBO[3];
glGenBuffers(3,VBO);

// 绑定第一个 VBO，拷贝顶点数组到显存
unsigned int m_VboIds[2];
glGenBuffers(2, m_VboIds);
glBindBuffer(GL_ARRAY_BUFFER, m_VboIds[0]);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
// 绑定第二个 VBO（EBO），拷贝图元索引数据到显存
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, m_VboIds[1]);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

//glBufferData是一个专门用来把用户定义的数据复制到当前绑定缓冲的函数。它的第一个参数是目标缓冲的类型：当前绑定到GL_ARRAY_BUFFER目标上的顶点缓冲对象。第二个参数指定传输数据的大小(以字节为单位)；用一个简单的sizeof计算出顶点数据大小就行。第三个参数是我们希望发送的实际数据。
//第四个参数指定了我们希望显卡如何管理给定的数据。它有三种形式：

//GL_STATIC_DRAW 标志标识缓冲区对象数据被修改一次，使用多次，用于绘制。
//GL_STATIC_DRAW ：数据不会或几乎不会改变。
//GL_DYNAMIC_DRAW：数据会被改变很多。
//GL_STREAM_DRAW ：数据每次绘制时都会改变。


本例中顶点着色器和片段着色器增加 color 属性：

//顶点着色器
#version 330 core
layout (location = 0) in vec3 aPos;
void main()
{
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}

//片段着色器
#version 300
  esprecision mediump float;
  in vec3 v_color;
  out vec4 o_fragColor;
void main(){ 
  o_fragColor = vec4(v_color, 1.0);
}

顶点数组数据和图元索引数据：
// 4 vertices, with(x,y,z) ,(r, g, b, a) per-vertex
GLfloat vertices[] = {-0.5f, 0.5f, 0.0f,    // v0        
                      1.0f, 0.0f, 0.0f,     // c0        
                      -0.5f, -0.5f, 0.0f,   // v1        
                      0.0f, 1.0f, 0.0f,     // c1        
                      0.5f, -0.5f, 0.0f,    // v2        
                      0.0f, 0.0f, 1.0f,     // c2        
                      0.5f, 0.5f, 0.0f,     // v3        
                      0.5f, 1.0f, 1.0f,     // c3
  };
  // Index buffer data 
  GLushort indices[6] = { 0, 1, 2, 0, 2, 3};
```



### FBO的介绍

在 OpenGL 中，FBO 是 Framebuffer Object 的缩写，用于渲染到纹理或者多重渲染目标的技术。FBO 允许将渲染结果输出到一个纹理或者多个纹理上，而不是直接输出到屏幕上。这种机制允许实现更复杂的图形效果，例如后期处理、阴影映射、抗锯齿等，可以有效地实现高级的渲染技术和后期处理效果，并提高渲染质量和灵活性。

##### 使用 FBO 的基本步骤：

###### 创建 FBO：
使用 glGenFramebuffers() 创建一个 FBO 对象。
使用 glBindFramebuffer(GL_FRAMEBUFFER, framebufferID) 绑定 FBO 对象，告诉 OpenGL 后续操作会针对这个 FBO。

###### 附加纹理：
使用 glGenTextures() 创建一个纹理对象并配置它。
使用 glFramebufferTexture2D(GL_FRAMEBUFFER, attachment, GL_TEXTURE_2D, textureID, 0) 将纹理附加到 FBO 上。

###### 检查 FBO 完整性：
使用 glCheckFramebufferStatus(GL_FRAMEBUFFER) 检查 FBO 是否完整。

###### 渲染到 FBO：
在渲染过程中，将 FBO 绑定为当前渲染目标。
进行渲染操作，结果将输出到附加的纹理。

###### 解绑 FBO：
在完成渲染后，使用 glBindFramebuffer(GL_FRAMEBUFFER, 0) 解绑 FBO。

###### 清理资源：
当 FBO 不再需要时，记得删除 FBO 对象和相关的纹理。

VBO的介绍
在 OpenGL 中，VBO 是 Vertex Buffer Object 的缩写，用于存储顶点数据（如位置、颜色、法线等）在显存中的缓冲区对象。使用 VBO 可以提高性能并减少 CPU 到 GPU 之间的数据传输。

使用 VBO 的基本步骤：
创建 VBO：

使用 glGenBuffers() 创建一个 VBO 对象。
使用 glBindBuffer(GL_ARRAY_BUFFER, vboID) 绑定 VBO 对象为顶点数组缓冲区。
填充数据到 VBO：

使用 glBufferData(GL_ARRAY_BUFFER, size, data, usage) 将顶点数据传输到 VBO 中。
size：数据大小
data：指向要上传的数据的指针
usage：指定数据如何被使用（GL_STATIC_DRAW、GL_DYNAMIC_DRAW 等）
绘制顶点数据：

在渲染过程中，使用 VBO 存储的顶点数据进行绘制。
使用 glDrawArrays() 或 glDrawElements() 等函数指定绘制方式和顶点范围。
解绑 VBO：

在结束使用 VBO 后，使用 glBindBuffer(GL_ARRAY_BUFFER, 0) 解绑 VBO。
清理资源：

当 VBO 不再需要时，记得删除 VBO 对象。

示例代码片段：

```glsl
// 创建 VBO
GLuint vbo;
glGenBuffers(1, &vbo);
glBindBuffer(GL_ARRAY_BUFFER, vbo);

// 填充数据到 VBO
GLfloat vertices[] = { /* 顶点数据 */ };
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// 在渲染循环中绘制数据
glBindBuffer(GL_ARRAY_BUFFER, vbo);
// 设置顶点属性指针
glVertexAttribPointer(attributeIndex, size, type, normalized, stride, offset);
// 启用顶点属性数组
glEnableVertexAttribArray(attributeIndex);
// 绘制顶点数据
glDrawArrays(GL_TRIANGLES, 0, numVertices);
// 解绑 VBO
glBindBuffer(GL_ARRAY_BUFFER, 0);

// 删除 VBO
glDeleteBuffers(1, &vbo);
```

通过使用 VBO，可以有效地管理顶点数据，并在显存中存储大量的渲染数据，从而加速渲染过程。这种技术可以提高性能，减少数据传输带来的开销，并支持更复杂的图形效果。







https://blog.csdn.net/u012861978/article/details/130953012

https://www.jianshu.com/p/22cda5b2eedd
