
### NCCL Tests는 NVIDIA에서 제공하는 벤치마크 및 검증 도구입니다:

### NCCL Tests 빌드
```
# 환경 설정 로드
source /fsx/gpu_env.sh

# NCCL 테스트 디렉토리로 이동
cd /fsx/nccl-tests

# 빌드 (헤드노드에서 실행 - 컴파일만)
echo "NCCL 테스트 빌드 시작..."
make MPI=1 \
  MPI_HOME=/opt/amazon/openmpi \
  NCCL_HOME=/opt/nccl/build \
  CUDA_HOME=/usr/local/cuda-12.4 \
  -j $(nproc)

# 빌드 확인
echo ""
echo "빌드 결과 확인:"
ls -lh /fsx/nccl-tests/build/
```

### 빌드가 끝나면 다음 확인
```
# 빌드 완료 후 확인
echo ""
echo "빌드 결과 확인:"
ls -lh /fsx/nccl-tests/build/

# 주요 실행 파일들
ls -lh /fsx/nccl-tests/build/*_perf

```


### 빌드 완료 확인
```
# 빌드 완료 후 확인
echo ""
echo "빌드 결과 확인:"
ls -lh /fsx/nccl-tests/build/

# 주요 실행 파일들
ls -lh /fsx/nccl-tests/build/*_perf

빌드 결과 확인:
total 59M
-rw-rw-r-- 1 ubuntu ubuntu  87K Dec  4 02:46 all_gather.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 all_gather_perf
-rw-rw-r-- 1 ubuntu ubuntu  86K Dec  4 02:46 all_reduce.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 all_reduce_perf
-rw-rw-r-- 1 ubuntu ubuntu  98K Dec  4 02:46 alltoall.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 alltoall_perf
-rw-rw-r-- 1 ubuntu ubuntu  89K Dec  4 02:46 broadcast.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 broadcast_perf
-rw-rw-r-- 1 ubuntu ubuntu 423K Dec  4 02:46 common.o
-rw-rw-r-- 1 ubuntu ubuntu 102K Dec  4 02:46 gather.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 gather_perf
-rw-rw-r-- 1 ubuntu ubuntu 104K Dec  4 02:46 hypercube.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 hypercube_perf
-rw-rw-r-- 1 ubuntu ubuntu  90K Dec  4 02:46 reduce.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 reduce_perf
-rw-rw-r-- 1 ubuntu ubuntu  91K Dec  4 02:46 reduce_scatter.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 reduce_scatter_perf
-rw-rw-r-- 1 ubuntu ubuntu  99K Dec  4 02:46 scatter.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 scatter_perf
-rw-rw-r-- 1 ubuntu ubuntu  99K Dec  4 02:46 sendrecv.o
-rwxrwxr-x 1 ubuntu ubuntu  18M Dec  4 02:47 sendrecv_perf
-rw-rw-r-- 1 ubuntu ubuntu  21K Dec  4 00:49 timer.o
-rw-rw-r-- 1 ubuntu ubuntu 198K Dec  4 02:46 util.o
drwxrwxr-x 2 ubuntu ubuntu  33K Dec  4 02:47 verifiable
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/all_gather_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/all_reduce_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/alltoall_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/broadcast_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/gather_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/hypercube_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/reduce_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/reduce_scatter_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/scatter_perf
-rwxrwxr-x 1 ubuntu ubuntu 18M Dec  4 02:47 /fsx/nccl-tests/build/sendrecv_perf
ubuntu@ip-10-0-108-46:/fsx/nccl-tests$
```
