## 클러스터 구성요약
```
클러스터 구성 요약
✅ GPU 구성 (완벽)

노드 1 & 노드 2 각각:

    GPU 개수: 8개
    모델: NVIDIA H100 80GB HBM3
    총 메모리: 81,559 MiB (약 80GB) × 8 = 652 GB per node
    사용 가능 메모리: 80,995 MiB per GPU (거의 전체)
    총 클러스터 GPU 메모리: 1,304 GB (1.27 TB)

✅ 소프트웨어 스택

    드라이버: 550.90.07
    CUDA: 12.4
    상태: 모든 GPU 정상 작동

🚀 NVLink 상태 (매우 중요!)
각 GPU당 18개의 NVLink 연결
각 GPU: 18 links × 26.562 GB/s = 478 GB/s 대역폭

이것은 엄청난 성능입니다!
NVLink 토폴로지:

    Link 0-17: 모두 26.562 GB/s로 활성화
    총 노드 내부 대역폭: 약 3.8 TB/s (8 GPUs × 478 GB/s / 2)
    완전 연결 메쉬 구조 (All-to-All)

이는 H100 NVSwitch 구성으로, 모든 GPU가 직접 연결되어 있습니다!

네트워크 구성 (EFA - Elastic Fabric Adapter)
32개의 EFA 디바이스 (각 노드)
uverbs0 ~ uverbs31 (총 32개)

이것은 매우 특별합니다!
네트워크 인터페이스:

    32개의 고속 네트워크 인터페이스 (enp71s0 ~ enp193s0)
    모두 10.1.x.x 대역 (AWS 내부 네트워크)
    Docker 네트워크: 172.17.0.1/16

예상 네트워크 구성:

    각 GPU당 4개의 EFA 어댑터 (8 GPUs × 4 = 32)
    노드 간 통신 대역폭: 약 3.2 Tbps (400 Gbps × 8 GPUs)
    GPUDirect RDMA 지원 (GPU 간 직접 통신)

아키텍처 분석
이것은 AWS p5.48xlarge 인스턴스 구성입니다!

사양:

    GPU: 8× NVIDIA H100 80GB (NVSwitch 연결)
    NVLink: 900 GB/s 양방향 대역폭
    EFA: 3,200 Gbps 네트워크 대역폭
    GPUDirect RDMA: 활성화 (추정)
    CPU: 192 vCPUs (Intel Xeon 또는 AMD EPYC)
    메모리: 2 TB 시스템 메모리 (추정)

클러스터 총 성능:
╔════════════════════════════════════════════════════════════╗
║           클러스터 성능 지표                                ║
╠════════════════════════════════════════════════════════════╣
║  총 GPU:              16개 (H100 80GB)                     ║
║  총 GPU 메모리:       1.27 TB                              ║
║  총 FP16 성능:        ~32 PetaFLOPS                       ║
║  총 FP8 성능:         ~64 PetaFLOPS                       ║
║  노드 내부 대역폭:    900 GB/s per node (NVLink)          ║
║  노드 간 대역폭:      3.2 Tbps (EFA)                      ║
║  GPUDirect RDMA:      지원 (GPU 간 직접 통신)             ║
╚════════════════════════════════════════════════════════════╝

```

### 테스트 방법
```
ubuntu@ip-10-0-108-46:~$ cat > validate_simple.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=validate
#SBATCH --nodes=2
#SBATCH --ntasks=2
#SBATCH --ntasks-per-node=1
#SBATCH --gres=gpu:8
#SBATCH --time=00:10:00
#SBATCH --output=logs/validate-%j.out

echo "╔════════════════════════════════════════╗"
echo "║  GPU/Network Validation Report         ║"
echo "║  Job ID: $SLURM_JOB_ID                 ║"
echo "╚════════════════════════════════════════╝"

srun bash -c '
NODE=$(hostname)
echo ""
echo "┌────────────────────────────────────────┐"
echo "│  Node: $NODE"
echo "│  Task: $SLURM_PROCID"
echo "└────────────────────────────────────────┘"

echo ""
echo "GPU Count and Model:"
nvidia-smi --query-gpu=index,name --format=csv,noheader

echo ""
echo "GPU Memory:"
nvidia-smi --query-gpu=index,memory.total,memory.free --format=csv,noheader

echo ""
echo "CUDA Version:"
nvidia-smi | grep "CUDA Version"

echo ""
echo "Driver Version:"
nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -1

echo ""
echo "NVLink Status:"
nvidia-smi nvlink --status 2>/dev/null || echo "NVLink info not available"

echo ""
echo "Network Interfaces:"
ip addr show | grep "inet " | awk "{print \$2, \$NF}"

echo ""
echo "EFA Devices:"
ls /dev/infiniband/ 2>/dev/null || echo "No EFA devices found"

echo ""
'

echo ""
echo "╔════════════════════════════════════════╗"
echo "║  Validation Complete                   ║"
echo "╚════════════════════════════════════════╝"
EOF

mkdir -p logs
chmod +x validate_simple.sh
sbatch validate_simple.sh
```

### 실행방법
```
# 1. 로그 디렉토리 먼저 생성
mkdir -p logs

# 로그 파일 확인
ls -lh logs/

# 작업 상태 확인
squeue -j 11

# 로그 내용 보기
cat logs/validate-11.out
```


### 예상출력
```
╔════════════════════════════════════════╗
║  GPU/Network Validation Report         ║
║  Job ID: 11                 ║
╚════════════════════════════════════════╝

┌────────────────────────────────────────┐
│  Node: compute-gpu-st-distributed-ml-1
│  Task: 0
└────────────────────────────────────────┘

GPU Count and Model:
0, NVIDIA H100 80GB HBM3
1, NVIDIA H100 80GB HBM3
2, NVIDIA H100 80GB HBM3
3, NVIDIA H100 80GB HBM3
4, NVIDIA H100 80GB HBM3
5, NVIDIA H100 80GB HBM3
6, NVIDIA H100 80GB HBM3
7, NVIDIA H100 80GB HBM3

GPU Memory:

┌────────────────────────────────────────┐
│  Node: compute-gpu-st-distributed-ml-2
│  Task: 1
└────────────────────────────────────────┘

GPU Count and Model:
0, 81559 MiB, 80995 MiB
1, 81559 MiB, 80995 MiB
2, 81559 MiB, 80995 MiB
3, 81559 MiB, 80995 MiB
4, 81559 MiB, 80995 MiB
5, 81559 MiB, 80995 MiB
6, 81559 MiB, 80995 MiB
7, 81559 MiB, 80995 MiB

CUDA Version:
0, NVIDIA H100 80GB HBM3
1, NVIDIA H100 80GB HBM3
2, NVIDIA H100 80GB HBM3
3, NVIDIA H100 80GB HBM3
4, NVIDIA H100 80GB HBM3
5, NVIDIA H100 80GB HBM3
6, NVIDIA H100 80GB HBM3
7, NVIDIA H100 80GB HBM3

GPU Memory:
0, 81559 MiB, 80995 MiB
1, 81559 MiB, 80995 MiB
2, 81559 MiB, 80995 MiB
3, 81559 MiB, 80995 MiB
4, 81559 MiB, 80995 MiB
5, 81559 MiB, 80995 MiB
6, 81559 MiB, 80995 MiB
7, 81559 MiB, 80995 MiB

CUDA Version:
| NVIDIA-SMI 550.90.07              Driver Version: 550.90.07      CUDA Version: 12.4     |

Driver Version:
550.90.07

NVLink Status:
| NVIDIA-SMI 550.90.07              Driver Version: 550.90.07      CUDA Version: 12.4     |

Driver Version:
550.90.07

NVLink Status:
GPU 0: NVIDIA H100 80GB HBM3 (UUID: GPU-a13fd687-df01-df63-bd60-ecc0de70cc45)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 1: NVIDIA H100 80GB HBM3 (UUID: GPU-f9552e2d-4881-ba4c-acc2-5b31f54d7369)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 2: NVIDIA H100 80GB HBM3 (UUID: GPU-20a2005b-2297-26f5-b9d1-03b1a7f0a657)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 3: NVIDIA H100 80GB HBM3 (UUID: GPU-acf327e1-d7fb-f426-b5d2-c017b8103351)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 4: NVIDIA H100 80GB HBM3 (UUID: GPU-6f85c50b-9190-1e5d-d108-21c0191c011b)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 5: NVIDIA H100 80GB HBM3 (UUID: GPU-1115f8b2-2525-01a2-b6f5-47558fc5292e)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 6: NVIDIA H100 80GB HBM3 (UUID: GPU-7e33e3c5-6511-3a15-3174-b38d99093e56)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s
GPU 7: NVIDIA H100 80GB HBM3 (UUID: GPU-904c6966-ccb1-735a-fb5b-718b34a7c0cd)
         Link 0: 26.562 GB/s
         Link 1: 26.562 GB/s
         Link 2: 26.562 GB/s
         Link 3: 26.562 GB/s
         Link 4: 26.562 GB/s
         Link 5: 26.562 GB/s
         Link 6: 26.562 GB/s
         Link 7: 26.562 GB/s
         Link 8: 26.562 GB/s
         Link 9: 26.562 GB/s
         Link 10: 26.562 GB/s
         Link 11: 26.562 GB/s
         Link 12: 26.562 GB/s
         Link 13: 26.562 GB/s
         Link 14: 26.562 GB/s
         Link 15: 26.562 GB/s
         Link 16: 26.562 GB/s
         Link 17: 26.562 GB/s

Network Interfaces:
127.0.0.1/8 lo
10.1.71.128/17 enp71s0
10.1.80.35/17 enp72s0
10.1.109.242/17 enp73s0
10.1.118.224/17 enp74s0
10.1.71.149/17 enp88s0
10.1.67.65/17 enp89s0
10.1.29.202/17 enp90s0
10.1.91.112/17 enp91s0
10.1.58.141/17 enp105s0
10.1.81.166/17 enp106s0
10.1.61.230/17 enp107s0
10.1.27.52/17 enp108s0
10.1.96.219/17 enp122s0
10.1.25.226/17 enp123s0
10.1.12.7/17 enp124s0
10.1.3.91/17 enp125s0
10.1.106.200/17 enp140s0
10.1.31.210/17 enp141s0
10.1.109.81/17 enp142s0
10.1.67.48/17 enp143s0
10.1.81.0/17 enp156s0
10.1.88.145/17 enp157s0
10.1.43.229/17 enp158s0
10.1.95.1/17 enp159s0
10.1.48.12/17 enp173s0
10.1.33.52/17 enp174s0
10.1.75.177/17 enp175s0
10.1.66.184/17 enp176s0
10.1.102.157/17 enp190s0
10.1.64.227/17 enp191s0
10.1.28.135/17 enp192s0
10.1.88.120/17 enp193s0
172.17.0.1/16 docker0

EFA Devices:
uverbs0
uverbs1
uverbs10
uverbs11
uverbs12
uverbs13
uverbs14
uverbs15
uverbs16
uverbs17
uverbs18
uverbs19
uverbs2
uverbs20
uverbs21
uverbs22
uverbs23
uverbs24
uverbs25
uverbs26
uverbs27
uverbs28
uverbs29
uverbs3
uverbs30
uverbs31
uverbs4
uverbs5
uverbs6
uverbs7
uverbs8
uverbs9
```
