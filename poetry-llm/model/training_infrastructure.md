---
type: Design
title: 학습 인프라 설계
description: Gemma 4 27B/31B 풀 파인튜닝 기준 분산 학습 설정, 비용 추정, 안정화 전략, 모니터링 지표.
tags: [model, training, infrastructure]
timestamp: 2026-06-28T00:00:00Z
---

# 학습 인프라 설계

## FSDP vs DeepSpeed ZeRO-3 선택 기준

두 프레임워크 모두 모델 파라미터·그래디언트·옵티마이저 상태를 GPU 간에 샤딩하여
단일 GPU VRAM 한계를 넘는 모델을 풀 파인튜닝할 수 있게 한다.
시 파인튜닝 맥락에서의 선택 기준은 다음과 같다.

### 비교표

| 항목 | FSDP (PyTorch 내장) | DeepSpeed ZeRO-3 |
|------|---------------------|------------------|
| 설치 복잡도 | 낮음 — PyTorch 2.x 기본 포함 | 높음 — 별도 패키지, CUDA 커스텀 커널 |
| HuggingFace 연동 | `Trainer` / `accelerate` 에서 바로 사용 | `deepspeed` config JSON 별도 필요 |
| 통신 효율 | NCCL AllGather 기반 — 안정적 | ZeRO Infinity 지원 (NVMe offload 가능) |
| GPU 2장 환경 | **추천** — 오버헤드 낮음 | 가능하나 설정 부담 대비 이득 적음 |
| GPU 8장 이상 | 가능하나 ZeRO보다 느릴 수 있음 | **추천** — 대규모 샤딩에 최적화 |
| CPU/NVMe offload | 지원 약함 | ZeRO-Infinity로 지원 |
| 디버깅 용이성 | PyTorch 표준 — 익숙함 | 별도 로그 파싱 필요 |

### 시 파인튜닝 권장 선택

```
Gemma 4 27B/31B + A100 80GB × 2장 → FSDP
  └── 이유: GPU 2장 규모에서 DeepSpeed 설정 복잡도 대비 이득이 없음
  └── HuggingFace accelerate + FSDP_AUTO_WRAP_POLICY로 충분
  └── 추후 GPU 4장 이상으로 확장 시 DeepSpeed ZeRO-3 재검토

만약 A100 80GB × 4장 이상 확보 시 → DeepSpeed ZeRO-3
  └── ZeRO Stage 3: 파라미터 + 그래디언트 + 옵티마이저 상태 모두 샤딩
  └── GPU당 메모리 요구량 ~1/N로 감소
```

### 분산 학습 설정 템플릿

#### 1. FSDP 설정 (Hugging Face Accelerate - `accelerate_config.yaml`)

```yaml
compute_environment: LOCAL_MACHINE
debug: false
distributed_type: FSDP
downcast_bf16: 'no'
fsdp_config:
  fsdp_auto_wrap_policy: TRANSFORMER_BASED_WRAP
  fsdp_backward_prefetch_policy: BACKWARD_PRE
  fsdp_cpu_ram_efficient_loading: true
  fsdp_forward_prefetch_limit: 2
  fsdp_offload_params: false          # VRAM 여유 시 false, OOM 발생 시 true (CPU Offload)
  fsdp_sharding_strategy: FULL_SHARD  # ZeRO-3와 동일한 파라미터/그래디언트/옵티마이저 샤딩
  fsdp_state_dict_type: SHARDED_STATE_DICT
  fsdp_transformer_layer_cls_to_wrap: GemmaDecoderLayer # Gemma 4 래핑 레이어 명시
machine_rank: 0
main_training_function: main
mixed_precision: bf16
num_machines: 1
num_processes: 2                       # A100 GPU 장수 (2장 기준)
rdzv_backend: static
same_network: true
use_cpu: false
```

#### 2. DeepSpeed ZeRO-3 설정 (`ds_config_zero3.json`)

DeepSpeed ZeRO-3는 매개변수, 그래디언트, 옵티마이저 상태를 모두 분할하여 대규모 GPU 클러스터에서 분산 학습 시 VRAM 오버헤드를 극적으로 제한한다.

```json
{
  "fp16": {
    "enabled": false
  },
  "bf16": {
    "enabled": true
  },
  "zero_optimization": {
    "stage": 3,
    "offload_optimizer": {
      "device": "none",
      "pin_memory": true
    },
    "offload_param": {
      "device": "none",
      "pin_memory": true
    },
    "overlap_comm": true,
    "contiguous_gradients": true,
    "sub_group_size": 1e9,
    "reduce_bucket_size": "auto",
    "stage3_prefetch_bucket_size": "auto",
    "stage3_param_persistence_threshold": "auto",
    "stage3_max_live_parameters": 1e9,
    "stage3_max_reuse_distance": 1e9,
    "stage3_gather_1d_tensor_gradient": true
  },
  "gradient_accumulation_steps": "auto",
  "gradient_clipping": "auto",
  "steps_per_print": 2000,
  "train_batch_size": "auto",
  "train_micro_batch_size_per_gpu": "auto",
  "wall_clock_breakdown": false
}
```

> [!TIP]
> GPU VRAM이 부족한 환경(예: 2x A100 80GB)에서 OOM이 발생할 경우, `"offload_optimizer"`의 `"device"`를 `"cpu"`로 변경하여 CPU Offload를 활성화할 수 있다.

---

## 백워드 프리페치(Backward Prefetch) 및 통신 중첩(Overlap) 비교

분산 학습 환경(FSDP 및 DeepSpeed ZeRO-3)에서 모델 파라미터가 GPU들에 분할(Sharding)되어 있으므로, 역전파(Backward Pass) 과정에서 각 레이어의 그래디언트를 계산하기 전에 해당 레이어의 파라미터를 다른 GPU들로부터 모으는(AllGather) 통신이 매번 발생한다. 이 통신 지연시간을 줄이기 위해 제공되는 핵심 기능이 **백워드 프리페치(Backward Prefetch)** 정책이다.

### 1. PyTorch FSDP 백워드 프리페치 정책

PyTorch FSDP는 `fsdp_backward_prefetch_policy`를 통해 다음 두 가지 모드를 제공한다:

*   **`BACKWARD_PRE` (Prefetch Before)**:
    *   **작동 방식**: 현재 레이어 $N$의 역전파 연산을 수행하는 동안, FSDP 엔진이 백그라운드 스트림에서 다음 연산 대상인 레이어 $N-1$의 파라미터를 모으는 AllGather 통신을 미리 시작한다.
    *   **통신 오버헤드**: 연산과 통신(AllGather)을 완전히 중첩(Overlap)시켜 GPU가 통신 완료를 기다리며 노는 대기 시간(Idle Time)을 대폭 줄인다.
    *   **VRAM 사용량**: 레이어 $N$의 언샤딩(Unsharded) 파라미터와 레이어 $N-1$의 프리페치된 언샤딩 파라미터가 VRAM에 동시에 존재해야 하므로, **피크 VRAM 사용량(Peak VRAM)이 증가**한다.
*   **`BACKWARD_POST` (Prefetch After)**:
    *   **작동 방식**: 현재 레이어 $N$의 연산이 끝난 직후 혹은 그래디언트 축소(ReduceScatter)가 시작될 때 레이어 $N-1$의 AllGather를 실행한다.
    *   **통신 오버헤드**: `BACKWARD_PRE`에 비해 연산과 통신의 중첩 범위가 좁아, 통신 지연이 학습 속도 병목으로 작용할 가능성이 크다.
    *   **VRAM 사용량**: 여러 레이어의 언샤딩 파라미터가 동시에 VRAM을 점유하는 기간이 짧아, 피크 VRAM 사용량을 억제할 수 있다.
*   **`None` (프리페치 비활성화)**:
    *   레이어 $N-1$의 연산 직전에 AllGather를 수행한다. VRAM 사용량은 최소화되지만, 통신 병목이 극대화되어 MFU가 심각하게 저하된다.

### 2. DeepSpeed ZeRO-3 프리페치 및 통신 중첩 정책

DeepSpeed는 `overlap_comm`과 여러 세부 버킷 설정을 조합하여 프리페치를 제어한다:

*   **`"overlap_comm": true`**:
    *   그래디언트 축소(ReduceScatter) 및 파라미터 프리페치(AllGather) 통신을 연산(Computation) 스트림과 동시에 병렬 실행한다. FSDP의 프리페치 메커니즘과 유사하게 통신 오버헤드를 대폭 완화한다.
*   **`stage3_prefetch_bucket_size`**:
    *   앞으로 필요한 파라미터를 미리 모을 때의 버킷 크기(파라미터 개수 단위)를 설정한다. 값이 클수록 프리페치 효율이 좋아지지만 VRAM 점유량이 증가한다.
*   **`stage3_max_live_parameters`**:
    *   VRAM 내에 복원(Unsharded)된 상태로 동시에 존재할 수 있는 최대 파라미터 수를 강제 제한한다. 프리페치가 과도하게 일어나 VRAM을 잠식하는 것을 막아 OOM을 방지하는 안전장치 역할을 한다.

### 3. FSDP vs DeepSpeed 비교 및 VRAM/통신 오버헤드 영향 요약

| 비교 항목 | PyTorch FSDP (`BACKWARD_PRE`) | DeepSpeed ZeRO-3 (`overlap_comm` + Prefetch) |
|---|---|---|
| **통신 중첩 방식** | 레이어 단위(Transformer Layer 블록) 프리페치 | 설정된 파라미터 버킷 크기 단위 비동기 프리페치 |
| **통신 오버헤드 감쇄** | 매우 우수 (대기 시간 최소화) | 매우 우수 (버킷 튜닝을 통해 통신 오버헤드 최적화 가능) |
| **VRAM 사용량** | 상대적으로 큼 (연속된 레이어들의 언샤딩 가중치가 상주) | 버킷 크기 및 `max_live_parameters` 제한으로 미세 조율 가능 |
| **2x A100 환경 권장** | **`BACKWARD_PRE` 적용 권장** (Offload 활성화로 부족한 VRAM을 충당하고 속도 저하를 최소화) | `"overlap_comm": true` 사용하되 버킷 크기를 `auto` 혹은 보수적으로 설정하여 OOM 회피 |

---

## bf16 Mixed Precision + Gradient Checkpointing 설정

### bf16 Mixed Precision

A100은 bf16 하드웨어 가속을 지원한다. fp16 대비 장점:
- 동적 범위가 fp32와 동일 → loss spike 위험 감소
- 별도 loss scaling 불필요 (fp16은 GradScaler 필요)

```python
# HuggingFace TrainingArguments
from transformers import TrainingArguments

args = TrainingArguments(
    bf16=True,
    bf16_full_eval=True,
    tf32=True,              # A100 Tensor Core 최대 활용
    # fp16은 사용하지 않음 — bf16으로 통일
)
```

**주의**: bf16은 Ampere(A100, A6000) 이상에서만 하드웨어 가속.
V100 등 구형 GPU라면 fp16 + GradScaler로 전환 필요.

### Gradient Checkpointing

32B 모델의 activation 메모리는 파라미터 메모리보다 클 수 있다.
Gradient checkpointing은 순전파 중 activation을 버리고 역전파 시 재계산하여
메모리를 ~60~70% 절감하는 대신 학습 시간이 ~30% 증가한다.

```python
model.gradient_checkpointing_enable(
    gradient_checkpointing_kwargs={"use_reentrant": False}
    # use_reentrant=False: FSDP와 호환성 높음, 최신 PyTorch 권장
)
```

### GPU 메모리 할당 분석 및 OOM 완화 전략

Gemma 4 27B/31B 모델(정밀도 bf16, 실제 파라미터 수 $\Phi \approx 27.0 \times 10^9$)의 풀 파인튜닝을 수행할 때, GPU 메모리는 크게 **모델 상태(Model States)**, **활성화(Activations)**, 그리고 **임시 버퍼(Temporary/Communication Buffers)**로 나뉜다.

#### 1. 모델 상태 메모리 수학적 분석

모델 상태는 파라미터, 그래디언트, 그리고 옵티마이저 상태(AdamW 기준)로 구성된다.

- **파라미터 ($\Phi$ - bf16)**: $2 \text{ bytes/parameter} \times 32.5\text{B} \approx 65.0\text{ GB}$
- **그래디언트 ($G$ - bf16)**: $2 \text{ bytes/parameter} \times 32.5\text{B} \approx 65.0\text{ GB}$
- **옵티마이저 상태 ($O$ - AdamW fp32)**: $12 \text{ bytes/parameter} \times 32.5\text{B} \approx 390.0\text{ GB}$  
  *(fp32 복사본 가중치 4 bytes + 모멘텀 4 bytes + 분산 4 bytes)*
- **8-bit AdamW 도입 시 옵티마이저 상태**: $6 \text{ bytes/parameter} \times 32.5\text{B} \approx 195.0\text{ GB}$  
  *(fp32 복사본 가중치 4 bytes + 8-bit 모멘텀 1 byte + 8-bit 분산 1 byte)*

#### 2. ZeRO 분산화 전략에 따른 GPU당 모델 상태 메모리 수식 ($N$ = GPU 개수)

- **ZeRO Stage 1 (Optimizer State Sharding)**:
  $$\text{Memory}_{\text{state}} = \Phi + G + \frac{O}{N} = 65.0 + 65.0 + \frac{390.0}{N}\text{ GB}$$
  - $N=8$인 경우 GPU당 $\approx 178.75\text{ GB}$ (A100 80GB에서 **OOM**)
- **ZeRO Stage 2 (Optimizer + Gradient Sharding)**:
  $$\text{Memory}_{\text{state}} = \Phi + \frac{G + O}{N} = 65.0 + \frac{65.0 + 390.0}{N}\text{ GB}$$
  - $N=8$인 경우 GPU당 $\approx 121.88\text{ GB}$ (A100 80GB에서 **OOM**)
- **ZeRO Stage 3 / FSDP Full Shard (Parameter + Gradient + Optimizer Sharding)**:
  $$\text{Memory}_{\text{state}} = \frac{\Phi + G + O}{N} = \frac{520.0}{N}\text{ GB}$$
  - $N=8$인 경우 GPU당 $\approx 65.0\text{ GB}$ (VRAM 여유 약 $15\text{ GB}$로 아슬아슬하게 학습 가능)
  - **8-bit AdamW 적용 및 ZeRO-3 ($N=8$)**:
    $$\text{Memory}_{\text{state}} = \frac{65.0 + 65.0 + 195.0}{8} = \frac{325.0}{8} \approx 40.63\text{ GB}$$
    (VRAM 여유 약 $39.37\text{ GB}$ 확보로 안전하게 학습 가능)

#### 3. 활성화 메모리(Activation Memory) 분석 및 OOM 임계점

활성화 메모리는 순전파 시 역전파를 위해 레이어의 입출력 및 중간 연산 결과를 VRAM에 유지하는 영역이다.

- **기본 활성화 메모리 추정 (시퀀스 길이 $L$, 배치 크기 $B$, 은닉 차원 $h$, 레이어 수 $a$)**:
  FlashAttention과 Activation Checkpointing이 없을 경우, 셀프 어텐션 행렬 연산으로 인해 시퀀스 길이 $L$에 대해 이차 연산 오버헤드($O(L^2)$)가 추가되어 기하급수적으로 메모리가 증가한다.
  - **OOM 임계점**: $B=1, L=4096$ 환경에서도 체크포인팅이 없으면 활성화 메모리가 $100\text{ GB}$를 초과하여 A100 80GB에서 즉시 OOM이 발생한다.

- **활성화 메모리 완화 전략**:
  1. **FlashAttention-2**: $O(L^2)$ 크기의 어텐션 행렬을 VRAM에 완전히 올리지 않고 SRAM 상에서 온라인 Softmax를 계산하여, 활성화 메모리 스케일링을 $O(L)$ 수준으로 대폭 축소한다.
  2. **Activation Checkpointing (선택적/전체 재계산)**: 모든 레이어의 중간 연산값을 들고 있는 대신 레이어 경계의 텐서만 보관하고 역전파 시 필요한 값만 재계산한다.
     - 전체 활성화 체크포인팅 적용 시 활성화 메모리 수식:
       $$\text{Memory}_{\text{activation}} \approx 2 \cdot B \cdot L \cdot h \cdot a + A_{\text{one\_layer\_activation}}$$
       Gemma 4 27B/31B ($h=4608, a=36$, $B=1, L=4096$) 기준 약 **$3.1\text{ GB}$** 내외로 대폭 감소한다.
  3. **ZeRO-Offload (CPU/NVMe Offload)**:
     - GPU VRAM이 부족한 극단적 환경(예: 2x A100 80GB)에서 옵티마이저 상태(390GB 또는 195GB)를 시스템 호스트 메모리(CPU DDR RAM)로 오프로드한다.
     - PCIe Gen4/Gen5를 통해 파라미터 업데이트 시에만 데이터를 통신하므로, 연산 속도는 대폭 저하(약 20%~40% 페널티)되지만 메모리 부족으로 인한 학습 실패를 완전히 방지할 수 있다.

```python
# 8-bit AdamW로 옵티마이저 메모리 절감 구현 예시
from bitsandbytes.optim import AdamW8bit

# optimizer 정의 시 bitsandbytes of 8-bit 구현체 사용
optimizer = AdamW8bit(model.parameters(), lr=1e-5, weight_decay=0.01)
```

#### 4. 2x A100 80GB 환경에서의 상세 메모리 요구량 및 CPU Offloading 분석

Gemma 4 27B/31B 모델의 파라미터 수 $\Phi \approx 27.0\text{B}$를 2x A100 80GB ($N=2$) 환경에서 풀 파인튜닝할 때의 상세 VRAM 요구량은 다음과 같다.

*   **가중치 및 그래디언트 (bf16)**:
    *   모델 파라미터 분할: $65.0\text{ GB} / 2 = 32.5\text{ GB/GPU}$
    *   그래디언트 분할: $65.0\text{ GB} / 2 = 32.5\text{ GB/GPU}$
*   **옵티마이저 상태 (AdamW)**:
    *   **Standard AdamW (fp32)**: $390.0\text{ GB} / 2 = 195.0\text{ GB/GPU}$
    *   **8-bit AdamW**: $195.0\text{ GB} / 2 = 97.5\text{ GB/GPU}$
*   **활성화 메모리 (FlashAttention-2 + Selective Activation Checkpointing)**:
    *   배치 크기 $B=1$ (GPU당), 시퀀스 길이 $L=4096$ 기준: 약 $3.7\text{ GB/GPU}$
*   **통신 버퍼 및 임시 메모리**:
    *   AllGather / ReduceScatter 프리페치 버퍼 등 추가 오버헤드: 약 $2.0 \sim 4.0\text{ GB/GPU}$

**종합 분석 (CPU Offload 없을 경우)**:
*   Standard AdamW 사용 시 필요한 GPU당 VRAM: $32.5 + 32.5 + 195.0 + 3.7 + 3.0 \approx 266.7\text{ GB}$ (A100 80GB 용량을 대폭 초과하여 **즉시 OOM**)
*   8-bit AdamW 사용 시 필요한 GPU당 VRAM: $32.5 + 32.5 + 97.5 + 3.7 + 3.0 \approx 169.2\text{ GB}$ (역시 80GB 용량을 초과하여 **OOM**)

**해결 방안: CPU Offloading 필수 적용**:
2x A100 80GB 환경에서 OOM 없이 학습을 완주하려면 **옵티마이저 상태 전체를 호스트 메모리(CPU RAM)로 오프로드**해야 한다.
*   **Optimizer State CPU Offload 적용 시 VRAM 요구량**:
    *   VRAM 점유 상태: 파라미터($32.5\text{ GB}$) + 그래디언트($32.5\text{ GB}$) + 활성화 메모리($3.7\text{ GB}$) + 통신 버퍼($3.0\text{ GB}$) $\approx 71.7\text{ GB/GPU}$
    *   여유 VRAM: 약 $8.3\text{ GB}$ (안정적인 학습 가능 범위 내 확보)
*   **호스트 RAM(CPU DDR5) 요구량**:
    *   Standard AdamW 사용 시: 최소 $390\text{ GB}$ 이상의 시스템 RAM 확보 필요 (시스템 기본 운영 메모리 고려 시 512GB RAM 추천)
    *   8-bit AdamW 사용 시: 최소 $195\text{ GB}$ 이상의 시스템 RAM 확보 필요 (256GB RAM 이상 노드에서 구동 가능)

#### 5. 2x A100 80GB 환경을 위한 메모리 프로파일링 지표 (Memory Profiling Metrics)

CPU Offload와 백워드 프리페치를 동시에 적용한 2x A100 80GB 극단적 환경에서는 메모리 누수와 미세한 단편화(Fragmentation)도 즉시 OOM으로 이어질 수 있다. 이를 감시하기 위한 필수 프로파일링 지표는 다음과 같다.

1.  **PyTorch 메모리 지표 (코드 레벨)**:
    *   `Allocated Memory` (`torch.cuda.memory_allocated()`): 현재 모델 상태 및 활성화 텐서가 점유 중인 실제 논리적 메모리 크기.
    *   `Max Allocated Memory` (`torch.cuda.max_memory_allocated()`): 학습 스텝(Forward/Backward) 전체를 통틀어 가장 높았던 피크 논리 메모리. 이 수치가 72GB를 초과하면 버퍼 공간 부족으로 위험하다.
    *   `Reserved Memory` (`torch.cuda.memory_reserved()`): PyTorch 캐싱 할당자(Caching Allocator)가 GPU로부터 대여해 확보해둔 물리 메모리 크기.
    *   `Max Reserved Memory` (`torch.cuda.max_memory_reserved()`): 캐싱 할당자의 피크 점유량. 이 값이 80GB에 도달했으나 `Max Allocated`가 훨씬 낮다면 메모리 단편화(Fragmentation)가 심하게 발생하고 있음을 뜻한다.
    *   `Active Bytes` / `Inactive Special Bytes` (PyTorch Profiler): `record_shapes=True`, `profile_memory=True` 옵션으로 수집하여 개별 레이어 연산(GemmaDecoderLayer 내 Self-Attention 등)에서 VRAM 급증을 추적.
2.  **하드웨어 및 시스템 지표 (인프라 레벨)**:
    *   `GPU VRAM 사용량` (`nvidia-smi` FB Memory): 물리적인 VRAM 총 점유량.
    *   `System Host Memory (RAM) 사용량`: CPU Offload된 옵티마이저 상태가 CPU 메모리를 초과하여 OS Swap을 유발하거나 Out-Of-Memory(OOM) Killer에 의해 프로세스가 강제 종료되는지 모니터링.
    *   `PCIe Read/Write Bandwidth` (DCGM or `nvidia-smi dmon`): CPU Offload 시 매 스텝마다 가중치와 옵티마이저 상태를 주고받는 PCIe 대역폭 점유율. 통신 병목으로 인해 GPU 활용률(GPU Utilization)이 저하되는 현상을 진단하는 핵심 지표.

---

## 클라우드 GPU 서버 및 학습 비용 추정 (Gemma 4 27B/31B 풀 파인튜닝)

Gemma 4 27B/31B 모델의 풀 파인튜닝 비용을 클라우드 인프라 제공업체(Lambda Labs, RunPod, 주요 CSP 등)의 단가와 하드웨어 스펙별 성능(MFU, 처리량)을 기반으로 추정한다.

### 1. GPU 서버 하드웨어 구성 및 단가 비교 (2026년 기준)

> 탐색중 — 클라우드 제공업체 및 GPU 구성 옵션은 시장 가격 변동에 따라 추가될 수 있음

| 노드 구성 | GPU 연결성 (Interconnect) | peak bf16 성능 (GPU당) | 시간당 예상 단가 (온디맨드) | 시간당 예상 단가 (스팟/예약) |
|---|---|---|---|---|
| **8x H100 (80GB) SXM5** | NVLink (900 GB/s) | 989 TFLOPs | $\$30.00 \sim \$45.00$ | $\$18.00 \sim \$25.00$ |
| **8x A100 (80GB) SXM4** | NVLink (600 GB/s) | 312 TFLOPs | $\$15.00 \sim \$22.00$ | $\$8.00 \sim \$12.00$ |
| **4x A100 (80GB) SXM4** | NVLink (600 GB/s) | 312 TFLOPs | $\$8.00 \sim \$11.00$ | $\$4.50 \sim \$6.50$ |
| **2x A100 (80GB) SXM4** | NVLink (600 GB/s) | 312 TFLOPs | $\$4.00 \sim \$5.50$ | $\$2.20 \sim \$3.50$ |
| **8x RTX 6000 Ada (48GB)** | PCIe Gen4 (64 GB/s) | 145 TFLOPs | $\$10.00 \sim \$14.00$ | $\$6.00 \sim \$9.00$ |

### 2. 학습 처리량 및 MFU(Model FLOPs Utilization) 분석

- **학습 연산량 공식**: 토큰당 연산량 $\approx 6 \times P$ (Forward 2회 + Backward 4회 연산).
  - Gemma 4 27B/31B ($P = 27.0 \times 10^9$) 기준: 토큰당 $\approx 1.62 \times 10^{11}$ FLOPs 요구.
- **실제 MFU 추정**:
  - **8x H100 SXM5 Node**: NVLink 대역폭이 넓고 분산 통신 병목이 적어 FlashAttention-2 + ZeRO-3 적용 시 **MFU 약 40%** 유지 가능.  
    $$\text{Tokens/sec/GPU} = \frac{989 \times 10^{12} \times 0.40}{1.95 \times 10^{11}} \approx 2,028 \text{ tokens/sec}$$
    8장 노드 총합 처리량: **$\approx 16,224$ tokens/sec**
  - **8x A100 SXM4 Node**: **MFU 약 42%** 달성 가능.  
    $$\text{Tokens/sec/GPU} = \frac{312 \times 10^{12} \times 0.42}{1.95 \times 10^{11}} \approx 672 \text{ tokens/sec}$$
    8장 노드 총합 처리량: **$\approx 5,376$ tokens/sec**
  - **4x A100 SXM4 Node**: **MFU 약 42%** 달성 가능. 4장 노드 총합 처리량: **$\approx 2,688$ tokens/sec**
  - **2x A100 SXM4 Node (CPU Offload 활성화)**: VRAM 제한 극복을 위해 optimizer offload를 사용하므로 PCIe 전송 병목이 추가되어 **MFU 약 30%**로 저하됨.  
    $$\text{Tokens/sec/GPU} = \frac{312 \times 10^{12} \times 0.30}{1.95 \times 10^{11}} \approx 480 \text{ tokens/sec}$$
    2장 노드 총합 처리량: **$\approx 960$ tokens/sec**

### 3. 데이터셋 규모별 학습 시간 및 비용 시뮬레이션

다양한 코퍼스 크기(50M, 100M, 500M tokens)와 학습 서버 설정에 따른 소요 시간 및 최종 비용을 산출한 표이다. (가정 단가: H100 8장 = $\$35.00/\text{hr}$, A100 8장 = $\$18.00/\text{hr}$, A100 4장 = $\$9.00/\text{hr}$, A100 2장(Offload) = $\$4.50/\text{hr}$)

| 노드 구성 | 지표 | 시나리오 A (50M Tokens) | 시나리오 B (100M Tokens) | 시나리오 C (500M Tokens) |
|---|---|---|---|---|
| **8x H100 SXM5** | 소요 시간 | 0.86 시간 | 1.71 시간 | 8.56 시간 |
| | **예상 비용** | **$\approx \$30.10$** | **$\approx \$59.85$** | **$\approx \$299.60$** |
| **8x A100 SXM4** | 소요 시간 | 2.58 시간 | 5.16 시간 | 25.81 시간 |
| | **예상 비용** | **$\approx \$46.44$** | **$\approx \$92.88$** | **$\approx \$464.58$** |
| **4x A100 SXM4** | 소요 시간 | 5.16 시간 | 10.32 시간 | 51.62 시간 |
| | **예상 비용** | **$\approx \$46.44$** | **$\approx \$92.88$** | **$\approx \$464.58$** |
| **2x A100 SXM4 (Offload)**| 소요 시간 | 14.47 시간 | 28.94 시간 | 144.68 시간 |
| | **예상 비용** | **$\approx \$65.12$** | **$\approx \$130.23$** | **$\approx \$651.06$** |

### 4. 비용 최적화 및 인프라 의사결정 전략

1. **학습 속도 vs 비용 균형**:
   - 4x A100 SXM4와 8x A100 SXM4는 시간당 비용 대비 처리량이 선형으로 증가하므로 총비용 차이가 크지 않다. 따라서 예산 범위 내에서 개발 속도 향상을 위해 **8x A100 SXM4 노드**를 대여하는 것이 작업 효율성 측면에서 권장된다.
   - 단기 실험이나 빠른 반복이 필요한 경우 **8x H100 SXM5**가 가장 탁월한 속도대비 가성비를 보여준다.
2. **2x A100 (Offload)의 비실용성**:
   - 2x A100 환경에서 오프로드를 사용하면 총학습 시간은 100M 토큰 기준 29시간에 달하며, 오버헤드로 인해 오히려 총 비용이 4x/8x A100보다 높아진다. 따라서 2장 환경에서의 풀 파인튜닝은 디버깅 목적으로만 제한하고, 실 학습은 4장 이상의 노드에서 수행하는 것이 타당하다.
3. **스팟 인스턴스 전략**: 스팟 계약을 통해 최대 50%의 추가 할인을 노리되, 체크포인트 저장 주기(`save_steps`)를 200스텝으로 타이트하게 설정하여 불시의 노드 회수에 대응한다.

---

## Loss Spike 방지 전략

Loss spike는 학습 중 loss가 갑자기 치솟는 현상으로,
대형 모델 파인튜닝에서 흔히 발생한다. 주요 원인: 큰 그래디언트, 비정상 데이터 배치.

### Gradient Clipping

```python
TrainingArguments(
    max_grad_norm=1.0,   # 표준값; 불안정 시 0.5로 낮춤
)
```

그래디언트 노름이 `max_grad_norm`을 초과하면 스케일다운.
32B 풀 파인튜닝에서는 1.0이 기본값이나, 초기 학습이 불안정하면 0.3~0.5로 조정.

### Warmup 설정

```python
TrainingArguments(
    warmup_ratio=0.03,        # 총 스텝의 3% — 절대값보다 비율 권장
    # warmup_steps=100,       # 피할 것: 총 스텝 수 모르면 의미 없음
    lr_scheduler_type="cosine",
)
```

**왜 비율이 중요한가**: 총 학습 스텝이 500이면 warmup 100스텝 = 20% (과도).
총 스텝이 10,000이면 100스텝 = 1% (부족). `warmup_ratio=0.03`이 일반적으로 안전.

### Learning Rate 선택

```
풀 파인튜닝 (32B): 5e-6 ~ 2e-5
  └── 너무 크면 catastrophic forgetting 위험
  └── 너무 작으면 시 도메인 적응 부족
  └── 권장 시작점: 1e-5

커리큘럼 Stage별 LR:
  Stage 1~2 (언어 기반): 1e-5
  Stage 3~4 (CoT/창작): 5e-6 (더 섬세한 조정 필요)
  Stage 5 (수정/반복): 3e-6
```

### 데이터 품질 사전 검증

spike의 숨은 원인 중 하나는 비정상 샘플(빈 텍스트, 이상 긴 시퀀스):

```python
# 전처리 시 토큰 길이 분포 확인
from collections import Counter
lengths = [len(tokenizer(text)["input_ids"]) for text in dataset]
# 최대 길이 outlier 제거: max_seq_length의 2배 초과 샘플 제거
```

---

## 체크포인트 전략

### 저장 주기

```python
TrainingArguments(
    save_strategy="steps",
    save_steps=200,             # 총 스텝의 약 2~5%마다 저장
    save_total_limit=5,         # 최신 5개만 보관 (디스크 절약)
    load_best_model_at_end=True,
    metric_for_best_model="eval_loss",
    greater_is_better=False,
)
```

### 디스크 용량 계산 (Gemma 4 27B/31B)

| 저장 형식 | 체크포인트 1개 크기 |
|-----------|-------------------|
| FSDP Sharded (bf16) | ~64 GB (샤드 파일 합산) |
| HuggingFace SafeTensors (bf16) | ~64 GB |
| LoRA 어댑터만 | ~500 MB~1 GB |

```
체크포인트 5개 보관 기준:
  풀 파인튜닝: 64 GB × 5 = ~320 GB 필요
  → NVMe SSD 500 GB 이상 확보 권장

커리큘럼 학습 (Stage 5개 × 최종 체크포인트 1개):
  64 GB × 5 = ~320 GB 추가
  → 총 ~640 GB (Stage 중간 저장 포함)
```

### 체크포인트 보관 정책

```
학습 중: save_total_limit=5 (롤링 보관)
Stage 완료 시: 해당 Stage 최종 체크포인트는 별도 보관 (삭제 금지)
  └── 이유: 커리큘럼 Stage 간 비교 실험 및 롤백을 위해
최종 모델: SafeTensors 포맷으로 별도 변환 후 보관
```

---

## 학습 모니터링 지표

### 필수 지표 (학습이 잘 되고 있는지 판단)

| 지표 | 정상 범위 | 이상 신호 |
|------|-----------|-----------|
| **train/loss** | 초기 대비 꾸준히 감소 | spike 또는 발산 |
| **eval/loss** | train/loss와 유사한 추세 | train < eval 격차 벌어짐 = 과적합 |
| **grad_norm** | 대부분 1.0 이하 | 지속적으로 1.0 클리핑 = LR 너무 큼 |
| **learning_rate** | warmup 후 cosine 감소 | 변동 없으면 scheduler 오류 |
| **GPU 메모리 사용률** | 70~90% | OOM 직전 신호: 95% 이상 |
| **처리량 (tokens/sec)** | 안정적 유지 | 갑자기 감소 = 통신 병목 또는 OOM |

### 시 도메인 특화 지표

```
매 500~1000 스텝마다 다음 프롬프트로 샘플 생성 후 수동 확인:
  1. "봄 / 비 / 어머니를 소재로 시를 써라"
  2. "다음 소재로 시를 써라: 새벽 지하철 첫 차"

확인 항목:
  - 행갈이(<행갈이>) 토큰이 자연스러운 위치에 나타나는가?
  - 시가 완결되는가 (중간에 끊기지 않는가)?
  - 반복 루프에 빠지는가? (생성 품질 저하 초기 신호)
  - 언어가 시적 감각을 유지하는가? (산문화 여부 확인)
```

### 권장 모니터링 도구

```
WandB (Weights & Biases):
  - loss curve, grad_norm, LR 자동 시각화
  - 생성 샘플 텍스트 로깅 가능
  - 무료 플랜으로 개인 프로젝트 충분

TensorBoard (대안):
  - 별도 서버 설정 필요, 기능은 WandB보다 제한적

설정:
  TrainingArguments(report_to="wandb", run_name="okf-poetry-stage1")
```

### 조기 종료 기준

```
eval_loss가 N 평가 주기 연속 개선되지 않으면 중단:
  early_stopping_patience=5  (500 스텝 × 5 = 2500 스텝 개선 없으면 중단)

추가 기준:
  - 생성 샘플에서 반복 루프 3회 이상 발생 → 즉시 학습 중단 검토
  - grad_norm이 10 이상으로 지속 → LR 낮추고 재시작
```

### 다중 노드 통신 효율 및 CPU Offload 병목 분석 (결정 사항)

**1. 다중 노드(2개 이상) 분산 학습 시 통신 병목 및 프레임워크 선택**
*   **대역폭 병목 이론치**: 단일 노드 내(NVLink) 통신은 600~900 GB/s에 달하나, 노드 간 통신(InfiniBand/RoCE)은 통상 100~400 Gbps (12.5~50 GB/s)로 약 10~50배 느리다.
*   **FSDP (`BACKWARD_PRE`)**: 레이어 단위 통신으로 노드 간 네트워크에 일시적인 트래픽 버스트(Micro-burst)를 유발하여 InfiniBand 혼잡(Congestion)이 발생할 수 있다. 통신-연산 중첩이 완벽하지 않을 경우 스텝 타임이 지연된다.
*   **DeepSpeed ZeRO-3**: `overlap_comm=true` 및 `stage3_prefetch_bucket_size` 조율을 통해 대역폭 사용을 평탄화(Communication Shaping)할 수 있어 다중 노드 환경에서 FSDP보다 통신 병목 완화에 유리하다.
*   **설계 결정**: 단일 노드(GPU 8장 이하)에서는 FSDP를 우선 적용하나, 2개 노드(GPU 16장) 이상으로 확장 시 DeepSpeed ZeRO-3로 마이그레이션하여 버킷 크기를 노드 간 대역폭에 맞춰 튜닝한다.

**2. CPU Offload 및 PCIe Gen4 병목에 따른 MFU 예측치**
*   **이론적 한계**: PCIe Gen4 x16의 실효 대역폭은 약 26 GB/s(단방향)이다. Gemma 4 27B/31B의 옵티마이저 상태(162GB, 8-bit 기준)를 매 스텝마다 CPU와 GPU 간에 전송해야 하므로 통신 오버헤드가 극심하다.
*   **MFU 감소율 예측**: NVLink 기반 A100 환경의 기준 MFU(약 42%) 대비, PCIe 통신 병목으로 인해 MFU가 약 30% 수준으로 하락(절대치 12%p 감소, 상대비율 약 28.5% 저하)할 것으로 예측된다.
*   **오버헤드 완화 대책 (Gradient Accumulation Tuning)**:
    *   통신 횟수를 줄이기 위해 마이크로 배치 크기는 1로 유지하되, 그래디언트 누적 스텝(Gradient Accumulation Steps, $K$)을 4~8 이상으로 증가시킨다.
    *   **효과 모델링**: 오프로드 통신(가중치 갱신 및 옵티마이저 스텝)은 $K$ 스텝마다 1회만 발생한다. 따라서 $K$ 스텝 동안의 평균 스텝 시간 $T_{\text{total}} \approx K \times T_{\text{compute}} + T_{\text{offload}}$가 되어, $K$가 커질수록 통신 오버헤드 비율 $\frac{T_{\text{offload}}}{K \times T_{\text{compute}}}$가 반비례하여 감소한다.
    *   이 방식을 통해 MFU를 부분적으로 복구할 수 있으나, 글로벌 배치 크기 증가로 인해 수렴 에포크 수가 늘어날 수 있으므로 학습 곡선 모니터링이 필수적이다.

---

## 미결 사항

- [User Review] **FSDP와 DeepSpeed ZeRO-3 간의 2개 노드 이상 분산 환경에서의 상호 대역폭 병목 및 통신 효율 측정**: 단일 노드(GPU 8장) 대비 다중 노드로 확장 시, FSDP의 `BACKWARD_PRE` 프리페치에 따른 노드 간 AllGather 통신 버스트와 DeepSpeed의 `overlap_comm` 및 `stage3_prefetch_bucket_size` 조율을 통한 통신 셰이핑(Communication Shaping)의 상호 대역폭 병목 및 VRAM 할당 변화 추이 검증 필요.
- [Ph1] **8-bit AdamW 사용 시의 가중치 정밀도 저하가 시적 novelty 및 수사적 특이성 학습에 미치는 장기적 영향 검증**: 32B 모델의 풀 파인튜닝에서 8-bit optimizer가 fp32 master weights와 8-bit gradient를 조합할 때, 미세한 가중치 업데이트 손실이 한국어 고유의 미학적 시 작법 학습에 부정적 영향을 미치는지에 대한 실험적 평가 필요.
- [User Review] **CPU Offload 활성화 시 PCIe Gen4 대역폭 병목으로 인한 MFU(Model FLOPs Utilization) 감소율 예측 및 A100 NVLink 미탑재 환경에서의 오버헤드 완화 대책**: 4x A100 PCIe 환경에서 optimizer offload 사용 시, NVLink SXM4 대비 PCIe Gen4 대역폭 한계로 인한 학습 속도 저하(MFU 30% 이하 예상)를 극복하기 위한 커스텀 gradient accumulation tuning의 타당성.
  - *Sub-question*: MFU 손실을 상쇄하기 위해 글로벌 배치 크기를 고정하고 그래디언트 누적(Gradient Accumulation) 단계를 늘릴 경우, 역전파 과정에서 통신 오버헤드를 얼마나 줄일 수 있으며 이에 따른 수렴 속도 지연은 수치적으로 어떻게 모델링할 수 있는가?
