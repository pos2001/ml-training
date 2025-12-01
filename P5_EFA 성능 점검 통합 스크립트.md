```
#!/bin/bash
#
# P5 Instance Performance Validation Script
# AWS HPC Reference Implementation
#
# Usage: ./p5_performance_check.sh [--full|--quick|--network|--gpu]
#

set -e

# 색상 정의
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# 로그 디렉토리
LOG_DIR="/var/log/p5_perf_check"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
LOG_FILE="${LOG_DIR}/perf_check_${TIMESTAMP}.log"
RESULT_FILE="${LOG_DIR}/results_${TIMESTAMP}.json"

mkdir -p ${LOG_DIR}

# 로깅 함수
log() { echo -e "${BLUE}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1" | tee -a ${LOG_FILE}
}

log_success() { echo -e "${GREEN}[✓]${NC} $1" | tee -a ${LOG_FILE}
}

log_warning() { echo -e "${YELLOW}[⚠]${NC} $1" | tee -a ${LOG_FILE}
}

log_error() { echo -e "${RED}[✗]${NC} $1" | tee -a ${LOG_FILE}
}

# JSON 결과 초기화
init_results() { cat > ${RESULT_FILE} << EOF
{ "timestamp": "$(date -Iseconds)", "instance_type": "$(ec2-metadata --instance-type | cut -d ' ' -f 2)", "instance_id": "$(ec2-metadata --instance-id | cut -d ' ' -f 2)", "checks": {}
}
EOF
}

# 결과 추가 함수
add_result() { local check_name=$1 local status=$2 local details=$3 python3 << EOF
import json
with open('${RESULT_FILE}', 'r') as f: data = json.load(f)
data['checks']['${check_name}'] = { 'status': '${status}', 'details': '''${details}'''
}
with open('${RESULT_FILE}', 'w') as f: json.dump(data, f, indent=2)
EOF
}

#=============================================================================
# 1. 시스템 기본 점검
#=============================================================================
check_system_basics() { log "=== 시스템 기본 점검 시작 ===" # 인스턴스 타입 확인 INSTANCE_TYPE=$(ec2-metadata --instance-type | cut -d ' ' -f 2) log "Instance Type: ${INSTANCE_TYPE}" if [[ ! "${INSTANCE_TYPE}" =~ ^p5\. ]]; then log_warning "이 스크립트는 P5 인스턴스용으로 최적화되어 있습니다" fi # 커널 버전 KERNEL_VERSION=$(uname -r) log "Kernel Version: ${KERNEL_VERSION}" # 권장 커널: 5.15 이상 KERNEL_MAJOR=$(echo ${KERNEL_VERSION} | cut -d. -f1) KERNEL_MINOR=$(echo ${KERNEL_VERSION} | cut -d. -f2) if [ ${KERNEL_MAJOR} -lt 5 ] || ([ ${KERNEL_MAJOR} -eq 5 ] && [ ${KERNEL_MINOR} -lt 15 ]); then log_warning "권장 커널 버전: 5.15 이상 (현재: ${KERNEL_VERSION})" else log_success "커널 버전 적합" fi add_result "system_basics" "pass" "Kernel: ${KERNEL_VERSION}, Instance: ${INSTANCE_TYPE}"
}

#=============================================================================
# 2. EFA 드라이버 및 설정 점검
#=============================================================================
check_efa() { log "=== EFA 드라이버 및 설정 점검 ===" # EFA 디바이스 확인 EFA_DEVICES=$(ls /sys/class/infiniband/ 2>/dev/null | grep -c efa || echo 0) log "EFA Devices Found: ${EFA_DEVICES}" if [ ${EFA_DEVICES} -eq 0 ]; then log_error "EFA 디바이스를 찾을 수 없습니다" add_result "efa_devices" "fail" "No EFA devices found" return 1 fi # P5.48xlarge는 32개 EFA 어댑터 보유 EXPECTED_EFA=32 if [ ${EFA_DEVICES} -ne ${EXPECTED_EFA} ]; then log_warning "예상 EFA 디바이스 수: ${EXPECTED_EFA}, 실제: ${EFA_DEVICES}" else log_success "EFA 디바이스 수 정상 (${EFA_DEVICES})" fi # EFA 드라이버 버전 EFA_VERSION=$(modinfo efa 2>/dev/null | grep ^version | awk '{print $2}') log "EFA Driver Version: ${EFA_VERSION}" # libfabric 버전 if command -v fi_info &> /dev/null; then LIBFABRIC_VERSION=$(fi_info --version 2>&1 | head -1) log "Libfabric Version: ${LIBFABRIC_VERSION}" log_success "libfabric 설치 확인" else log_error "libfabric이 설치되지 않았습니다" add_result "libfabric" "fail" "Not installed" return 1 fi # EFA 인터페이스 상태 for efa_dev in $(ls /sys/class/infiniband/ | grep efa); do STATE=$(cat /sys/class/infiniband/${efa_dev}/ports/1/state) log "EFA Device ${efa_dev}: ${STATE}" done add_result "efa_check" "pass" "Devices: ${EFA_DEVICES}, Driver: ${EFA_VERSION}"
}

#=============================================================================
# 3. GDRCopy 점검
#=============================================================================
check_gdrcopy() { log "=== GDRCopy 점검 ===" # GDRCopy 모듈 로드 확인 if lsmod | grep -q gdrdrv; then GDRCOPY_VERSION=$(modinfo gdrdrv 2>/dev/null | grep ^version | awk '{print $2}') log_success "GDRCopy 모듈 로드됨 (버전: ${GDRCOPY_VERSION})" # /dev/gdrdrv 디바이스 확인 if [ -c /dev/gdrdrv ]; then log_success "/dev/gdrdrv 디바이스 존재" else log_error "/dev/gdrdrv 디바이스를 찾을 수 없습니다" add_result "gdrcopy" "fail" "Device not found" return 1 fi add_result "gdrcopy" "pass" "Version: ${GDRCOPY_VERSION}" else log_error "GDRCopy 모듈이 로드되지 않았습니다" add_result "gdrcopy" "fail" "Module not loaded" return 1 fi
}

#=============================================================================
# 4. NVIDIA GPU 및 드라이버 점검
#=============================================================================
check_nvidia() { log "=== NVIDIA GPU 및 드라이버 점검 ===" # nvidia-smi 실행 가능 여부 if ! command -v nvidia-smi &> /dev/null; then log_error "nvidia-smi를 찾을 수 없습니다" add_result "nvidia_driver" "fail" "nvidia-smi not found" return 1 fi # GPU 개수 확인 (P5.48xlarge = 8x H100) GPU_COUNT=$(nvidia-smi --query-gpu=count --format=csv,noheader | head -1) log "GPU Count: ${GPU_COUNT}" EXPECTED_GPU=8 if [ ${GPU_COUNT} -ne ${EXPECTED_GPU} ]; then log_warning "예상 GPU 수: ${EXPECTED_GPU}, 실제: ${GPU_COUNT}" else log_success "GPU 수 정상 (${GPU_COUNT})" fi # 드라이버 버전 DRIVER_VERSION=$(nvidia-smi --query-gpu=driver_version --format=csv,noheader | head -1) log "NVIDIA Driver Version: ${DRIVER_VERSION}" # CUDA 버전 CUDA_VERSION=$(nvidia-smi | grep "CUDA Version" | awk '{print $9}') log "CUDA Version: ${CUDA_VERSION}" # GPU 상태 확인 nvidia-smi --query-gpu=index,name,temperature.gpu,utilization.gpu,memory.used,memory.total \ --format=csv,noheader | while read line; do log "GPU Status: ${line}" done # GPU Topology 확인 log "GPU Topology Matrix:" nvidia-smi topo -m 2>&1 | tee -a ${LOG_FILE} add_result "nvidia_gpu" "pass" "GPUs: ${GPU_COUNT}, Driver: ${DRIVER_VERSION}, CUDA: ${CUDA_VERSION}"
}

#=============================================================================
# 5. NCCL 설정 점검
#=============================================================================
check_nccl() { log "=== NCCL 설정 점검 ===" # NCCL 라이브러리 확인 if ldconfig -p | grep -q libnccl; then NCCL_PATH=$(ldconfig -p | grep libnccl.so | head -1 | awk '{print $NF}') log_success "NCCL 라이브러리 발견: ${NCCL_PATH}" # NCCL 버전 (파일명에서 추출) NCCL_VERSION=$(basename ${NCCL_PATH} | grep -oP 'libnccl\.so\.\K[0-9.]+') log "NCCL Version: ${NCCL_VERSION}" else log_warning "NCCL 라이브러리를 찾을 수 없습니다" fi # 권장 NCCL 환경변수 log "권장 NCCL 환경변수:" cat << 'EOF' | tee -a ${LOG_FILE}
export NCCL_DEBUG=INFO
export NCCL_SOCKET_IFNAME=^docker0,lo
export NCCL_PROTO=simple
export FI_EFA_USE_DEVICE_RDMA=1
export FI_PROVIDER=efa
export FI_EFA_FORK_SAFE=1
export NCCL_ALGO=Ring
export NCCL_NET_GDR_LEVEL=PHB
EOF add_result "nccl" "pass" "Version: ${NCCL_VERSION:-unknown}"
}

#=============================================================================
# 6. NUMA 설정 점검
#=============================================================================
check_numa() { log "=== NUMA 설정 점검 ===" # NUMA 노드 수 NUMA_NODES=$(numactl --hardware | grep "available:" | awk '{print $2}') log "NUMA Nodes: ${NUMA_NODES}" # CPU-GPU NUMA 매핑 log "CPU-GPU NUMA Affinity:" for gpu in $(seq 0 $((GPU_COUNT-1))); do NUMA_NODE=$(nvidia-smi topo -m | grep "GPU${gpu}" | awk '{print $NF}') log "GPU${gpu} -> NUMA Node: ${NUMA_NODE}" done # NUMA balancing 상태 (비활성화 권장) NUMA_BALANCING=$(cat /proc/sys/kernel/numa_balancing) if [ ${NUMA_BALANCING} -eq 0 ]; then log_success "NUMA balancing 비활성화됨 (권장)" else log_warning "NUMA balancing 활성화됨 (비활성화 권장)" log "비활성화 명령: echo 0 | sudo tee /proc/sys/kernel/numa_balancing" fi add_result "numa" "pass" "Nodes: ${NUMA_NODES}, Balancing: ${NUMA_BALANCING}"
}

#=============================================================================
# 7. PCIe 설정 점검
#=============================================================================
check_pcie() { log "=== PCIe 설정 점검 ===" # GPU PCIe 링크 속도 및 폭 log "GPU PCIe Link Status:" nvidia-smi --query-gpu=index,pcie.link.gen.current,pcie.link.width.current \ --format=csv,noheader | while read line; do log "GPU PCIe: ${line}" done # EFA PCIe 링크 속도 log "EFA PCIe Link Status:" for efa_dev in $(ls /sys/class/infiniband/ | grep efa | head -3); do PCI_DEV=$(readlink /sys/class/infiniband/${efa_dev}/device | xargs basename) LINK_SPEED=$(lspci -vv -s ${PCI_DEV} 2>/dev/null | grep "LnkSta:" | awk '{print $2, $3}') log "EFA ${efa_dev} (${PCI_DEV}): ${LINK_SPEED}" done # PCIe ACS (Access Control Services) 확인 log "PCIe ACS Status:" lspci -vvv 2>/dev/null | grep -A 5 "Access Control Services" | head -20 | tee -a ${LOG_FILE} add_result "pcie" "pass" "Link status checked"
}

#=============================================================================
# 8. 네트워크 MTU 및 튜닝 점검
#=============================================================================
check_network_tuning() { log "=== 네트워크 MTU 및 튜닝 점검 ===" # EFA 인터페이스 MTU (9000 권장) for efa_if in $(ip link | grep efa | awk -F: '{print $2}' | tr -d ' '); do MTU=$(ip link show ${efa_if} | grep mtu | awk '{print $5}') if [ ${MTU} -eq 9000 ]; then log_success "EFA ${efa_if} MTU: ${MTU} (권장값)" else log_warning "EFA ${efa_if} MTU: ${MTU} (권장: 9000)" log "설정 명령: sudo ip link set ${efa_if} mtu 9000" fi done # TCP 튜닝 파라미터 log "TCP 튜닝 파라미터 점검:" # TCP window size TCP_RMEM=$(cat /proc/sys/net/ipv4/tcp_rmem) TCP_WMEM=$(cat /proc/sys/net/ipv4/tcp_wmem) log "tcp_rmem: ${TCP_RMEM}" log "tcp_wmem: ${TCP_WMEM}" # TCP congestion control TCP_CONGESTION=$(cat /proc/sys/net/ipv4/tcp_congestion_control) log "TCP Congestion Control: ${TCP_CONGESTION}" if [ "${TCP_CONGESTION}" != "bbr" ]; then log_warning "권장 Congestion Control: bbr (현재: ${TCP_CONGESTION})" fi # 권장 sysctl 설정 log "권장 네트워크 튜닝 설정:" cat << 'EOF' | tee -a ${LOG_FILE}
# /etc/sysctl.d/99-efa-tuning.conf
net.core.rmem_max = 536870912
net.core.wmem_max = 536870912
net.ipv4.tcp_rmem = 4096 87380 536870912
net.ipv4.tcp_wmem = 4096 65536 536870912
net.core.netdev_max_backlog = 30000
net.ipv4.tcp_congestion_control = bbr
net.core.default_qdisc = fq
EOF add_result "network_tuning" "pass" "MTU and TCP parameters checked"
}

#=============================================================================
# 9. NCCL all_reduce 성능 테스트
#=============================================================================
test_nccl_performance() { log "=== NCCL all_reduce 성능 테스트 ===" # nccl-tests 경로 확인 NCCL_TEST_PATH="/opt/nccl-tests/build/all_reduce_perf" if [ ! -f "${NCCL_TEST_PATH}" ]; then log_warning "NCCL tests를 찾을 수 없습니다: ${NCCL_TEST_PATH}" log "설치 방법:" cat << 'EOF' | tee -a ${LOG_FILE}
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests
make MPI=1 MPI_HOME=/opt/amazon/openmpi CUDA_HOME=/usr/local/cuda
EOF add_result "nccl_perf_test" "skip" "nccl-tests not found" return 0 fi # 단일 노드 8-GPU all_reduce 테스트 log "단일 노드 all_reduce 테스트 실행 중..." export NCCL_DEBUG=INFO export NCCL_SOCKET_IFNAME=^docker0,lo export FI_PROVIDER=efa export FI_EFA_USE_DEVICE_RDMA=1 NCCL_OUTPUT="${LOG_DIR}/nccl_allreduce_${TIMESTAMP}.log" ${NCCL_TEST_PATH} -b 8 -e 8G -f 2 -g 8 2>&1 | tee ${NCCL_OUTPUT} # 결과 파싱 (8GB 전송 시 대역폭) BW_8GB=$(grep "8589934592" ${NCCL_OUTPUT} | tail -1 | awk '{print $(NF-2)}') if [ ! -z "${BW_8GB}" ]; then log_success "NCCL all_reduce 대역폭 (8GB): ${BW_8GB} GB/s" # P5.48xlarge 기대 성능: ~3000 GB/s (NVLink) EXPECTED_BW=2500 if (( $(echo "${BW_8GB} > ${EXPECTED_BW}" | bc -l) )); then log_success "성능 목표 달성 (>${EXPECTED_BW} GB/s)" else log_warning "성능 목표 미달 (기대: >${EXPECTED_BW} GB/s, 실제: ${BW_8GB} GB/s)" fi add_result "nccl_perf_test" "pass" "Bandwidth: ${BW_8GB} GB/s" else log_error "NCCL 테스트 결과를 파싱할 수 없습니다" add_result "nccl_perf_test" "fail" "Could not parse results" fi
}

#=============================================================================
# 10. OSU Micro-Benchmarks (Latency/Bandwidth)
#=============================================================================
test_osu_benchmarks() { log "=== OSU Micro-Benchmarks 테스트 ===" OSU_PATH="/opt/osu-micro-benchmarks/build.openmpi/mpi" if [ ! -d "${OSU_PATH}" ]; then log_warning "OSU Micro-Benchmarks를 찾을 수 없습니다" log "설치 방법:" cat << 'EOF' | tee -a ${LOG_FILE}
wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-7.3.tar.gz
tar xzf osu-micro-benchmarks-7.3.tar.gz
cd osu-micro-benchmarks-7.3
./configure CC=mpicc CXX=mpicxx --prefix=/opt/osu-micro-benchmarks/build.openmpi
make && make install
EOF add_result "osu_benchmarks" "skip" "OSU benchmarks not found" return 0 fi # Point-to-Point Latency 테스트 log "OSU Latency 테스트 실행 중..." OSU_LAT_OUTPUT="${LOG_DIR}/osu_latency_${TIMESTAMP}.log" mpirun -np 2 --host localhost:2 \ -x FI_PROVIDER=efa \ -x FI_EFA_USE_DEVICE_RDMA=1 \ -x LD_LIBRARY_PATH \ ${OSU_PATH}/pt2pt/osu_latency 2>&1 | tee ${OSU_LAT_OUTPUT} # 4KB 메시지 latency 추출 LAT_4K=$(grep "^4096" ${OSU_LAT_OUTPUT} | awk '{print $2}') if [ ! -z "${LAT_4K}" ]; then log_success "OSU Latency (4KB): ${LAT_4K} μs" # P5 EFA 기대 latency: < 10 μs if (( $(echo "${LAT_4K} < 10" | bc -l) )); then log_success "Latency 목표 달성 (<10 μs)" else log_warning "Latency 높음 (기대: <10 μs, 실제: ${LAT_4K} μs)" fi fi # Point-to-Point Bandwidth 테스트 log "OSU Bandwidth 테스트 실행 중..." OSU_BW_OUTPUT="${LOG_DIR}/osu_bw_${TIMESTAMP}.log" mpirun -np 2 --host localhost:2 \ -x FI_PROVIDER=efa \ -x FI_EFA_USE_DEVICE_RDMA=1 \ -x LD_LIBRARY_PATH \ ${OSU_PATH}/pt2pt/osu_bw 2>&1 | tee ${OSU_BW_OUTPUT} # 4MB 메시지 bandwidth 추출 BW_4M=$(grep "^4194304" ${OSU_BW_OUTPUT} | awk '{print $2}') if [ ! -z "${BW_4M}" ]; then log_success "OSU Bandwidth (4MB): ${BW_4M} MB/s" # P5 EFA 기대 bandwidth: > 25000 MB/s (200 Gbps) if (( $(echo "${BW_4M} > 25000" | bc -l) )); then log_success "Bandwidth 목표 달성 (>25000 MB/s)" else log_warning "Bandwidth 낮음 (기대: >25000 MB/s, 실제: ${BW_4M} MB/s)" fi fi add_result "osu_benchmarks" "pass" "Latency: ${LAT_4K} μs, Bandwidth: ${BW_4M} MB/s"
}

#=============================================================================
# 11. EFA 통계 및 에러 점검
#=============================================================================
check_efa_stats() { log "=== EFA 통계 및 에러 점검 ===" for efa_dev in $(ls /sys/class/infiniband/ | grep efa); do log "EFA Device: ${efa_dev}" # Port counters PORT_DIR="/sys/class/infiniband/${efa_dev}/ports/1/counters" if [ -d "${PORT_DIR}" ]; then # RX/TX 패킷 RX_PACKETS=$(cat ${PORT_DIR}/port_rcv_packets 2>/dev/null || echo 0) TX_PACKETS=$(cat ${PORT_DIR}/port_xmit_packets 2>/dev/null || echo 0) log "  RX Packets: ${RX_PACKETS}" log "  TX Packets: ${TX_PACKETS}" # 에러 카운터 RX_ERRORS=$(cat ${PORT_DIR}/port_rcv_errors 2>/dev/null || echo 0) TX_ERRORS=$(cat ${PORT_DIR}/port_xmit_discards 2>/dev/null || echo 0) if [ ${RX_ERRORS} -gt 0 ] || [ ${TX_ERRORS} -gt 0 ]; then log_warning "  에러 발견 - RX Errors: ${RX_ERRORS}, TX Errors: ${TX_ERRORS}" else log_success "  에
```
