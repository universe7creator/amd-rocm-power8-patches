# PowerElyan HSA Kernel Build

## Status: IN PROGRESS 🔄

Building kernel 6.1.x with `CONFIG_HSA_AMD=y` for POWER8.

### Build Environment
- **Host**: IBM POWER8 S824 running Ubuntu 20.04 (kernel 5.4.0-216)
- **Container**: Debian Bookworm (via containerd)
- **Target**: kernel 6.1.x with HSA/KFD enabled

### Progress
Currently building - see [power8-projects BUILD_STATUS.md](https://github.com/Scottcjn/power8-projects/blob/main/BUILD_STATUS.md) for live updates.

### What This Enables
Once complete, the custom kernel will provide:
- `/dev/kfd` - HSA Kernel Fusion Driver
- ROCm compute support for AMD GPUs on POWER8
- HIP, rocBLAS, MIOpen compatibility

### Expected Timeline
- Dependency installation: ~10 minutes
- Kernel source download: ~5 minutes
- Kernel compilation: 2-4 hours (128 threads)
- Packaging: ~10 minutes

---
*PowerElyan by Elyan Labs*
