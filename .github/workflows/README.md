# SWUpdate Buildroot Build Workflow

This directory contains GitHub Actions workflows for automating the build and creation of SWUpdate Buildroot images for the NTC CHIP device.

## Overview

The **Build SWUpdate Image** workflow (`build-swupdate.yml`) automates the complete build process for creating bootable SWUpdate images. It handles compilation of Buildroot, the Linux kernel, and packaging them into an ITB (FIT Image) format that can be booted by U-Boot.

## What the Workflow Does

### Build Pipeline

The workflow executes the following steps in sequence:

1. **Environment Setup**
   - Installs all required build tools and cross-compilation utilities
   - Sets up ARM EABIHF cross-compiler for Cortex-A8 target

2. **Buildroot Configuration**
   - Downloads Buildroot 2021.05.1
   - Applies your custom configuration (`swupdate/buildroot.config`)
   - Configures BusyBox with custom settings
   - Builds the root filesystem as `rootfs.cpio.xz`

3. **Kernel Compilation**
   - Downloads Linux 5.15 kernel source
   - Applies kernel patches from `kernel_files/` directory
   - Cross-compiles kernel with your custom configuration
   - Generates zImage and device tree binaries (.dtb files)

4. **ITB Image Creation**
   - Combines kernel zImage, device tree, and root filesystem
   - Uses your `swupdate.its` specification to create a FIT image (ITB)
   - Produces a single bootable image for U-Boot

5. **SWU Image Generation** (optional)
   - If build scripts exist in `swupdate-gen-image/`, generates the final SWU format
   - Creates the update package for swupdate runtime

6. **Artifact Management**
   - Uploads all generated images as build artifacts
   - Creates GitHub releases with downloadable images when you push version tags
   - Archives build logs for debugging failed builds

## Key Features

✅ **Automated Builds** - Triggers automatically on changes to `swupdate/` directory  
✅ **Complete Tool Chain** - Includes ARM cross-compiler and all build dependencies  
✅ **Kernel Patching** - Applies patches from `kernel_files/` directory automatically  
✅ **Configuration Management** - Uses your exact buildroot, kernel, and busybox configs  
✅ **ITB Packaging** - Creates U-Boot compatible FIT images  
✅ **SWU Format Support** - Generates swupdate-compatible update packages  
✅ **Release Management** - Auto-creates GitHub releases with built images  
✅ **Build Artifacts** - Stores images for 30 days for easy download  
✅ **Failure Diagnostics** - Archives build logs when compilation fails  
✅ **Manual Trigger** - Run builds on-demand via workflow dispatch  

## How to Use

### Automatic Builds

Builds trigger automatically when you push changes to the `swupdate/` directory:

```bash
# Make changes to any swupdate config
vim swupdate/buildroot.config
git add swupdate/buildroot.config
git commit -m "Update buildroot configuration"
git push origin main
# Workflow triggers automatically!
```
### Manual Builds

Trigger a build manually from GitHub:

1. Go to **Actions** tab in your repository
2. Select **Build SWUpdate Image** workflow
3. Click **Run workflow**
4. Select branch and click **Run workflow**

### Creating Releases

To create a GitHub Release with downloadable images:

```bash
# Tag your commit
git tag v1.0.0 -m "SWUpdate Image Release v1.0.0"
git push origin v1.0.0
# Workflow creates release and uploads images automatically
```

## Workflow Triggers

| Trigger | Condition |
|---------|-----------|
| **Push to main** | Any file in `swupdate/` directory changes OR workflow file changes |
| **Workflow Dispatch** | Manual trigger from GitHub Actions UI |
| **Release Creation** | When you push a git tag (releases created automatically) |

## Build Requirements

The workflow automatically installs these dependencies:

### Build Tools
- `build-essential` - GCC, make, and development tools
- `libncurses-dev` - Terminal UI support for menuconfig
- `pkg-config` - Package configuration utility
- `bc` - Calculator for build scripts
- `git` - Version control
- `wget` - File downloading

### Specialized Tools
- `u-boot-tools` - Creates ITB/FIT images with mkimage
- `device-tree-compiler` - Compiles .dts to .dtb
- `gcc-arm-linux-gnueabihf` - ARM cross-compiler
- `binutils-arm-linux-gnueabihf` - ARM binutils

### Required Libraries
- `libssl-dev` - OpenSSL library for kernel build
- `bison`, `flex` - Parser generators for kernel build

## Configuration Files Used

The workflow uses these configuration files from your repository:

### Buildroot Configuration
- **Path**: `swupdate/buildroot.config`
- **Purpose**: Buildroot build settings (toolchain, packages, filesystem)
- **Target**: Cortex-A8 ARM processor with uClibc libc

### Kernel Configuration
- **Path**: `swupdate/kernel.config`
- **Purpose**: Linux kernel build options
- **Version**: Linux 5.15

### BusyBox Configuration
- **Path**: `swupdate/busybox.config`
- **Purpose**: BusyBox utilities and applets selection

### Kernel Patches
- **Path**: `kernel_files/*.patch`
- **Purpose**: Custom patches applied before kernel compilation
- **Examples**: NAND support, HDMI fixes, i2c shutdown fixes

### Build Overlay
- **Path**: `swupdate/buildroot-overlay/`
- **Purpose**: Files overlaid into the root filesystem after build

### Image Specification
- **Path**: `swupdate/swupdate.its`
- **Purpose**: FIT image tree specification for ITB creation

## Build Outputs

### Generated Artifacts

The workflow produces the following outputs:

| Artifact | Format | Purpose |
|----------|--------|---------|
| `swupdate.itb` | FIT Image | Bootable kernel + rootfs for U-Boot |
| `swupdate.swu` | SWU Format | Update package for swupdate utility (if generated) |
| `build.log` | Text Log | Buildroot build output (on failure) |
| `kernel-build.log` | Text Log | Kernel compilation output (on failure) |

### Artifact Storage

- **Location**: GitHub Actions artifacts section
- **Retention**: 30 days
- **Download**: Via Actions tab or Release page

## Environment Variables

The workflow sets these environment variables:

```bash
BUILDROOT_VERSION=2021.05.1
KERNEL_VERSION=5.15
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-
```

You can modify the Buildroot or Kernel version by editing the workflow file.

## Build Times

Expected build durations:

| Stage | Duration | Notes |
|-------|----------|-------|
| Setup & Dependencies | 2-3 minutes | Faster on cached runs |
| Buildroot Build | 60-120 minutes | Depends on package selection |
| Kernel Compilation | 30-60 minutes | ARM cross-compilation |
| ITB Creation | < 1 minute | Image packaging |
| **Total** | **90-180 minutes** | First run will be slower |

**Pro Tip**: Buildroot will be cached between runs if packages don't change, reducing subsequent build times by 50%.

## Troubleshooting

### Build Fails - Memory Issues

If the build runs out of memory:

1. Reduce parallel jobs in the workflow:
   ```bash
   make -j2  # Instead of -j$(nproc)
   ```

2. Check build logs for specific errors:
   - Go to **Actions** → failed workflow
   - Download build logs artifact

### Kernel Patch Application Fails

If a kernel patch doesn't apply:

1. Check patch compatibility with kernel version
2. Ensure patches are in `kernel_files/` directory
3. Verify patch format (unified diff format required)
4. Review kernel-build.log for details

### ITB Image Generation Fails

If mkimage fails:

1. Verify `swupdate.its` file syntax
2. Ensure kernel and DTB files are generated
3. Check that rootfs image exists

### Missing buildroot-overlay

If overlay directory is empty or missing:

1. Ensure `swupdate/buildroot-overlay/` exists
2. Add required overlay files
3. Verify buildroot.config points to correct path (auto-updated by workflow)

### Build Artifacts Not Appearing

If no artifacts are generated:

1. Check workflow logs for build failures
2. Verify all config files exist and are valid
3. Download build logs to identify the issue

## Customization

### Change Buildroot Version

Edit `build-swupdate.yml`:

```yaml
env:
  BUILDROOT_VERSION: 2021.08.1  # Change version here
```

### Change Kernel Version

Edit `build-swupdate.yml`:

```yaml
env:
  KERNEL_VERSION: 5.16  # Change version here
```

### Add Custom Build Steps

To add steps before or after the build, edit the workflow file and insert new steps in the appropriate section.

### Modify Build Parameters

Adjust the parallel job count:

```yaml
make -j4  # Change from -j$(nproc)
```

## Performance Tips

1. **Cache Management**: GitHub Actions automatically caches downloaded sources between builds
2. **Reduce Scope**: Disable unnecessary packages in buildroot.config to speed up builds
3. **Incremental Builds**: Subsequent builds are faster if only configs change
4. **Manual Triggers**: Use manual dispatch only when necessary to preserve Actions minutes

## Security Considerations

- All builds run in isolated GitHub Actions environments
- No secrets are exposed in logs
- Build artifacts are stored securely with GitHub
- Releases are published only when you explicitly tag commits

## Logs and Debugging

### Accessing Build Logs

1. Go to **Actions** tab
2. Click on the failed/completed workflow run
3. Expand the step that failed
4. Scroll to see detailed output
5. Or download the artifact logs after completion

### Build Log Contents

- **build.log**: Buildroot compilation output and warnings
- **kernel-build.log**: Linux kernel build messages

### Common Log Messages

| Message | Meaning | Solution |
|---------|---------|----------|
| `No such file or directory` | Missing config file | Verify `swupdate/` files exist |
| `Permission denied` | Script not executable | Not critical for shell scripts |
| `Compilation terminated` | Build error | Check specific error lines above |

## Related Documentation

- **Buildroot Documentation**: https://buildroot.org/downloads/manual/
- **Linux Kernel Build Guide**: https://www.kernel.org/doc/html/latest/kbuild/
- **U-Boot FIT Image Format**: https://source.denx.de/u-boot/u-boot/-/blob/master/doc/uImage.FIT/
- **SWUpdate Documentation**: https://sbabic.github.io/swupdate/

## Support

For issues or questions:

1. Check the **Troubleshooting** section above
2. Review workflow logs in the **Actions** tab
3. Open an issue on the repository with:
   - Workflow run link
   - Build logs (artifacts)
   - Exact error message
   - Steps to reproduce

## Contributing

To improve this workflow:

1. Test changes locally if possible
2. Create a pull request with improvements
3. Include explanation of changes
4. Test in your fork before merging

## License

This workflow documentation is part of the chip-debroot project.




