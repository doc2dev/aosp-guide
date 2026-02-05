# Chapter 2: Development Environment & Workflow

## Introduction

Building Android from source is significantly more complex than typical application development. You're not just compiling code—you're building an entire operating system with millions of lines of code, generating bootable system images, and managing a sophisticated build environment.

This chapter will guide you through setting up a proper AOSP development environment, understanding the build process, and establishing an efficient workflow. By the end, you'll be able to build AOSP from source, make modifications, and test your changes on actual hardware or emulators.

## System Requirements

AOSP builds are resource-intensive. Before starting, ensure your system meets these requirements:

### Minimum Requirements
- **OS**: Linux (Ubuntu 20.04 LTS or newer recommended, or Debian)
- **RAM**: 16GB minimum, 32GB recommended, 64GB+ for optimal performance
- **Disk Space**: 250GB minimum, 400GB+ recommended
  - Source code: ~100GB
  - Build artifacts: 100-200GB depending on target
  - Additional space for multiple branches/builds
- **CPU**: 64-bit x86 processor, multi-core (8+ cores recommended)
- **Internet**: Fast, stable connection for initial sync

### Recommended Hardware
```
CPU: AMD Ryzen 9 / Intel i9 or better (16+ cores)
RAM: 64GB DDR4
SSD: 1TB NVMe SSD (critical for build speed)
Network: Gigabit Ethernet
```

### Why These Requirements?

- **RAM**: The build system loads many files into memory; insufficient RAM causes swapping and dramatically slows builds
- **Disk Space**: Multiple build targets and branches quickly consume space
- **SSD**: Random I/O operations during build are drastically faster on SSD
- **CPU cores**: Build parallelism scales nearly linearly with core count

## Setting Up the Build Environment

### Installing Required Packages

For Ubuntu 20.04 or newer:

```bash
sudo apt-get update
sudo apt-get install -y \
    git-core gnupg flex bison build-essential zip curl zlib1g-dev \
    gcc-multilib g++-multilib libc6-dev-i386 libncurses5 \
    lib32ncurses5-dev x11proto-core-dev libx11-dev lib32z1-dev \
    libgl1-mesa-dev libxml2-utils xsltproc unzip fontconfig \
    python3 python-is-python3
```

For Ubuntu 22.04+ you may also need:

```bash
sudo apt-get install -y \
    libncurses5 libncurses6 libncurses-dev
```

### Configuring Git

Git is essential for working with AOSP:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"

# Optional but recommended
git config --global color.ui auto
git config --global core.editor vim  # or your preferred editor
```

### Installing repo

The `repo` tool manages the multiple git repositories that comprise AOSP:

```bash
# Create a bin directory in your home folder
mkdir -p ~/bin
export PATH=~/bin:$PATH

# Add to your .bashrc or .zshrc to make it permanent
echo 'export PATH=~/bin:$PATH' >> ~/.bashrc

# Download repo
curl https://storage.googleapis.com/git-repo-downloads/repo > ~/bin/repo
chmod a+x ~/bin/repo
```

Verify installation:

```bash
repo version
```

### Setting Up ccache (Optional but Recommended)

`ccache` dramatically speeds up rebuilds by caching compilation results:

```bash
sudo apt-get install ccache

# Add to ~/.bashrc
export USE_CCACHE=1
export CCACHE_DIR=/path/to/ccache  # Choose a location with enough space
export CCACHE_EXEC=/usr/bin/ccache

# Set cache size (50-100GB recommended)
ccache -M 50G
```

Check ccache statistics:

```bash
ccache -s
```

### File System Considerations

For optimal performance:

1. **Use ext4 or f2fs**: These filesystems perform best for AOSP builds
2. **Avoid encryption on build directory**: Encryption adds overhead
3. **Disable access time updates**: Add `noatime` mount option
4. **Consider separate partition**: Isolate AOSP from system files

Example `/etc/fstab` entry:

```
/dev/sda2  /aosp  ext4  noatime,errors=remount-ro  0  1
```

## Downloading the AOSP Source

### Choosing a Branch

Android releases are tagged with specific version identifiers. Visit https://source.android.com/docs/setup/about/build-numbers to see available branches.

Common branch naming:
- `android-14.0.0_r1` - Android 14 initial release
- `android-13.0.0_r50` - Android 13 with patches
- `main` - Development branch (unstable)

### Initializing the Repository

Create a directory for AOSP and initialize:

```bash
mkdir -p ~/aosp
cd ~/aosp

# Initialize repo for Android 14
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1

# Or for a specific device (e.g., Pixel)
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1 \
    -g default,-x86,-mips
```

The `-g` flag specifies groups to sync. Common groups:
- `default`: Standard AOSP
- `pdk`: Platform Development Kit
- `-x86,-mips`: Exclude x86 and MIPS architectures

### Syncing the Source

The initial sync downloads ~100GB and can take hours:

```bash
# Sync with maximum parallelism
repo sync -c -j$(nproc) --force-sync --no-clone-bundle --no-tags

# Options explained:
# -c              : Fetch current branch only (saves space)
# -j$(nproc)      : Use all CPU cores
# --force-sync    : Overwrite local changes
# --no-clone-bundle : Disable bundle downloads (sometimes faster)
# --no-tags       : Don't fetch git tags (saves time)
```

### Handling Sync Issues

Syncing can fail due to network issues. If interrupted:

```bash
# Resume sync
repo sync -c -j$(nproc)

# If completely stuck, try:
repo sync -c -j1 --force-sync  # Single-threaded, force overwrite
```

### Verifying the Sync

After sync completes:

```bash
# Check repo status
repo status

# Should show all projects up-to-date
# Any project with changes indicates incomplete sync
```

## Understanding Build Targets

Before building, you need to choose a target device configuration.

### Listing Available Targets

```bash
cd ~/aosp
source build/envsetup.sh
lunch
```

This displays available lunch combos in format: `<product_name>-<build_variant>`

Example targets:
```
aosp_arm64-eng          Generic ARM64 device (engineering build)
aosp_x86_64-userdebug   Generic x86_64 device (userdebug build)
sdk_phone_x86_64-eng    SDK emulator
aosp_cf_x86_64_phone-userdebug  Cuttlefish virtual device
```

### Build Variants Explained

**eng** (Engineering):
- Debug enabled
- Root access by default
- Additional debugging tools
- Less optimized
- Use for active development

**userdebug**:
- Root access available (adb root)
- Optimized like user builds
- Some debugging capability
- Use for testing and QA

**user**:
- Production configuration
- No root access
- Fully optimized
- Limited logging
- Use for release builds

### Product vs. Device

- **Product**: Defines what gets built (apps, libraries, configs)
- **Device**: Hardware-specific settings (kernel, drivers, partitions)

Common products:
- `aosp_*`: Generic AOSP without proprietary apps
- `sdk_*`: Emulator targets
- `aosp_cf_*`: Cuttlefish virtual device

## Building AOSP

### Full System Build

The complete process:

```bash
# 1. Navigate to AOSP root
cd ~/aosp

# 2. Initialize environment
source build/envsetup.sh

# 3. Choose target
lunch aosp_x86_64-userdebug

# 4. Start build
m -j$(nproc)
```

The `m` command is a wrapper around `make` with AOSP-specific settings.

### Build Time Expectations

First build times (varies greatly by hardware):
- High-end workstation (32 cores, 64GB RAM, NVMe): 1-2 hours
- Mid-range (16 cores, 32GB RAM, SSD): 3-4 hours
- Minimum spec (8 cores, 16GB RAM, HDD): 6-8+ hours

Subsequent builds with ccache: 10-30 minutes for minor changes

### Understanding Build Output

During build, you'll see:

```
[  1% 234/15234] target  C: libfoo <= foo.c
[ 12% 2456/15234] target  Java: framework (frameworks/base/core/java)
[ 45% 6789/15234] target  SharedLib: libbinder (out/target/.../libbinder.so)
[100% 15234/15234] Create system image
```

Format: `[percentage completed/total] action: target`

### Monitoring Build Progress

In another terminal:

```bash
# Watch build progress
watch -n 1 'tail -20 /tmp/build.log'

# Monitor system resources
htop

# Check ccache effectiveness
watch -n 5 ccache -s
```

### Build Errors

Common errors and solutions:

**Out of memory**:
```
c++: internal compiler error: Killed (program cc1plus)
```
Solution: Reduce parallelism: `m -j8` instead of `m -j$(nproc)`

**Disk space**:
```
No space left on device
```
Solution: Free space or increase disk allocation

**Missing dependencies**:
```
/bin/bash: bison: command not found
```
Solution: Install missing package: `sudo apt-get install bison`

**Repo out of sync**:
```
error: revision refs/heads/android-14.0.0_r1 not found
```
Solution: Re-sync: `repo sync -c -j$(nproc)`

### Cleaning Builds

When needed:

```bash
# Clean everything (full rebuild required)
m clean

# Clean specific module
m clean-<module_name>

# Clean output directory (nuclear option)
rm -rf out/

# Remove build artifacts but keep ccache
m installclean
```

## Building Specific Modules

You don't always need to rebuild the entire system. AOSP provides tools for incremental builds.

### Module Build Commands

```bash
# Build a specific module
m <module_name>

# Examples:
m SystemUI              # Build SystemUI
m framework            # Build framework.jar
m services             # Build services.jar
m Settings             # Build Settings app
```

### Finding Module Names

Module names are defined in `Android.bp` or `Android.mk` files:

```bash
# Find module definition
grep -r "name: \"SystemUI\"" frameworks/base/

# Or check module-info
m module-info | grep -i systemui
```

### Directory-Specific Builds

```bash
# Build everything in current directory
mm

# Build everything in current directory and dependencies
mmm path/to/directory

# Examples:
cd frameworks/base/packages/SystemUI
mm

# Or from AOSP root
mmm frameworks/base/packages/SystemUI
```

### The m/mm/mmm Family

```bash
m       # Build from top of tree (can run from any directory)
mm      # Build current directory only
mmm     # Build specific directory
mma     # Build current directory and dependencies
mmma    # Build specific directory and dependencies
```

### Faster Iteration with Module Builds

Typical workflow for framework development:

```bash
# Initial full build
m -j$(nproc)

# Make changes to ActivityManagerService
vim frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java

# Rebuild just the services module
m services

# Push to device and restart
adb root
adb remount
adb push out/target/product/generic_x86_64/system/framework/services.jar /system/framework/
adb reboot
```

## Working with Build Environment Variables

### Essential Variables

After `source build/envsetup.sh` and `lunch`, important variables are set:

```bash
# Check current target
echo $TARGET_PRODUCT        # e.g., aosp_x86_64
echo $TARGET_BUILD_VARIANT  # e.g., userdebug

# Build paths
echo $ANDROID_BUILD_TOP     # AOSP root directory
echo $ANDROID_PRODUCT_OUT   # Output directory for current target
echo $ANDROID_HOST_OUT      # Host tools output directory

# Navigate quickly
cd $ANDROID_BUILD_TOP                    # Jump to AOSP root
cd $ANDROID_PRODUCT_OUT                  # Jump to build output
cd $ANDROID_PRODUCT_OUT/system/framework # Jump to framework JARs
```

### Useful Shell Functions

`envsetup.sh` provides many helpful functions:

```bash
# Search for files
croot     # Change to AOSP root
cgrep     # Grep in C/C++ files
jgrep     # Grep in Java files
resgrep   # Grep in resource files
mangrep   # Grep in AndroidManifest.xml files
sepgrep   # Grep in SELinux policy files

# Examples:
jgrep "ActivityManagerService"
cgrep "SurfaceFlinger"
resgrep "status_bar_height"
```

### Custom Environment Setup

Create `~/.aosprc` for personal aliases:

```bash
# ~/.aosprc
export ANDROID_ROOT=~/aosp
export USE_CCACHE=1
export CCACHE_DIR=~/.ccache

alias aosp='cd $ANDROID_ROOT'
alias aospbuild='aosp && source build/envsetup.sh'
alias build-full='m -j$(nproc)'
alias build-framework='m framework services SystemUI'

# Function to quickly rebuild and push framework
function push-framework() {
    m services && \
    adb root && \
    adb remount && \
    adb push $ANDROID_PRODUCT_OUT/system/framework/services.jar /system/framework/ && \
    adb reboot
}
```

Source it from `.bashrc`:

```bash
echo 'source ~/.aosprc' >> ~/.bashrc
```

## Flashing and Testing

### Using the Emulator

The AOSP emulator is the easiest way to test changes:

```bash
# Launch emulator
emulator

# Or specify a target
emulator -system $ANDROID_PRODUCT_OUT/system.img \
         -ramdisk $ANDROID_PRODUCT_OUT/ramdisk.img \
         -kernel prebuilts/qemu-kernel/x86_64/kernel-qemu2
```

The emulator looks for images in `$ANDROID_PRODUCT_OUT` by default.

### Using Cuttlefish (Recommended)

Cuttlefish is Google's virtual Android device, offering better performance than the emulator:

```bash
# Build for Cuttlefish
lunch aosp_cf_x86_64_phone-userdebug
m -j$(nproc)

# Install Cuttlefish tools (one-time setup)
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils

# Launch Cuttlefish
launch_cvd

# Connect via adb
adb connect localhost:6520

# Web interface available at
# https://localhost:8443
```

Cuttlefish advantages:
- Faster boot and runtime
- Better graphics performance
- Easy snapshotting
- Web-based UI

### Flashing Physical Devices

For supported devices (Pixel phones):

```bash
# Unlock bootloader first (if not already)
adb reboot bootloader
fastboot flashing unlock

# Flash all partitions
fastboot flashall

# Or flash specific partitions
fastboot flash boot boot.img
fastboot flash system system.img
fastboot flash vendor vendor.img
fastboot flash userdata userdata.img
```

**Warning**: Flashing will erase all data. Backup first!

### Incremental Flashing

When developing, you often only need to update specific partitions:

```bash
# Rebuild system image only
m systemimage

# Flash just system partition
adb reboot bootloader
fastboot flash system $ANDROID_PRODUCT_OUT/system.img
fastboot reboot
```

For framework changes that don't require full system image:

```bash
# Make device writable
adb root
adb remount

# Push specific files
adb push $ANDROID_PRODUCT_OUT/system/framework/framework.jar /system/framework/
adb push $ANDROID_PRODUCT_OUT/system/framework/services.jar /system/framework/

# Restart system server (faster than reboot)
adb shell stop
adb shell start

# Or reboot
adb reboot
```

## Debugging Tools and Techniques

### ADB Essentials

ADB is your primary interface to the device:

```bash
# Check connected devices
adb devices

# Multiple devices? Specify target
adb -s <serial> <command>

# Root access (userdebug/eng builds)
adb root

# Remount partitions as read-write
adb remount

# Shell access
adb shell

# Pull/push files
adb pull /system/framework/services.jar .
adb push services.jar /system/framework/

# Port forwarding
adb forward tcp:8080 tcp:8080

# Screen capture
adb shell screencap /sdcard/screen.png
adb pull /sdcard/screen.png
```

### Logcat

Viewing system logs:

```bash
# View all logs
adb logcat

# Clear log buffer first
adb logcat -c && adb logcat

# Filter by tag
adb logcat ActivityManager:V *:S

# Filter by priority (V/D/I/W/E/F)
adb logcat *:W  # Warnings and above only

# Grep for specific content
adb logcat | grep -i "my search term"

# Save to file
adb logcat > system.log

# View specific buffer
adb logcat -b system     # System logs
adb logcat -b events     # Event logs
adb logcat -b crash      # Crash logs
```

Log priority levels:
- V: Verbose
- D: Debug
- I: Info
- W: Warning
- E: Error
- F: Fatal

### Adding Logging to Your Code

In Java:

```java
import android.util.Log;

private static final String TAG = "MyService";

Log.v(TAG, "Verbose message");
Log.d(TAG, "Debug message");
Log.i(TAG, "Info message");
Log.w(TAG, "Warning message");
Log.e(TAG, "Error message");
Log.wtf(TAG, "What a Terrible Failure");  // For impossible conditions
```

In C++:

```cpp
#define LOG_TAG "MyNativeService"
#include <log/log.h>

ALOGV("Verbose message");
ALOGD("Debug message");
ALOGI("Info message");
ALOGW("Warning message");
ALOGE("Error message");
```

### Debugging System Services

Attach debugger to system_server:

```bash
# Find system_server PID
adb shell ps -A | grep system_server

# Forward debugging port
adb forward tcp:8700 jdwp:<pid>

# Connect with your IDE (Android Studio, IntelliJ)
# Use remote debugger on localhost:8700
```

### Understanding Tombstones

When native code crashes, tombstones are generated:

```bash
# View tombstones
adb shell ls /data/tombstones/

# Pull latest tombstone
adb pull /data/tombstones/tombstone_01

# Tombstone contains:
# - Stack trace
# - Register state
# - Memory maps
# - Signal information
```

Use `stack` tool to symbolicate:

```bash
# In AOSP directory
development/scripts/stack < tombstone_01
```

### Performance Profiling

**systrace**: System-wide performance trace

```bash
# Capture 10-second trace
python $ANDROID_BUILD_TOP/external/chromium-trace/systrace.py \
    --time=10 -o trace.html \
    sched freq idle am wm gfx view binder_driver hal dalvik input res

# Open trace.html in Chrome
```

**simpleperf**: Native profiling

```bash
# Profile system_server
adb shell simpleperf record -p $(adb shell pidof system_server) -g -o /data/perf.data
adb pull /data/perf.data
simpleperf report -i perf.data
```

**perfetto**: Modern tracing framework

```bash
# Web UI
https://ui.perfetto.dev

# Capture trace
adb shell perfetto -o /data/misc/perfetto-traces/trace.perfetto-trace \
    -t 20s sched freq idle am wm

adb pull /data/misc/perfetto-traces/trace.perfetto-trace
# Upload to ui.perfetto.dev
```

## Development Workflow Best Practices

### Iterative Development Cycle

Efficient AOSP development follows this pattern:

1. **Make targeted changes**: Edit specific files
2. **Incremental build**: Build only affected modules
3. **Fast deployment**: Push changes without full flash
4. **Quick test**: Verify specific functionality
5. **Iterate**: Repeat until satisfied

Example workflow for SystemUI change:

```bash
# 1. Make changes
vim frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java

# 2. Rebuild module
m SystemUI

# 3. Install on device
adb root && adb remount
adb shell rm -rf /system/priv-app/SystemUI/oat  # Clear compiled code
adb push $ANDROID_PRODUCT_OUT/system/priv-app/SystemUI/SystemUI.apk /system/priv-app/SystemUI/

# 4. Restart SystemUI
adb shell killall com.android.systemui

# 5. Test and verify
adb logcat -b main -b system | grep SystemUI
```

### Branch Management

Use branches for different features:

```bash
# Start new feature branch
repo start my-feature --all

# Or for specific projects
repo start my-feature frameworks/base system/core

# Check status across repos
repo status

# Switch branches
repo checkout main
repo checkout my-feature

# Abandon branch
repo abandon my-feature
```

### Creating Patches

Generate patches for sharing or review:

```bash
# Create patch from working directory
repo diff > my-changes.patch

# Create patch for specific project
cd frameworks/base
git diff > framework-changes.patch

# Apply patch
patch -p1 < my-changes.patch

# Or with git
git apply my-changes.patch
```

### Using Git Effectively

For individual repository work:

```bash
cd frameworks/base

# Create local branch
git checkout -b my-feature

# Make commits
git add .
git commit -m "Add custom feature to framework"

# View history
git log --oneline

# Interactive rebase for clean history
git rebase -i HEAD~5

# Create patch series
git format-patch HEAD~5
```

### Code Review Integration

If working with Gerrit (Android's code review system):

```bash
# Configure git hooks (one-time)
gitdir=$(git rev-parse --git-dir)
scp -p -P 29418 review.example.com:hooks/commit-msg ${gitdir}/hooks/

# Make changes and commit
git add .
git commit

# Upload for review
repo upload

# Or with git
git push origin HEAD:refs/for/main
```

## Optimizing Build Performance

### Parallelization Tuning

The `-j` flag controls parallel jobs. Optimal value depends on your system:

```bash
# Rule of thumb: 1.5x to 2x your core count
# 16 cores → use -j24 to -j32

# Too many jobs can slow down due to context switching
# Monitor with htop to find sweet spot

# During active development, leave cores free
m -j$(($(nproc) - 2))
```

### Using RAM Disk (tmpfs)

For systems with abundant RAM (64GB+):

```bash
# Create tmpfs mount for build output
sudo mkdir -p /mnt/aosp-build
sudo mount -t tmpfs -o size=64G tmpfs /mnt/aosp-build

# Symlink output directory
cd ~/aosp
rm -rf out
ln -s /mnt/aosp-build out

# Add to fstab for persistence
echo "tmpfs /mnt/aosp-build tmpfs defaults,size=64G 0 0" | sudo tee -a /etc/fstab
```

This dramatically speeds up builds but output is lost on reboot.

### Distributed Builds

For teams or multi-machine setups, consider distcc or icecc:

```bash
# Install distcc
sudo apt-get install distcc

# Configure build to use distcc
export CCACHE_PREFIX=distcc

# Set up distcc hosts
export DISTCC_HOSTS="localhost/8 otherhost/16"
```

### Monitoring Build Efficiency

Track build times to measure improvements:

```bash
# Time a build
time m -j$(nproc)

# Check ccache hit rate
ccache -s | grep "cache hit rate"

# Goal: >90% hit rate after first build
```

### Incremental Build Tips

1. **Avoid clean builds**: Only clean when necessary
2. **Use module builds**: Don't rebuild everything for small changes
3. **Maximize ccache**: Ensure ccache is properly configured
4. **Keep source clean**: Avoid modifying generated files
5. **Separate build outputs**: Use separate out/ directories for different targets

## Troubleshooting Common Issues

### Build Hangs

If build appears frozen:

```bash
# Check for zombie processes
ps aux | grep defunct

# Kill hanging jobs
killall java cc1 cc1plus

# Restart build with lower parallelism
m -j4
```

### Inconsistent Builds

If builds produce different results:

```bash
# Clean specific output
m installclean

# Or nuclear option
rm -rf out/
```

### Permission Errors

```bash
# Fix ownership
sudo chown -R $USER:$USER ~/aosp

# Fix permissions
chmod -R u+rw ~/aosp
```

### Network Timeout During Sync

```bash
# Increase git timeout
git config --global http.postBuffer 524288000
git config --global http.lowSpeedLimit 0
git config --global http.lowSpeedTime 999999

# Retry sync
repo sync -c -j4 --force-sync
```

## Advanced Build Configuration

### Product Configuration

Create custom product configuration in `device/manufacturer/product/`:

```makefile
# device.mk
$(call inherit-product, $(SRC_TARGET_DIR)/product/aosp_base_telephony.mk)

PRODUCT_NAME := my_custom_product
PRODUCT_DEVICE := my_device
PRODUCT_BRAND := MyBrand
PRODUCT_MODEL := My Custom Device
PRODUCT_MANUFACTURER := MyCompany

# Add custom packages
PRODUCT_PACKAGES += \
    MyCustomApp \
    MyCustomService

# Set properties
PRODUCT_PROPERTY_OVERRIDES += \
    ro.my.custom.property=value
```

### Build Variants

Create custom build variant:

```makefile
# In BoardConfig.mk
# Define custom variant
CUSTOM_BUILD_VARIANTS := myvariant

# In device.mk
ifeq ($(TARGET_BUILD_VARIANT),myvariant)
    # Custom configuration for myvariant
    PRODUCT_PACKAGES += DebugTools
endif
```

### Conditional Compilation

Use build flags for conditional features:

```makefile
# In Android.bp
cc_library {
    name: "libmylib",
    srcs: ["mylib.cpp"],
    cflags: [
        "-DFEATURE_ENABLED",
    ],
}
```

In code:

```cpp
#ifdef FEATURE_ENABLED
    // Feature-specific code
#endif
```

## Development Environment Automation

### Build Scripts

Create `build.sh` for common operations:

```bash
#!/bin/bash
# build.sh - AOSP build automation

set -e  # Exit on error

AOSP_ROOT=~/aosp
PRODUCT=aosp_x86_64-userdebug

build_full() {
    echo "Building full system..."
    cd $AOSP_ROOT
    source build/envsetup.sh
    lunch $PRODUCT
    m -j$(nproc)
}

build_framework() {
    echo "Building framework..."
    cd $AOSP_ROOT
    source build/envsetup.sh
    lunch $PRODUCT
    m framework services SystemUI
}

flash_device() {
    echo "Flashing device..."
    adb reboot bootloader
    fastboot flashall -w
}

push_framework() {
    echo "Pushing framework to device..."
    adb root
    adb remount
    adb push $AOSP_ROOT/out/target/product/*/system/framework/services.jar /system/framework/
    adb reboot
}

case "$1" in
    full)
        build_full
        ;;
    framework)
        build_framework
        ;;
    flash)
        flash_device
        ;;
    push)
        push_framework
        ;;
    *)
        echo "Usage: $0 {full|framework|flash|push}"
        exit 1
        ;;
esac
```

Make executable:

```bash
chmod +x build.sh
./build.sh framework
```

## Key Takeaways

1. **Hardware matters**: Invest in SSD, RAM, and multi-core CPU for productive development
2. **ccache is essential**: Configure it properly for 10x+ faster rebuilds
3. **Module builds save time**: Learn to build only what you changed
4. **Master the tools**: adb, logcat, and build environment functions are your daily drivers
5. **Iterate quickly**: Push changes incrementally rather than flashing full images
6. **Automate repetitive tasks**: Create scripts for common workflows
7. **Monitor resources**: Use htop, ccache stats to optimize your setup

## Next Steps

In Chapter 3, we'll dive deep into Android's init process and boot sequence. You'll learn how the system starts from the bootloader through to the fully running Android environment, and we'll create a custom init service to see this knowledge in action.

## Quick Reference

### Essential Commands Cheat Sheet

```bash
# Environment setup
source build/envsetup.sh
lunch <target>

# Building
m -j$(nproc)           # Full build
m <module>             # Module build
mm                     # Build current directory
mmm <path>             # Build specific directory

# Cleaning
m clean                # Clean all
m installclean         # Clean output, keep ccache

# Device interaction
adb root               # Root access
adb remount            # Remount partitions R/W
adb push <src> <dst>   # Copy to device
adb pull <src> <dst>   # Copy from device
adb reboot             # Reboot device
adb logcat             # View logs

# Flashing
fastboot flashall      # Flash all partitions
fastboot flash <part>  # Flash specific partition

# Debugging
adb shell              # Shell access
adb logcat             # System logs
adb bugreport          # Generate bug report

# Repository management
repo sync              # Sync all repos
repo start <branch>    # Create branch
repo status            # Check status
```

### Recommended Directory Structure

```
~/aosp/                          # AOSP source
~/.ccache/                       # ccache directory
~/bin/                           # Personal tools
~/aosp-scripts/                  # Build automation scripts
/mnt/aosp-build/ (optional)      # tmpfs for fast builds
```
