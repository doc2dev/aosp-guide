# Chapter 12: Build System Mastery

## Contents

- [Introduction](#introduction)
- [Build System Evolution](#build-system-evolution)
- [Android.bp Syntax](#androidbp-syntax)
- [Build Configuration](#build-configuration)
- [Module Dependencies](#module-dependencies)
- [Compiler Flags](#compiler-flags)
- [Advanced Build Features](#advanced-build-features)
- [Build Performance Optimization](#build-performance-optimization)
- [Practical Example 1: Custom Module with Dependencies](#practical-example-1-custom-module-with-dependencies)
- [Practical Example 2: Build Variant Configuration](#practical-example-2-build-variant-configuration)
- [Build System Debugging](#build-system-debugging)
- [Key Takeaways](#key-takeaways)
- [Quick Reference](#quick-reference)
- [Conclusion](#conclusion)

## Introduction

The Android build system is one of the most complex aspects of AOSP development. Understanding it deeply enables you to:
- Create custom modules and libraries
- Configure product builds
- Optimize build performance
- Debug build failures
- Integrate third-party components
- Customize build flavors and variants

This chapter explores Android's build system architecture, module types, advanced configurations, and practical examples of mastering the build system.

## Build System Evolution

### Historical Overview

**Make-based (Android 1.0 - 7.x):**
```makefile
# Android.mk
LOCAL_PATH := $(call my-dir)
include $(CLEAR_VARS)
LOCAL_MODULE := myapp
LOCAL_SRC_FILES := $(call all-java-files-under, src)
include $(BUILD_PACKAGE)
```

**Soong/Blueprint (Android 8.0+):**
```go
// Android.bp
android_app {
    name: "myapp",
    srcs: ["src/**/*.java"],
    platform_apis: true,
}
```

**Why the change?**
- Make is slow for large projects
- Hard to parallelize effectively
- Difficult to analyze dependencies
- Blueprint/Soong provides better performance

### Current Architecture

```
┌────────────────────────────────────────┐
│         Blueprint (meta-build)          │
│  Parses Android.bp files               │
│  Generates Ninja build files           │
└──────────────┬─────────────────────────┘
               │
┌──────────────▼─────────────────────────┐
│         Soong (build logic)            │
│  Android-specific build rules          │
│  Module types and processing           │
└──────────────┬─────────────────────────┘
               │
┌──────────────▼─────────────────────────┐
│         Ninja (build execution)        │
│  Fast parallel build execution         │
│  Incremental builds                    │
└────────────────────────────────────────┘
```

## Android.bp Syntax

### Basic Structure

```go
// Module definition
module_type {
    name: "module_name",
    property: "value",
    list_property: ["item1", "item2"],
    nested: {
        sub_property: "value",
    },
}
```

### Common Module Types

**Java/Android:**
```go
// Java library
java_library {
    name: "mylib",
    srcs: ["src/**/*.java"],
    sdk_version: "current",
}

// Android library (with resources)
android_library {
    name: "myandroidlib",
    srcs: ["src/**/*.java"],
    resource_dirs: ["res"],
    manifest: "AndroidManifest.xml",
}

// Android app
android_app {
    name: "MyApp",
    srcs: ["src/**/*.java"],
    resource_dirs: ["res"],
    manifest: "AndroidManifest.xml",
    platform_apis: true,
    certificate: "platform",
    privileged: true,
}
```

**Native (C/C++):**
```go
// Shared library
cc_library_shared {
    name: "libmylib",
    srcs: ["mylib.cpp"],
    shared_libs: ["liblog", "libutils"],
    export_include_dirs: ["include"],
}

// Static library
cc_library_static {
    name: "libmystatic",
    srcs: ["static.cpp"],
}

// Binary/executable
cc_binary {
    name: "mydaemon",
    srcs: ["daemon.cpp"],
    shared_libs: ["liblog", "libmylib"],
    init_rc: ["mydaemon.rc"],
}
```

**Prebuilts:**
```go
// Prebuilt APK
android_app_import {
    name: "PrebuiltApp",
    apk: "app.apk",
    privileged: true,
    presigned: true,
}

// Prebuilt library
cc_prebuilt_library_shared {
    name: "libprebuilt",
    srcs: ["libprebuilt.so"],
    strip: {
        none: true,
    },
}
```

### Properties

**Common properties:**
```go
module {
    name: "mymodule",              // Module name (required)
    
    // Source files
    srcs: ["src/**/*.java"],       // Source files (glob patterns)
    exclude_srcs: ["src/bad.java"], // Exclude specific files
    
    // Dependencies
    static_libs: ["lib1"],         // Static dependencies
    shared_libs: ["lib2"],         // Shared dependencies
    libs: ["framework"],           // SDK/provided libraries
    
    // SDK version
    sdk_version: "current",        // Use current SDK
    min_sdk_version: "21",         // Minimum API level
    target_sdk_version: "33",      // Target API level
    
    // Build variants
    compile_multilib: "both",      // Build for 32 and 64-bit
    vendor: true,                  // Install to vendor partition
    proprietary: true,             // Proprietary module
    
    // Other
    enabled: true,                 // Enable/disable module
    required: ["dep1", "dep2"],    // Runtime dependencies
}
```

### Defaults

Share common properties:

```go
// Define defaults
cc_defaults {
    name: "myapp_defaults",
    cflags: ["-Wall", "-Werror"],
    shared_libs: ["liblog", "libutils"],
}

// Use defaults
cc_binary {
    name: "app1",
    defaults: ["myapp_defaults"],
    srcs: ["app1.cpp"],
}

cc_binary {
    name: "app2",
    defaults: ["myapp_defaults"],
    srcs: ["app2.cpp"],
}
```

### Filegroups

Group files for reuse:

```go
filegroup {
    name: "common_sources",
    srcs: ["common/**/*.cpp"],
}

cc_library {
    name: "lib1",
    srcs: [":common_sources", "lib1.cpp"],
}

cc_library {
    name: "lib2",
    srcs: [":common_sources", "lib2.cpp"],
}
```

## Build Configuration

### Product Configuration

**Product makefiles define what gets built:**

**File:** `device/manufacturer/product/device.mk`

```makefile
# Inherit from common configuration
$(call inherit-product, $(SRC_TARGET_DIR)/product/core_64_bit.mk)
$(call inherit-product, $(SRC_TARGET_DIR)/product/full_base_telephony.mk)

# Device specifics
PRODUCT_NAME := my_product
PRODUCT_DEVICE := my_device
PRODUCT_BRAND := MyBrand
PRODUCT_MODEL := My Device
PRODUCT_MANUFACTURER := MyCompany

# Packages to include
PRODUCT_PACKAGES += \
    Camera2 \
    Gallery2 \
    Launcher3 \
    Settings \
    SystemUI

# Properties
PRODUCT_PROPERTY_OVERRIDES += \
    ro.build.fingerprint=MyBrand/my_product/my_device:13/ABC123/1234567:user/release-keys \
    ro.product.locale=en-US

# Copy files
PRODUCT_COPY_FILES += \
    device/manufacturer/product/configs/audio_policy.conf:$(TARGET_COPY_OUT_VENDOR)/etc/audio_policy.conf

# Overlays
PRODUCT_PACKAGE_OVERLAYS := device/manufacturer/product/overlay

# Inherit vendor configuration
$(call inherit-product, vendor/manufacturer/product/vendor.mk)
```

### Board Configuration

**Hardware-specific settings:**

**File:** `device/manufacturer/product/BoardConfig.mk`

```makefile
# Architecture
TARGET_ARCH := arm64
TARGET_ARCH_VARIANT := armv8-a
TARGET_CPU_ABI := arm64-v8a
TARGET_CPU_VARIANT := generic

# Secondary architecture (32-bit)
TARGET_2ND_ARCH := arm
TARGET_2ND_ARCH_VARIANT := armv8-a
TARGET_2ND_CPU_ABI := armeabi-v7a
TARGET_2ND_CPU_VARIANT := generic

# Kernel
TARGET_KERNEL_CONFIG := mydevice_defconfig
BOARD_KERNEL_CMDLINE := console=ttyMSM0,115200n8
BOARD_KERNEL_BASE := 0x80000000
BOARD_KERNEL_PAGESIZE := 4096

# Partitions
BOARD_BOOTIMAGE_PARTITION_SIZE := 67108864
BOARD_SYSTEMIMAGE_PARTITION_SIZE := 3221225472
BOARD_USERDATAIMAGE_PARTITION_SIZE := 10737418240
BOARD_FLASH_BLOCK_SIZE := 131072

# Filesystem
TARGET_USERIMAGES_USE_EXT4 := true
BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE := ext4

# SELinux
BOARD_SEPOLICY_DIRS += device/manufacturer/product/sepolicy

# Display
TARGET_SCREEN_DENSITY := 480

# Audio
BOARD_USES_ALSA_AUDIO := true

# Bluetooth
BOARD_HAVE_BLUETOOTH := true

# WiFi
BOARD_WLAN_DEVICE := qcwcn
BOARD_HOSTAPD_DRIVER := NL80211
```

### Build Variants

Three main variants:

```makefile
# Engineering build (eng)
- Root access enabled
- Debugging enabled
- ADB enabled by default
- For active development

# User-debug build (userdebug)
- Root access available via adb root
- Some debugging enabled
- Optimizations enabled
- For testing/QA

# User build (user)
- No root access
- Minimal debugging
- Full optimizations
- For production/release
```

**Select variant:**
```bash
lunch my_product-userdebug
```

## Module Dependencies

### Static vs Shared Libraries

**Static linking:**
```go
cc_library_static {
    name: "libstatic",
    srcs: ["static.cpp"],
}

cc_binary {
    name: "app",
    srcs: ["app.cpp"],
    static_libs: ["libstatic"],  // Compiled into binary
}
```

**Shared linking:**
```go
cc_library_shared {
    name: "libshared",
    srcs: ["shared.cpp"],
}

cc_binary {
    name: "app",
    srcs: ["app.cpp"],
    shared_libs: ["libshared"],  // Separate .so file
}
```

### Whole Static Libraries

Include entire static library (don't strip unused symbols):

```go
cc_binary {
    name: "app",
    whole_static_libs: ["libplugins"],  // Include everything
}
```

### Export Dependencies

Make dependencies available to dependents:

```go
cc_library {
    name: "liba",
    export_include_dirs: ["include"],
    export_shared_lib_headers: ["libb"],
}

cc_binary {
    name: "app",
    shared_libs: ["liba"],
    // Automatically gets liba's includes and libb headers
}
```

## Compiler Flags

### C/C++ Flags

```go
cc_library {
    name: "mylib",
    
    // All C/C++ files
    cflags: [
        "-Wall",
        "-Werror",
        "-O2",
    ],
    
    // C++ only
    cppflags: [
        "-std=c++17",
        "-fno-rtti",
    ],
    
    // C only
    conlyflags: [
        "-std=c11",
    ],
    
    // Linker flags
    ldflags: [
        "-Wl,--no-undefined",
    ],
}
```

### Conditional Compilation

```go
cc_library {
    name: "mylib",
    srcs: ["base.cpp"],
    
    target: {
        android: {
            srcs: ["android_specific.cpp"],
            cflags: ["-DANDROID"],
        },
        host: {
            srcs: ["host_specific.cpp"],
        },
    },
    
    arch: {
        arm: {
            srcs: ["arm_specific.cpp"],
        },
        arm64: {
            srcs: ["arm64_specific.cpp"],
        },
        x86: {
            srcs: ["x86_specific.cpp"],
        },
    },
}
```

## Advanced Build Features

### Generated Sources

```go
genrule {
    name: "generate_version",
    srcs: ["version.txt"],
    out: ["version.h"],
    cmd: "$(location gen_version.sh) $(in) > $(out)",
    tool_files: ["gen_version.sh"],
}

cc_library {
    name: "mylib",
    srcs: ["mylib.cpp"],
    generated_headers: ["generate_version"],
}
```

### AIDL Integration

```go
java_library {
    name: "myservice-interface",
    srcs: [
        "src/**/*.java",
        "src/**/*.aidl",  // AIDL files automatically compiled
    ],
    aidl: {
        local_include_dirs: ["src"],
        export_include_dirs: ["src"],
    },
}
```

### Protocol Buffers

```go
java_library {
    name: "myproto",
    proto: {
        type: "lite",  // or "full"
    },
    srcs: ["**/*.proto"],
}
```

### Resource Overlays

Override framework resources:

```go
android_app {
    name: "SystemUI",
    resource_dirs: ["res"],
    
    // Apply overlays from product
    product_specific: true,
}
```

**File:** `device/manufacturer/product/overlay/frameworks/base/core/res/res/values/config.xml`

```xml
<resources>
    <!-- Override status bar height -->
    <dimen name="status_bar_height">28dp</dimen>
</resources>
```

## Build Performance Optimization

### Parallel Builds

```bash
# Use all CPU cores
m -j$(nproc)

# Use specific number
m -j16

# Leave cores free for system
m -j$(($(nproc) - 2))
```

### ccache

```bash
# Enable ccache
export USE_CCACHE=1
export CCACHE_DIR=/path/to/ccache

# Set size
ccache -M 50G

# Check stats
ccache -s
```

### Incremental Builds

```bash
# Build specific module
m SystemUI

# Build and install
m SystemUI
adb sync

# Clean specific module
m clean-SystemUI
```

### Build Server

Distributed builds with distcc:

```bash
# Setup distcc
export DISTCC_HOSTS="localhost/4 server1/8 server2/8"
export CC="distcc gcc"
export CXX="distcc g++"
```

## Practical Example 1: Custom Module with Dependencies

Let's create a complete module with multiple components.

### Step 1: Project Structure

```
device/mycompany/myproject/
├── Android.bp
├── app/
│   ├── Android.bp
│   ├── AndroidManifest.xml
│   ├── src/
│   └── res/
├── lib/
│   ├── Android.bp
│   └── mylib.cpp
├── service/
│   ├── Android.bp
│   ├── service.cpp
│   └── IMyService.aidl
└── config/
    └── my_config.xml
```

### Step 2: Native Library

**File:** `device/mycompany/myproject/lib/Android.bp`

```go
cc_library_shared {
    name: "libmyproject",
    vendor: true,
    
    srcs: [
        "mylib.cpp",
        "utils.cpp",
    ],
    
    shared_libs: [
        "liblog",
        "libutils",
        "libcutils",
    ],
    
    export_include_dirs: ["include"],
    
    cflags: [
        "-Wall",
        "-Werror",
        "-DLOG_TAG=\"MyProject\"",
    ],
    
    cppflags: [
        "-std=c++17",
    ],
}
```

### Step 3: Service with AIDL

**File:** `device/mycompany/myproject/service/Android.bp`

```go
// AIDL interface
filegroup {
    name: "myservice_aidl",
    srcs: ["IMyService.aidl"],
}

// Service binary
cc_binary {
    name: "myservice",
    vendor: true,
    
    srcs: [
        "service.cpp",
        ":myservice_aidl",
    ],
    
    shared_libs: [
        "liblog",
        "libutils",
        "libbinder",
        "libmyproject",
    ],
    
    init_rc: ["myservice.rc"],
    
    cflags: ["-Wall", "-Werror"],
}
```

### Step 4: Android App

**File:** `device/mycompany/myproject/app/Android.bp`

```go
android_app {
    name: "MyProjectApp",
    
    srcs: [
        "src/**/*.java",
        ":myservice_aidl",  // Include AIDL interface
    ],
    
    resource_dirs: ["res"],
    manifest: "AndroidManifest.xml",
    
    platform_apis: true,
    certificate: "platform",
    privileged: true,
    
    static_libs: [
        "androidx.appcompat_appcompat",
    ],
    
    jni_libs: ["libmyproject"],
    
    optimize: {
        enabled: true,
        shrink: true,
        proguard_flags_files: ["proguard.flags"],
    },
}
```

### Step 5: Package Everything

**File:** `device/mycompany/myproject/Android.bp`

```go
// Convenience target to build everything
phony {
    name: "myproject",
    required: [
        "libmyproject",
        "myservice",
        "MyProjectApp",
    ],
}
```

### Step 6: Include in Product

**File:** `device/mycompany/mydevice/device.mk`

```makefile
# Include MyProject
PRODUCT_PACKAGES += myproject

# Copy configuration
PRODUCT_COPY_FILES += \
    device/mycompany/myproject/config/my_config.xml:$(TARGET_COPY_OUT_VENDOR)/etc/my_config.xml
```

### Step 7: Build

```bash
# Build everything
m myproject

# Or build individually
m libmyproject
m myservice
m MyProjectApp
```

## Practical Example 2: Build Variant Configuration

Create different build configurations.

### Step 1: Define Variants

**File:** `device/mycompany/mydevice/BoardConfig.mk`

```makefile
# Define custom variants
CUSTOM_BUILD_VARIANTS := debug release performance

# Variant-specific flags
ifeq ($(TARGET_BUILD_VARIANT),debug)
    TARGET_CFLAGS += -DDEBUG_BUILD
    TARGET_CPPFLAGS += -DDEBUG_BUILD
endif

ifeq ($(TARGET_BUILD_VARIANT),performance)
    TARGET_CFLAGS += -O3 -DPERFORMANCE_BUILD
    TARGET_CPPFLAGS += -O3 -DPERFORMANCE_BUILD
endif
```

### Step 2: Conditional Module Inclusion

**File:** `device/mycompany/mydevice/device.mk`

```makefile
# Base packages
PRODUCT_PACKAGES += \
    Camera2 \
    Settings

# Debug-only packages
ifeq ($(TARGET_BUILD_VARIANT),debug)
PRODUCT_PACKAGES += \
    DebugTools \
    Perfetto \
    strace
endif

# Performance-only packages
ifeq ($(TARGET_BUILD_VARIANT),performance)
PRODUCT_PACKAGES += \
    PerformanceMonitor
endif
```

### Step 3: Variant-Specific Properties

```makefile
# Base properties
PRODUCT_PROPERTY_OVERRIDES += \
    ro.product.name=$(PRODUCT_NAME)

# Debug properties
ifeq ($(TARGET_BUILD_VARIANT),debug)
PRODUCT_PROPERTY_OVERRIDES += \
    ro.debuggable=1 \
    persist.sys.usb.config=adb \
    debug.performance.tuning=1
endif

# Performance properties
ifeq ($(TARGET_BUILD_VARIANT),performance)
PRODUCT_PROPERTY_OVERRIDES += \
    dalvik.vm.dex2oat-flags=--compiler-filter=speed \
    debug.sf.hw=1
endif
```

### Step 4: Building Variants

```bash
# Debug variant
lunch mydevice-debug
m -j$(nproc)

# Release variant
lunch mydevice-release
m -j$(nproc)

# Performance variant
lunch mydevice-performance
m -j$(nproc)
```

## Build System Debugging

### Verbose Output

```bash
# Show commands being executed
m showcommands

# Verbose build
m -v

# Dry run (show what would be built)
m -n
```

### Dependency Analysis

```bash
# Show module dependencies
m deps-packages-MyApp

# Show why module is being built
m why-module-MyApp

# Show all modules
m modules
```

### Build Graph

```bash
# Generate build graph
m --build-graph=build_graph.dot

# Convert to image
dot -Tpng build_graph.dot -o build_graph.png
```

### Common Build Errors

**Missing dependency:**
```
error: MyApp (java:sdk) missing dependencies: SomeLib
```
**Solution:** Add to `static_libs` or `libs`

**Multiple definitions:**
```
error: multiple definitions of module MyModule
```
**Solution:** Check for duplicate module definitions

**Ninja error:**
```
ninja: error: unknown target 'MyModule'
```
**Solution:** Check module name and ensure it's in build graph

## Key Takeaways

1. **Soong/Blueprint replaced Make**: Faster, more maintainable
2. **Android.bp is declarative**: Simpler than Make syntax
3. **Module types handle specifics**: cc_library, android_app, etc.
4. **Dependencies are explicit**: Static, shared, export
5. **Product configuration is flexible**: Multiple devices/variants
6. **Optimization is important**: ccache, parallel builds
7. **Debugging tools exist**: Verbose, deps, graph analysis

## Quick Reference

### Build Commands

```bash
# Setup environment
source build/envsetup.sh
lunch <product>-<variant>

# Full build
m -j$(nproc)

# Module build
m <module_name>

# Clean
m clean
m clean-<module>

# Install
m <module>
adb sync

# Debugging
m showcommands
m deps-packages-<module>
```

### Common Module Types

```go
// Native
cc_binary            // Executable
cc_library           // Library (both static and shared)
cc_library_shared    // Shared library (.so)
cc_library_static    // Static library (.a)

// Java
java_library         // Java library (JAR)
android_library      // Android library (with resources)
android_app          // Android application (APK)

// Other
filegroup           // Group of files
genrule             // Generate files
sh_binary           // Shell script
prebuilt_etc        // Configuration files
```

### Module Properties

```go
name                // Module name (required)
srcs                // Source files
static_libs         // Static dependencies
shared_libs         // Shared dependencies
sdk_version         // SDK version
platform_apis       // Use platform APIs
vendor             // Install to vendor
required           // Runtime dependencies
```

## Conclusion

You now have 12 comprehensive chapters covering AOSP development from foundational concepts through advanced build system mastery. This guide provides the knowledge needed to build, customize, and optimize Android systems at a professional level.
