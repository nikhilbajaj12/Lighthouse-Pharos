# Lighthouse Attention: Hierarchical Selection for Long-Context LLMs

## The Problem
Transformer attention scales quadratically with sequence length (O(n²)). At 98K tokens, full attention becomes prohibitively expensive.

## The Solution
**Lighthouse Attention** = hierarchical token selection via multi-level pooling and top-K scoring.

---

## How It Works

### 1. **Build a Pyramid**
Pool the sequence into hierarchical levels (coarser → finer):
- **Level 1 (coarsest):** Tokens grouped by factor `p` (e.g., p=4 means 4 tokens per pool)
- **Level 2:** Tokens grouped by p²
- **Level L (finest):** Full resolution

### 2. **Score Each Position**
Two lightweight "scorers" compute importance:
- **QK score**: How much a query wants a key
- **KQ score**: How much a key wants a query

Available scorers: `norm` (default), `dilated`, `gla`

### 3. **Select Top-K Hierarchically**
Starting from the coarsest level, "unpool" winners to the next finer level:
```
Coarse Level:  [Select top-K=64]
                 ↓↓↓↓ (unpool by p)
Fine Level:    [Select top-K=512 from expanded children]
                 ↓↓↓↓ (repeat to finest)
Final Level:   [Select top-K=2048 actual tokens]
```

### 4. **Attend Only to Selected Tokens**
Standard attention on the sparse selected set (e.g., 2048/98K = 2%)
- **Complexity:** O(n · k) instead of O(n²)
- **~50× speedup** while maintaining quality

### 5. **Scatter Results Back**
Fan-out attention outputs to original sequence positions.

---

## Key Parameters

| Parameter | Default | Role | Examples |
|-----------|---------|------|----------|
| `topk` | 2048 | Final number of selected tokens | 1536, 2048, 3072, 4096, 6144 |
| `pooling_factor` (p) | 4 | Grouping ratio per level | 2, 4, 8 |
| `num_levels` (L) | 3 | Number of pyramid levels | 3, 4, 5 |
| `scorer` | norm | Scoring function | norm, dilated, gla |
| `seq_len` | 98304 | Training context length | - |

---

## Architecture

```
torchtitan (PyTorch distributed training framework)
    ↓ (patch + 2 source files)
    ├── TransformerModelArgs (new params: topk, pooling_factor, etc.)
    ├── lighthouse_selection.py (Triton GPU kernel)
    ├── lighthouse_selection_cuda.py (CUDA NVRTC kernel)
    └── Scorer dispatch (norm | dilated | gla)
```

---

## Quick Start

### Setup
```bash
git clone https://github.com/pytorch/torchtitan.git && cd torchtitan
git checkout 61c25f8d
cp /path/to/lighthouse-attention/src/*.py torchtitan/models/llama3/model/
git apply /path/to/lighthouse-attention/lighthouse-attention.patch
pip install -r /path/to/lighthouse-attention/requirements.txt
```

### Train
```bash
sed 's|<DUMP_FOLDER>|/scratch/runs|; s|<HF_ASSETS_PATH>|/path/to/tokenizer|' \
    configs/topk/topk2048.toml > /tmp/run.toml
torchrun --nproc-per-node 8 ./torchtitan/train.py --job.config_file /tmp/run.toml
```

---

## Requirements

- **Hardware:** 8× NVIDIA GPU (B200, H100, A100; sm_80+)
- **CUDA:** 12.8
- **Python:** 3.13
- **Torch:** 2.11.0+cu128
- **Dependencies:** Triton, einops, flash-linear-attention (for GLA scorer)

---

## Validation

- **Training:** Logs loss curves to WandB
- **Configs provided:** Ablations over (k, p, L) grid
- **Stage 1 (Lighthouse):** 10K steps at sparse attention
- **Stage 2 (SDPA-resume):** 6K steps at dense attention (dim-matched baseline)
- **GPU:** Tested on NVIDIA B200 (sm_100)

---

## Paper & Citation

**"Long Context Pre-Training with Lighthouse Attention"**  
[arXiv:2605.06554](https://arxiv.org/pdf/2605.06554v1)

Original implementation. All results in paper produced by this codebase.

---

## Key Insight

> Attention doesn't need to be dense to be effective. By hierarchically filtering tokens, Lighthouse reduces complexity from O(n²) to O(n·k) with minimal quality loss, enabling LLMs to train efficiently at extreme context lengths (98K+ tokens).
