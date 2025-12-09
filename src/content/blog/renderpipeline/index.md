---
title: Minecraft模组自定义渲染管线详解————以Fabric为例
publishDate: 2025-12-06
updatedDate: 2025-12-09
description: ''
tags: 
    - game
    - minecraft
    - fabric
    - render
    - development
language: 'Chinese'
draft: true
---
# 前言

# 渲染管线 (Render Pipe Line)

# DepthTestFunction
`com.mojang.blaze3d.platform.DepthTestFunction` 定义了几种枚举常量，用于指定片段如何与深度缓冲区进行比较，即哪种深度的元素会被渲染。

|枚举常量|OpenGL常量|含义|场景|
|---|---|---|---|
|NO_DEPTH_TEST|GL_ALWAYS|总是通过,不进行深度测试|透视效果(穿墙显示)|
|EQUAL_DEPTH_TEST|GL_EQUAL|深度值相等时通过|特殊遮罩效果|
|LEQUAL_DEPTH_TEST|GL_LEQUAL|深度值小于等于缓冲区时通过|天空盒、UI 叠加|
|LESS_DEPTH_TEST|GL_LESS|深度值小于缓冲区时通过|标准 3D 渲染|
|GREATER_DEPTH_TEST|GL_GREATER|深度值大于缓冲区时通过|反向深度测试|

## NO_DEPTH_TEST

```java
public FilledRenderPipeline() {
    super(RenderPipelines.register(RenderPipeline
            // Filled Shader
            .builder(RenderPipelines.DEBUG_FILLED_SNIPPET)
            .withLocation(ResourceLocation.fromNamespaceAndPath(Fabricdemo.MOD_ID, "pipeline/filled_through_walls"))
            .withVertexFormat(DefaultVertexFormat.POSITION_COLOR, VertexFormat.Mode.TRIANGLE_STRIP)
            // Disable depth testing so shapes are drawn through walls
            .withDepthTestFunction(DepthTestFunction.NO_DEPTH_TEST)
            .build())
    );
}
@Override
public void onInitializeClient() {
    FilledRenderPipeline.instance = this;
    // 在半透明渲染前绘制
    WorldRenderEvents.BEFORE_TRANSLUCENT.register(this::extraAndDraw);
}
```

## EQUAL_DEPTH_TEST

## LEQUAL_DEPTH_TEST

## LESS_DEPTH_TEST
这种用于最常见的标准 3D 渲染。

```java
@Override
public void onInitializeClient() {
    FilledRenderPipeline.instance = this;
    // 在实体渲染后绘制
    WorldRenderEvents.AFTER_ENTITIES.register(this::extraAndDraw);
}
```

## GREATER_DEPTH_TEST


# Reference Links
[绘图流水线 - Wikipedia](https://zh.wikipedia.org/wiki/%E7%B9%AA%E5%9C%96%E7%AE%A1%E7%B7%9A)