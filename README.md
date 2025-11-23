# ML Training Project

AWS ParallelCluster 환경에서 대규모 분산 학습을 수행하기 위한 프로젝트입니다.

## 프로젝트 개요

이 프로젝트는 GPU 클러스터를 활용한 머신러닝 모델 학습을 위한 샘플 환경 설정과 코드를 포함합니다.


## 사전 고려 사항

## 진행절차(end-to-end)
### Phase 0: 사전 준비 및 계획(예시)
- 클러스터 사양 확정

  - 인스턴스: P5e.48xlarge × 80개
  - 리전: ap-southeast-3 (자카르타)
  - 총 GPU: 640개 (8 GPU × 80 노드)
  - 총 vCPU: 15,360개 (192 vCPU × 80)
  - 총 메모리: 160 TB (2 TB × 80)
  - 시간당 비용: $2,768
- 훈련 계획:
  - 7개 모델 동시 훈련
  - 노드 할당: 모델 1-6 각 xx노드, 모델 7은 xx노드
  - 예상 훈련 기간: 모델당 y주

- 훈련 데이터 S3 업로드:
```
  - #!/bin/bash
set -e  # 에러 발생시 스크립트 중단

# 리전 설정
REGION="ap-southeast-3"

# 버킷 이름 설정
TRAINING_BUCKET="training-data-jakarta"
CHECKPOINT_BUCKET="checkpoint-jakarta"

# 로컬 경로 설정
LOCAL_DATA_PATH="/local/training-data/"
SAMPLE_DIR="./sample"

echo "Starting S3 operations..."

# 1. S3 버킷 생성
echo "Creating S3 buckets..."
for BUCKET in "$TRAINING_BUCKET" "$CHECKPOINT_BUCKET"; do
    if ! aws s3api head-bucket --bucket "$BUCKET" --region "$REGION" 2>/dev/null; then
        aws s3 mb "s3://$BUCKET" --region "$REGION"
        echo "Created bucket: $BUCKET"
    else
        echo "Bucket already exists: $BUCKET"
    fi
done

# 2. 데이터 업로드 (멀티파트)
echo "Uploading data to S3..."
aws s3 sync "$LOCAL_DATA_PATH" "s3://$TRAINING_BUCKET/datasets/" \
    --region "$REGION" \
    --storage-class STANDARD \
    --delete \
    --only-show-errors \
    --metadata md5=$(md5sum "$LOCAL_DATA_PATH"/* | sort | md5sum | cut -d' ' -f1)

# 3. 업로드 확인 및 요약
echo "Verifying upload..."
aws s3 ls "s3://$TRAINING_BUCKET/datasets/" \
    --recursive \
    --summarize \
    --human-readable

# 4. 데이터 검증
echo "Performing data validation..."

# 샘플 디렉토리 생성
mkdir -p "$SAMPLE_DIR"

# 샘플 데이터 다운로드
echo "Downloading sample data..."
aws s3 cp "s3://$TRAINING_BUCKET/datasets/sample.tar.gz" \
    "$SAMPLE_DIR/" \
    --region "$REGION"

# 체크섬 검증
echo "Validating checksum..."
LOCAL_MD5=$(md5sum "$SAMPLE_DIR/sample.tar.gz" | cut -d' ' -f1)
S3_MD5=$(aws s3api head-object \
    --bucket "$TRAINING_BUCKET" \
    --key "datasets/sample.tar.gz" \
    --query 'Metadata.md5' \
    --output text)

if [ "$LOCAL_MD5" = "$S3_MD5" ]; then
    echo "Checksum validation passed"
else
    echo "Checksum validation failed"
    exit 1
fi

# 압축 해제
echo "Extracting sample data..."
tar -xzf "$SAMPLE_DIR/sample.tar.gz" -C "$SAMPLE_DIR"

# 데이터셋 검증
echo "Running dataset validation..."
if [ -f validate_dataset.py ]; then
    python validate_dataset.py --data-dir "$SAMPLE_DIR"
else
    echo "Error: validate_dataset.py not found"
    exit 1
fi

echo "All operations completed successfully"
```

### Phase 1: 인프라 구축 (예시)
- 1.1 VPC 및 네트워크 생성
  - VPC 생성
  - 서브네 생성
  - 인터넷 게이트웨이 및 NAT 게이트웨이
  - 라우팅 테이블 설정
- 1.2 Security Group 설정
  - 헤드 노드와 컴퓨트 노드간 통신
  - 헤드 노드 SSH 접속용
  - Grafana 접근용 SG(Grafana 설치한다면)
- 1.3 키페어 설정(헤드 노드 접속용)

### Phase 2: 클러스터 배포(예시)
- 2.1 ParallelCluster 설치(최신 버전 권장)
  -  https://github.com/aws/aws-parallelcluster/releases
- 2.2 Yaml 설정 파일 작성(단순 예시이며,고객 상황에 따라 옵션이 다를 수 있음)
```
Region: ap-southeast-3                # AWS 리전 지정 (자카르타)
Version: 3.7.0                        # ParallelCluster 버전

Image:
  Os: alinux2                        # Amazon Linux 2 OS 선택 (AWS 최적화 OS)

HeadNode:                            # 클러스터 관리 노드 설정
  InstanceType: c6i.4xlarge          # 헤드노드 인스턴스 타입 (16vCPU, 32GB 메모리)
  Networking:                        # 네트워크 설정
    SubnetId: subnet-xxxxxxxxx       # 퍼블릭 서브넷 ID (외부 접속 가능)
    SecurityGroups:                  # 보안 그룹 설정
      - sg-zzzzzzzzz                # 헤드노드용 보안그룹 ID
  Ssh:
    KeyName: your-keypair-name       # SSH 접속용 키페어 이름
  Iam:                              # IAM 권한 설정
    AdditionalIamPolicies:          # 추가 IAM 정책
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore  # Systems Manager 접속용
      - Policy: arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy  # CloudWatch 로그 수집용
  LocalStorage:                      # 로컬 스토리지 설정
    RootVolume:                     # 루트 볼륨 설정
      Size: 200                     # 볼륨 크기 (GB)
      VolumeType: gp3              # SSD 볼륨 타입
      Iops: 16000                  # 초당 I/O 작업 수
      Throughput: 1000             # 처리량 (MB/s)

Scheduling:                         # 작업 스케줄링 설정
  Scheduler: slurm                  # Slurm 스케줄러 사용 (HPC 작업 관리)
  ScalingStrategy: best-effort      # 최선노력 기반 스케일링
  SlurmSettings:                    # Slurm 세부 설정
    ScaledownIdletime: 10          # 유휴시간 후 노드 종료 (분)
    QueueUpdateStrategy: DRAIN      # 작업 종료 후 노드 정리 방식

SlurmQueues:                        # Slurm 작업 큐 설정
  - Name: gpu-queue                 # GPU 작업 전용 큐 이름
    CapacityType: ONDEMAND          # 온디맨드 인스턴스 사용
    AllocationStrategy: lowest-price # 최저가 인스턴스 선택 전략
    ComputeSettings:                # 컴퓨트 노드 설정
      LocalStorage:                 # 컴퓨트 노드 스토리지 설정
        RootVolume:                # 루트 볼륨 설정
          Size: 200                # 볼륨 크기 (GB)
          VolumeType: gp3         # SSD 볼륨 타입
          Iops: 16000             # 초당 I/O 작업 수
          Throughput: 1000        # 처리량 (MB/s)
    Networking:                    # 컴퓨트 노드 네트워크 설정
      SubnetIds:                  # 서브넷 설정
        - subnet-yyyyyyyyy        # 프라이빗 서브넷 ID
      PlacementGroup:            # 배치 그룹 설정
        Enabled: true            # 네트워크 성능 최적화를 위한 배치 그룹 활성화
      SecurityGroups:            # 보안 그룹 설정
        - sg-zzzzzzzzz          # 컴퓨트 노드용 보안그룹 ID
    HealthChecks:                # 상태 확인 설정
      Gpu:                      # GPU 상태 확인
        Enabled: true          # GPU 상태 모니터링 활성화
    ComputeResources:           # 컴퓨트 리소스 설정
      - Name: p5e-nodes        # 노드 그룹 이름
        InstanceType: p5e.48xlarge  # GPU 인스턴스 타입 (48xlarge = 8 GPU)
        MinCount: 80            # 최소 노드 수
        MaxCount: 80           # 최대 노드 수
        
        Efa:                   # EFA(Elastic Fabric Adapter) 설정
          Enabled: true       # 고성능 네트워킹 활성화
          GdrSupport: ENABLED # GPU Direct RDMA 지원 활성화
        CapacityReservationTarget:  # 용량 예약 설정
          CapacityReservationId: cr-xxxxxxxxxxxxx  # 용량 예약 ID
    CustomActions:              # 사용자 정의 작업
      OnNodeStart:             # 노드 시작 시 실행할 스크립트
        Script: s3://your-bucket/scripts/node-setup.sh
      OnNodeConfigured:        # 노드 구성 완료 시 실행할 스크립트
        Script: s3://your-bucket/scripts/node-configured.sh
    Iam:                       # 컴퓨트 노드 IAM 설정
      AdditionalIamPolicies:   # 추가 IAM 정책
        - Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess  # S3 읽기 권한
        - Policy: arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy  # CloudWatch 로그 수집

SharedStorage:                 # 공유 스토리지 설정
  - MountDir: /fsx1          # 마운트 위치 (학습 데이터용)
    Name: fsx-data          # 스토리지 이름
    StorageType: FsxLustre  # FSx for Lustre 타입
    FsxLustreSettings:      # Lustre 설정
      StorageCapacity: 7200 # 용량 (GB)
      DeploymentType: PERSISTENT_2  # 영구 배포 타입
      PerUnitStorageThroughput: 1000  # 처리량 (MB/s)
      DataCompressionType: LZ4  # 데이터 압축 방식
      AutoImportPolicy: NEW    # 새 파일 자동 임포트
      DriveCacheType: READ    # 드라이브 캐시 타입

  - MountDir: /fsx2         # 마운트 위치 (체크포인트용)
    Name: fsx-checkpoints   # 스토리지 이름
    StorageType: FsxLustre  # FSx for Lustre 타입
    FsxLustreSettings:      # Lustre 설정
      StorageCapacity: 7200 # 용량 (GB)
      DeploymentType: PERSISTENT_2  # 영구 배포 타입
      PerUnitStorageThroughput: 1000  # 처리량 (MB/s)
      DataCompressionType: LZ4  # 데이터 압축 방식
      AutoImportPolicy: NEW    # 새 파일 자동 임포트
      DriveCacheType: READ    # 드라이브 캐시 타입

  - MountDir: /fsx3         # 마운트 위치 (로그용)
    Name: fsx-logs         # 스토리지 이름
    StorageType: FsxLustre # FSx for Lustre 타입
    FsxLustreSettings:     # Lustre 설정
      StorageCapacity: 4800  # 용량 (GB)
      DeploymentType: PERSISTENT_2  # 영구 배포 타입
      PerUnitStorageThroughput: 500  # 처리량 (MB/s)
      DataCompressionType: LZ4  # 데이터 압축 방식
      AutoImportPolicy: NEW    # 새 파일 자동 임포트
      DriveCacheType: READ    # 드라이브 캐시 타입

Monitoring:                   # 모니터링 설정
  DetailedMonitoring: true   # EC2 상세 모니터링 활성화
  Logs:                      # 로그 설정
    CloudWatch:              # CloudWatch 로그
      Enabled: true         # 활성화
      RetentionInDays: 14   # 로그 보관 기간 (일)
  Dashboards:               # 대시보드 설정
    CloudWatch:             # CloudWatch 대시보드
      Enabled: true        # 활성화

DevSettings:                # 개발 설정
  Timeouts:                # 타임아웃 설정
    HeadNodeBootstrapTimeout: 3600  # 헤드노드 부트스트랩 타임아웃 (초)

Tags:                      # 리소스 태그
  - Key: Project          # 프로젝트 태그
    Value: DistributedTraining
  - Key: Environment      # 환경 태그
    Value: Production
```
- 주요 yaml 특징

```
1. 컴퓨팅 인프라:
   ├── 헤드노드: c6i.4xlarge (컴퓨트 최적화)
   └── 컴퓨트노드: p5e.48xlarge
       ├── 총 80대 (ODCR)
       ├── 노드당 8개 H100 GPU
       └── 총 640개 GPU 사용 가능

2. 네트워킹:
   ├── 헤드노드: 퍼블릭 서브넷
   ├── 컴퓨트노드: 프라이빗 서브넷
   ├── EFA 지원 활성화
   └── 배치그룹 사용 (네트워크 지연 최소화)

3. 스토리지 구성:
   ├── /fsx1: 데이터 (7.2TB, 1000MB/s)
   ├── /fsx2: 체크포인트 (7.2TB, 1000MB/s)
   └── /fsx3: 로그 (4.8TB, 500MB/s)

4. 작업 관리:
   ├── Slurm 스케줄러
   ├── 메모리 기반 스케줄링
   └── 10분 유휴 자동 스케일다운

5. 모니터링/관리:
   ├── CloudWatch 상세 모니터링
   ├── GPU 상태 체크
   └── 14일 로그 보관
```

- 2.3 Yaml 검증 (옵션)
```
# 1. YAML 문법과 설정 검증(YAML 파일의 문법 오류 검사, 설정값의 유효성 검사, 필수 파라미터 누락 여부 확인)
pcluster create-cluster --dryrun true -n test-cluster -c config.yaml

# 2. 클러스터 생성 시뮬레이션 (실제 생성하지 않고 시물레이션 만 수행, AWS 리소스 생성 가능 여부 확인, 권한 문제 사전 확인 등)
: 이러한 사전 검증을 거치지 않으면, 생성 시도 후 다시 클러스터 삭제하는데 많은 시간 및 비용 소모
pcluster create-cluster \
    --dryrun true \
    --cluster-name ml-cluster \
    --cluster-configuration config.yaml \
    --region ap-southeast-3
```
- 2.4 클러스터 생성
```
# 클러스터 생성 
pcluster create-cluster \
    --cluster-name p5e-80node-cluster \
    --cluster-configuration config.yaml \
    --region ap-southeast-3 \
    --rollback-on-failure false
# 실시간 상태 확인
watch -n 10 'pcluster describe-cluster \
--cluster-name p5e-80node-cluster \
--region ap-southeast-3 \
--query "clusterStatus"'
```
```
# 클러스터 생성 명령어 설명
1. 클러스터 생성:
   ├── 수 십여분 소요
   └── 백그라운드 실행

2. 상태 모니터링:
   ├── watch 명령어로 실시간 확인
   ├── 10초 간격 갱신
   └── 가능한 상태값:
       ├── CREATE_IN_PROGRESS
       ├── CREATE_COMPLETE
       └── CREATE_FAILED
```
- 2.5 헤드 노드 접속
```
# 1. 헤드노드 IP 확인
HEAD_NODE_IP=$(pcluster describe-cluster \
    --cluster-name p5e-80node-cluster \
    --region ap-southeast-3 \
    --query "headNode.publicIpAddress" \
    --output text)

# 2. 접속 방법 (4 가지 중 선택)

# 방법 1: pcluster ssh 사용
pcluster ssh \
    --cluster-name p5e-80node-cluster \
    --region ap-southeast-3 \
    -i ~/.ssh/p5e-cluster-key.pem

# 방법 2: 일반 ssh 사용
ssh -i ~/.ssh/p5e-cluster-key.pem ec2-user@$HEAD_NODE_IP

# 방법 3: EC2 콘솔에서 헤드노드를 Session Manager를 통해 접근(가장 간편, 핸즈온 실습에서 사용한 방법)

# 방법 4: ParallelCluster UI (별도 설치 필요)
```

### Phase 3: 검증 및 테스트(예시)
- 3.1 기본 시스템 검증
```
# 1. SLURM 클러스터 상태 확인
sinfo
# 예상 출력:
# PARTITION AVAIL TIMELIMIT NODES STATE NODELIST
# gpu-queue    up   infinite     0  idle~  
# (MinCount가 0이므로 초기에는 노드가 없음)

# 2. FSx Lustre 마운트 확인
df -h | grep fsx
# 예상 출력:
# lustre-xxx  14T  1.1T  13T   8% /fsx1
# lustre-xxx  14T  1.1T  13T   8% /fsx2
# lustre-xxx  7.2T  100G  7.1T  2% /fsx3
# lustre-xxx  7.2T  100G  7.1T  2% /fsx4

# 3. S3 접근 테스트
aws s3 ls s3://training-data-jakarta/
# 예상 출력: 버킷 내용 리스트
```

- 3.2 소규모 노드 테스트 (2 노드)
  - 3.2.1 2 노드 기본 동작 테스트
    - 테스트 목적: 클러스터 기본 기능 검증
      - 노드 프로비저닝 작동 확인
      - Slurm 스케줄러 작동 확인
      -  GPU 할당 확인 
   
```
# 1. 테스트 작업 스크립트 생성
cat > test_2nodes.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=test-2nodes    # 작업 이름
#SBATCH --nodes=2                 # 2개 노드 요청
#SBATCH --ntasks-per-node=1       # 노드당 1개 태스크
#SBATCH --gres=gpu:1             # 노드당 1개 GPU 요청
#SBATCH --time=00:30:00          # 최대 실행 시간 30분
#SBATCH --output=/fsx3/logs/test-2nodes-%j.out  # 로그 저장 위치

echo "Node: $(hostname)"
nvidia-smi
EOF

# 2. 작업 제출
sbatch test_2nodes.sh

# 3. 노드 상태 모니터링
watch -n 10 sinfo
```

     - 2 노드 기본 테스트 예상 결과(sinfo)


  
  - 3.2.2 2 노드 GPU 및 EFA 검증
    - GPU와 EFA(Elastic Fabric Adapter) 네트워크의 상태를 종합적으로 검증
```
# GPU 및 EFA 검증 스크립트 생성
#!/bin/bash
# GPU/EFA 검증 스크립트 생성
cat > validate_gpu_efa.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=validate
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --gres=gpu:8
#SBATCH --time=00:30:00
#SBATCH --output=logs/validate-%j.out
#SBATCH --error=logs/validate-%j.err

# 로그 디렉토리 생성
mkdir -p logs

echo "======================================"
echo "GPU/EFA Validation Started"
echo "Date: $(date)"
echo "Job ID: $SLURM_JOB_ID"
echo "Nodes: $SLURM_JOB_NODELIST"
echo "======================================"

# 1. GPU 정보 수집 (노드별)
echo -e "\n=== GPU Information ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  nvidia-smi --query-gpu=index,name,memory.total,driver_version --format=csv,noheader
'

# 2. GPU 상세 정보
echo -e "\n=== GPU Topology ==="
srun --ntasks-per-node=1 nvidia-smi topo -m

# 3. CUDA 버전 확인
echo -e "\n=== CUDA Version ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  nvcc --version 2>/dev/null || echo "CUDA not found in PATH"
'

# 4. EFA 어댑터 확인
echo -e "\n=== EFA Adapters ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  fi_info -p efa -t FI_EP_RDM | grep -A 5 "provider:"
'

# 5. EFA 디바이스 확인
echo -e "\n=== EFA Devices ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  ls -la /dev/infiniband/
'

# 6. NVLink 상태 확인
echo -e "\n=== NVLink Status ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  nvidia-smi nvlink -s
'

# 7. GPUDirect RDMA 확인
echo -e "\n=== GPUDirect RDMA Status ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  if [ -e /sys/kernel/mm/memory_peers/nv_mem/version ]; then
    echo "GPUDirect RDMA: Enabled"
    cat /sys/kernel/mm/memory_peers/nv_mem/version
  else
    echo "GPUDirect RDMA: Not found"
  fi
'

# 8. 네트워크 인터페이스 확인
echo -e "\n=== Network Interfaces ==="
srun --ntasks-per-node=1 bash -c '
  echo "Node: $(hostname)"
  ip link show | grep -E "^[0-9]+: (eth|ens|efa)"
'

# 9. 간단한 통신 테스트 (선택사항)
echo -e "\n=== MPI Communication Test ==="
if command -v mpirun &> /dev/null; then
  srun hostname
else
  echo "MPI not available for communication test"
fi

echo -e "\n======================================"
echo "Validation Completed"
echo "Date: $(date)"
echo "======================================"
EOF

# 로그 디렉토리 생성
mkdir -p logs

# 실행 권한 부여
chmod +x validate_gpu_efa.sh

# 작업 제출
echo "Submitting validation job..."
JOB_ID=$(sbatch validate_gpu_efa.sh | awk '{print $4}')

echo "Job ID: $JOB_ID"
echo "Log file: logs/validate-$JOB_ID.out"
echo ""
echo "Monitor with:"
echo "  tail -f logs/validate-$JOB_ID.out"
echo ""
echo "Or check job status:"
echo "  squeue -j $JOB_ID"

```




  - 3.2.3 2 노드 NCCL 테스트
    - NCCL(NVIDIA Collective Communications Library)의 다양한 분산 통신 패턴들의 성능을 검증하는 종합 테스트
    - 이 스크립트는 6개의 다른 NCCL 통신 패턴을 순차적으로 테스트하며, 각각의 결과가 출력
```
cat > nccl_test_all.sh <<'EOF'    # 테스트 스크립트 파일 생성
#!/bin/bash                        # bash 스크립트 선언

#SBATCH --job-name=nccl-test-all  # Slurm 작업 이름
#SBATCH --nodes=2                  # 2개 노드 사용
#SBATCH --ntasks-per-node=8       # 노드당 8개 태스크 (GPU당 1개)
#SBATCH --gres=gpu:8              # 노드당 8개 GPU 요청
#SBATCH --time=01:00:00           # 최대 실행 시간 1시간
#SBATCH --output=/fsx3/logs/nccl-test-%j.out  # 로그 파일 위치

# NCCL 환경변수 설정
export FI_PROVIDER=efa              # EFA 네트워크 사용
export FI_EFA_USE_DEVICE_RDMA=1    # EFA RDMA 활성화
export FI_EFA_FORK_SAFE=1          # Fork 안전성 보장
export NCCL_SOCKET_NTHREADS=4      # 소켓 통신 쓰레드 수
export NCCL_CROSS_NIC=1            # 다중 NIC 사용
export NCCL_MIN_NCHANNELS=32       # 최소 통신 채널 수
export NCCL_MAX_NCHANNELS=64       # 최대 통신 채널 수
export NCCL_P2P_LEVEL=NVL          # NVLink 통신 활성화
export NCCL_NET_GDR_LEVEL=PHB      # GPUDirect RDMA 설정
export NCCL_BUFFSIZE=8388608       # 통신 버퍼 크기
export NCCL_P2P_NET_CHUNKSIZE=524288  # P2P 청크 크기
export NCCL_DEBUG=INFO             # 디버그 정보 출력

# 테스트할 collective operations 배열 정의
TESTS=(
    "all_reduce_perf"              # 그래디언트 평균 계산 테스트
    "all_gather_perf"              # 배치 데이터 공유 테스트
    "broadcast_perf"               # 파라미터 동기화 테스트
    "reduce_perf"                  # 손실 값 집계 테스트
    "reduce_scatter_perf"          # 샤딩된 그래디언트 처리 테스트
    "alltoall_perf"               # 피처 재분배 테스트
)

# 각 테스트 순차적 실행
for test in "${TESTS[@]}"; do
    echo "=== Starting $test ==="  # 테스트 시작 표시
    echo "Date: $(date)"           # 시작 시간 기록
    
    srun --mpi=pmix /opt/nccl-tests/build/$test \  # MPI로 테스트 실행
        -b 8 -e 4G -f 2 -g 1 -n 50                 # 테스트 파라미터
        # -b 8: 시작 크기 8B
        # -e 4G: 최대 크기 4GB
        # -f 2: 2배수로 증가
        # -g 1: GPU당 1프로세스
        # -n 50: 50회 반복
    
    echo "=== Completed $test ===" # 테스트 완료 표시
    echo
done
EOF

sbatch --wait nccl_test_all.sh     # 작업 제출 및 완료 대기
```

  - 3.2.4 2 노드 FSx for Lustre 성능 테스트
   ```
cat > test_fsx_performance.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=fsx-test
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --time=00:30:00
#SBATCH --output=/fsx3/logs/fsx-test-%j.out

echo "=== Starting FSx Performance Tests ==="
date

# 테스트할 FSx 마운트 포인트들
FSX_MOUNTS=("/fsx1" "/fsx2" "/fsx3" "/fsx4")

for mount in "${FSX_MOUNTS[@]}"; do
    echo "=== Testing $mount ==="
    
    # 기본 파일시스템 정보
    echo "Mount info:"
    df -h $mount
    
    # IOR 순차 읽기/쓰기 테스트
    echo "Sequential I/O test:"
    srun ior -a POSIX -w -r \
        -o ${mount}/ior_test \
        -t 1m -b 16m -s 100 -F
    
    # mdtest 메타데이터 테스트
    echo "Metadata test:"
    srun mdtest -n 1000 -d ${mount}/mdtest
    
    echo "=== Completed testing $mount ==="
    echo
done

echo "=== All FSx tests completed ==="
date
EOF
```


- 3.3 중규모 노드 테스트 (10 노드)
  - 3.3.1 위 테스트 반복  

### Phase 4: 프로덕션 트레이닝(예시)
- 4.1 배치 스크립트
```
#!/bin/bash

#SBATCH --job-name=model1-training    # 작업 이름
#SBATCH --nodes=80                    # 전체 80개 노드 사용
#SBATCH --ntasks-per-node=8          # 노드당 8개 태스크
#SBATCH --gres=gpu:8                 # 노드당 8개 GPU
#SBATCH --cpus-per-task=12           # 태스크당 12개 CPU 코어
#SBATCH --partition=gpu-queue        
#SBATCH --output=/fsx3/model1/logs/%j.out  
#SBATCH --error=/fsx3/model1/logs/%j.err   
#SBATCH --exclusive                   
#SBATCH --time=7-00:00:00            
#SBATCH --container-image=YOUR_ACCOUNT_ID.dkr.ecr.ap-southeast-3.amazonaws.com/my-training
#SBATCH --container-mounts=/fsx1:/fsx1,/fsx2:/fsx2,/fsx3:/fsx3,/fsx4:/fsx4

# 초기 설정
set -e
set -x

# 디렉토리 생성
mkdir -p /fsx3/model1/logs
mkdir -p /fsx3/model1/gpu_stats

# EFA/NCCL 환경 설정
export PATH=$PATH:/opt/amazon/efa/bin
export LD_LIBRARY_PATH=/opt/amazon/openmpi/lib:/opt/amazon/efa/lib:/opt/aws-ofi-nccl/install/lib:$LD_LIBRARY_PATH

# EFA 설정
export FI_PROVIDER=efa
export FI_EFA_USE_DEVICE_RDMA=1
export FI_EFA_FORK_SAFE=1
export FI_EFA_ENABLE_SHM_TRANSFER=1

# NCCL 설정 (80노드 최적화)
export NCCL_SOCKET_NTHREADS=4
export NCCL_CROSS_NIC=1
export NCCL_IFNAME=eth0,eth1,eth2,eth3
export NCCL_MIN_NCHANNELS=128
export NCCL_MAX_NCHANNELS=256
export NCCL_P2P_LEVEL=NVL
export NCCL_NET_GDR_LEVEL=PHB
export NCCL_BUFFSIZE=16777216
export NCCL_P2P_NET_CHUNKSIZE=1048576
export NCCL_TREE_THRESHOLD=2048
export NCCL_ALGO=Tree,Ring
export NCCL_COMM_TIMEOUT_SECONDS=3600
export NCCL_TUNER_PLUGIN=/opt/aws-ofi-nccl/install/lib/libnccl-ofi-tuner.so
export NCCL_DEBUG=WARN
export NCCL_DEBUG_FILE=/fsx3/model1/logs/nccl-rank-%r.log

# MPI 설정
export OMPI_MCA_pml=^cm,ucx
export OMPI_MCA_btl=tcp,self
export OMPI_MCA_btl_tcp_if_exclude=lo,docker0

# PyTorch 분산 설정
export MASTER_ADDR=$(scontrol show hostname $SLURM_JOB_NODELIST | head -n 1)
export MASTER_PORT=29500
export WORLD_SIZE=$SLURM_NTASKS
export RANK=$SLURM_PROCID
export LOCAL_RANK=$SLURM_LOCALID
export CUDA_DEVICE_ORDER=PCI_BUS_ID
export CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

# GPU 모니터링 함수
monitor_gpu() {
    while true; do
        nvidia-smi --query-gpu=timestamp,name,utilization.gpu,memory.used,memory.total,temperature.gpu \
                  --format=csv,noheader \
        >> /fsx3/model1/gpu_stats/gpu_stats_${SLURM_NODEID}_${SLURM_PROCID}.csv
        sleep 60
    done
}

# 훈련 시작
echo "=== Training Start ==="
echo "Start time: $(date)"
echo "Total nodes: $SLURM_JOB_NUM_NODES"
echo "GPUs per node: 8"
echo "Total GPUs: $((SLURM_JOB_NUM_NODES * 8))"
echo "Master node: $MASTER_ADDR"

# GPU 모니터링 시작
monitor_gpu &
MONITOR_PID=$!

# 메인 훈련 실행
srun python train.py \
    --data-dir /fsx1/datasets/model1 \
    --checkpoint-dir /fsx2/model1/checkpoints \
    --log-dir /fsx3/model1/logs \
    --output-dir /fsx2/model1/outputs \
    --model-name model1 \
    --batch-size 32 \
    --learning-rate 3e-4 \
    --epochs 100 \
    --checkpoint-freq 5 \
    --distributed-backend nccl

EXIT_CODE=$?

# GPU 모니터링 중지
kill $MONITOR_PID

# 훈련 완료 처리
echo "=== Training Complete ==="
echo "End time: $(date)"
echo "Exit code: $EXIT_CODE"

# 성공 시 체크포인트 백업
if [ $EXIT_CODE -eq 0 ] && [ $SLURM_PROCID -eq 0 ]; then
    echo "Backing up final checkpoint to S3..."
    aws s3 sync /fsx2/model1/checkpoints/final/ \
        s3://checkpoint-jakarta/model1/final/ \
        --region ap-southeast-3
fi

# 임시 파일 정리
if [ $SLURM_PROCID -eq 0 ]; then
    find /fsx4 -name "*.tmp" -type f -mtime +1 -delete
fi

exit $EXIT_CODE
```


# Q&A
## YAML 파일 자체에서는 FSx Lustre의 디렉토리 구조를 사전 정의할 수 있는지?
- A: YAML 파일 자체에서는 FSx Lustre의 디렉토리 구조를 사전 정의할 수 없습니다. 대신 두 가지 접근 방법이 있습니다:
- 이렇게 구성하면 FSx 스토리지 생성과 디렉토리 구조 설정이 클러스터 생성 시 자동생성
```
1. CustomActions 사용:
# 장점: 클러스터 생성되었을 때 자동으로 사용자가 원하는 스토리지 구조를 만드는 것이 가능
# 단점: 설치에 너무 많은 시간이 소요되어 HeadNodeWaitCondition 타임 아웃 발생 가능성 존재, 설정 에러
   HeadNode:
     CustomActions:
       OnNodeConfigured:
         Script: s3://your-bucket/scripts/setup_dirs.sh

2. 각 FSx 마운트 포인트에 용도별 구분:
# 장점: 상대적으로 빠르게 클러스터 생성 가능, 구성 단순, 설정 실패 위험 감소
# 단점: 스토리지 구조는 클러스터 생성 후 수동으로 설정해야 함 
   SharedStorage:
     - MountDir: /fsx1          # 데이터셋용
       Name: fsx-data
       ...
     
     - MountDir: /fsx2          # 체크포인트용
       Name: fsx-checkpoints
       ...
     
     - MountDir: /fsx3          # 로그용
       Name: fsx-logs
       ...
```

```
# 이런 FSx for Lustre 스토리지 정책을 원한다면, 아래 스크립트 파일 사용

/fsx1/
├── shared_datasets/     # 공유 데이터셋
└── model1/
    ├── raw_data/       # 원본 데이터
    ├── processed_data/ # 전처리 데이터
    ├── cache/         # 캐시
    └── temp/          # 임시 데이터
└── model2/
    ...

/fsx2/
├── model1/
│   ├── checkpoints/
│   │   ├── latest/
│   │   └── archive/
│   └── outputs/
│       ├── eval/
│       └── predictions/
└── model2/
    ...

/fsx3/
├── model1/
│   ├── training_logs/
│   ├── tensorboard/
│   ├── metrics/
│   └── debug/
└── model2/
    ...
```

```
# setup_dirs.sh
#!/bin/bash

# 로그 파일 설정
LOGFILE="/var/log/parallelcluster/setup_dirs.log"
mkdir -p /var/log/parallelcluster
exec 1> >(tee -a "$LOGFILE") 2>&1
echo "Starting directory setup at $(date)"

# FSx 마운트 포인트가 준비될 때까지 대기
echo "Checking FSx mount points..."
for i in {1..30}; do  # 최대 5분 대기 (10초 × 30)
    if mountpoint -q /fsx1 && mountpoint -q /fsx2 && mountpoint -q /fsx3; then
        echo "All FSx mount points are ready"
        break
    fi
    echo "Waiting for FSx mount points... Attempt $i/30"
    sleep 10
done

# 마운트 상태 최종 확인
if ! mountpoint -q /fsx1 || ! mountpoint -q /fsx2 || ! mountpoint -q /fsx3; then
    echo "Error: FSx mount points not ready after timeout"
    exit 1
fi

# FSx 마운트 정보 기록
echo "FSx mount details:"
df -h | grep fsx

# 디렉토리 구조 생성
echo "Creating directory structure..."

# FSx1 (데이터셋) 구조 생성
echo "Setting up FSx1 (datasets)..."
mkdir -p /fsx1/shared_datasets
chmod 755 /fsx1/shared_datasets

for i in {1..7}; do
    mkdir -p /fsx1/model${i}/{raw_data,processed_data,cache,temp}
    chmod -R 750 /fsx1/model${i}
done

# FSx2 (체크포인트) 구조 생성
echo "Setting up FSx2 (checkpoints)..."
for i in {1..7}; do
    mkdir -p /fsx2/model${i}/{checkpoints/{latest,archive},outputs/{eval,predictions}}
    chmod -R 750 /fsx2/model${i}
done

# FSx3 (로그) 구조 생성
echo "Setting up FSx3 (logs)..."
for i in {1..7}; do
    mkdir -p /fsx3/model${i}/{training_logs,tensorboard,metrics,debug}
    chmod -R 700 /fsx3/model${i}
done

# 소유권 설정
echo "Setting ownership..."
chown -R ec2-user:ec2-user /fsx1
chown -R ec2-user:ec2-user /fsx2
chown -R ec2-user:ec2-user /fsx3

# 스트라이핑 설정
echo "Setting Lustre striping..."
# 큰 파일을 위한 스트라이핑 (체크포인트)
lfs setstripe --stripe-count 8 --stripe-size 4M /fsx2/*/checkpoints
# 중간 크기 파일을 위한 스트라이핑 (처리된 데이터)
lfs setstripe --stripe-count 4 --stripe-size 1M /fsx1/*/processed_data
# 작은 파일을 위한 스트라이핑 (로그)
lfs setstripe --stripe-count 1 --stripe-size 1M /fsx3/*/training_logs

# 디렉토리 구조 확인
echo "Created directory structure:"
echo "FSx1 (datasets):"
tree -L 3 /fsx1
echo "FSx2 (checkpoints/outputs):"
tree -L 4 /fsx2
echo "FSx3 (logs):"
tree -L 3 /fsx3

echo "Directory setup completed at $(date)"

# 권한 확인
echo "Verifying permissions..."
ls -la /fsx1
ls -la /fsx2
ls -la /fsx3

echo "Setup completed successfully"
```

```
# 각 모델을 훈련시킬 때 SLURM 스크립트에서 고유한 체크포인트 경로를 지정
#!/bin/bash
#SBATCH --job-name=model1_training
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --gres=gpu:8
#SBATCH --partition=gpu-queue

# 모델별 디렉토리 설정
MODEL_ID=1

# 수정된 경로들
SHARED_DATA_DIR="/fsx1/shared_datasets"                    # 공유 데이터셋
PROCESSED_DATA_DIR="/fsx1/model${MODEL_ID}/processed_data" # 전처리된 데이터
CHECKPOINT_DIR="/fsx2/model${MODEL_ID}/checkpoints/latest" # 최신 체크포인트
OUTPUT_DIR="/fsx2/model${MODEL_ID}/outputs"                # 출력
LOG_DIR="/fsx3/model${MODEL_ID}/training_logs"            # 학습 로그
TENSORBOARD_DIR="/fsx3/model${MODEL_ID}/tensorboard"      # Tensorboard

# 분산 훈련 실행
srun python train.py \
    --shared-data-dir ${SHARED_DATA_DIR} \
    --processed-data-dir ${PROCESSED_DATA_DIR} \
    --checkpoint-dir ${CHECKPOINT_DIR} \
    --output-dir ${OUTPUT_DIR} \
    --log-dir ${LOG_DIR} \
    --tensorboard-dir ${TENSORBOARD_DIR} \
    --model-name "model${MODEL_ID}"
```


```

## ParallelCluster 참고 링크
- Launch instances with On-Demand Capacity Reservations (ODCR)
  - https://docs.aws.amazon.com/parallelcluster/latest/ug/launch-instances-odcr-v3.html
-   Launch instances with Capacity Blocks (CB)
  - https://docs.aws.amazon.com/parallelcluster/latest/ug/launch-instances-capacity-blocks.html

