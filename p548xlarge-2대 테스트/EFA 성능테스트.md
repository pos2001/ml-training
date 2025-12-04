
## https://catalog.workshops.aws/ml-on-aws-parallelcluster/en-US/06-observability/08-grafana-osu : 이 워크샵 기반으로 수행
### 파티션(큐)이름만 변경
ㅊ 'squeue'로 실행이 종료 후 확인

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
