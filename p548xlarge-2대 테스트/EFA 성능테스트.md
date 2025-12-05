
```
OSU Micro-Benchmarks EFA 테스트 완벽 가이드
🎯 전제 조건

    AWS P5 인스턴스 (또는 EFA 지원 인스턴스)
    2개 이상의 컴퓨트 노드
    공유 파일시스템 (/fsx 또는 NFS)
    SSH 키 기반 인증 설정

1️⃣ 환경 준비 (최초 1회)
A. 헤드 노드에서 실행
# 1. 공유 디렉토리로 이동
cd /fsx

# 2. 모듈 시스템 확인
module av

# 3. OpenMPI 모듈 로드
module load openmpi/4.1.7  # 또는 사용 가능한 버전

# 4. OSU Micro-Benchmarks 다운로드
wget http://mvapich.cse.ohio-state.edu/download/mvapich/osu-micro-benchmarks-7.3.tar.gz
tar xzf osu-micro-benchmarks-7.3.tar.gz
cd osu-micro-benchmarks-7.3

# 5. 빌드
./configure CC=mpicc CXX=mpicxx
make -j$(nproc)

# 6. 빌드 확인
find . -name "osu_bw" -type f
# 출력: ./c/mpi/pt2pt/standard/osu_bw

B. SSH 키 설정 확인
# 헤드 노드에서 컴퓨트 노드로 비밀번호 없이 접속 가능해야 함
ssh compute-gpu-st-distributed-ml-1 hostname
ssh compute-gpu-st-distributed-ml-2 hostname

# 만약 비밀번호를 요구하면:
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
for node in compute-gpu-st-distributed-ml-1 compute-gpu-st-distributed-ml-2; do
    ssh-copy-id $node
done

2️⃣ Point-to-Point 대역폭 테스트 (단일 EFA)
명령어
cd /fsx/osu-micro-benchmarks-7.3

# 모듈 로드
module load openmpi/4.1.7

# 대역폭 테스트
mpirun -np 2 \
    -H compute-gpu-st-distributed-ml-1:1,compute-gpu-st-distributed-ml-2:1 \
    --mca pml cm \
    --mca mtl ofi \
    --mca mtl_ofi_provider_include efa \
    -x FI_PROVIDER=efa \
    -x FI_EFA_USE_DEVICE_RDMA=1 \
    ./c/mpi/pt2pt/standard/osu_bw

예상 결과
# OSU MPI Bandwidth Test v7.3
# Size      Bandwidth (MB/s)
1                       0.63
...
1048576             11,992.28
2097152             12,097.30
4194304             12,148.06  ← 12 GB/s (97 Gbps)

```

```
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$ find /fsx/osu-micro-benchmarks-7.3 -name "osu_bw" -type f
/fsx/osu-micro-benchmarks-7.3/c/mpi/pt2pt/standard/osu_bw
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$ module load openmpi/4.1.7

cd /fsx/osu-micro-benchmarks-7.3

mpirun -np 2 \
    -H compute-gpu-st-distributed-ml-1:1,compute-gpu-st-distributed-ml-2:1 \
    --mca pml cm \
    --mca mtl ofi \
    --mca mtl_ofi_provider_include efa \
    -x FI_PROVIDER=efa \
    -x FI_EFA_USE_DEVICE_RDMA=1 \
    ./c/mpi/pt2pt/standard/osu_bw
Warning: Permanently added 'compute-gpu-st-distributed-ml-1' (ED25519) to the list of known hosts.
Warning: Permanently added 'compute-gpu-st-distributed-ml-2' (ED25519) to the list of known hosts.
# OSU MPI Bandwidth Test v7.3
# Size      Bandwidth (MB/s)
# Datatype: MPI_CHAR.
1                       0.63
2                       1.78
4                       3.60
8                       7.08
16                     14.14
32                     28.00
64                     56.77
128                   112.36
256                   223.37
512                   437.77
1024                  877.84
2048                 1704.13
4096                 3237.71
8192                 5657.81
16384                7578.72
32768                9453.47
65536               10671.50
131072              10856.99
262144              11541.96
524288              11933.70
1048576             11992.28
2097152             12097.30
4194304             12148.06
ubuntu@ip-10-0-31-195:/fsx/osu-micro-benchmarks-7.3$
```



## 아래 내용은 주의 필요, 게런티 못함








## https://catalog.workshops.aws/ml-on-aws-parallelcluster/en-US/06-observability/08-grafana-osu : 이 워크샵 기반으로 수행
### 파티션(큐)이름 변경 및 , 스크립트 변경 필요(hpc7g.16xlarge 용)

'squeue'로 실행이 종료 후 확인

```
-rw-rw-r-- 1 ubuntu ubuntu   1220 Dec  4 00:01 osu_bw.out
-rw-rw-r-- 1 ubuntu ubuntu    182 Dec  3 23:52 osu_bw.sbatch
drwxrwxr-x 3 ubuntu ubuntu  33280 Dec  3 23:52 upc
drwxrwxr-x 3 ubuntu ubuntu  33280 Dec  3 23:52 upcxx
drwxrwxr-x 3 ubuntu ubuntu  33280 Dec  3 23:52 util
ubuntu@ip-10-0-108-46:/fsx/osu-micro-benchmarks-5.6.2$
ubuntu@ip-10-0-108-46:/fsx/osu-micro-benchmarks-5.6.2$
ubuntu@ip-10-0-108-46:/fsx/osu-micro-benchmarks-5.6.2$ squeue
             JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
ubuntu@ip-10-0-108-46:/fsx/osu-micro-benchmarks-5.6.2$
```
```
tail -f osu_bw.out
8192                49015.97        5983395.38
16384               48475.68        2958720.86
32768               46352.10        1414553.75
65536               47379.69         722956.68
131072              48631.34         371027.68
262144              47404.00         180831.91
524288              47086.55          89810.47
1048576             46182.53          44043.09
2097152             45900.40          21887.02
4194304             46471.92          11079.77
```
```
메시지 크기	대역폭 (MB/s)	대역폭 (Gbps)	메시지율 (msg/s)
8 KB	49,016	392 Gbps	5,983,395
16 KB	48,476	388 Gbps	2,958,721
32 KB	46,352	371 Gbps	1,414,554
64 KB	47,380	379 Gbps	722,957
128 KB	48,631	389 Gbps	371,028
256 KB	47,404	379 Gbps	180,832
512 KB	47,087	377 Gbps	89,810
1 MB	46,183	369 Gbps	44,043
2 MB	45,900	367 Gbps	21,887
4 MB	46,472	372 Gbps	11,080
```
## 성능 요약
```
╔═══════════════════════════════════════════════════════════╗
║        노드 간 네트워크 대역폭 벤치마크 결과                ║
╠═══════════════════════════════════════════════════════════╣
║  최대 대역폭:         49.0 GB/s (392 Gbps)               ║
║  평균 대역폭:         ~47 GB/s (376 Gbps)                ║
║  최적 메시지 크기:    8-128 KB                            ║
║  테스트 구성:         128 tasks (64 per node)            ║
║  EFA 활용률:          ~98% (400 Gbps 링크 기준)          ║
╠═══════════════════════════════════════════════════════════╣
║  평가:               ⭐⭐⭐⭐⭐ 매우 우수                  ║
╚═══════════════════════════════════════════════════════════╝
성능 분석
✅ 매우 우수한 성능

1. 최대 대역폭: 392 Gbps

    단일 400 Gbps EFA 링크의 98% 활용률
    이론적 최대치에 거의 도달

2. 안정적인 성능

    메시지 크기에 관계없이 일관된 성능 (46-49 GB/s)
    큰 메시지에서도 성능 저하 없음

3. 높은 메시지 처리율

    작은 메시지 (8 KB): 약 600만 msg/s
    네트워크 오버헤드가 매우 낮음

```

```
