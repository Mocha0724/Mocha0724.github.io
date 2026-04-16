---
title: 【技术笔记】CuPy 库使用笔记 - GPU 加速计算
date: 2024-01-11 16:42:43
tags:
  - GPU
  - 技术笔记
  - CuPy
  - CUDA
  - NumPy
---
CuPy 是一个基于 CUDA 的 NumPy 替代品，可以在 NVIDIA GPU 上执行高性能的数值计算。由于其语法与NumPy高度一致，因此可以很简单地将原本NumPy的代码替换为CuPy

本文记录CuPy 的安装、配置和使用方法。

## 安装和配置

### 1. 前置要求

#### 硬件要求

- **NVIDIA GPU**

#### 软件要求

- **CUDA Toolkit**：10.2+ （推荐 11.x 或 12.x）
- **cuDNN**：7.0+ （可选，用于深度学习加速）

### 2. 检查 GPU 和 CUDA

**检查是否有 NVIDIA GPU：**

```bash
# Linux
nvidia-smi
```

**查找 CUDA 安装位置：**

```bash
# Linux
which nvcc
nvcc --version
```

### 3. 安装 CuPy

**使用 pip**

```bash
# 通过 pip 安装预编译版本（自动匹配 CUDA 版本）
pip install cupy-cuda11x

# 验证安装
python -c "import cupy; print(cupy.cuda.runtime.getDeviceCount())"
# 应该输出 GPU 数量
```

**使用 Conda**

```bash
# 使用 conda-forge 频道
conda install -c conda-forge cupy

# 或官方 cupy 频道
conda install -c cupy cupy

# 验证
python -c "import cupy as cp; print(cp.cuda.Device())"
```

### 4. 验证安装

创建 `test_cupy.py`：

```python
import cupy as cp
import numpy as np

# 检查 GPU
print(f"GPU 数量: {cp.cuda.runtime.getDeviceCount()}")
print(f"当前 GPU: {cp.cuda.get_device_id()}")
print(f"GPU 名称: {cp.cuda.Device().get_device_properties()['name'].decode()}")

# 检查显存
gpu_mem = cp.cuda.Device().mem_info
print(f"总显存: {gpu_mem[0] / 1e9:.2f} GB")
print(f"可用显存: {gpu_mem[1] / 1e9:.2f} GB")

# 简单计算测试
a_gpu = cp.array([1, 2, 3, 4, 5])
print(f"GPU 数组: {a_gpu}")
print(f"平方和: {cp.sum(a_gpu ** 2)}")

print("\n✓ CuPy 安装成功！")
```

---

## 基本使用方法

**核心思想就是把np.全部换成cp.基本就可以了**


### 多 GPU 使用

```python
import cupy as cp

# 获取 GPU 数量
num_gpus = cp.cuda.runtime.getDeviceCount()
print(f"可用 GPU 数: {num_gpus}")

# 在特定 GPU 上执行计算
for gpu_id in range(num_gpus):
    with cp.cuda.Device(gpu_id):
        arr = cp.random.rand(1000, 1000)
        result = cp.sum(arr)
        print(f"GPU {gpu_id}: sum = {result}")

# 数据在不同 GPU 间转移
arr_gpu0 = cp.array([1, 2, 3])  # 默认在 GPU 0
with cp.cuda.Device(1):
    arr_gpu1 = cp.asarray(arr_gpu0)  # 转移到 GPU 1
```

---

## 需要注意的问题

✅ **推荐做法：**

```python
# 1. 数据批量转移到 GPU
data_gpu = cp.asarray(data_cpu)

# 2. 在 GPU 上进行所有计算
result_gpu = apply_operations(data_gpu)

# 3. 最后才转回 CPU
result_cpu = cp.asnumpy(result_gpu)
```

❌ **不推荐做法：**

```python
# 频繁在 CPU 和 GPU 间转移
for i in range(1000):
    data_gpu = cp.asarray(data_cpu[i])          # ❌ 低效
    result = cp.asnumpy(cp.sum(data_gpu))
```

---

## 相关资源

- [CuPy 官方文档](https://docs.cupy.dev/)
- [CuPy GitHub 仓库](https://github.com/cupy/cupy)
- [CUDA C++ 编程指南](https://docs.nvidia.com/cuda/cuda-c-programming-guide/)
- [NVIDIA 开发者文档](https://developer.nvidia.com/cuda-toolkit)
