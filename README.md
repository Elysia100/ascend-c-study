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
├── .gitattributes           强制 LF 换行
├── .gitignore
├── README.md
└── TanhCustom/              算子工程
    ├── build.sh             编译入口
    ├── CMakePresets.json    CANN 路径等编译配置
    ├── framework/           TensorFlow 插件（脚手架自带）
    ├── op_host/             host 侧：Tiling 结构体、TilingFunc、OpDef
    └── op_kernel/           kernel 侧：核函数实现
```

`.gitattributes` 固定 `eol=lf`。若检出成 CRLF，Linux 下执行 `build.sh` 会报
`bad interpreter: /bin/bash^M`，这是换行符导致的问题，与代码无关。

## 环境要求

- 昇腾 NPU（本工程面向 910B 系列）
- CANN 9.x
- Python 3 + numpy
- cmake ≥ 3.16

## 编译与测试

```bash
source /opt/atomgit/Ascend/cann-9.2.0-beta.2/set_env.sh

# CMakePresets.json 的 ASCEND_CANN_PACKAGE_PATH 需指向同一路径。
# build.sh 走 cmake --preset=default，这是唯一配置入口，命令行无法覆盖。

cd TanhCustom
rm -rf build_out
bash build.sh

cd build_out
./custom_opp_ubuntu_aarch64.run

cd ../../AclNNInvocation
ASCEND_HOME_DIR=/opt/atomgit/Ascend/cann-9.2.0-beta.2 bash run.sh
```

改完配置必须 `rm -rf build_out`，否则 CMake 缓存里的旧路径继续生效。

## 算子说明

| 项 | 值 |
|---|---|
| 输入 `x` | `float16`，shape `(8, 2048)` |
| 输出 `y` | `float16`，shape `(8, 2048)` |
| 中间计算 | `float32` |
| 精度判据 | 相对误差 `1e-3`，最小比较阈值 `1e-3` |
| Tiling | `BLOCK_DIM=8`、`TILE_NUM=8`、`BUFFER_NUM=2` |

算法：`tanh(x) = (exp(x) - exp(-x)) / (exp(x) + exp(-x))`

## 两个关键设计点

**中间量用 fp32。** float16 只有 10 位尾数，直接计算 `e^x - e^-x` 会因有效位抵消产生误差；这种误差绝对值很小，能通过 `1e-3` 判据，却会让算子在真实模型里失效。因此先 `Cast` 到 fp32 全程计算，最后再收窄回 fp16。

**收窄用 CAST_RINT。** 即 IEEE754 就近舍入到偶数，与 numpy `astype(np.float16)` 行为一致——精度比对的真值正是 numpy 生成的，舍入方式不对齐会在无意义的地方产生误差。

## Tiling 参数推导

```
blockLength = totalLength / GetBlockNum()          = 16384 / 8 = 2048
tileLength  = blockLength / tileNum / BUFFER_NUM   = 2048 / 8 / 2 = 128
loopCount   = tileNum * BUFFER_NUM                 = 16
```

自洽性校验：`loopCount × tileLength = 16 × 128 = 2048 = blockLength`，每核循环 16 次正好覆盖其负责的 2048 个元素。

`tileLength` 要除以 `BUFFER_NUM`，因为 `Process()` 的循环次数是 `tileNum * BUFFER_NUM` 而非 `tileNum`；不除会导致单核搬运总量越界。
