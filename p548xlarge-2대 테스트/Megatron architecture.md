
## 아키텍처
```
┌─────────────────────────────────────────────────────────────────────┐
│                         HEAD NODE                                    │
├─────────────────────────────────────────────────────────────────────┤
│  HOST (AMI)                                                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ ❌ CUDA Driver (GPU 없음)                                   │    │
│  │ ✅ EFA 1.26.1 (설치됨, 사용 안 함)                          │    │
│  │ ✅ Docker (이미지 빌드용)                                   │    │
│  │ ✅ Enroot (이미지 변환용)                                   │    │
│  │ ✅ Slurm Controller (slurmctld)                            │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  Instance: m5.8xlarge                                                │
│  Network: 10 Gbps                                                    │
└─────────────────────────────────────────────────────────────────────┘

                              ↓ sbatch
                              ↓ srun

┌─────────────────────────────────────────────────────────────────────┐
│                      COMPUTE NODE 1 (p5.48xlarge)                    │
├─────────────────────────────────────────────────────────────────────┤
│  HOST (AMI)                                                          │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ ✅ CUDA Driver 535.54.03+ (8 x H100 GPU 제어)             │    │
│  │ ✅ EFA 1.26.1 (32 x 100 Gbps = 3200 Gbps)                 │    │
│  │ ✅ Docker + Enroot                                         │    │
│  │ ✅ Slurm Daemon (slurmd)                                   │    │
│  │ ✅ /scratch (8 x 3.84TB NVMe = 30.72TB)                   │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  CONTAINER (Enroot/Pyxis로 실행)                                    │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  ✅ CUDA Toolkit 12.2                                      │    │
│  │  ✅ NCCL 2.18.5-1+cuda12.2                                 │    │
│  │  ✅ AWS OFI NCCL v1.8.1-aws (EFA 플러그인)                 │    │
│  │  ✅ EFA libfabric (사용자 공간)                            │    │
│  │  ✅ PyTorch + Megatron-LM                                  │    │
│  │                                                             │    │
│  │  프로세스: Rank 0-7 (각 GPU에 하나씩)                      │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GPU Topology (NVSwitch 기반):                                      │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  GPU 0 ◄─┐                                           │          │
│  │  GPU 1   │                                           │          │
│  │  GPU 2   │  NVLink 4.0 (900 GB/s bidirectional)     │          │
│  │  GPU 3   │  All-to-All 연결 (NVSwitch)              │          │
│  │  GPU 4   │                                           │          │
│  │  GPU 5   │                                           │          │
│  │  GPU 6   │                                           │          │
│  │  GPU 7 ◄─┘                                           │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
│  Network: 32 x EFA (100 Gbps each)                                  │
│  ├─ EFA 0-31 ◄──────────┐                                          │
│  │                       │                                          │
│  │  Total: 3200 Gbps     │ GPUDirect RDMA                          │
│  │        (400 GB/s)     │ (GPU ↔ EFA 직접 통신)                   │
│  └───────────────────────┘                                          │
└──────────────────────────┼──────────────────────────────────────────┘
                           │
                           │ EFA Network
                           │ 3200 Gbps (400 GB/s)
                           │ 노드 간 고속 통신
                           │
┌──────────────────────────┼──────────────────────────────────────────┐
│                      COMPUTE NODE 2 (p5.48xlarge)          │        │
├──────────────────────────┼──────────────────────────────────────────┤
│  HOST (AMI)              │                                          │
│  ┌───────────────────────┼────────────────────────────────────┐    │
│  │ ✅ CUDA Driver        │                                    │    │
│  │ ✅ EFA 1.26.1 ◄───────┘ (32 x 100 Gbps = 3200 Gbps)      │    │
│  │ ✅ Docker + Enroot                                        │    │
│  │ ✅ Slurm Daemon                                           │    │
│  │ ✅ /scratch (30.72TB NVMe)                                │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  CONTAINER (Enroot/Pyxis로 실행)                                    │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  ✅ CUDA Toolkit 12.2                                      │    │
│  │  ✅ NCCL 2.18.5-1+cuda12.2                                 │    │
│  │  ✅ AWS OFI NCCL v1.8.1-aws                                │    │
│  │  ✅ EFA libfabric                                          │    │
│  │  ✅ PyTorch + Megatron-LM                                  │    │
│  │                                                             │    │
│  │  프로세스: Rank 8-15 (각 GPU에 하나씩)                     │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  GPU Topology (NVSwitch 기반):                                      │
│  ┌──────────────────────────────────────────────────────┐          │
│  │  GPU 0 ◄─┐                                           │          │
│  │  GPU 1   │                                           │          │
│  │  GPU 2   │  NVLink 4.0 (900 GB/s bidirectional)     │          │
│  │  GPU 3   │  All-to-All 연결 (NVSwitch)              │          │
│  │  GPU 4   │                                           │          │
│  │  GPU 5   │                                           │          │
│  │  GPU 6   │                                           │          │
│  │  GPU 7 ◄─┘                                           │          │
│  └──────────────────────────────────────────────────────┘          │
│                                                                      │
│  Network: 32 x EFA (100 Gbps each)                                  │
│  └─ Total: 3200 Gbps (400 GB/s)                                    │
└─────────────────────────────────────────────────────────────────────┘

                    ↓ 공유 스토리지 ↓


┌─────────────────────────────────────────────────────────────────────┐
│                      SHARED STORAGE                                  │
├─────────────────────────────────────────────────────────────────────┤
│  /fsx (FSx Lustre)                                                   │
│  ├── megatron-training.sqsh (컨테이너 이미지)                        │
│  ├── data/ (학습 데이터)                                             │
│  ├── checkpoints/ (모델 체크포인트)                                  │
│  ├── logs/ (학습 로그)                                               │
│  └── scripts/ (실행 스크립트)                                        │
│                                                                      │
│  /home (FSx OpenZFS)                                                 │
│  └── ubuntu/ (사용자 홈 디렉토리)                                    │
└─────────────────────────────────────────────────────────────────────┘


```


```
헤드 노드 (Head Node)

역할: 관리 및 조정

    GPU 없음 (m5.8xlarge)
    작업 제출 및 스케줄링
    Docker 이미지 빌드
    데이터 준비

설치 항목:
HOST:
├── EFA 1.26.1 (설치되지만 사용 안 함)
├── Docker (이미지 빌드)
├── Enroot (이미지 변환)
└── Slurm Controller (slurmctld)

CONTAINER: 실행 안 함

주요 작업:
# 헤드 노드에서 실행
docker build -t megatron-training .
enroot import -o megatron-training.sqsh dockerd://megatron-training:latest
sbatch train.sbatch
squeue
tail -f logs/train.out

📍 컴퓨트 노드 (Compute Nodes)

역할: 실제 학습 실행

    GPU 8개 (p5.48xlarge)
    분산 학습 워커
    고속 네트워크 통신

설치 항목:
HOST:
├── CUDA Driver 535.54.03+ (GPU 제어)
├── EFA 1.26.1 (노드 간 통신)
├── Docker (컨테이너 런타임)
├── Enroot (컨테이너 실행)
├── Slurm Daemon (slurmd)
└── /scratch (30.72TB NVMe)

CONTAINER (실행 시):
├── CUDA Toolkit 12.2
├── NCCL 2.18.5-1+cuda12.2
├── AWS OFI NCCL v1.8.1-aws
├── EFA libfabric
├── PyTorch
└── Megatron-LM

실행 프로세스:
Node 1:
├── Rank 0 → GPU 0
├── Rank 1 → GPU 1
├── Rank 2 → GPU 2
├── Rank 3 → GPU 3
├── Rank 4 → GPU 4
├── Rank 5 → GPU 5
├── Rank 6 → GPU 6
└── Rank 7 → GPU 7

Node 2:
├── Rank 8  → GPU 0
├── Rank 9  → GPU 1
├── Rank 10 → GPU 2
├── Rank 11 → GPU 3
├── Rank 12 → GPU 4
├── Rank 13 → GPU 5
├── Rank 14 → GPU 6
└── Rank 15 → GPU 7

통신 경로
노드 내부 (Intra-Node):
GPU 0 ◄─────────┐
GPU 1           │
GPU 2           │ NVLink
GPU 3           │ (900 GB/s)
GPU 4           │
GPU 5           │
GPU 6           │
GPU 7 ◄─────────┘

노드 간 (Inter-Node):
Node 1                     Node 2
GPU 0-7 ◄──────EFA─────► GPU 0-7
            (400 Gbps)
            
NCCL + AWS OFI NCCL + EFA
= 고속 GPU 간 통신

검증 체크리스트
✅ 헤드 노드 검증:
# 헤드 노드에서 실행
bash /fsx/scripts/scan-host-environment.sh head

# 확인 항목:
# - Docker: ✅
# - Enroot: ✅
# - Slurm: ✅
# - GPU: ❌ (없음, 정상)
# - EFA: ✅ (설치됨, 사용 안 함)

✅ 컴퓨트 노드 검증 (호스트):
# 컴퓨트 노드에서 실행
srun -N 2 bash /fsx/scripts/scan-host-environment.sh compute

# 확인 항목:
# - CUDA Driver: ✅ 535.54.03+
# - EFA: ✅ 1.26.1+
# - GPU: ✅ 8개
# - Docker: ✅
# - Enroot: ✅
# - /scratch: ✅ 30.72TB

✅ 컴퓨트 노드 검증 (컨테이너):
# 컨테이너 내부에서 실행
srun -N 2 \
  --container-image=/fsx/megatron-training.sqsh \
  --container-mounts=/fsx:/fsx \
  bash /fsx/scripts/scan-container-environment.sh

# 확인 항목:
# - CUDA Toolkit: ✅ 12.2
# - NCCL: ✅ 2.18.5-1+cuda12.2
# - AWS OFI NCCL: ✅ v1.8.1-aws
# - EFA libfabric: ✅
# - PyTorch: ✅
# - Megatron-LM: ✅

워크플로우
1. 헤드 노드:
   └─ Docker 이미지 빌드
   └─ Enroot 변환
   └─ sbatch 작업 제출

2. Slurm Controller (헤드 노드):
   └─ 작업 스케줄링
   └─ 컴퓨트 노드 할당

3. 컴퓨트 노드 1, 2:
   └─ Enroot로 컨테이너 시작
   └─ GPU 할당
   └─ NCCL 초기화
   └─ EFA 네트워크 설정
   └─ 분산 학습 시작

4. 학습 중:
   └─ 노드 내: NVLink (GPU 간)
   └─ 노드 간: EFA + NCCL (고속 통신)
   └─ 체크포인트: /fsx에 저장
   └─ 임시 데이터: /scratch 사용

5. 헤드 노드:
   └─ 로그 모니터링
   └─ 작업 상태 확인

이제 헤드 노드와 컴퓨트 노드의 역할과 구성이 명확합니다! 🎯
```
