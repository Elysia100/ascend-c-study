# ascend-c-study

昇腾 Ascend C 算子开发学习记录。

## 已完成

**TanhCustom** —— 自定义 Tanh 算子，采用 Tiling 模板编程，已在 **Ascend 910B4 + CANN 9.2.0-beta.2** 上完成端到端精度校验。

```
INFO: you have passed the Precision!
```

## 目录结构

```
.
├── TanhCustom/              算子工程
│   ├── build.sh             编译入口
│   ├── CMakePresets.json    CANN 路径等编译配置
│   ├── framework/           TensorFlow 插件（脚手架自带）
│   ├── op_host/             host 侧：Tiling 结构体、TilingFunc、OpDef
│   └── op_kernel/           kernel 侧：核函数实现
└── docs/
    ├── 01-环境搭建与踩坑.md
    └── 02-算子设计说明.md
```

## 环境要求

- 昇腾 NPU（本工程面向 910B 系列）
- CANN 9.x
- Python 3 + numpy
- cmake ≥ 3.16

## 编译与测试

```bash
# 1. 配置环境变量（路径按实际安装位置修改）
source /opt/atomgit/Ascend/cann-9.2.0-beta.2/set_env.sh

# 2. 修改 CMakePresets.json 中的 ASCEND_CANN_PACKAGE_PATH 指向同一路径
#    build.sh 走 cmake --preset=default，这是唯一配置入口，命令行无法覆盖

# 3. 编译
cd TanhCustom
rm -rf build_out
bash build.sh

# 4. 安装算子包（不要加 --install）
cd build_out
./custom_opp_ubuntu_aarch64.run

# 5. 精度测试（AclNNInvocation 为测试工程，需与 TanhCustom 同级）
cd ../../AclNNInvocation
ASCEND_HOME_DIR=/opt/atomgit/Ascend/cann-9.2.0-beta.2 bash run.sh
```

## 算子说明

| 项 | 值 |
|---|---|
| 输入 `x` | `float16`，shape `(8, 2048)` |
| 输出 `y` | `float16`，shape `(8, 2048)` |
| 中间计算 | `float32` |
| 精度判据 | 相对误差 `1e-3`，最小比较阈值 `1e-3` |
| Tiling | `BLOCK_DIM=8`、`TILE_NUM=8`、`BUFFER_NUM=2` |

算法：`tanh(x) = (exp(x) - exp(-x)) / (exp(x) + exp(-x))`
