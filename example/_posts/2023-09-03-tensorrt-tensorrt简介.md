---
layout: post
title: 介绍tensorrt 8.6.1

description: >
  tensorrt是常用算法部署工具
sitemap: false

---



以下介绍tensorrt 8.6.1版本的内容

# 目录

1. TensorRT简介
   1. 基本特性和用法
   2. 工作流：使用tensorrt api搭建
   3. 工作流：使用onnx-parser——使用parser解析onnx模型
   4. 工作流：使用框架内tensorrt接口
2. 开发辅助工具
   1. trtexec
   2. Netron
   3. polygraphy
   4. onnx-graphsurgeon
   5. Nsight Systems
   6. 性能测试，网络可视化，模型迁移，精度检验，计算图编辑，性能优化
3. 插件书写
   1. 使用plugin的简单例子
   2. 关键API
   3. 结合使用parser和plugin
   4. plugin高级话题
   5. 使用plugin的例子
4. tensorrt高级用法
   1. 多optimization profile
   2. 多stream
   3. 多context
   4. cuda graph
   5. timing cache
   6. algorithm selector
   7. refit
   8. tactic source
   9. 硬件兼容和版本兼容
5. 常见优化策略
   1. 概述
   2. 性能分析工具
   3. 性能优化实例

## tensorrt工作流

1. 使用框架自带的TRT接口

TF-TRT，Torch-TensorRT

特点：易用但是性能不佳，兼容性有限，需要返回原框架计算

2. 使用Parser——最常用，也是我们目前采用的

TF/Torch/...->ONNX->TensorRT

特点：性能兼容性更好，不支持时可以改网络，改parser或者写plugin

当遇到重新需要生成的，需要parser onnx后，进行serialize生成trt

3. 使用原生API搭建网络

![image-20230829103956511](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230829103956511.png)

**以下只介绍使用Parser的tensorrt工作流**



# 实际工程通用流程

## workflow

**pytorch/tensorflow->C->onnx->A->trt->B->使用**

C为python转C++的最重要过程

构建引擎需要时间，为了构建一次来反复使用，需要A，B两个工作流

### 解析onnx

A：**parse/deserialize onnx，serialize trt**

解析对象为onnx文件：没有已经转好的trt或者已有的trt不适配，需要将onnx序列化trt

### 解析trt

B：**parse/deserialize trt**

解析对象为trt文件：已有trt，直接导入然后得到engine和context，使用

---

A，B两者最重要的**中间产物**都是parseTRT或者onnx得到的**engine和context**

### python转C++

C：将python模型转化为C++

pytorch

1. 创建网络，保存为.pt
2. pytorch内部API将.pt转化为.onnx
3. tensorrt中读取onnx构建engine

TensorFlow

1. 创建网络保存为.pb文件
2. 使用tf2onnx将.pb转化为onnx文件
3. tensorrt中读取onnx构建engine

## 使用trt的限制

直接使用B的限制条件，以下环境需要一致

1. 硬件环境必须一样，因为engine会根据硬件做优化，不能跨硬件平台
2. CUDA
3. cuDNN
4. tensorRT环境，不同版本tensorrt生成的engine不能够相互兼容
5. **同平台同环境**多次生成的engine**可能不同**

第4点：tensorrt runtime版本和engine版本不同时的报错信息：

![image-20230824150423797](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824150423797.png)

对于第5点，可以采用以下手段，见教程第四部分

![image-20230824150613993](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824150613993.png)

## ONNX介绍

官网：[ONNX | Home](https://onnx.ai/)

open neural network exchange

onnx是

1. 当前tensorrt导入模型的主要途径
2. TF/pytorch模型转tensorrt的中间表示
3. 用于存储训练好的模型

onnxruntime

1. 利用onnx格式模型进行推理计算的框架
2. 可以用于检查tensorflow和torch模型导出到onnx的正确性

# A. 解析onnx

已有的trt不适配，需要将onnx转为trt

1. parse onnx
2. serialize trt
3. 保存trt文件

注意：如果不使用Int8模式，onnx的parser代码几乎通用

## 概览

![image-20230824150915553](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824150915553.png)

构建阶段

1. 建立logger（日志）
2. 建立builder（网络元数据)
3. 创建network（计算图）（API独需）
4. 生成序列化的网络（网络的trt内部表示）

运行阶段

1. 建立engine（可执行代码）
2. 创建context（gpu进程）
3. buffer准备（host+device）
4. 拷贝host to device
5. 执行推理execute
6. 拷贝device to host
7. 善后

## A.1 构建阶段

### 1. 创建logger

记录器

`getTRTLogger();`

### 2. 创建builder

模型搭建的入口，网络的trt内部表示和引擎都是builder的成员方法生成的

builder.create_optimization_profile()：创建用于dynamic shape输入的配置器

`createInferBuilder()`



builder.create_network()：创建tensorrt网络对象

`createNetworkV2()`



在builderconfig下面进行细节设置

---

另外builder需要创建optimazation profile

在给定输入张量的最小最常见最大尺寸后，将设置的profile传给config

```cpp
auto profile = builder->createOptimizationProfile();
profile->setDimensions();
config->addOptimizationProfile(profile);
```



### 3. 设置builder config

进行设置网络属性

config=builder.create_builder_config()

```cpp
auto config = std::unique_ptr<nvinfer1::IBuilderConfig, samplesCommon::InferDeleter>(builder->createBuilderConfig());
```

1. 指定构建期可用显存
2. 设置标志位开关
3. 指定校正器
4. 添加用于dynamic shape输入的配置器

```cpp
config->addOptimizationProfile(profile);//添加用于dynamic shape输入的配置器
config->setFlag();
```

### 4. 搭建network

创建network（计算图）是API独需的因为其他两种方法使用parser从onnx导入，不用一层层添加

network=builder.create_network()

在onnx-parser中一旦模型parser解析完成，network就自动填好了，成为了serialized network

**onnx-parser解析**

```cpp
createParser(*network, sample::gLogger.getTRTLogger();

parser->parseFromFile(modelFile.c_str(), static_cast<int>(sample::gLogger.getReportableSeverity()));
```

## A.2 运行阶段 runtime

### 5. 生成TRT内部表示-serialized network

build_serialized_network(network,config)



### 6. 生成engine

推理引擎，可执行的代码段

生成engine：

```cpp
m_engine = std::unique_ptr<nvinfer1::ICudaEngine, samplesCommon::InferDeleter>(builder->buildEngineWithConfig(*network, *config), samplesCommon::InferDeleter());
```

### 7. 创建context

context即GPU进程

创建context：

python:`engine.create_execution_context()`

```cpp
 m_context = std::unique_ptr<nvinfer1::IExecutionContext, samplesCommon::InferDeleter>(m_engine->createExecutionContext(), samplesCommon::InferDeleter());
```

### 绑定输入输出

仅dynamic shape需要

### 8. 准备buffer

1. 内存和显存的分别申请
2. 拷贝
3. 释放

python:`cudart.cudaMalloc(inputHost.nbytes)[1]`

**课程第四部分会对buffer部分的优化做介绍**

![image-20230824114005386](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824114005386.png)



![image-20230824145433795](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824145433795.png)

### 9. 执行计算-execute

拷贝到cuda buffer上执行再拷贝回host，这一步一般是**B.解析trt**中做，但是读取onnx后也可以做

`m_context->enqueueV2()`

![image-20230824114125832](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824114125832.png)

### 10. 序列化引擎

`engine->serialize()`

### 11. 导出trt

## 特殊情况

遇到tensorrt不支持的onnx模型节点

1. 修改源模型
2. 修改onnx计算图，onnx-surgeon
3. tensorrt中实现plugin
4. 修改parser：修改源码，重新编译trt，因为tensorrt部分开源

# B. 解析trt

已有trt，直接导入然后使用

parse TRT后得到engine和context

### 1. 创建logger

`getTRTLogger()`

### 2. 创建cudaruntime

`createInferRuntime()`

### 3. 解析/反序列化trt文件，生成引擎

`runtime->deserializeCudaEngine()`

### 4. 创建context

`engine->createExecutionContext()`

### 5. 使用



# 背景知识

### CUDA异构计算

![image-20230824145343446](https://raw.githubusercontent.com/lanyily/Cplusplus-pic/main/2023/image-20230824145343446.png)

### Explicit Batch Implicit Batch

Explicit Batch为TensorRT主流网络构建方法，可以用dynamic shape

从onnx中导入的模型默认使用explicit batch模式

implicit batch单纯用于向旧版兼容

---

### Dynamic Shape模式

适用于输入张量形状在推理时才决定网络

除了batch维度，其他维度可以在推理时决定

**前提**

1. 需要explicit batch
2. 需要optimazation profile帮助网络优化
3. 需要context.set_input_shape绑定实际输入数据形状

### 一些建议

1. 使用最新的tensorrt8
2. 使用explicit batch模式，是onnx格式默认格式
3. 使用dynamic shape模式

