---
title: OpenGL学习笔记（一）
date: 2019-12-08 19:44:51
tags:
keywords:
categories:
description:
summary_img:
---

## OpenGL Kick Start

**窗口创建测试**

```cpp
#include <iostream>
#include <glad/glad.h>
#include <GLFW/glfw3.h>

/*
 OpenGL内部实现使用标准化设备坐标系(Normalize Device Coordinates)
 该坐标系下的xyz坐标 ∈ [ -1.0, 1.0 ]
 glViewport用于设置从该坐标系投影到二维平面上的视口
 即从一个三维坐标由矩阵变化为一个二维坐标
 这一步的设置极为重要 因为该部分决定了图像在窗口上的具体表现
 如果更改了窗口大小却不调用glViewport会导致窗口中的图像相对于屏幕不改变位置而相对于窗口改变位置
 */

/**
 * 该方法用于每次改变窗口大小时改变渲染位置
 @param window : 接受的窗口
 @param width : 新的宽度
 @param height : 新的高度
 */

void framebuffer_size_callback(GLFWwindow* window, int width, int height){
    glViewport(0, 0, width, height);
}

int main(int argc, const char * argv[]) {
    glfwInit();
    // MacOS需要设置
    glfwWindowHint(GLFW_OPENGL_FORWARD_COMPAT, GL_TRUE);
    // 设置GLFW库的主次要的Version
    glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 3);
    glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 3);
    // 设置OpenGL的核心资料部分
    glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
    // 新建窗口 窗口宽高 窗口标题 全屏模式下的监视器 与该窗口共享窗口的指针
    GLFWwindow* window = glfwCreateWindow(800, 600, "OpenGL learning", NULL, NULL);
    // GLFW库主要负责OpenGL中人机交互的部分
    // 例如显示窗口、获取键鼠输入
    if (window == NULL) {
        std::cout << "FATAL: Fail to create a GLFW window!" << std::endl;
        glfwTerminate();
        return -1;
    }
    // 设置窗口为当前线程渲染的主目标
    glfwMakeContextCurrent(window);
    
    if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress))
    {
        std::cout << "FATAL: Failed to initialize GLAD!" << std::endl;
        return -1;
    }
    // 指定视口 具体解释在上面
    glViewport(0, 0, 800, 600);
    
    // 告诉GLFW在每次窗口改变大小时都改变渲染区域 注册事件
    glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);
    
    // 如果没有按下关闭键
    while(!glfwWindowShouldClose(window)) {
        // 前后台双缓冲交换缓冲
        glfwSwapBuffers(window);
        // 处理输入输出
        glfwPollEvents();
    }
    // 终止程序
    glfwTerminate();
    return 0;
}

```

___所有的注释均在代码中___

