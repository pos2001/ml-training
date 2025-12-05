
```
✅ Step 3 통과! 중요한 발견!
확인된 사항
✅ GPU Topology (NVLink):

    NV18: 모든 GPU가 NVLink 18개 연결로 서로 연결됨
    Full mesh topology: 8개 GPU가 all-to-all로 연결 (최고 성능)
    NUMA: GPU 0-3은 NUMA 0, GPU 4-7은 NUMA 1

✅ GPUDirect RDMA:

    gdrdrv 모듈 로드됨! ✅
    gdrdrv는 GPUDirect RDMA를 위한 커널 드라이버
    nvidia_peermem 대신 gdrdrv 사용 중 (정상)

해석:
gdrdrv                 24576  0
nvidia               8781824  36 nvidia_uvm,gdrdrv,nvidia_modeset
                                              ^^^^^^
                                         GPUDirect RDMA 활성화!

NVLink 성능
NV18 = NVLink 4.0 (18개 링크):

    대역폭: 900 GB/s per GPU
    레이턴시: ~1-2 us (매우 낮음)
    Full mesh: 모든 GPU 간 직접 연결

예상 성능 (단일 노드 8 GPUs):

    All-reduce 대역폭: ~300-400 GB/s
    노드 내 통신: 매우 빠름 (NVLink)
```

```
ubuntu@ip-10-0-31-195:/fsx$ ssh compute-gpu-st-distributed-ml-1 << 'EOF'
echo "=========================================="
echo "Step 3: GPU Topology & GPUDirect"
echo "=========================================="
echo ""

echo "=== GPU Topology (NVLink) ==="
nvidia-smi topo -m

echo ""
echo "=== NVIDIA Kernel Modules ==="
lsmod | grep nvidia

echo ""
echo "=== GPUDirect RDMA Check ==="
# nvidia-peermem 모듈 확인 (GPUDirect RDMA용)
if lsmod | grep -q nvidia_peermem; then
    echo "✓ nvidia_peermem loaded (GPUDirect RDMA enabled)"
else
    echo "✗ nvidia_peermem not loaded"
    echo "Checking alternative..."
    lsmod | grep -E "gdrdrv|nv_peer_mem"
fi

echo ""
echo "=========================================="
EOF
Pseudo-terminal will not be allocated because stdin is not a terminal.
Warning: Permanently added 'compute-gpu-st-distributed-ml-1' (ED25519) to the list of known hosts.
Welcome to Ubuntu 22.04.5 LTS (GNU/Linux 6.8.0-1043-aws x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro

 System information as of Fri Dec  5 10:38:16 UTC 2025

  System load: 0.08               Memory usage: 2%   Processes:       2343
  Usage of /:  4.9% of 496.03GB   Swap usage:   0%   Users logged in: 1


Expanded Security Maintenance for Applications is not enabled.

130 updates can be applied immediately.
3 of these updates are standard security updates.
To see these additional updates run: apt list --upgradable

2 additional security updates can be applied with ESM Apps.
Learn more about enabling ESM Apps service at https://ubuntu.com/esm

New release '24.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.


==========================================
Step 3: GPU Topology & GPUDirect
==========================================

=== GPU Topology (NVLink) ===
        GPU0    GPU1    GPU2    GPU3    GPU4    GPU5    GPU6    GPU7    CPU Affinity    NUMA Affinity   GPU NUMA ID
GPU0     X      NV18    NV18    NV18    NV18    NV18    NV18    NV18    0-47,96-143     0               N/A
GPU1    NV18     X      NV18    NV18    NV18    NV18    NV18    NV18    0-47,96-143     0               N/A
GPU2    NV18    NV18     X      NV18    NV18    NV18    NV18    NV18    0-47,96-143     0               N/A
GPU3    NV18    NV18    NV18     X      NV18    NV18    NV18    NV18    0-47,96-143     0               N/A
GPU4    NV18    NV18    NV18    NV18     X      NV18    NV18    NV18    48-95,144-191   1               N/A
GPU5    NV18    NV18    NV18    NV18    NV18     X      NV18    NV18    48-95,144-191   1               N/A
GPU6    NV18    NV18    NV18    NV18    NV18    NV18     X      NV18    48-95,144-191   1               N/A
GPU7    NV18    NV18    NV18    NV18    NV18    NV18    NV18     X      48-95,144-191   1               N/A

Legend:

  X    = Self
  SYS  = Connection traversing PCIe as well as the SMP interconnect between NUMA nodes (e.g., QPI/UPI)
  NODE = Connection traversing PCIe as well as the interconnect between PCIe Host Bridges within a NUMA node
  PHB  = Connection traversing PCIe as well as a PCIe Host Bridge (typically the CPU)
  PXB  = Connection traversing multiple PCIe bridges (without traversing the PCIe Host Bridge)
  PIX  = Connection traversing at most a single PCIe bridge
  NV#  = Connection traversing a bonded set of # NVLinks

=== NVIDIA Kernel Modules ===
nvidia_uvm           5025792  0
nvidia_drm            122880  0
nvidia_modeset       1507328  1 nvidia_drm
video                  77824  1 nvidia_modeset
nvidia               8781824  36 nvidia_uvm,gdrdrv,nvidia_modeset
ecc                    45056  1 nvidia

=== GPUDirect RDMA Check ===
✗ nvidia_peermem not loaded
Checking alternative...
gdrdrv                 24576  0
nvidia               8781824  36 nvidia_uvm,gdrdrv,nvidia_modeset
```

==========================================
