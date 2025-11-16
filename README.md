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
Region: ap-southeast-3

Image:
  Os: alinux2

HeadNode:
  InstanceType: c5.9xlarge
  LocalStorage:
    RootVolume:
      Size: 100
      VolumeType: gp3
      Iops: 10000
      Snapshot: true  # 헤드노드 백업 활성화
  
  Networking:
    SubnetId: subnet-xxxxxxxxx  # PUBLIC_SUBNET
    SecurityGroups:
      - sg-internal-xxxxxx  # INTERNAL_SG
    AdditionalSecurityGroups:
      - sg-ssh-xxxxxx  # SSH_SG
      - sg-grafana-xxxxxx  # GRAFANA_SG

  Ssh:
    KeyName: p5e-cluster-key

  Iam:
    S3Access:
      - BucketName: training-data-jakarta
        EnableWriteAccess: false
      - BucketName: checkpoint-jakarta
        EnableWriteAccess: true
      - BucketName: logs-jakarta
        EnableWriteAccess: true
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
      - Policy: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  CustomActions:
    OnNodeConfigured:
      Script: s3://scripts-jakarta/setup_headnode.sh

Scheduling:
  Scheduler: slurm
  ScalingStrategy: all-or-nothing
  SlurmSettings:
    ScaledownIdletime: 30  # 30분으로 증가
    EnableMemoryBasedScheduling: true
    CustomSlurmSettings:
      - SelectType: select/cons_tres
      - SelectTypeParameters: CR_Core_Memory
      - ResumeTimeout: 1800
      - SuspendTimeout: 300
      - SuspendTime: 300
      - ResumeRate: 10
      - SuspendRate: 10

SlurmQueues:
  - Name: gpu-queue
    CapacityType: ONDEMAND
    ComputeSettings:
      LocalStorage:
        RootVolume:
          Size: 200
          VolumeType: gp3
          Iops: 16000
          Throughput: 1000
          Snapshot: true
    
    Networking:
      SubnetIds:
        - subnet-yyyyyyyyy1  # PRIVATE_SUBNET_1
        - subnet-yyyyyyyyy2  # PRIVATE_SUBNET_2
      SecurityGroups:
        - sg-internal-xxxxxx  # INTERNAL_SG

    HealthChecks:
      Gpu:
        Enabled: true
      FileSystem:
        Enabled: true
      
    ComputeResources:
      - Name: p5e-training
        InstanceType: p5e.48xlarge
        MinCount: 0
        MaxCount: 80
        DisableSimultaneousMultithreading: false
        Efa:
          Enabled: true
          GdrSupport: true
        CapacityReservationTarget:
          CapacityReservationId: cr-xxxxxxxxxxxxx  # ODCR
        Networking:
          PlacementGroup:
            Enabled: true
            Name: pg-training

    Iam:
      S3Access:
        - BucketName: training-data-jakarta
          EnableWriteAccess: false
        - BucketName: checkpoint-jakarta
          EnableWriteAccess: true
      AdditionalIamPolicies:
        - Policy: arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - Policy: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

    CustomActions:
      OnNodeConfigured:
        Script: s3://scripts-jakarta/setup_compute_node.sh

SharedStorage:
  - MountDir: /fsx1
    Name: fsx-data-1
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 14400
      DeploymentType: PERSISTENT_1
      StorageType: SSD
      PerUnitStorageThroughput: 200
      DataCompressionType: LZ4
      ImportPath: s3://training-data-jakarta/datasets
      AutoImportPolicy: NEW_CHANGED
      BackupRetentionDays: 7
      AutomaticBackupRetentionDays: 7
      DailyAutomaticBackupStartTime: "00:00"

  - MountDir: /fsx2
    Name: fsx-checkpoint
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 14400
      DeploymentType: PERSISTENT_1
      StorageType: SSD
      PerUnitStorageThroughput: 200
      DataCompressionType: LZ4
      ExportPath: s3://checkpoint-jakarta
      AutoExportPolicy: NEW_CHANGED
      BackupRetentionDays: 7
      AutomaticBackupRetentionDays: 7
      DailyAutomaticBackupStartTime: "01:00"

  - MountDir: /fsx3
    Name: fsx-logs
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 7200
      DeploymentType: PERSISTENT_1
      StorageType: SSD
      PerUnitStorageThroughput: 200
      BackupRetentionDays: 3
      AutomaticBackupRetentionDays: 3
      DailyAutomaticBackupStartTime: "02:00"

  - MountDir: /fsx4
    Name: fsx-scratch
    StorageType: FsxLustre
    FsxLustreSettings:
      StorageCapacity: 7200
      DeploymentType: PERSISTENT_1
      StorageType: SSD
      PerUnitStorageThroughput: 200
      BackupRetentionDays: 1
      AutomaticBackupRetentionDays: 1
      DailyAutomaticBackupStartTime: "03:00"

Monitoring:
  DetailedMonitoring: true
  Logs:
    CloudWatch:
      Enabled: true
      RetentionInDays: 30
    Rotation:
      Enabled: true
      RetentionInDays: 15
  
  Dashboards:
    CloudWatch:
      Enabled: true

Tags:
  - Key: Project
    Value: DistributedTraining
  - Key: Environment
    Value: Production
  - Key: Owner
    Value: YourTeam
```
- 주요 yaml 특징
```
1. 컴퓨팅:
   └── 단일 ComputeResource로 80대 P5e 관리

2. 스토리지:
   ├── 훈련 데이터: 14.4TB
   ├── 체크포인트: 14.4TB
   ├── 로그: 7.2TB
   └── 스크래치: 7.2TB

3. 네트워킹:
   ├── EFA 활성화
   └── 단일 플레이스먼트 그룹

4. 안정성:
   ├── 자동 백업
   ├── 헬스체크
   └── 상세 모니터링
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









## 주요 기능

- 다중 GPU 분산 학습 지원
- NCCL 기반 고성능 통신
- SLURM 워크로드 관리
- EFA 네트워킹 최적화

## 개발 환경

- **클라우드**: AWS ParallelCluster
- **인스턴스**: P5e.48xlarge
- **OS**: Amazon Linux 2 / Ubuntu
- **프레임워크**: PyTorch
- **통신**: NCCL, MPI

## 설치 방법

필요한 패키지 설치
pip install -r requirements.txt

환경 변수 설정
export NCCL_DEBUG=INFO


## 사용 방법
단일 노드 실행
python train.py

분산 학습 실행
sbatch distributed_train.s

## 프로젝트 구조
```
ml-training/
├── train.py # 학습 스크립트
├── config.yaml # 설정 파일
├── requirements.txt # 필요 패키지
└── README.md # 프로젝트 문서
```

## 참고 자료

- [AWS ParallelCluster 문서](https://docs.aws.amazon.com/parallelcluster/)
- [PyTorch 분산 학습 가이드](https://pytorch.org/tutorials/beginner/dist_overview.html)

## 라이선스

MIT License

웹에서 문서 올리는 방법
화면에 보이는 편집기에서 바로 작업하시면 됩니다:​

1단계: 파일명 입력
"Name your file..." 입력란에 README.md 입력

2단계: 내용 작성
아래 편집 영역에 위의 샘플 내용을 복사하여 붙여넣기

필요에 따라 내용을 수정

3단계: 미리보기 확인
"Preview" 탭을 클릭하여 작성한 내용이 어떻게 보이는지 확인

4단계: 커밋(저장)
우측 상단의 "Commit changes..." 버튼 클릭

커밋 메시지 입력 (예: "첫 번째 README 파일 추가")

"Commit changes" 버튼을 눌러 최종 저장​

5단계: 확인
저장 후 자동으로 레포지토리 메인 페이지로 이동

README.md 파일이 자동으로 화면에 표시됨​

이 방법은 Git 명령어를 전혀 몰라도 웹 브라우저에서 바로 파일을 작성하고 업로드할 수 있는 가장 간단한 방법입니다.​
