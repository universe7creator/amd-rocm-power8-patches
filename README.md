# ROCm for AMD GPUs on IBM POWER8

**World's First ROCm Stack Built from Source for POWER8/PPC64LE!**

This repository documents the successful build of the ROCm (Radeon Open Compute) stack on IBM POWER8 systems, enabling AMD GPU compute on legacy PowerPC servers.

Successfully compiled January 10, 2025.

## ROCm Build Status

| Component | Version | Status | Notes |
|-----------|---------|--------|-------|
| **ROCT-Thunk-Interface** | 5.7.1 | ✅ Built & Installed | HSA kernel interface |
| **LLVM/Clang** | 17.0.0 | ✅ Built & Installed | AMDGPU + PowerPC targets |
| **ROCR-Runtime** | 5.7.1 | ✅ Built & Installed | HSA runtime library (libhsa-runtime64.so) |
| **ROCm Drivers** | - | ⚠️ Partial | amdgpu loads, KFD needs kernel rebuild |

## Current Status: RX 580 Detected!

The RX 580 (Polaris/Ellesmere) is detected and working with the amdgpu kernel driver:

```
Device: 1002:67df (AMD Ellesmere - RX 580)
Driver: amdgpu loaded
VRAM: 8192MB detected
DRM: /dev/dri/card0, /dev/dri/renderD128 available
```

**Issue**: `/dev/kfd` is missing - the stock Ubuntu 20.04 kernel for ppc64le does not have `CONFIG_HSA_AMD` enabled.

### What Works
- GPU detected on PCIe bus via OCuLink
- amdgpu kernel module loads
- 8GB VRAM recognized
- Display/graphics output functional

### What's Missing for ROCm Compute
- `/dev/kfd` (HSA Kernel Fusion Driver)
- Requires `CONFIG_HSA_AMD=y` in kernel config
- Stock kernel: `CONFIG_DRM_AMDGPU=m` but no HSA

### Solution: Custom Kernel Build

The Ubuntu ppc64le kernel needs to be rebuilt with HSA support. Required config:

```
CONFIG_HSA_AMD=y
CONFIG_DRM_AMDGPU=m
CONFIG_DRM_AMDGPU_USERPTR=y
```

**Kernel build dependencies** (built from source due to broken apt):
- m4 1.4.19
- flex 2.6.4
- bison 3.8.2
- OpenSSL headers (needed for module signing)

**ROCm Installation Path**: `/opt/rocm/`
- Libraries: `/opt/rocm/lib/` (libhsa-runtime64.so, libhsakmt.a)
- Headers: `/opt/rocm/include/hsa/`
- Clang: `/opt/rocm/llvm/bin/clang`

## Test System

| Component | Specification |
|-----------|---------------|
| **Server** | IBM Power System S824 (8286-42A) |
| **CPU** | Dual 8-core POWER8 (16 cores, 128 threads with SMT8) |
| **RAM** | 576 GB DDR3 (4 NUMA nodes) |
| **OS** | Ubuntu 20.04 LTS (Focal Fossa) |
| **Kernel** | 5.4.0-216-generic |

## Prerequisites Built from Source

Ubuntu 20.04 on POWER8 has broken packages (wrong glibc 2.32-2.34 vs system 2.31). All prerequisites built from source:

| Tool | Version | Why Needed |
|------|---------|------------|
| cmake | 3.25.3 | System cmake broken (glibc mismatch) |
| numactl | 2.0.16 | ROCT dependency |
| pkg-config | 0.29.2 | Build system |
| libdrm | 2.4.100 | DRM userspace library (AMD-only, no nouveau) |
| libelf headers | 0.189 | ELF manipulation (from elfutils) |

## Build Instructions

### 1. Build Prerequisites

```bash
# cmake 3.25.3
cd /tmp && wget https://github.com/Kitware/CMake/releases/download/v3.25.3/cmake-3.25.3.tar.gz
tar xzf cmake-3.25.3.tar.gz && cd cmake-3.25.3
./bootstrap --prefix=/usr/local -- -DCMAKE_USE_OPENSSL=OFF
make -j$(nproc) && sudo make install

# numactl 2.0.16
cd /tmp && wget https://github.com/numactl/numactl/releases/download/v2.0.16/numactl-2.0.16.tar.gz
tar xzf numactl-2.0.16.tar.gz && cd numactl-2.0.16
./configure --prefix=/usr/local && make -j$(nproc) && sudo make install

# pkg-config 0.29.2
cd /tmp && wget https://pkg-config.freedesktop.org/releases/pkg-config-0.29.2.tar.gz
tar xzf pkg-config-0.29.2.tar.gz && cd pkg-config-0.29.2
./configure --prefix=/usr/local --with-internal-glib && make -j$(nproc) && sudo make install

# libdrm 2.4.100 (AMD only)
cd /tmp && wget https://dri.freedesktop.org/libdrm/libdrm-2.4.100.tar.bz2
tar xjf libdrm-2.4.100.tar.bz2 && cd libdrm-2.4.100
./configure --prefix=/usr/local --disable-nouveau --enable-amdgpu
make -j$(nproc) && sudo make install

# libelf headers (from elfutils)
cd /tmp && wget https://sourceware.org/elfutils/ftp/0.189/elfutils-0.189.tar.bz2
tar xjf elfutils-0.189.tar.bz2
sudo cp elfutils-0.189/libelf/libelf.h /usr/include/
sudo cp elfutils-0.189/libelf/gelf.h /usr/include/
sudo ln -sf /usr/lib/powerpc64le-linux-gnu/libelf.so.1 /usr/lib/powerpc64le-linux-gnu/libelf.so
```

### 2. Build ROCT-Thunk-Interface

```bash
cd /tmp/rocm-build
wget https://github.com/RadeonOpenCompute/ROCT-Thunk-Interface/archive/refs/tags/rocm-5.7.1.tar.gz -O roct.tar.gz
tar xzf roct.tar.gz && cd ROCT-Thunk-Interface-rocm-5.7.1
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/opt/rocm
make -j$(nproc) && sudo make install
```

### 3. Build LLVM/Clang with AMDGPU

```bash
cd /tmp/rocm-build
wget https://github.com/RadeonOpenCompute/llvm-project/archive/refs/tags/rocm-5.7.1.tar.gz -O llvm.tar.gz
tar xzf llvm.tar.gz && cd llvm-project-rocm-5.7.1
mkdir build && cd build

cmake ../llvm \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/opt/rocm/llvm \
    -DLLVM_ENABLE_PROJECTS="clang;lld" \
    -DLLVM_TARGETS_TO_BUILD="AMDGPU;PowerPC" \
    -DLLVM_ENABLE_ASSERTIONS=OFF \
    -DLLVM_PARALLEL_LINK_JOBS=8

# Build with 64 threads (POWER8 optimal, not 128)
make -j64
sudo make install
```

### 4. Build ROCR-Runtime

```bash
cd /tmp/rocm-build
wget https://github.com/RadeonOpenCompute/ROCR-Runtime/archive/refs/tags/rocm-5.7.1.tar.gz -O rocr.tar.gz
tar xzf rocr.tar.gz && cd ROCR-Runtime-rocm-5.7.1
mkdir build && cd build

cmake ../src \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/opt/rocm \
    -DCMAKE_C_COMPILER=/opt/rocm/llvm/bin/clang \
    -DCMAKE_CXX_COMPILER=/opt/rocm/llvm/bin/clang++ \
    -DCMAKE_PREFIX_PATH=/opt/rocm \
    -DLIBELF_INCLUDE_DIRS=/usr/include \
    -DLIBELF_LIBRARIES=/usr/lib/powerpc64le-linux-gnu/libelf.so

make -j32 && sudo make install
```

## Verified Clang Output

```
$ /opt/rocm/llvm/bin/clang --version
clang version 17.0.0
Target: powerpc64le-unknown-linux-gnu
Thread model: posix
```

## Supported AMD GPUs (Target)

| Architecture | GPUs | ROCm Support |
|-------------|------|--------------|
| **RDNA 2/3** | RX 6800, 7900 XTX | Full |
| **RDNA 1** | RX 5700 | Full |
| **Polaris (GCN4)** | RX 580, 480 | Legacy (5.7.x) |
| **Vega** | Radeon VII, MI50 | Full |

## AMD GPU Connection via OCuLink

Same method as NVIDIA GPUs:
- **Host Card**: Supermicro AOC-SLG3-2E4T (PCIe 3.0 x8)
- **GPU Dock**: Minisforum DEG1 or similar
- **Power**: ATX PSU (650W+ recommended)
- **Cable**: SFF-8611 OCuLink cable

## Installation Verification

```bash
# Check libraries
ls -la /opt/rocm/lib/libhsa*

# Check Clang
/opt/rocm/llvm/bin/clang --version

# Check HSA headers
ls /opt/rocm/include/hsa/
```

## Why ROCm on POWER8?

1. **True Open Source**: AMD's GPU compute stack is fully open
2. **Legacy GPU Support**: Polaris/Vega cards are affordable
3. **HIP Portability**: Write once, run on AMD or NVIDIA
4. **576GB RAM**: Load massive models without GPU VRAM limits
5. **Alternative Ecosystem**: Not locked to NVIDIA CUDA

## Next Steps

- Build amdgpu kernel driver for POWER8
- Test RX 580 via OCuLink
- Build HIP runtime

## Credits

- **Development**: Elyan Labs (Scott Boudreaux)
- **Testing Platform**: IBM POWER8 S824 with 576GB RAM
- **AI Assistance**: Claude AI (Anthropic)

## License

Build instructions and patches provided under MIT license.
ROCm components retain their original AMD licenses (primarily MIT).

## Contact

For questions or contributions:
- Open an issue on this repository
- Email: scott@elyanlabs.ai
