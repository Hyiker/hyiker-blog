# 使用 OpenGL 绘图（三角形）

## OpenGL 渲染创建流程

<img width="40%" height="40%" src='/assets/RenderingPipeline.png'></img>

**流程详解：**

-   顶点着色（生成相应的顶点(vertex) ）
-   形状联结（将顶点连接成线）
-   几何着色（可选）
-   光栅化（将连续的线段离散化为像素点）
-   片段着色（对光栅化产生的像素点进行着色）
-   使用

以上的每一步几乎都是不可省略的，其中至少有两个 shader 的步骤 即 vertex shader（顶点着色器） 和 fragment shader（片段着色器）

<!-- more -->

## 实例：绘制三角形

### 编写 GLSL 创建 Vertex Shader 以及 Fragment Shader

```cpp
    // 使用GLSL(OpenGL Shading Language (OpenGL着色语言)) 来编写对于的渲染语句
    // 第一行不可或缺 一定要指定版本
    const string vertex_shader_code = "#version 330 core"+
    +"layout (location = 0) in vec3 aPos;"
    +"void main()"
    +"{"
    +"    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);"
    "}";

    // 存储shader的ID
    unsigned int vertexShader_ID;
    vertexShader = glad_glCreateShader(GL_VERTEX_SHADER);
    // 输入对应的shader_ID以及source_code
    glShaderSource(vertexShader_ID, 1, vertex_shader_code, NULL);
    // 编译对应的shader_code
    glCompileShader(vertexShader);


```

### 创建 Shader Program 连接各个 shader

```cpp
// 创建shader program连接各个shader
    unsigned int shaderProgram_ID;
    shaderProgram_ID = glCreateProgram();
    glAttachShader(shaderProgram_ID, vertexShader_ID);
    glAttachShader(shaderProgram_ID, fragmentShader_ID);
    glLinkProgram(shaderProgram_ID);
```

### 创建 VBO 对象

```cpp
    // VBO:vertex buffer object 顶点缓存对象
    // 用于储存顶点的一个数据
    // 对应VBO的ID
    unsigned int VBO_ID;
    // 在内部产生VBO对象 并且将对应的ID储存在VBO_ID中
    glGenBuffers(1, &VBO_ID);
    // 将对应VBO对象绑定为GL_ARRAY_BUFFER类型
    glBindBuffer(GL_ARRAY_BUFFER, VBO_ID);
    // 将vertices中的数据拷贝入缓存中 指定使用STATIC模式h来绘制
    glBufferData(GL_ARRAY_BUFFER,   sizeof(vertices), vertices, GL_STATIC_DRAW);
```

`With the vertex data defined we'd like to send it as input to the first process of the graphics pipeline: the vertex shader. This is done by creating memory on the GPU where we store the vertex data, configure how OpenGL should interpret the memory and specify how to send the data to the graphics card. The vertex shader then processes as much vertices as we tell it to from its memory.`

"`我们希望能够将我们定义的顶点数据作为输入发送给GPU的渲染流程中：即顶点着色。这是通过创建一块存储顶点数据的内存区域、指定OpenGL应该如何翻译这些内存数据并且指定如何将数据发送给GPU完成的。顶点着色器之后根据我们通过内存制定的数目来处理相应的数个顶点`"

`We manage this memory via so called vertex buffer objects (VBO) that can store a large number of vertices in the GPU's memory. The advantage of using those buffer objects is that we can send large batches of data all at once to the graphics card without having to send data a vertex a time. Sending data to the graphics card from the CPU is relatively slow, so wherever we can, we try to send as much data as possible at once. Once the data is in the graphics card's memory the vertex shader has almost instant access to the vertices making it extremely fast`

"`我们通过VBO (顶点缓冲对象) 这样一种可以存储大量顶点数据的对象来管理这一块内存。使用这些缓冲对象的优点在于我们可以一次性从CPU向GPU中传输大量数据而无需一个一个地传输。因为从CPU中向GPU中传输数据相对较慢，所以我们应该一次性尽量地传输更多的数据。一旦这些数据被传输到了GPU中，那么顶点着色器便可以以极快的速度处理相应的数据`"

### 创建 VAO 对象

`A vertex array object (also known as VAO) can be bound just like a vertex buffer object and any subsequent vertex attribute calls from that point on will be stored inside the VAO. This has the advantage that when configuring vertex attribute pointers you only have to make those calls once and whenever we want to draw the object, we can just bind the corresponding VAO. This makes switching between different vertex data and attribute configurations as easy as binding a different VAO. All the state we just set is stored inside the VAO.`

"`VAO（顶点数组对象）可以将VBO对象绑定在一起，并且任何随后的属性调用都会被储存在VAO中。这样使得VAO有着一定的优势：设置顶点属性的时候你只需要调用一次，并且不管什么时候我们想要绘制对象的时候，我们只需要绑定相应的VAO。这样使得在不同的顶点数据以及属性配置之间切换起来像绑定一个不同的VAO一样简单。我们设置的所有参数都被存储在了VAO中`"

```cpp
 // VAO对象必须要有
    unsigned int VAO_ID;

    glGenVertexArrays(1, &VAO_ID);

    glBindVertexArray(VAO_ID);
    // 设置顶点指针（指向vertices）
    glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
    glEnableVertexAttribArray(0);

    [in the while loop]
    glUseProgram(shaderProgram_ID);
    glBindVertexArray(VAO_ID);

```
