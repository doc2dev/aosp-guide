# Chapter 3: Android Init & Boot Process

## Contents

- [Introduction](#introduction)
- [Boot Sequence Overview](#boot-sequence-overview)
- [Stage 1: Boot ROM](#stage-1-boot-rom)
- [Stage 2: Bootloader](#stage-2-bootloader)
- [Stage 3: Linux Kernel](#stage-3-linux-kernel)
- [Stage 4: Init Process](#stage-4-init-process)
- [Actions in Init Language](#actions-in-init-language)
- [Services in Init Language](#services-in-init-language)
- [Real-World Init.rc Example](#real-world-initrc-example)
- [Property System](#property-system)
- [SELinux in Boot Context](#selinux-in-boot-context)
- [Practical Example 1: Adding a Custom Init Service](#practical-example-1-adding-a-custom-init-service)
- [Practical Example 2: Custom Boot Action](#practical-example-2-custom-boot-action)
- [Debugging Init Issues](#debugging-init-issues)
- [Boot Time Optimization](#boot-time-optimization)
- [Advanced Init Features](#advanced-init-features)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)
- [Quick Reference](#quick-reference)

## Introduction

Understanding the Android boot process is fundamental to AOSP development. From the moment you press the power button to a fully functioning Android system, a complex sequence of events unfolds across multiple layers of software. Each stage has specific responsibilities, and understanding this flow allows you to customize boot behavior, add system services, debug boot issues, and optimize startup time.

This chapter provides a comprehensive walkthrough of the Android boot sequence, deep dives into the init process and its configuration language, explains SELinux's role during boot, and concludes with practical examples of adding custom init services.

## Boot Sequence Overview

The Android boot process can be divided into several distinct stages:

```
Power On
   ↓
Boot ROM (BootLoader Stage 1)
   ↓
Bootloader (BootLoader Stage 2)
   ↓
Kernel
   ↓
Init (First User-Space Process)
   ↓
SELinux Setup
   ↓
Init Stage 2 (Property Service, etc.)
   ↓
Zygote
   ↓
System Server
   ↓
System Services
   ↓
Boot Complete
```

Let's explore each stage in detail.

## Stage 1: Boot ROM

### What Happens

When you press the power button, the CPU begins executing code from a small ROM embedded in the SoC (System on Chip). This code is immutable and burned into the hardware during manufacturing.

### Responsibilities

1. **Basic hardware initialization**
   - Initialize CPU and memory controllers
   - Set up minimal I/O for loading next stage

2. **Find and load bootloader**
   - Check predefined locations (usually internal storage)
   - Verify bootloader signature (on locked devices)
   - Load bootloader into RAM

3. **Transfer control to bootloader**

### Developer Impact

You generally cannot modify this stage, but understanding it helps debug low-level boot failures. If a device doesn't reach the bootloader, the issue is often hardware-related or requires specialized recovery tools.

## Stage 2: Bootloader

### What Happens

The bootloader is the first software stage you can typically interact with. Common Android bootloaders include:
- **U-Boot**: Open-source bootloader, common in development boards
- **LittleKernel (LK)**: Used in many Qualcomm-based devices
- **Proprietary bootloaders**: Manufacturer-specific implementations

### Responsibilities

1. **Hardware initialization**
   - Initialize display (for boot logo)
   - Initialize storage controllers
   - Set up memory

2. **Boot mode selection**
   - Normal boot → Load boot partition
   - Recovery mode → Load recovery partition
   - Fastboot mode → Enter fastboot protocol for flashing
   - Download mode → Manufacturer-specific flash mode

3. **Verified Boot (if enabled)**
   - Verify boot image signature
   - Check rollback index
   - Enforce bootloader lock state

4. **Load kernel and ramdisk**
   - Read boot.img from boot partition
   - Extract kernel and ramdisk
   - Load into memory

5. **Pass control to kernel**
   - Set up kernel command line arguments
   - Jump to kernel entry point

### Boot Image Structure

The boot partition contains a boot.img file with this structure:

```
boot.img
├── Header
│   ├── Magic number
│   ├── Kernel size
│   ├── Ramdisk size
│   ├── Page size
│   └── Command line arguments
├── Kernel (zImage or Image.gz)
├── Ramdisk (cpio.gz)
├── Second stage bootloader (optional)
└── Device tree blob (DTB)
```

You can examine boot.img with:

```bash
# Extract boot image
cd $ANDROID_PRODUCT_OUT
abootimg -i boot.img

# Or use unpack_bootimg.py
python3 $ANDROID_BUILD_TOP/system/tools/mkbootimg/unpack_bootimg.py \
    --boot_img boot.img \
    --out extracted/
```

### Kernel Command Line

The bootloader passes arguments to the kernel via the command line. View on a running device:

```bash
adb shell cat /proc/cmdline
```

Example output:
```
console=ttyMSM0,115200,n8 androidboot.hardware=qcom 
androidboot.console=ttyMSM0 androidboot.memcg=1 
lpm_levels.sleep_disabled=1 androidboot.selinux=permissive
```

Key parameters:
- `console=`: Kernel console output device
- `androidboot.hardware=`: Hardware platform name (used by init)
- `androidboot.selinux=`: SELinux mode (enforcing/permissive)
- `androidboot.serialno=`: Device serial number

## Stage 3: Linux Kernel

### What Happens

The Linux kernel initializes the core operating system functionality. Android uses a modified Linux kernel with Android-specific patches.

### Key Kernel Responsibilities

1. **Memory management initialization**
   - Set up page tables
   - Initialize memory allocators
   - Configure low memory killer

2. **Process scheduler**
   - Initialize task scheduler
   - Set up CPU affinity and scheduling classes

3. **Driver initialization**
   - Initialize built-in drivers
   - Mount initramfs

4. **Mount rootfs**
   - Extract and mount the ramdisk (initramfs)
   - Ramdisk becomes the initial root filesystem

5. **Execute init**
   - Locate `/init` in the ramdisk
   - Execute it as PID 1

### Android-Specific Kernel Features

**Binder Driver** (`/dev/binder`):
- IPC mechanism for Android
- Facilitates communication between processes
- Critical for system services

**Ashmem** (Anonymous Shared Memory):
- Shared memory allocator
- Allows memory sharing between processes
- Used extensively by Android framework

**Low Memory Killer**:
- More aggressive than Linux OOM killer
- Kills processes based on oom_adj score
- Maintains responsive system under memory pressure

**Wakelocks** (Legacy) / Wakeup Sources:
- Prevent device from sleeping
- Power management mechanism
- Application-level sleep prevention

**ION Memory Allocator** (Legacy, replaced by DMA-BUF):
- Manages graphics and multimedia memory
- Provides zero-copy buffer sharing

### Viewing Kernel Logs

```bash
# View kernel messages
adb shell dmesg

# Or from kernel log buffer
adb shell cat /proc/kmsg

# During early boot (requires root)
adb shell cat /dev/kmsg
```

## Stage 4: Init Process

### Introduction to Init

Init is the first user-space process started by the kernel. It has PID 1 and is the ancestor of all other processes. Init's responsibilities include:

- Mounting filesystems
- Starting essential services
- Setting up SELinux
- Managing system properties
- Responding to system events

### Init Source Code Location

```
system/core/init/
├── init.cpp              # Main entry point
├── init.h                # Core declarations
├── service.cpp           # Service management
├── action.cpp            # Action execution
├── parser.cpp            # .rc file parser
├── property_service.cpp  # Property system
├── ueventd.cpp          # Device node manager
├── first_stage_init.cpp # First stage init
├── selinux.cpp          # SELinux setup
└── README.md            # Documentation
```

### Init Execution Stages

Init actually runs in two stages:

#### First Stage Init

Located in the ramdisk, runs before SELinux is fully set up.

**Responsibilities:**
1. Mount critical filesystems (tmpfs, devtmpfs)
2. Create device nodes
3. Set up initial SELinux policy
4. Mount system partitions with dm-verity if enabled
5. Switch root and exec into second stage init

Code location: `system/core/init/first_stage_init.cpp`

#### Second Stage Init

Runs after SELinux is enforced, from the system partition.

**Responsibilities:**
1. Initialize property system
2. Parse .rc files
3. Execute early-init actions
4. Set up SELinux contexts
5. Start services
6. Respond to events

Code location: `system/core/init/init.cpp`

### Init Process Flow

```
Kernel executes /init (first stage)
   ↓
Mount essential filesystems
   ↓
Load SELinux policy
   ↓
Re-exec into second stage init
   ↓
Initialize property service
   ↓
Parse init.rc and other .rc files
   ↓
Execute early-init actions
   ↓
Execute init actions
   ↓
Start early services (ueventd)
   ↓
Execute late-init actions
   ↓
Start boot services
   ↓
Wait for properties and events
   ↓
Enter main event loop
```

### Init Configuration Files (.rc files)

Init's behavior is controlled by `.rc` (run command) files written in Android Init Language (AIL). These files are parsed during boot.

**Key .rc files location:**

```
system/core/rootdir/
├── init.rc               # Main init configuration
├── init.zygote32.rc      # 32-bit Zygote
├── init.zygote64.rc      # 64-bit Zygote
└── init.usb.rc          # USB configuration

# Device-specific
device/manufacturer/device/
└── init.device.rc

# Vendor-specific
vendor/manufacturer/
└── init.vendor.rc
```

### Android Init Language (AIL)

AIL is a simple declarative language for configuring init. It consists of four main types of statements:

1. **Actions**: Commands executed when triggers fire
2. **Services**: Programs managed by init
3. **Imports**: Include other .rc files
4. **Options**: Modify behavior of services

## Actions in Init Language

### Action Syntax

```
on <trigger> [&& <trigger>]*
    <command>
    <command>
    ...
```

### Triggers

Triggers determine when an action executes:

**Boot triggers** (execute once):
```
on early-init      # Very early, before SELinux fully set up
on init            # After basic initialization
on late-init       # After all init tasks
on boot            # When system is ready to start services
on charger         # When booting into charger mode
```

**Property triggers** (execute when property changes):
```
on property:sys.boot_completed=1
on property:ro.debuggable=1
```

**Combined triggers**:
```
on boot && property:ro.debuggable=1
```

### Commands

Common commands used in actions:

**Filesystem operations:**
```
mkdir <path> [mode] [owner] [group]     # Create directory
mount <type> <device> <path> [flags]    # Mount filesystem
chmod <octal-mode> <path>               # Change permissions
chown <owner> <group> <path>            # Change ownership
```

**Property operations:**
```
setprop <name> <value>                  # Set property
```

**Process control:**
```
start <service>                         # Start service
stop <service>                          # Stop service
restart <service>                       # Restart service
```

**Security:**
```
seclabel <label>                        # Set SELinux context
setcon <context>                        # Set process SELinux context
```

**Other:**
```
write <path> <string>                   # Write to file
copy <src> <dst>                        # Copy file
exec <path> [args]*                     # Execute program
symlink <target> <path>                 # Create symlink
wait <path> [timeout]                   # Wait for file to exist
```

### Example Actions

```rc
# Create directories for runtime data
on post-fs-data
    mkdir /data/vendor/wifi 0770 wifi wifi
    mkdir /data/misc/dhcp 0770 dhcp dhcp
    chown system system /data/app
    chmod 0771 /data/app

# Start services when boot completes
on property:sys.boot_completed=1
    start my_background_service
    setprop my.custom.property ready

# Debug mode initialization
on property:ro.debuggable=1
    write /proc/sys/kernel/printk "8 8 8 8"
    start logd
```

## Services in Init Language

### Service Syntax

```
service <name> <pathname> [<argument>]*
    <option>
    <option>
    ...
```

### Service Options

**Class:**
```
class <name>         # Service belongs to a class (for group start/stop)
```

Common classes:
- `core`: Core system services
- `main`: Main services
- `late_start`: Services started later in boot
- `hal`: HAL services

**User and Group:**
```
user <username>      # Run as specific user
group <groupname> [<groupname>]*   # Run with specific groups
```

**Capabilities:**
```
capabilities <capability> [<capability>]*
```

Example: `capabilities NET_ADMIN NET_RAW`

**Priority:**
```
priority <priority>  # Process priority (-20 to 19, lower is higher priority)
```

**One-shot:**
```
oneshot             # Don't restart when it exits
```

**Disabled:**
```
disabled            # Don't start automatically, must be started explicitly
```

**Critical:**
```
critical            # Reboot device if service crashes in first 4 minutes
```

**Restart policy:**
```
restart <period>    # How to handle crashes (default, periodic, on-failure)
```

**Console:**
```
console             # Provide console for service
```

**On restart:**
```
onrestart <command> # Execute command when service restarts
```

**Seclabel:**
```
seclabel <context>  # SELinux context for service
```

**Writepid:**
```
writepid <file>     # Write PID to file when service starts
```

**Socket:**
```
socket <name> <type> <perm> [<user> [<group> [<seclabel>]]]
```

Creates a socket before starting the service.

### Example Services

**Simple service:**
```rc
service my_daemon /system/bin/my_daemon
    class main
    user system
    group system
```

**Service with socket:**
```rc
service my_service /system/bin/my_service
    class main
    user system
    group system
    socket my_socket stream 0660 system system
    seclabel u:r:my_service:s0
```

**HAL service:**
```rc
service vendor.my_hal /vendor/bin/hw/android.hardware.my@1.0-service
    class hal
    user system
    group system
    disabled
    seclabel u:r:hal_my:s0
```

**One-shot service:**
```rc
service setup_once /system/bin/setup.sh
    class main
    user root
    group root
    oneshot
    disabled
```

**Critical service with restart handling:**
```rc
service system_server /system/bin/system_server
    class main
    user system
    group system
    critical
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart cameraserver
```

## Real-World Init.rc Example

Let's examine a portion of the actual `init.rc`:

```rc
# system/core/rootdir/init.rc

# Mount filesystems
on early-init
    # Set init and its forked children's oom_adj.
    write /proc/1/oom_score_adj -1000

    # Disable sysrq from keyboard
    write /proc/sys/kernel/sysrq 0

    start ueventd

on init
    # Set up cgroup mount points
    mkdir /dev/cpuctl
    mount cgroup none /dev/cpuctl cpu
    chown system system /dev/cpuctl
    chown system system /dev/cpuctl/tasks
    chmod 0666 /dev/cpuctl/tasks

    # Create basic filesystem structure
    mkdir /system
    mkdir /data 0771 system system
    mkdir /cache 0770 system cache
    mkdir /config 0500 root root

    # Mount tmpfs on /mnt so we can create mount points
    mount tmpfs tmpfs /mnt mode=0755,uid=1000,gid=1000

on late-init
    # Now we can start Zygote
    trigger zygote-start

on zygote-start
    # Start Zygote based on the ro.zygote property
    start zygote
    start zygote_secondary

on boot
    # Define default values for properties
    setprop ro.config.nocheckin yes

    # Start essential services
    class_start core

on property:sys.boot_completed=1
    # Boot completed, start late services
    class_start late_start
```

## Property System

The property system is a key-value store used throughout Android for configuration and inter-process communication.

### Property Characteristics

- **Name-value pairs**: `ro.build.version.sdk=33`
- **Persistent or ephemeral**: Some persist across reboots
- **Type prefixes**:
  - `ro.*`: Read-only (set during boot)
  - `persist.*`: Persistent across reboots
  - `sys.*`: System runtime properties
  - `dev.*`: Device-specific properties

### Property Service

Init runs the property service that manages property access:

**Location:** `system/core/init/property_service.cpp`

**Socket:** `/dev/socket/property_service`

Processes set properties by connecting to this socket and sending requests. Only authorized processes can set certain properties (controlled by SELinux).

### Accessing Properties

**From shell:**
```bash
# Get property
adb shell getprop ro.build.version.sdk

# Set property (if allowed)
adb shell setprop debug.my.property value

# List all properties
adb shell getprop

# Watch property changes
adb shell watchprops
```

**From C++:**
```cpp
#include <android-base/properties.h>

// Get property
std::string value = android::base::GetProperty("ro.build.version.sdk", "default");

// Set property
android::base::SetProperty("debug.my.property", "value");

// Wait for property
bool success = android::base::WaitForProperty("sys.boot_completed", "1",
                                               std::chrono::seconds(60));
```

**From Java:**
```java
import android.os.SystemProperties;

// Get property
String value = SystemProperties.get("ro.build.version.sdk", "default");

// Set property
SystemProperties.set("debug.my.property", "value");
```

**From init.rc:**
```rc
# Set property
on boot
    setprop my.custom.property initialized

# Trigger on property
on property:my.custom.property=initialized
    start my_service
```

### Property Contexts (SELinux)

Properties have SELinux contexts that control access:

**Location:** `system/sepolicy/private/property_contexts`

```
# Format: <property_name> <security_context>

net.rmnet               u:object_r:net_radio_prop:s0
net.gprs                u:object_r:net_radio_prop:s0
persist.sys.locale      u:object_r:system_prop:s0
persist.sys.timezone    u:object_r:system_prop:s0
```

## SELinux in Boot Context

SELinux (Security-Enhanced Linux) is mandatory access control enforced from early boot.

### SELinux Boot Stages

1. **Kernel loads policy** from `/sepolicy` in ramdisk
2. **First-stage init runs** in `u:r:init:s0` context
3. **Policy reloaded** from system partition if needed
4. **Enforcement enabled** (or permissive for debugging)
5. **Processes labeled** according to policy

### SELinux Modes

**Enforcing:**
- Denials are enforced
- Violations are logged and blocked
- Production mode

**Permissive:**
- Denials are logged but not enforced
- Used for policy development and debugging
- Can be set via kernel command line: `androidboot.selinux=permissive`

Check current mode:
```bash
adb shell getenforce
```

Change mode (requires root, not persistent):
```bash
adb shell setenforce 0  # Permissive
adb shell setenforce 1  # Enforcing
```

### SELinux Contexts

Everything in Android has an SELinux context (label):

**Processes:**
```bash
adb shell ps -Z
```

Output:
```
u:r:init:s0                root      1     0   ... /init
u:r:zygote:s0             root      123   1   ... zygote
u:r:system_server:s0      system    234   123 ... system_server
```

**Files:**
```bash
adb shell ls -Z /system/bin/
```

Output:
```
u:object_r:system_file:s0 app_process
u:object_r:init_exec:s0   init
u:object_r:shell_exec:s0  sh
```

### SELinux Policy Files

**Location:** `system/sepolicy/`

```
system/sepolicy/
├── private/          # Private policy (internal to platform)
│   ├── init.te      # Init process policy
│   ├── zygote.te    # Zygote policy
│   ├── file_contexts # File labeling
│   └── ...
├── public/           # Public policy (for vendor)
├── vendor/           # Vendor-specific policy
└── prebuilts/        # Prebuilt policy components
```

**Policy language:**

Example from `init.te`:
```
# Allow init to execute shell scripts
allow init shell_exec:file rx_file_perms;

# Allow init to set all properties
allow init property_type:property_service set;

# Allow init to create device nodes
allow init device:dir create_dir_perms;
allow init device:chr_file create_file_perms;
```

### Viewing SELinux Denials

When SELinux blocks an operation, a denial is logged:

```bash
adb shell dmesg | grep avc
# or
adb logcat -b events | grep avc
```

Example denial:
```
avc: denied { read } for pid=1234 comm="my_service" 
name="config.xml" dev="dm-0" ino=123456 
scontext=u:r:my_service:s0 tcontext=u:object_r:system_file:s0 
tclass=file permissive=0
```

This denial means `my_service` was denied read access to a file labeled `system_file`.

## Practical Example 1: Adding a Custom Init Service

Let's create a simple native daemon that starts during boot.

### Step 1: Create the Service Binary

**File:** `device/mycompany/mydevice/mydaemon/mydaemon.cpp`

```cpp
#include <android-base/logging.h>
#include <android-base/properties.h>
#include <unistd.h>

int main() {
    android::base::InitLogging(nullptr);
    LOG(INFO) << "MyDaemon starting...";
    
    // Set property to indicate daemon is running
    android::base::SetProperty("init.svc.mydaemon", "running");
    
    // Main daemon loop
    while (true) {
        LOG(INFO) << "MyDaemon heartbeat";
        sleep(60);  // Sleep for 60 seconds
        
        // Do useful work here
        // For example, monitor system state, manage hardware, etc.
    }
    
    return 0;
}
```

**File:** `device/mycompany/mydevice/mydaemon/Android.bp`

```go
cc_binary {
    name: "mydaemon",
    srcs: ["mydaemon.cpp"],
    shared_libs: [
        "libbase",
        "liblog",
    ],
    vendor: true,  // Install to vendor partition
    init_rc: ["mydaemon.rc"],
}
```

### Step 2: Create Init Configuration

**File:** `device/mycompany/mydevice/mydaemon/mydaemon.rc`

```rc
service mydaemon /vendor/bin/mydaemon
    class main
    user system
    group system
    seclabel u:r:mydaemon:s0
```

### Step 3: Create SELinux Policy

**File:** `device/mycompany/mydevice/sepolicy/mydaemon.te`

```
# Define mydaemon type
type mydaemon, domain;
type mydaemon_exec, exec_type, vendor_file_type, file_type;

# Allow init to start mydaemon
init_daemon_domain(mydaemon)

# Allow mydaemon to use logging
allow mydaemon kernel:system syslog_read;
allow mydaemon init:unix_stream_socket connectto;
allow mydaemon property_socket:sock_file write;

# Allow setting properties
set_prop(mydaemon, system_prop)
```

**File:** `device/mycompany/mydevice/sepolicy/file_contexts`

```
/vendor/bin/mydaemon    u:object_r:mydaemon_exec:s0
```

### Step 4: Build and Deploy

```bash
# Build the module
m mydaemon

# Push to device (for testing)
adb root
adb remount
adb push $ANDROID_PRODUCT_OUT/vendor/bin/mydaemon /vendor/bin/
adb push device/mycompany/mydevice/mydaemon/mydaemon.rc /vendor/etc/init/

# Reload init configuration
adb shell stop
adb shell start

# Or reboot
adb reboot
```

### Step 5: Verify Service

```bash
# Check if service is running
adb shell ps -A | grep mydaemon

# Check property
adb shell getprop init.svc.mydaemon

# View logs
adb logcat -s mydaemon

# Manually control service
adb shell start mydaemon
adb shell stop mydaemon
```

## Practical Example 2: Custom Boot Action

Let's add a custom action that runs during boot to perform device-specific initialization.

### Step 1: Create Init Script

**File:** `device/mycompany/mydevice/init.mydevice.rc`

```rc
# Custom initialization for mydevice

on early-init
    # Set up custom device nodes
    mkdir /dev/mydevice 0755 system system

on init
    # Create custom directories
    mkdir /data/mydevice 0770 system system
    chown system system /data/mydevice
    
    # Set custom properties
    setprop ro.mydevice.version 1.0
    setprop ro.mydevice.variant production

on late-init
    # Trigger custom initialization
    trigger mydevice-init

on mydevice-init
    # Load custom configuration
    exec - system system -- /vendor/bin/mydevice-setup.sh
    
    # Set up custom networking
    write /proc/sys/net/ipv4/ip_forward 1
    
    # Custom hardware initialization
    write /sys/class/leds/myled/brightness 255

on boot
    # Start custom services
    start mydevice-monitor

on property:sys.boot_completed=1
    # Post-boot initialization
    exec - system system -- /vendor/bin/mydevice-postboot.sh
    setprop mydevice.initialized 1

# Custom service definition
service mydevice-monitor /vendor/bin/mydevice-monitor
    class late_start
    user system
    group system
    disabled
```

### Step 2: Create Setup Script

**File:** `device/mycompany/mydevice/mydevice-setup.sh`

```bash
#!/system/bin/sh

# Custom device setup script

LOG_TAG="mydevice-setup"

log() {
    echo "$LOG_TAG: $1"
}

log "Starting device initialization"

# Detect hardware variant
if [ -f /sys/class/mydevice/variant ]; then
    VARIANT=$(cat /sys/class/mydevice/variant)
    setprop ro.mydevice.detected_variant "$VARIANT"
    log "Detected variant: $VARIANT"
fi

# Configure based on variant
case "$VARIANT" in
    "premium")
        log "Configuring premium variant"
        setprop mydevice.features.advanced true
        ;;
    "standard")
        log "Configuring standard variant"
        setprop mydevice.features.advanced false
        ;;
    *)
        log "Unknown variant, using defaults"
        ;;
esac

log "Device initialization complete"
exit 0
```

### Step 3: Include in Device Build

**File:** `device/mycompany/mydevice/device.mk`

```makefile
# Include custom init scripts
PRODUCT_COPY_FILES += \
    device/mycompany/mydevice/init.mydevice.rc:$(TARGET_COPY_OUT_VENDOR)/etc/init/init.mydevice.rc \
    device/mycompany/mydevice/mydevice-setup.sh:$(TARGET_COPY_OUT_VENDOR)/bin/mydevice-setup.sh

# Ensure scripts are executable
PRODUCT_PACKAGES += \
    mydevice-monitor
```

### Step 4: Test the Configuration

The workflow here depends on your target device.

**On a physical device:**
```bash
# Build and flash via fastboot
m -j$(nproc)
adb reboot bootloader
fastboot flashall

# After boot, verify
adb shell getprop | grep mydevice
adb shell ls -l /data/mydevice
adb logcat -s mydevice-setup
```

**On Cuttlefish (CVD):**

Cuttlefish does not have a `bootloader.img`, so `adb reboot bootloader` is a no-op and `fastboot` cannot be used to flash images. Instead, rebuild the relevant images and relaunch the virtual device:

```bash
# Build the images
m -j$(nproc)

# Stop the running Cuttlefish instance
stop_cvd

# Relaunch with the new images (from $ANDROID_PRODUCT_OUT)
launch_cvd

# After boot, verify
adb shell getprop | grep mydevice
adb shell ls -l /data/mydevice
adb logcat -s mydevice-setup
```

For quick iterative testing of init scripts and pushed binaries without a full relaunch, you can push files directly and use `adb reboot`:

```bash
adb root
adb remount
adb push $ANDROID_PRODUCT_OUT/vendor/bin/mydevice-monitor /vendor/bin/
adb push device/mycompany/mydevice/init.mydevice.rc /vendor/etc/init/
adb reboot
```

## Debugging Init Issues

### Common Problems and Solutions

**Service won't start:**

1. Check init logs:
```bash
adb logcat -b main | grep init
```

2. Verify service definition:
```bash
adb shell cat /vendor/etc/init/mydaemon.rc
```

3. Check SELinux denials:
```bash
adb shell dmesg | grep avc
```

4. Verify binary exists and is executable:
```bash
adb shell ls -l /vendor/bin/mydaemon
```

**Service crashes immediately:**

1. Check crash logs:
```bash
adb shell logcat -b crash
```

2. Check tombstones:
```bash
adb shell ls /data/tombstones/
adb pull /data/tombstones/tombstone_XX
```

3. Run manually to see errors:
```bash
adb shell
su
/vendor/bin/mydaemon
```

**SELinux denials:**

1. Set permissive mode for debugging:
```bash
adb shell setenforce 0
```

2. Reproduce issue and check denials:
```bash
adb logcat -b events | grep avc
```

3. Generate policy from denials:
```bash
# Use audit2allow (on development machine)
adb shell dmesg | grep avc | audit2allow
```

4. Add rules to policy and rebuild

**Property not being set:**

1. Check property permissions:
```bash
adb shell ls -lZ /dev/__properties__
```

2. Verify SELinux allows property access:
```bash
# Check if your service can set the property
adb shell dmesg | grep avc | grep property
```

3. Check property_contexts file

### Init Debugging Tools

**Verbose init logging:**

Add to kernel command line:
```
androidboot.init_log_level=debug
```

**Init console:**

Some devices support init console for early debugging:
```
androidboot.console=ttyMSM0
```

**Manual service control:**

```bash
# List all services
adb shell service list

# Start/stop services
adb shell start <service_name>
adb shell stop <service_name>
adb shell restart <service_name>
```

**Property watching:**

```bash
# Watch property changes in real-time
adb shell watchprops
```

## Boot Time Optimization

Understanding boot flow enables optimization. Here are key techniques:

### 1. Parallelize Service Startup

Use different classes to control startup order:

```rc
# Fast services start early
service quick_service /vendor/bin/quick_service
    class early_hal
    
# Slow services can start later
service slow_service /vendor/bin/slow_service
    class late_start
```

### 2. Lazy Service Loading

Use `disabled` and start services on-demand:

```rc
service optional_service /vendor/bin/optional_service
    class main
    disabled

on property:sys.feature.needed=1
    start optional_service
```

### 3. Reduce Dependencies

Minimize service dependencies to allow parallel startup:

```rc
# Instead of:
on boot
    start serviceA
    start serviceB  # Waits for serviceA

# Use:
on boot
    start serviceA

on boot
    start serviceB  # Starts in parallel
```

### 4. Optimize Init Scripts

- Remove unnecessary commands
- Combine multiple actions where possible
- Use `exec` instead of starting shell for simple tasks

### 5. Profile Boot Time

```bash
# Boot time analysis
adb logcat -b events | grep boot_progress

# Detailed timing
adb shell dmesg | grep -i timing
```

## Advanced Init Features

### Importing Files

```rc
import /vendor/etc/init/hw/init.${ro.hardware}.rc
import /vendor/etc/init/*.rc
```

### File-Based Triggers

Wait for files to exist:

```rc
on property:init.svc.myservice=running
    wait /dev/mydevice 5
    write /dev/mydevice "initialized"
```

### Service Groups (Classes)

Start/stop services as a group:

```rc
service service1 /vendor/bin/service1
    class myclass

service service2 /vendor/bin/service2
    class myclass

on boot
    class_start myclass  # Starts both services

on shutdown
    class_stop myclass   # Stops both services
```

### Dependency Management

Using `onrestart`:

```rc
service critical_service /vendor/bin/critical
    class main
    critical

service dependent_service /vendor/bin/dependent
    class main
    
on property:init.svc.critical_service=restarting
    restart dependent_service
```

## Key Takeaways

1. **Boot is multi-stage**: Boot ROM → Bootloader → Kernel → Init → Android
2. **Init is central**: First user-space process, manages all services and system setup
3. **Init language is declarative**: Actions and services defined in .rc files
4. **Properties enable communication**: Key-value store for configuration and signaling
5. **SELinux is enforced from boot**: Security contexts assigned early, denials must be addressed
6. **Services can be customized**: Add your own daemons and startup logic
7. **Understanding boot flow enables optimization**: Profile and parallelize for faster boot

## Next Steps

In Chapter 4, we'll explore Binder IPC in depth—the communication mechanism that enables all of Android's inter-process communication. You'll learn how Binder works at a deep level and how to create custom system services that integrate with the Android framework.

## Quick Reference

### Common Init Commands

```bash
# Service control
adb shell start <service>
adb shell stop <service>
adb shell restart <service>

# Property management
adb shell getprop <name>
adb shell setprop <name> <value>
adb shell watchprops

# Init debugging
adb logcat -b main | grep init
adb shell dmesg | grep init

# SELinux
adb shell getenforce
adb shell setenforce [0|1]
adb shell dmesg | grep avc
```

### Init.rc Quick Syntax

```rc
# Action
on <trigger>
    <command>

# Service
service <name> <path> [args]
    class <class>
    user <user>
    group <group>
    
# Import
import <path>

# Property
setprop <name> <value>
```

### Important Boot Properties

```
sys.boot_completed      # 1 when boot finishes
ro.boottime.*          # Boot timing information
init.svc.<service>     # Service state
ro.build.*             # Build information
ro.hardware            # Hardware platform
```
