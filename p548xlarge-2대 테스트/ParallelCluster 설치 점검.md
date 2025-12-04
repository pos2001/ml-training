
```
#!/bin/bash
# 파일명: full_cluster_check.sh

echo "=========================================="
echo "ParallelCluster 완전 진단"
echo "=========================================="
echo ""

# 헤드노드 체크
echo "===== HEAD NODE ====="
echo "Hostname: $(hostname)"
echo "Instance Type: m5.8xlarge (예상)"
echo ""
echo "설치된 컴포넌트:"
echo "  Docker: $(docker --version 2>/dev/null || echo 'Not found')"
echo "  NCCL: $(ls /opt/nccl/build/lib/libnccl.so 2>/dev/null && echo 'Installed' || echo 'Not found')"
echo "  AWS OFI NCCL: $(ls /opt/aws-ofi-nccl/lib/libnccl-net.so 2>/dev/null && echo 'Installed' || echo 'Not found')"
echo "  GPU: $(nvidia-smi 2>/dev/null && echo 'Present' || echo 'Not present (정상)')"
echo "  EFA: $(ls /dev/infiniband/uverbs* 2>/dev/null | wc -l) devices (0 = 정상)"
echo ""

# 컴퓨트 노드 체크
echo "===== COMPUTE NODES ====="
srun -N 2 --ntasks-per-node=1 bash << 'EOFCOMPUTE'
source /fsx/setup_gpu_env.sh 2>/dev/null || true

echo "--- Node: $(hostname) ---"
echo "Instance Type: p5.48xlarge (예상)"
echo ""
echo "하드웨어:"
echo "  GPU: $(nvidia-smi --query-gpu=count --format=csv,noheader 2>/dev/null || echo '0') x $(nvidia-smi --query-gpu=name --format=csv,noheader 2>/dev/null | head -1)"
echo "  EFA Devices: $(ls /dev/infiniband/uverbs* 2>/dev/null | wc -l)"
echo "  EFA Interfaces: $(ip link show | grep -c efa)"
echo ""
echo "소프트웨어:"
echo "  Docker: $(docker --version 2>/dev/null | cut -d' ' -f3 || echo 'Not found')"
echo "  CUDA: $(nvcc --version 2>/dev/null | grep release | awk '{print $5}' | tr -d ',' || echo 'Not in PATH')"
echo "  NCCL: $(ls /opt/nccl/build/lib/libnccl.so.* 2>/dev/null | head -1 | xargs basename)"
echo "  AWS OFI NCCL: $(ls /opt/aws-ofi-nccl/lib/libnccl-net.so 2>/dev/null && echo 'Present' || echo 'Not found')"
echo "  Libfabric: $(fi_info --version 2>/dev/null | head -1)"
echo ""
echo "환경 변수:"
echo "  CUDA_HOME: ${CUDA_HOME:-'Not set'}"
echo "  FI_PROVIDER: ${FI_PROVIDER:-'Not set'}"
echo "  LD_PRELOAD: ${LD_PRELOAD:-'Not set'}"
echo ""
echo "EFA Provider 테스트:"
fi_info -p efa 2>/dev/null | head -3 || echo "  ERROR: EFA provider not available"
echo ""
EOFCOMPUTE

echo "=========================================="
echo "진단 완료"
echo "=========================================="
```
### 진단결과
```
==========================================
ParallelCluster 완전 진단
==========================================

===== HEAD NODE =====
Hostname: ip-10-0-108-46
Instance Type: m5.8xlarge (예상)

설치된 컴포넌트:
  Docker: Docker version 29.1.2, build 890dcca
  NCCL: /opt/nccl/build/lib/libnccl.so
Installed
  AWS OFI NCCL: /opt/aws-ofi-nccl/lib/libnccl-net.so
Installed
  GPU: NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running.

Not present (정상)
  EFA: 0 devices (0 = 정상)

===== COMPUTE NODES =====
--- Node: compute-gpu-st-distributed-ml-2 ---
--- Node: compute-gpu-st-distributed-ml-1 ---
Instance Type: p5.48xlarge (예상)

하드웨어:
Instance Type: p5.48xlarge (예상)

하드웨어:
  GPU: 8
8
8
8
8
8
8
8 x NVIDIA H100 80GB HBM3
  GPU: 8
8
8
8
8
8
8
8 x NVIDIA H100 80GB HBM3
  EFA Devices: 32
  EFA Devices: 32
  EFA Interfaces: 34

소프트웨어:
  EFA Interfaces: 34

소프트웨어:
  Docker: 29.1.2,
  CUDA:
  NCCL: libnccl.so.2
  AWS OFI NCCL: /opt/aws-ofi-nccl/lib/libnccl-net.so
Present
  Libfabric: fi_info: 1.22.0amzn1.0

환경 변수:
  CUDA_HOME: 'Not set'
  FI_PROVIDER: 'Not set'
  LD_PRELOAD: 'Not set'

EFA Provider 테스트:
  Docker: 29.1.2,
  CUDA:
  NCCL: libnccl.so.2
  AWS OFI NCCL: /opt/aws-ofi-nccl/lib/libnccl-net.so
Present
  Libfabric: fi_info: 1.22.0amzn1.0

환경 변수:
  CUDA_HOME: 'Not set'
  FI_PROVIDER: 'Not set'
  LD_PRELOAD: 'Not set'

EFA Provider 테스트:
provider: efa
    fabric: efa
    domain: rdmap79s0-rdm
provider: efa
    fabric: efa
    domain: rdmap79s0-rdm


==========================================
진단 완료
==========================================
```
