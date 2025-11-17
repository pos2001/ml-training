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
  - 노드 할당: 모델 1-6 각 11노드, 모델 7은 14노드
  - 예상 훈련 기간: 모델당 1-2주

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
- 1.2 SG 설정
  - 헤드 노드와 컴퓨트 노드간 통신
  - 헤드 노드 SSH 접속용
  - Grafana 접근용 SG(설치한다면)
- 1.3 키페어 설정(헤드 노드 접속용)

### Phase 2: 클러스터 배포(예시)
- 2.1 ParallelCluster 설치(최신 버전 권장)
  -  https://github.com/aws/aws-parallelcluster/releases
- 2.2 Yaml 설정 파일 작성
```
Region: ap-southeast-3                # 자카르타 리전 지정

Image:
  Os: alinux2                        # Amazon Linux 2 운영체제

HeadNode:
  InstanceType: c6i.4xlarge          # 헤드노드용 컴퓨트 최적화 인스턴스
  Networking:
    SubnetId: subnet-xxxxxxxxx       # 헤드노드용 퍼블릭 서브넷
  Ssh:
    KeyName: your-keypair-name       # SSH 접속용 키페어
  Iam:
    AdditionalIamPolicies:           # 헤드노드 추가 IAM 정책
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore  # SSM 접속용

Scheduling:
  Scheduler: slurm                   # Slurm 스케줄러 사용
  ScalingStrategy: best-effort       # 최선노력 스케일링 전략
  SlurmSettings:
    ScaledownIdletime: 10           # 10분 유휴 후 스케일다운


SlurmQueues:
  - Name: gpu-queue                  # GPU 작업 전용 큐
    CapacityType: ONDEMAND          # 온디맨드 인스턴스 사용
    AllocationStrategy: lowest-price # 최저가 할당 전략
    ComputeSettings:
      LocalStorage:
        RootVolume:                  # 컴퓨트 노드 루트 볼륨 설정
          Size: 200                  # 200GB 크기
          VolumeType: gp3           # gp3 타입 사용
          Iops: 16000               # 16000 IOPS
          Throughput: 1000          # 1000 MB/s 처리량
    
    Networking:
      SubnetIds:
        - subnet-yyyyyyyyy          # 컴퓨트 노드용 프라이빗 서브넷
      PlacementGroup:
        Enabled: true               # 네트워크 성능 최적화를 위한 배치 그룹
      SecurityGroups:
        - sg-zzzzzzzzz             # 컴퓨트 노드 보안 그룹

    HealthChecks:
      Gpu:
        Enabled: true              # GPU 상태 모니터링 활성화

    ComputeResources:
      - Name: p5e-nodes            # P5e 노드 그룹
        InstanceType: p5e.48xlarge # P5e 인스턴스 타입 (8 GPU)
        MinCount: 0                # 최소 0개 노드
        MaxCount: 80               # 최대 80개 노드 (ODCR 예약 수량)
        DisableSimultaneousMultithreading: false  # SMT 유지
        Efa:
          Enabled: true            # EFA 네트워킹 활성화
        CapacityReservationTarget:
          CapacityReservationId: cr-xxxxxxxxxxxxx  # ODCR 예약 ID

    CustomActions:
      OnNodeStart:
        Script: s3://your-bucket/scripts/node-setup.sh  # 노드 시작 시 실행할 스크립트

Iam:
  AdditionalIamPolicies:            # 컴퓨트 노드 추가 IAM 정책
    - Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess  # S3 읽기 권한

SharedStorage:
  - MountDir: /fsx1                 # 학습 데이터용 FSx
    Name: fsx-data
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 7200        # 7.2TB 용량
      DeploymentType: PERSISTENT   # Jakarta 리전은 PERSISTENT만 지원
      PerUnitStorageThroughput: 1000  # 1000MB/s 처리량
      DataCompressionType: LZ4     # LZ4 압축 사용

  - MountDir: /fsx2                # 체크포인트용 FSx
    Name: fsx-checkpoints
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 7200        # 7.2TB 용량
      DeploymentType: PERSISTENT
      PerUnitStorageThroughput: 1000  # 1000MB/s 처리량
      DataCompressionType: LZ4

  - MountDir: /fsx3                # 로그용 FSx
    Name: fsx-logs
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 4800        # 4.8TB 용량
      DeploymentType: PERSISTENT
      PerUnitStorageThroughput: 500   # 500MB/s 처리량
      DataCompressionType: LZ4

Monitoring:
  DetailedMonitoring: true         # EC2 상세 모니터링 활성화
  Logs:
    CloudWatch:
      Enabled: true                # CloudWatch 로그 활성화
      RetentionInDays: 14         # 로그 14일 보관

  Dashboards:
    CloudWatch:
      Enabled: true               # CloudWatch 대시보드 활성화

Tags:
  - Key: Project                   # 프로젝트 태그
    Value: DistributedTraining
  - Key: Environment               # 환경 태그
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

- 2.3 Yaml 검증
```
# 1. YAML 문법과 설정 검증
pcluster configure --config cluster-config.yaml

# 2. 클러스터 생성 시뮬레이션 (실제 생성하지 않음)
pcluster create-cluster \
    --cluster-name p5e-80node-cluster \
    --cluster-configuration cluster-config.yaml \
    --region ap-southeast-3 \
    --dryrun
```
- 2.4 클러스터 생성
```
# 클러스터 생성 (30-60분 소요)
pcluster create-cluster \
--cluster-name p5e-80node-cluster \
--cluster-configuration cluster-config.yaml \
--region ap-southeast-3
# 실시간 상태 확인
watch -n 10 'pcluster describe-cluster \
--cluster-name p5e-80node-cluster \
--region ap-southeast-3 \
--query "clusterStatus"'
```
```
# 클러스터 생성 명령어 설명
1. 클러스터 생성:
   ├── 30-60분 소요
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

# 2. 접속 방법 (세 가지 중 선택)

# 방법 1: pcluster ssh 사용
pcluster ssh \
    --cluster-name p5e-80node-cluster \
    --region ap-southeast-3 \
    -i ~/.ssh/p5e-cluster-key.pem

# 방법 2: 일반 ssh 사용
ssh -i ~/.ssh/p5e-cluster-key.pem ec2-user@$HEAD_NODE_IP

# 방법 3: EC2 콘솔에서 헤드노드를 Session Manager를 통해 접근(가장 간편)
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
  - 3.2.1 2노드 기본 테스트  
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


     - 3.2.1.1 2노드 기본 테스트 예상 결과
```
1. 초기 상태:
   PARTITION  AVAIL  NODES  STATE
   gpu-queue   up     0     none

2. 노드 시작 중:
   PARTITION  AVAIL  NODES  STATE
   gpu-queue   up     2     alloc+

3. 작업 실행 중:
   PARTITION  AVAIL  NODES  STATE
   gpu-queue   up     2     allocated

4. 작업 완료 후:
   PARTITION  AVAIL  NODES  STATE
   gpu-queue   up     2     idle~
```
  
  - 3.2.2 2노드 GPU 및 EFA 검증 
```
# GPU 및 EFA 검증 스크립트 생성
cat > validate_gpu_efa.sh <<'EOF'
#!/bin/bash
#SBATCH --job-name=validate
#SBATCH --nodes=2
#SBATCH --ntasks-per-node=8
#SBATCH --gres=gpu:8
#SBATCH --time=00:30:00
#SBATCH --output=/fsx3/logs/validate-%j.out

echo "=== GPU Information ==="
srun nvidia-smi --query-gpu=name,memory.total --format=csv

echo -e "\n=== EFA Status ==="
srun fi_info -p efa -t FI_EP_RDM

echo -e "\n=== GPUDirect RDMA Status ==="
srun nvidia-smi nvlink --status
EOF

# 작업 제출
sbatch validate_gpu_efa.sh

# 로그 모니터링
tail -f /fsx3/logs/validate-*.out
```

  - 3.2.3 2노드 NCCL 테스트
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

  - 3.2.4 2노드 FSx for Lustre 성능 테스트
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
```
1. CustomActions 사용:
   HeadNode:
     CustomActions:
       OnNodeConfigured:
         Script: s3://your-bucket/scripts/setup_dirs.sh

2. 각 FSx 마운트 포인트에 용도별 구분:
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
/fsx2/                    # 체크포인트용 FSx
├── model1/
│   ├── checkpoints/
│   └── outputs/
├── model2/
│   ├── checkpoints/
│   └── outputs/
...
└── model7/
    ├── checkpoints/
    └── outputs/
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

# 데이터셋 디렉토리 (/fsx1)
mkdir -p /fsx1/datasets
chmod 775 /fsx1/datasets

# 모델별 디렉토리 생성 (/fsx2, /fsx3)
for i in {1..7}; do
    # 체크포인트 및 출력 디렉토리
    mkdir -p /fsx2/model${i}/{checkpoints,outputs}
    chmod -R 775 /fsx2/model${i}

    # 로그 디렉토리
    mkdir -p /fsx3/model${i}/{training_logs,tensorboard}
    chmod -R 775 /fsx3/model${i}
done

# 소유권 설정
echo "Setting ownership..."
chown -R ec2-user:ec2-user /fsx1
chown -R ec2-user:ec2-user /fsx2
chown -R ec2-user:ec2-user /fsx3

# 디렉토리 구조 확인
echo "Created directory structure:"
echo "FSx1 (datasets):"
tree -L 2 /fsx1
echo "FSx2 (checkpoints/outputs):"
tree -L 3 /fsx2
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
CHECKPOINT_DIR="/fsx2/model${MODEL_ID}/checkpoints"
OUTPUT_DIR="/fsx2/model${MODEL_ID}/outputs"
LOG_DIR="/fsx3/model${MODEL_ID}/training_logs"
TENSORBOARD_DIR="/fsx3/model${MODEL_ID}/tensorboard"

# 훈련 데이터 경로
DATA_DIR="/fsx1/datasets"

# 분산 훈련 실행
srun python train.py \
    --checkpoint-dir ${CHECKPOINT_DIR} \
    --output-dir ${OUTPUT_DIR} \
    --log-dir ${LOG_DIR} \
    --tensorboard-dir ${TENSORBOARD_DIR} \
    --data-dir ${DATA_DIR} \
    --model-name "model${MODEL_ID}"
```

## 하나의 클러스터에 여러개의 모델을 훈련시킬 때PARALLELCLUSTER YAML 파일 입장에서 고려해야할 사항은?
- A: 메모리 기반 스케줄링 활성화
-   여러 작업이 동일한 노드를 공유할 수 있도록 메모리 기반 스케줄링을 활성화
```
Scheduling:
  Scheduler: slurm                    # Slurm 스케줄러 사용
  SlurmSettings:
    EnableMemoryBasedScheduling: true # 메모리 기반 스케줄링 활성화 (GPU 공유 가능)
    ScaledownIdletime: 10            # 10분 유휴 후 스케일다운
```

```
# 이 설정이 없으면 작업이 GPU 일부만 사용하더라도 전체 노드를 점유
활용 시나리오:
1. 효율적인 리소스 관리:
   ├── 대규모 모델: 여러 GPU/노드 사용
   ├── 중소형 모델: GPU 일부만 사용
   └── 동시에 여러 실험/모델 학습 가능

2. 리소스 최적화:
   예시 1) 노드 당 작업 분할
   ├── 대형 모델A: 4 GPU
   └── 중형 모델B: 4 GPU
   
   예시 2) 다양한 구성
   ├── 노드1: 모델A(8 GPU)
   ├── 노드2: 모델B(4 GPU) + 모델C(4 GPU)
   └── 노드3: 모델D(2 GPU) + 모델E(3 GPU) + 모델F(3 GPU)
```

## 분산 트레이닝에서 스토리지 정책은?
- 원본 훈련 데이터셋
- 체크 포인트
- 모델 가중치
- 옵티마이저 선택
- 로그 및 메트릭
- 중간 결과 및 임시 파일
