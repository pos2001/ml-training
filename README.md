# ML Training Project

AWS ParallelCluster 환경에서 대규모 분산 학습을 수행하기 위한 프로젝트입니다.

## 프로젝트 개요

이 프로젝트는 GPU 클러스터를 활용한 머신러닝 모델 학습을 위한 환경 설정과 코드를 포함합니다.

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
