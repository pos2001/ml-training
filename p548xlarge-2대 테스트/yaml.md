
### https://catalog.workshops.aws/ml-on-aws-parallelcluster/en-US/03-cluster/02-setup-cluster => 기반으로 하면 문제 발생하여 아래 버전 사용

### 이 스크립트를 이용하면 NCCL이 헤드노드와 컴퓨트노드에 설치가 안됨
```
가능한 원인:

1. 노드가 스크립트 실행 전에 생성됨

    YAML에 스크립트 추가 전에 클러스터 생성
    기존 노드는 새 설정 적용 안 됨

2. CustomActions 실행 실패

    네트워크 문제로 스크립트 다운로드 실패
    조용히 실패 (에러 로그 없음)

3. OnNodeConfigured 타이밍 문제

    노드 설정 단계에서 스크립트 실행 누락
```

```
# Copyright Amazon.com, Inc. or its affiliates. All Rights Reserved.
# SPDX-License-Identifier: MIT-0

Imds:
  ImdsSupport: v2.0
Image:
  Os: ubuntu2204
HeadNode:
  InstanceType: m5.8xlarge
  Networking:
    SubnetId: subnet-0130ab15fccb143fe
    AdditionalSecurityGroups:
      - sg-006ede02c2d0d293e
  LocalStorage:
    RootVolume:
      Size: 500
      DeleteOnTermination: true
  Iam:
    AdditionalIamPolicies:
      - Policy: arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      - Policy: arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
      - Policy: arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly
      - Policy: arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess
  CustomActions:
    OnNodeConfigured:
      Sequence:
        - Script: 'https://raw.githubusercontent.com/aws-samples/aws-parallelcluster-post-install-scripts/main/docker/postinstall.sh'
        - Script: 'https://raw.githubusercontent.com/aws-samples/aws-parallelcluster-post-install-scripts/main/nccl/postinstall.sh'
          Args:
            - v2.23.4-1
            - v1.11.0-aws
  Imds:
    Secured: false
Scheduling:
  Scheduler: slurm
  SlurmSettings:
    ScaledownIdletime: 60
    QueueUpdateStrategy: DRAIN
    EnableMemoryBasedScheduling: true
    CustomSlurmSettings:
      # Timeout settings - 증가된 값으로 노드 안정성 향상
      - SlurmdTimeout: 600          # 10분 (기본 180초에서 증가)
      - MessageTimeout: 120         # 2분 (기본 60초에서 증가)
      - SuspendTimeout: 600         # 10분
      - UnkillableStepTimeout: 300  # 5분
      # Debug logging - 문제 진단용
      - SlurmctldDebug: info
      - SlurmdDebug: info
  SlurmQueues:
    - Name: compute-gpu
      CapacityType: CAPACITY_BLOCK
      CapacityReservationTarget:
        CapacityReservationId: cr-0eaf243f6d60fcc9a
      Networking:
        SubnetIds:
          - subnet-0b578c46a39edb5d3
        PlacementGroup:
          Enabled: false
        AdditionalSecurityGroups:
          - sg-006ede02c2d0d293e
      ComputeSettings:
        LocalStorage:
          EphemeralVolume:
            MountDir: /scratch
          RootVolume:
            Size: 200
      JobExclusiveAllocation: true
      ComputeResources:
        - Name: distributed-ml
          InstanceType: p5.48xlarge
          MinCount: 2  # Capacity Block은 MinCount = MaxCount 필수
          MaxCount: 2
          Efa:
            Enabled: true
      CustomActions:
        OnNodeConfigured:
          Sequence:
            # Docker 설치
            - Script: 'https://raw.githubusercontent.com/aws-samples/aws-parallelcluster-post-install-scripts/main/docker/postinstall.sh'
            # NCCL 설치
            - Script: 'https://raw.githubusercontent.com/aws-samples/aws-parallelcluster-post-install-scripts/main/nccl/postinstall.sh'
              Args:
                - v2.23.4-1
                - v1.11.0-aws
SharedStorage:
  - Name: HomeDirs
    MountDir: /home
    StorageType: FsxOpenZfs
    FsxOpenZfsSettings:
      VolumeId: fsvol-07748ca1eb68e3e1c
  - MountDir: /fsx
    Name: fsx
    StorageType: FsxLustre
    FsxLustreSettings:
      FileSystemId: fs-02fe3015d20156a04
Monitoring:
  DetailedMonitoring: true
  Logs:
    CloudWatch:
      Enabled: true
      RetentionInDays: 7
  Dashboards:
    CloudWatch:
      Enabled: true
Tags:
  - Key: 'Grafana'
    Value: 'true'
```
