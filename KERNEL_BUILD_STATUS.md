# PowerElyan HSA Kernel Build

## Status: ✅ BUILD COMPLETE!

Successfully built kernel 6.1.159 with `CONFIG_HSA_AMD=y` for POWER8 on January 10, 2025.

### Build Time
- **Total Time**: ~20 minutes (on 128-thread POWER8)
- **Container**: Debian Bookworm via containerd
- **Host**: IBM POWER8 S824 running Ubuntu 20.04

### Generated Packages
| Package | Size | Description |
|---------|------|-------------|
| `linux-image-6.1.159-powerelyan-hsa_..._ppc64el.deb` | 102 MB | Kernel + modules |
| `linux-headers-6.1.159-powerelyan-hsa_..._ppc64el.deb` | 8.3 MB | Headers for module builds |
| `linux-libc-dev_..._ppc64el.deb` | 1.3 MB | Libc development files |

### Key Kernel Options Enabled
```
CONFIG_HSA_AMD=y             # HSA Kernel Fusion Driver
CONFIG_DRM_AMDGPU=m          # AMDGPU driver as module
CONFIG_DRM_AMDGPU_USERPTR=y  # Userspace pointers
CONFIG_DRM_AMDGPU_SI=y       # Southern Islands (GCN 1.0)
CONFIG_DRM_AMDGPU_CIK=y      # Sea Islands (GCN 2.0)
CONFIG_MMU_NOTIFIER=y        # Required by HSA_AMD
```

### AMDGPU Module Verified
```
./lib/modules/6.1.159-powerelyan-hsa/kernel/drivers/gpu/drm/amd/amdgpu/amdgpu.ko (15.4 MB)
```

### What This Enables
Once installed and booted, this kernel will provide:
- `/dev/kfd` - HSA Kernel Fusion Driver
- ROCm compute support for AMD GPUs on POWER8
- HIP, rocBLAS, MIOpen compatibility

### Installation (on PowerElyan/Debian ppc64el)
```bash
sudo dpkg -i linux-image-6.1.159-powerelyan-hsa_6.1.159-powerelyan-hsa-1_ppc64el.deb
sudo dpkg -i linux-headers-6.1.159-powerelyan-hsa_6.1.159-powerelyan-hsa-1_ppc64el.deb
sudo reboot
```

### Verification After Reboot
```bash
# Check kernel version
uname -r
# Expected: 6.1.159-powerelyan-hsa

# Check for KFD device
ls -la /dev/kfd
# Expected: crw-rw---- 1 root render 234, 0 /dev/kfd

# Check AMDGPU module loaded
lsmod | grep amdgpu
```

---
*PowerElyan by Elyan Labs - First ROCm on POWER8*
*Built in a Docker container, while simultaneously compiling PyTorch on the host!*
