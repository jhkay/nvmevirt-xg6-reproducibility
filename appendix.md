# 부록: NVMeVirt XG6 성능 재현성 실험 자료

이 문서는 논문 **"NVMe 에뮬레이터 NVMeVirt의 SSD 성능 재현 한계와 원인 분석"**에서 참조하는 부록 자료를 정리한 것이다.

<a id="a1"></a>
## A.1 순차쓰기 워크로드의 벤치마크 로그

순차쓰기 워크로드의 원시 실행 로그를 가독성을 위해 재구성한 벤치마크 로그를 제시한다. 로그는 실험 환경 스냅샷, 준비, 반복 측정, 쿨다운 등을 포함한다.

```text
Formatted benchmark log for the sequential write workload
==== Workload Start: sequential write ====

==== Environment Snapshot ====
[host] server-b
[kernel] 5.19.17
[governor] performance
[turbo] disabled
[THP] never
[ASLR] 0
[scheduler] none
[NUMA] single node

==== Preparation ====
blkdiscard -> full fill
post-full-fill cooldown: 600 s
blkdiscard test region (offset=1 GiB, length=16 GiB)
sequential write preconditioning
pre-run cooldown before run 1: 300 s

==== Measurement ====
run 1: write, size=16 GiB, QD=32, 1200 s
reinitialize test region
per-run cooldown before run 2: 300 s
run 2: write, size=16 GiB, QD=32, 1200 s
reinitialize test region
per-run cooldown before run 3: 300 s
run 3: write, size=16 GiB, QD=32, 1200 s

==== Workload End ====
workload cooldown: 300 s
```

<a id="a2"></a>
## A.2 실험 환경 통제를 위한 설정 명령

최소 사용자 공간 실행 환경 구성과 런타임 환경 통제에 사용한 명령들을 제시한다. Turbo Boost는 부팅 전에 Basic Input/Output System (BIOS)에서 비활성화하였다.

```bash
# Minimal user-space setup
sudo systemctl isolate rescue.target

# Stop unnecessary background services
sudo systemctl stop irqbalance cron atd rsyslog || true
sudo systemctl disable irqbalance || true

# Runtime environment controls
for p in /sys/devices/system/cpu/cpufreq/policy*/scaling_governor; do
    echo performance | sudo tee "$p" >/dev/null
done

echo 0 | sudo tee /sys/module/nvme_core/parameters/default_ps_max_latency_us >/dev/null
echo never | sudo tee /sys/kernel/mm/transparent_hugepage/enabled >/dev/null
sudo sysctl -w kernel.randomize_va_space=0
echo none | sudo tee /sys/block/<dev>/queue/scheduler >/dev/null
```

<a id="a3"></a>
## A.3 통제 항목 변경에 따른 상대 성능 변화

3장의 서버 B 벤치마크 결과를 baseline으로 두고, 각 환경 통제 항목을 하나씩 변경했을 때의 상대 성능 변화를 제시한다.

순차쓰기(write)가 가장 큰 영향을 받았고, 순차읽기(read)는 변화가 거의 없었다. 랜덤읽기(randread)에서는 mq-deadline 적용 시 상대적으로 큰 성능 저하가 나타났다.

| Modified setting | ΔIOPS randread (%, ↑ better) | ΔIOPS write (%, ↑ better) | ΔIOPS read (%, ↑ better) | Δp50 randread (%, ↑ worse) | Δp50 write (%, ↑ worse) | Δp50 read (%, ↑ worse) | Δp99 randread (%, ↑ worse) | Δp99 write (%, ↑ worse) | Δp99 read (%, ↑ worse) |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| APST enabled | -0.1 | -9.9 | 0.0 | 0.0 | +21.3 | 0.0 | 0.0 | +9.2 | +2.2 |
| Ondemand governor | +0.4 | -10.1 | 0.0 | 0.0 | +21.7 | 0.0 | -2.7 | +12.3 | +2.9 |
| mq-deadline scheduler | -4.5 | -11.0 | 0.0 | +4.5 | +21.7 | 0.0 | -2.9 | +5.9 | -2.2 |
| THP enabled | +0.3 | -8.9 | -0.1 | 0.0 | +21.7 | 0.0 | -2.7 | +3.3 | +4.4 |

<a id="a4"></a>
## A.4 반복 측정 변동성 요약

3장의 XG6 벤치마크 결과에 대해, 서버 A와 B에서 각 워크로드를 3회 반복 실행하여 얻은 성능 지표를 제시한다. kIOPS, p50 및 p99 지연시간의 런별 측정값, 평균 그리고 변동계수(Coefficient of Variation, CV)를 제시하였다.

전반적으로 랜덤읽기(randread)와 순차읽기(read)는 두 서버 모두 낮은 변동성을 나타냈으며, 순차쓰기(write)는 상대적으로 큰 변동성을 보였다. 그러나 서버 간 평균 성능 수준은 유사하게 유지되어, 재현 가능한 범위의 편차를 보인 것으로 해석된다.

| Metric | Run / Statistic | randread A | randread B | read A | read B | write A | write B |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: |
| kIOPS | run 1 | 263.70 | 264.76 | 23.68 | 23.69 | 22.64 | 21.11 |
| kIOPS | run 2 | 263.46 | 263.73 | 23.68 | 23.70 | 20.01 | 22.31 |
| kIOPS | run 3 | 263.22 | 263.18 | 23.68 | 23.68 | 20.98 | 20.61 |
| kIOPS | mean | 263.46 | 263.89 | 23.68 | 23.69 | 21.21 | 21.34 |
| kIOPS | CV (%) | 0.09 | 0.30 | 0.00 | 0.03 | 6.26 | 4.09 |
| p50 latency (ms) | run 1 | 0.11 | 0.11 | 1.34 | 1.34 | 1.30 | 1.32 |
| p50 latency (ms) | run 2 | 0.11 | 0.11 | 1.34 | 1.34 | 1.52 | 1.32 |
| p50 latency (ms) | run 3 | 0.11 | 0.11 | 1.34 | 1.34 | 1.32 | 1.32 |
| p50 latency (ms) | mean | 0.11 | 0.11 | 1.34 | 1.34 | 1.38 | 1.32 |
| p50 latency (ms) | CV (%) | 0.00 | 0.00 | 0.00 | 0.00 | 8.60 | 0.00 |
| p99 latency (ms) | run 1 | 0.26 | 0.25 | 1.61 | 1.48 | 5.60 | 6.00 |
| p99 latency (ms) | run 2 | 0.26 | 0.26 | 1.63 | 1.43 | 5.54 | 5.60 |
| p99 latency (ms) | run 3 | 0.26 | 0.26 | 1.53 | 1.53 | 5.93 | 6.26 |
| p99 latency (ms) | mean | 0.26 | 0.26 | 1.59 | 1.48 | 5.69 | 5.95 |
| p99 latency (ms) | CV (%) | 0.00 | 1.23 | 3.31 | 3.31 | 3.70 | 5.54 |

<a id="a5"></a>
## A.5 측정 단계에 사용한 fio job 파일

측정 단계에서 사용한 fio 파라미터를 제시한다. 이는 블록 크기, 큐 깊이, 실행 시간, ramp-up 시간 등의 실행 조건을 요약한 것이다.

```ini
[global]
ioengine=io_uring
direct=1
offset=1G
size=16G
iodepth=32
numjobs=1
time_based=1
runtime=1200
ramp_time=30

[read_qd32]
rw=read
bs=128k

[write_qd32]
rw=write
bs=128k

[randread_qd32]
rw=randread
bs=4k
```
