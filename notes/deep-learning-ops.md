# 深度学习运维 (Deep Learning Ops)

## 📋 概述

深度学习运维是 MLOps 的核心组成部分，专注于深度学习模型的训练、部署、监控和全生命周期管理。

## 🎯 学习目标

- 理解深度学习系统架构
- 掌握 GPU 资源管理
- 配置分布式训练环境
- 实施模型部署和服务化
- 构建 MLOps 流水线

## 🏗️ 深度学习系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                    深度学习运维平台                          │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              数据层 (Data Layer)                     │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ 数据采集│  │ 数据标注│  │ 数据存储│            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              训练层 (Training Layer)                 │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ 单机训练│  │ 分布式  │  │ 超参调优│            │   │
│  │  │         │  │ 训练    │  │ AutoML  │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              部署层 (Serving Layer)                  │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ 模型转换│  │ 模型服务│  │ A/B 测试 │            │   │
│  │  │ 优化    │  │ 器      │  │         │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
│                          │                                  │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              监控层 (Monitoring Layer)               │   │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐            │   │
│  │  │ 性能    │  │ 数据    │  │ 模型    │            │   │
│  │  │ 监控    │  │ 漂移    │  │ 版本    │            │   │
│  │  └─────────┘  └─────────┘  └─────────┘            │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 📦 核心组件

### 1. 硬件资源

| 组件 | 说明 | 推荐配置 |
|------|------|----------|
| **GPU** | 训练加速 | NVIDIA A100/V100/RTX 3090 |
| **CPU** | 数据预处理 | AMD EPYC/Intel Xeon 32+ 核 |
| **内存** | 批量数据加载 | 256GB+ DDR4 |
| **存储** | 数据集存储 | NVMe SSD 2TB+ |
| **网络** | 分布式训练 | 100GbE/InfiniBand |

### 2. 软件栈

```
┌─────────────────────────────────────┐
│         应用层                       │
│  PyTorch / TensorFlow / JAX         │
├─────────────────────────────────────┤
│         框架层                       │
│  CUDA / cuDNN / NCCL                │
├─────────────────────────────────────┤
│         驱动层                       │
│  NVIDIA Driver                      │
├─────────────────────────────────────┤
│         硬件层                       │
│  GPU / CPU / NVLink                 │
└─────────────────────────────────────┘
```

## 🔧 GPU 资源管理

### 1. 驱动和 CUDA 安装

```bash
# 查看 GPU 信息
nvidia-smi
nvidia-smi -L

# 安装 NVIDIA 驱动
wget https://us.download.nvidia.com/XFree86/Linux-x86_64/470.57.02/NVIDIA-Linux-x86_64-470.57.02.run
sh NVIDIA-Linux-x86_64-470.57.02.run

# 安装 CUDA Toolkit
wget https://developer.download.nvidia.com/compute/cuda/11.4.2/local_installers/cuda_11.4.2_470.57.02_linux.run
sh cuda_11.4.2_470.57.02_linux.run

# 验证安装
nvcc --version
nvidia-smi
```

### 2. Docker GPU 支持

```bash
# 安装 NVIDIA Container Toolkit
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list > /etc/apt/sources.list.d/nvidia-docker.list

apt-get update
apt-get install -y nvidia-container-toolkit
systemctl restart docker

# 测试 GPU 容器
docker run --gpus all nvidia/cuda:11.4.2-base-ubuntu20.04 nvidia-smi
```

### 3. GPU 资源隔离

```bash
# 指定 GPU 运行
docker run --gpus '"device=0,1"' pytorch/pytorch:1.9.0-cuda11.1-cudnn8-runtime

# Kubernetes GPU 调度
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda-container
    image: nvidia/cuda:11.4.2-base
    resources:
      limits:
        nvidia.com/gpu: 2  # 请求 2 个 GPU
```

## 🚀 分布式训练

### 1. 数据并行

```python
# PyTorch DataParallel
import torch.nn as nn

model = MyModel()
model = nn.DataParallel(model, device_ids=[0, 1, 2, 3])
model.to('cuda:0')

# PyTorch DistributedDataParallel (推荐)
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel

dist.init_process_group(backend='nccl')
model = MyModel().to(local_rank)
model = DistributedDataParallel(model, device_ids=[local_rank])
```

### 2. 模型并行

```python
# 模型并行示例
class ModelParallel(nn.Module):
    def __init__(self):
        super().__init__()
        self.part1 = nn.Linear(10000, 5000).to('cuda:0')
        self.part2 = nn.Linear(5000, 1000).to('cuda:1')
    
    def forward(self, x):
        x = self.part1(x.to('cuda:0'))
        x = x.to('cuda:1')
        return self.part2(x)
```

### 3. 混合精度训练

```python
# PyTorch AMP
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for data, target in dataloader:
    optimizer.zero_grad()
    
    with autocast():
        output = model(data)
        loss = criterion(output, target)
    
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

## 📊 训练监控

### 1. TensorBoard

```python
from torch.utils.tensorboard import SummaryWriter

writer = SummaryWriter('runs/experiment_1')

for epoch in range(epochs):
    for batch, (data, target) in enumerate(dataloader):
        loss = train_step(data, target)
        writer.add_scalar('Loss/train', loss, epoch * len(dataloader) + batch)
    
    accuracy = validate()
    writer.add_scalar('Accuracy/valid', accuracy, epoch)

writer.close()

# 启动 TensorBoard
tensorboard --logdir=runs --port=6006
```

### 2. Weights & Biases

```python
import wandb

wandb.init(project="my-project", config={
    "learning_rate": 0.001,
    "batch_size": 32,
    "epochs": 100
})

for epoch in range(epochs):
    loss = train_step()
    wandb.log({"loss": loss, "epoch": epoch})

wandb.finish()
```

### 3. 训练指标

```yaml
# 关键监控指标
metrics:
  # 资源使用
  - gpu_utilization
  - gpu_memory_usage
  - cpu_utilization
  - memory_usage
  
  # 训练性能
  - samples_per_second
  - steps_per_second
  - loss_value
  - learning_rate
  
  # 模型质量
  - validation_loss
  - validation_accuracy
  - f1_score
```

## 🎯 模型部署

### 1. 模型导出

```python
# PyTorch 导出 ONNX
import torch

model = MyModel()
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    export_params=True,
    opset_version=11,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={'input': {0: 'batch_size'}, 'output': {0: 'batch_size'}}
)

# TensorRT 优化
import tensorrt as trt

def build_engine(onnx_path):
    logger = trt.Logger(trt.Logger.WARNING)
    builder = trt.Builder(logger)
    network = builder.create_network(1 << int(trt.NetworkDefinitionCreationFlag.EXPLICIT_BATCH))
    parser = trt.OnnxParser(network, logger)
    
    with open(onnx_path, 'rb') as f:
        parser.parse(f.read())
    
    config = builder.create_builder_config()
    config.max_workspace_size = 1 << 30  # 1GB
    engine = builder.build_engine(network, config)
    
    with open('model.engine', 'wb') as f:
        f.write(engine.serialize())
```

### 2. Triton Inference Server

```yaml
# config.pbtxt
name: "my_model"
platform: "onnxruntime_onnx"
max_batch_size: 8
input [
  {
    name: "input"
    data_type: TYPE_FP32
    dims: [3, 224, 224]
  }
]
output [
  {
    name: "output"
    data_type: TYPE_FP32
    dims: [1000]
  }
]

instance_group [
  {
    count: 2
    kind: KIND_GPU
    gpus: [0, 1]
  }
]
```

```bash
# 启动 Triton
docker run --gpus all --rm \
  -p 8000:8000 \
  -p 8001:8001 \
  -p 8002:8002 \
  -v $(pwd)/models:/models \
  nvcr.io/nvidia/tritonserver:21.09-py3 \
  tritonserver --model-repository=/models
```

### 3. 客户端调用

```python
import tritonclient.grpc as grpcclient

# 创建客户端
client = grpcclient.InferenceServerClient(url='localhost:8001')

# 准备输入
inputs = []
inputs.append(grpcclient.InferInput('input', [1, 3, 224, 224], 'FP32'))
inputs[0].set_data_from_numpy(input_data)

# 推理
outputs = []
outputs.append(grpcclient.InferRequestedOutput('output'))
response = client.infer('my_model', inputs, outputs=outputs)

result = response.as_numpy('output')
```

## 🔄 MLOps 流水线

### 1. Kubeflow Pipelines

```python
import kfp
from kfp import dsl

@dsl.component
def preprocess_data():
    # 数据预处理
    pass

@dsl.component
def train_model(epochs: int = 10):
    # 模型训练
    pass

@dsl.component
def evaluate_model():
    # 模型评估
    pass

@dsl.pipeline(name='ml-pipeline')
def pipeline(epochs: int = 10):
    preprocess_task = preprocess_data()
    train_task = train_model(epochs=epochs)
    evaluate_task = evaluate_model()

# 提交流水线
kfp.Client().create_run_from_pipeline_func(
    pipeline,
    arguments={'epochs': 10}
)
```

### 2. 模型版本管理

```yaml
# MLflow 模型注册
import mlflow

mlflow.set_tracking_uri("http://localhost:5000")
mlflow.set_experiment("my_experiment")

with mlflow.start_run():
    mlflow.log_param("learning_rate", 0.001)
    mlflow.log_metric("accuracy", 0.95)
    mlflow.pytorch.log_model(model, "model")
    
    # 注册模型
    model_uri = "runs:/<run_id>/model"
    mlflow.register_model(model_uri, "my_model")
```

### 3. 持续训练

```yaml
# GitHub Actions for ML
name: ML Pipeline

on:
  push:
    paths:
      - 'data/**'
      - 'src/**'

jobs:
  train:
    runs-on: gpu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Setup Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.9'
      
      - name: Install dependencies
        run: pip install -r requirements.txt
      
      - name: Train model
        run: python train.py
      
      - name: Evaluate model
        run: python evaluate.py
      
      - name: Deploy if improved
        if: steps.evaluate.outputs.accuracy > 0.95
        run: ./deploy.sh
```

## 📈 性能优化

### 1. 数据加载优化

```python
from torch.utils.data import DataLoader

# 优化数据加载
dataloader = DataLoader(
    dataset,
    batch_size=32,
    num_workers=8,           # 多进程加载
    pin_memory=True,         # 锁定内存加速传输
    prefetch_factor=2,       # 预取批次
    persistent_workers=True  # 持久化工作进程
)
```

### 2. 混合精度训练

```python
# 使用 FP16 加速
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for batch in dataloader:
    optimizer.zero_grad()
    
    with autocast():
        output = model(batch)
        loss = criterion(output, target)
    
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

### 3. 梯度累积

```python
# 模拟大批次训练
accumulation_steps = 4

for i, (data, target) in enumerate(dataloader):
    output = model(data)
    loss = criterion(output, target) / accumulation_steps
    loss.backward()
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

## 🔒 安全与合规

### 1. 数据隐私

```python
# 差分隐私
from opacus import PrivacyEngine

model = MyModel()
optimizer = torch.optim.SGD(model.parameters(), lr=0.1)

privacy_engine = PrivacyEngine()
model, optimizer, train_loader = privacy_engine.make_private(
    module=model,
    optimizer=optimizer,
    data_loader=train_loader,
    noise_multiplier=1.0,
    max_grad_norm=1.0,
)
```

### 2. 模型安全

```python
# 模型加密
from cryptography.fernet import Fernet

# 加密模型权重
key = Fernet.generate_key()
cipher = Fernet(key)

with open('model.pth', 'rb') as f:
    model_data = f.read()

encrypted = cipher.encrypt(model_data)

with open('model.enc', 'wb') as f:
    f.write(encrypted)
```

## 📚 参考资料

- [NVIDIA Deep Learning](https://developer.nvidia.com/deep-learning)
- [PyTorch 官方文档](https://pytorch.org/docs/)
- [TensorFlow 官方文档](https://www.tensorflow.org/)
- [MLOps 最佳实践](https://ml-ops.org/)

---

*学习笔记创建时间：2026-05-01 | ClawsOps ⚙️*
