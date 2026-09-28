# architecture_overview

FourCastNet 是二维 patch 预测模型，核心思想是在 patch 网格上做频域混合，再恢复为完整气象场。
输入不包含压力层维度，主干不是注意力 Transformer，而是 AFNO 频域混合加逐位置前馈网络。关键调用约定是 embedding 输出展平 token 序列，进入 trunk 前必须恢复为二维 patch 网格。

# parameter_scale

- 默认输入尺寸 `(720, 1440)`，patch `(8, 8)`，patch 网格为 `90 x 180`。
- 默认 `in_chans=19`，`out_chans=19`，`embed_dim=768`，`depth=12`。
- 默认 AFNO 分块数 `num_blocks=8`。
- 参数主要集中在 patch embedding、12 层 fuser 与线性恢复头。

# architecture_structure

```text
输入通道组织
  x: (Batch, Channels, Height, Width)
    默认 Channels=19
    默认 Height=720, Width=1440

二维 patch embedding
  x
    -> OneEmbedding(style="FourCastNetEmbedding", patch_size=(8, 8), embed_dim=768)
    -> (Batch, 16200, 768)

位置编码与网格恢复
  patch tokens
    -> + pos_embed: (1, 16200, 768)
    -> dropout
    -> reshape
    -> (Batch, 90, 180, 768)

AFNO patch-grid trunk
  (Batch, 90, 180, 768)
    -> OneFuser(style="FourCastNetFuser") block 1
       内部: FourCastNetAFNO2D + FourCastNetFC
    -> FourCastNetFuser block 2
    -> ...
    -> FourCastNetFuser block 12
    -> (Batch, 90, 180, 768)

patch 输出头与恢复
  trunk 输出
    -> linear head
    -> (Batch, 90, 180, out_chans * 8 * 8)
    -> einops.rearrange
    -> (Batch, out_chans, 720, 1440)
```

# input_schema

- `x`: `(Batch, in_chans, Height, Width)`。
- 默认 `Height=720`，`Width=1440`。
- `Height` 和 `Width` 应能被 patch 尺寸整除，以避免 token 与恢复网格不一致。

# output_schema

- 输出：`(Batch, out_chans, Height, Width)`。
- 输出网格由 patch 恢复得到，不额外插值。

# shape_transformations

1. 输入 `(B,C,H,W)`。
2. patch embedding 得到 `(B, num_patches, embed_dim)`。
3. 加位置编码并 dropout。
4. reshape 为 `(B, H/ph, W/pw, embed_dim)`。
5. 逐层 fuser 保持 patch 网格尺寸。
6. 线性头输出 `(B, H/ph, W/pw, ph*pw*out_chans)`。
7. 重排为 `(B,out_chans,H,W)`。

# key_dependencies

- `fourcastnetembedding`
- `fourcastnetfuser`
- `fourcastnetafno`
- `fourcastnetfc`

# common_modification_points

- 修改 `in_chans/out_chans` 支持不同变量集合。
- 修改 `patch_size` 在分辨率、速度和局地细节之间折中。
- 调整 `depth`、`embed_dim`、`mlp_ratio` 改变模型容量。
- 调整 `num_blocks`、`sparsity_threshold`、`hard_thresholding_fraction` 影响频域混合行为。
- 可借鉴 Fuxi 添加时间 patch 维度，支持多历史步输入。

# implementation_risks

- 位置编码长度与 patch 数必须一致，变更分辨率需重建或插值位置编码。
- `rearrange` 假设 patch 网格严格匹配输入尺寸。
- AFNO 稀疏阈值过高可能损失小尺度天气信号。
- 当前模型不显式建模压力层或变量族结构。
- `embed_dim` 必须能被 AFNO 的 `num_blocks` 整除。

# code_references

- `{onescience_path}/onescience/src/onescience/models/fourcastnet/fourcastnet.py`
- `{onescience_path}/onescience/src/onescience/modules/fuser/fourcastnetfuser.py`
- `{onescience_path}/onescience/src/onescience/modules/afno/fourcastnetafno.py`
- `{onescience_path}/onescience/src/onescience/modules/embedding/fourcastnetembedding.py`
- `{onescience_path}/onescience/src/onescience/modules/fc/fourcastnetfc.py`

# resource_acquisition

区域化移植 FourCastNet 需要**预训练权重 + 再分析数据 + 运行库**，三者均不随场景包提供。
orchestrator 应按本契约预检并主动获取，取不到再降级/BLOCKED，不得停在方案阶段。

```yaml
resource_acquisition:
  - dep: fourcastnet-weights                 # FourCastNet v2 (SFNO) 预训练权重
    kind: model_weights
    source:
      - {provider: hf-mirror,   id: OneScience-Group/FourCastNet_v2, endpoint: "https://hf-mirror.com"}
      - {provider: huggingface, id: OneScience-Group/FourCastNet_v2}
      - {provider: ngc,         id: nvidia/modulus/modulus_fcnv2_sm, note: "NGC Catalog 官方 checkpoint 包"}
      - {provider: huggingface, id: nvidia/fourcastnet3, note: "如需 v3 概率预报版"}
    command: |
      python3 -m pip install -U "huggingface_hub[cli]"
      export HF_ENDPOINT=https://hf-mirror.com
      hf download OneScience-Group/FourCastNet_v2 --local-dir ./fcn_weights
      # 或经 earth2studio：python3 -m pip install earth2studio，其 px.SFNO 会从 NGC 拉取
    sha256: 待核验
    on_missing: fetch
    degrade_to: "无权重时用小分辨率随机初始化跑通前向流程冒烟，声明未取得预训练权重、非预报精度"

  - dep: era5-reanalysis                     # 30 年高精度再分析数据（迁移训练/微调）
    kind: dataset
    source:
      - {provider: cds, id: "ERA5 (Copernicus CDS)", note: "需 API token；0.25° 等距柱状网格"}
      - {provider: nasa, id: "MERRA-2 / GES DISC", note: "替代再分析源"}
    command: |
      python3 -m pip install cdsapi
      # 配置 ~/.cdsapirc 后按变量/时段检索下载；大体积建议分年分块
    on_missing: fetch
    degrade_to: "无 token/无网络：用样本子集或合成场做数据管线冒烟，声明未取得全量再分析"

  - dep: modulus / earth2studio + torch      # 运行库
    kind: tool
    source:
      - {provider: pypi, id: earth2studio}
    command: "python3 -m pip install torch numpy earth2studio"
    on_missing: fetch
    verify: "python3 -c 'import torch; print(torch.__version__, torch.cuda.is_available())'"

  - dep: GPU 集群（曙光）                     # 迁移训练/微调算力
    kind: compute
    source:
      - {provider: hpc, id: "目标曙光 GPU 集群", note: "训练/微调需 GPU；提交前按分区与配额预检"}
    on_missing: block
    degrade_to: "CPU-only 环境不可行全量迁移训练；仅能做小样本前向/推理冒烟，须显式声明算力受限"
```

**要点**：区域化移植涉及开放边界约束与位置编码分辨率匹配（见 implementation_risks），
下载权重后先核对 patch 网格与目标区域分辨率，再进入迁移训练；GPU 缺失时按 `degrade_to` 限定为冒烟。

