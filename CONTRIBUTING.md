# Contributing to AMD ROCm POWER8 Patches

Thank you for your interest in contributing to the ROCm stack for AMD GPUs on IBM POWER8! This is the first ppc64le build of ROCm, and your contributions help bring AMD GPU support to POWER8 systems.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Environment](#development-environment)
- [How to Contribute](#how-to-contribute)
- [Patch Guidelines](#patch-guidelines)
- [Testing](#testing)
- [Pull Request Process](#pull-request-process)
- [Community](#community)

## Code of Conduct

This project and everyone participating in it is governed by our commitment to:

- Be respectful and inclusive
- Welcome newcomers and help them learn
- Focus on constructive feedback
- Respect different viewpoints and experiences

## Getting Started

### Prerequisites

To contribute patches, you'll need:

- **IBM POWER8 or POWER9 system** with ppc64le architecture
- **AMD GPU** (MI50, MI100, or MI200 series recommended)
- **Linux distribution** with ppc64le support (RHEL 8/9, Ubuntu 20.04/22.04, or SLES)
- **Development tools**:
  ```bash
  sudo apt-get install build-essential git cmake ninja-build
  sudo apt-get install rocm-dev rocm-libs  # If available
  ```

### Repository Structure

```
amd-rocm-power8-patches/
├── patches/              # Individual patch files
│   ├── hip/             # HIP runtime patches
│   ├── rocblas/         # rocBLAS patches
│   ├── rocfft/          # rocFFT patches
│   └── ...
├── scripts/             # Build and utility scripts
├── docs/                # Documentation
├── tests/               # Test cases and validation
└── README.md
```

## Development Environment

### Setting Up POWER8 Development System

1. **QEMU Emulation** (if no physical hardware):
   ```bash
   # Install QEMU for ppc64le
   sudo apt-get install qemu-system-ppc qemu-user-static
   
   # Create POWER8 VM
   qemu-system-ppc64 -machine power8 -cpu power8 -m 8G \
     -drive file=ubuntu-ppc64le.qcow2,format=qcow2
   ```

2. **IBM Cloud POWER8 Instance**:
   ```bash
   # Provision POWER8 VM on IBM Cloud
   ibmcloud sl vs create --hostname rocm-dev --domain example.com \
     --cpu 4 --memory 16384 --os UBUNTU_20_64 \
     --datacenter dal13 --flavor B1_4X16X25
   ```

3. **Docker/Podman Container**:
   ```bash
   # Run ppc64le container on x86_64 host
   docker run --rm -it --platform linux/ppc64le \
     -v $(pwd):/workspace ppc64le/ubuntu:22.04
   ```

### ROCm Source Setup

```bash
# Clone ROCm components
mkdir -p ~/rocm-src && cd ~/rocm-src

git clone https://github.com/ROCm-Developer-Tools/HIP.git
git clone https://github.com/ROCmSoftwarePlatform/rocBLAS.git
git clone https://github.com/ROCmSoftwarePlatform/rocFFT.git
git clone https://github.com/ROCmSoftwarePlatform/rocRAND.git

# Apply POWER8 patches
cd HIP
patch -p1 < /path/to/amd-rocm-power8-patches/patches/hip/0001-power8-support.patch
```

## How to Contribute

### Types of Contributions

We welcome:

- **New Patches**: Support for additional ROCm components
- **Patch Updates**: Fixes for existing patches on newer ROCm versions
- **Documentation**: Guides, examples, and troubleshooting
- **Test Cases**: Validation scripts and benchmarks
- **Build Scripts**: Automation for building on POWER8

### Finding Issues to Work On

- Check [GitHub Issues](../../issues) for `good first issue` or `help wanted` labels
- Look for TODOs in existing patches
- Test with newer ROCm versions and report compatibility issues

### Making Changes

1. **Fork the repository** on GitHub
2. **Clone your fork**:
   ```bash
   git clone https://github.com/YOUR_USERNAME/amd-rocm-power8-patches.git
   cd amd-rocm-power8-patches
   ```
3. **Create a branch**:
   ```bash
   git checkout -b feature/your-feature-name
   # or
   git checkout -b fix/issue-description
   ```

## Patch Guidelines

### Patch Format Standards

All patches must follow these conventions:

1. **File Naming**:
   ```
   patches/<component>/####-short-description.patch
   ```
   - Use 4-digit sequence numbers
   - Use kebab-case descriptions
   - Example: `patches/hip/0001-add-power8-intrinsics.patch`

2. **Patch Headers**:
   ```diff
   From: Your Name <your.email@example.com>
   Date: Mon, 1 Jan 2024 00:00:00 +0000
   Subject: [PATCH] Component: Brief description
   
   Longer description explaining what and why.
   
   Signed-off-by: Your Name <your.email@example.com>
   ---
   ```

3. **Commit Message Style**:
   - First line: Component: Brief description (max 50 chars)
   - Blank line
   - Detailed explanation (wrap at 72 chars)
   - Signed-off-by line

### Power8-Specific Considerations

When writing patches for POWER8:

1. **Vector Intrinsics**:
   - Map AVX/SSE intrinsics to VSX (Vector Scalar Extension)
   - Use `vec_xl` / `vec_xst` for vector loads/stores
   - Reference: `patches/hip/0001-power8-intrinsics.patch`

2. **Memory Alignment**:
   - POWER8 requires stricter alignment for vector operations
   - Use `__builtin_assume_aligned()` where appropriate
   - Handle 128-bit alignment for VSX operations

3. **Atomic Operations**:
   - Use GCC built-in atomics: `__atomic_*`
   - Avoid x86-specific assembly
   - Test with `ppc64le` endianness

4. **Cache Line Size**:
   - POWER8 has 128-byte cache lines (vs 64-byte on x86)
   - Adjust cache-aligned structures accordingly

### Example Patch Structure

```diff
From: Developer Name <dev@example.com>
Date: Mon, 15 Jan 2024 10:30:00 +0000
Subject: [PATCH] rocBLAS: Add POWER8 VSX support for GEMM kernels

Implement VSX-optimized GEMM kernels for ppc64le architecture.
Maps x86 AVX2 intrinsics to POWER8 VSX equivalents.

- Add vec_madd() replacement for _mm256_fmadd_ps()
- Handle 128-byte cache line alignment
- Update CMake detection for power8+ target

Signed-off-by: Developer Name <dev@example.com>
---
 library/src/blas3/CMakeLists.txt     |  5 ++++
 library/src/blas3/gemm_power8.hpp    | 89 +++++++++++++++++++++++++++++
 2 files changed, 94 insertions(+)
```

## Testing

### Test Requirements

Before submitting patches:

1. **Patch Applies Cleanly**:
   ```bash
   cd /path/to/rocm-source
   patch --dry-run -p1 < /path/to/your.patch
   # Should report: "patching file ..." with no errors
   ```

2. **Builds Successfully**:
   ```bash
   mkdir build && cd build
   cmake .. -DCMAKE_BUILD_TYPE=Release
   make -j$(nproc)
   # Should complete without errors
   ```

3. **Passes Basic Tests**:
   ```bash
   # Run component tests
   ctest --output-on-failure
   
   # Or run specific test
   ./test_hip_vector_operations
   ```

### POWER8-Specific Testing

```bash
# Verify ppc64le architecture
uname -m  # Should output: ppc64le

# Check CPU features
cat /proc/cpuinfo | grep -m1 "model"  # POWER8 or POWER9

# Test with different optimization levels
for opt in O0 O1 O2 O3; do
  cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_FLAGS="-mcpu=power8 -$opt"
  make clean && make -j$(nproc)
  ctest --output-on-failure
done
```

### GPU Testing (if available)

```bash
# Check AMD GPU detection
rocm-smi

# Run HIP samples
/opt/rocm/hip/samples/0_Intro/square

# Test with rocBLAS
./rocblas-test --gtest_filter="*gemm*"
```

## Pull Request Process

1. **Update Documentation**:
   - Add entry to `docs/CHANGELOG.md`
   - Update relevant README sections
   - Document any new dependencies

2. **Test Your Changes**:
   ```bash
   ./scripts/validate-patches.sh
   ```

3. **Submit PR**:
   - Fill out the PR template completely
   - Reference any related issues
   - Include test results

4. **Review Process**:
   - Maintainers will review within 5 business days
   - Address feedback promptly
   - Re-request review after updates

### PR Checklist

- [ ] Patch