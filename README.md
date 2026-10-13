# ascend-c-study

## 目录结构

```
.
├── .gitattributes
├── .gitignore
├── README.md
└── ops/
    └── TanhCustom/
        ├── build.sh
        ├── CMakeLists.txt
        ├── CMakePresets.json
        ├── framework/
        ├── op_host/
        └── op_kernel/
```

## 环境要求

- 昇腾 NPU（本工程面向 910B 系列）
- CANN 9.x
- Python 3 + numpy
- cmake ≥ 3.16

## 算子说明

**TanhCustom** —— 自定义 Tanh 算子，采用 Tiling 模板编程。已在 Ascend 910B4 + CANN 9.2.0-beta.2 上完成端到端精度校验。

| 项 | 值 |
|---|---|
| 输入 `x` | `float16`，shape `(8, 2048)` |
| 输出 `y` | `float16`，shape `(8, 2048)` |
| 中间计算 | `float32` |
| 精度判据 | 相对误差 `1e-3`，最小比较阈值 `1e-3` |
| Tiling | `BLOCK_DIM=8`、`TILE_NUM=8`、`BUFFER_NUM=2` |

算法：`tanh(x) = (exp(x) - exp(-x)) / (exp(x) + exp(-x))`
