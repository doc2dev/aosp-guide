# Chapter 1: AOSP Project Structure

## Introduction

Understanding the Android Open Source Project (AOSP) structure is fundamental to becoming an effective AOSP developer. Unlike Android app development where you work within a predefined framework, AOSP development requires intimate knowledge of how the entire Android system is organized, built, and deployed.

This chapter provides a comprehensive overview of the AOSP source tree, its organization principles, and the build system that ties everything together. By the end of this chapter, you'll be able to navigate the AOSP codebase confidently and understand where to look for specific functionality.

## The AOSP Source Tree Overview

When you sync the AOSP repository using `repo sync`, you'll end up with a directory structure containing hundreds of thousands of files organized across dozens of top-level directories. The structure follows a logical organization based on functionality, layer, and ownership.

### Top-Level Directory Structure

```
aosp/
├── art/                    # Android Runtime
├── bionic/                 # Android's C library
├── bootable/               # Boot and recovery related code
├── build/                  # Build system files
├── cts/                    # Compatibility Test Suite
├── dalvik/                 # Dalvik VM (legacy, mostly deprecated)
├── development/            # Development tools and samples
├── device/                 # Device-specific configurations
├── external/               # External open source projects
├── frameworks/             # Android framework
├── hardware/               # Hardware Abstraction Layer
├── kernel/                 # Linux kernel sources
├── libcore/                # Core libraries (java.*, javax.*)
├── libnativehelper/        # JNI helper functions
├── packages/               # Standard Android packages/apps
├── pdk/                    # Platform Development Kit
├── platform_testing/       # Platform-level tests
├── prebuilts/              # Prebuilt binaries (toolchains, SDKs)
├── sdk/                    # Android SDK
├── system/                 # Low-level system components
├── test/                   # Testing infrastructure
├── toolchain/              # Compiler toolchains
├── tools/                  # Development tools
└── vendor/                 # Vendor-specific code (if any)
```

Let's explore the most critical directories in detail.

## Critical Directories Deep Dive

### frameworks/

The `frameworks/` directory is the heart of Android. It contains the Java and native code that implements the Android framework APIs and system services.

```
frameworks/
├── base/                   # Core framework
│   ├── api/               # API definitions
│   ├── core/              # Core libraries and services
│   │   ├── java/          # Java framework code
│   │   ├── jni/           # JNI implementations
│   │   └── res/           # Framework resources
│   ├── services/          # System services
│   │   ├── core/          # Core system services
│   │   ├── accessibility/ # Accessibility service
│   │   ├── appwidget/     # AppWidget service
│   │   └── ...
│   ├── packages/          # System packages
│   │   ├── SystemUI/      # System UI implementation
│   │   ├── SettingsProvider/
│   │   └── ...
│   ├── cmds/              # Command-line tools
│   └── libs/              # Native libraries
├── av/                    # Audio/Video frameworks
├── native/                # Native framework services
│   ├── services/          # Native services
│   │   ├── surfaceflinger/
│   │   ├── inputflinger/
│   │   └── ...
│   └── libs/              # Native framework libraries
├── opt/                   # Optional components
└── compile/               # Compilation-related tools
```

#### Key Framework Components

**frameworks/base/core/java/**: This is where most of the Android framework Java code lives. Key packages include:

- `android.app.*` - Application framework (Activity, Service, etc.)
- `android.content.*` - Content providers, intents, loaders
- `android.view.*` - View system, window management
- `android.widget.*` - UI widgets
- `android.os.*` - OS services interfaces
- `com.android.internal.*` - Internal framework APIs (not part of SDK)

**frameworks/base/services/**: System services implementations:

- `core/java/com/android/server/` - Core system services
  - `ActivityManagerService.java` - Activity/process management
  - `PackageManagerService.java` - Package management
  - `WindowManagerService.java` - Window management
  - `PowerManagerService.java` - Power management
  - And many more...

**frameworks/native/**: Native-level framework code:

- `services/surfaceflinger/` - Display composition
- `services/inputflinger/` - Input event handling
- `libs/binder/` - Binder IPC implementation
- `libs/gui/` - Graphics UI libraries

### system/

The `system/` directory contains low-level system components that run before or alongside the Android framework.

```
system/
├── core/                   # Core system utilities
│   ├── init/              # Init process
│   ├── rootdir/           # Root filesystem files
│   ├── debuggerd/         # Debug daemon (crash dumps)
│   ├── adb/               # Android Debug Bridge
│   ├── fastboot/          # Fastboot protocol
│   ├── liblog/            # Logging library
│   ├── logd/              # Log daemon
│   └── ...
├── bt/                    # Bluetooth stack
├── extras/                # Extra system utilities
├── media/                 # Media frameworks
├── netd/                  # Network daemon
├── sepolicy/              # SELinux policy
├── update_engine/         # OTA update engine
└── vold/                  # Volume daemon (storage management)
```

#### Key System Components

**system/core/init/**: The first user-space process that starts during boot. It:
- Parses `.rc` files to start services
- Sets up initial filesystem
- Enforces SELinux policies
- Manages system services lifecycle

**system/core/adb/**: Android Debug Bridge implementation, essential for:
- Installing and debugging apps
- Accessing shell
- Port forwarding
- File transfers

**system/sepolicy/**: SELinux policy definitions that enforce security:
- Process isolation
- File access controls
- Service access permissions
- Type enforcement rules

### hardware/

The Hardware Abstraction Layer (HAL) interface definitions and some implementations.

```
hardware/
├── interfaces/            # HIDL/AIDL interface definitions
│   ├── audio/            # Audio HAL interfaces
│   ├── camera/           # Camera HAL interfaces
│   ├── sensors/          # Sensor HAL interfaces
│   ├── graphics/         # Graphics HAL interfaces
│   └── ...
├── libhardware/          # Legacy HAL headers
└── ...
```

Modern Android uses HIDL (Hardware Interface Definition Language) or AIDL (Android Interface Definition Language) to define HAL interfaces. These provide versioned, stable interfaces between the Android framework and vendor-specific hardware implementations.

### packages/

Standard Android applications and providers.

```
packages/
├── apps/                  # Standard Android apps
│   ├── Settings/         # Settings app
│   ├── Launcher3/        # Default launcher
│   ├── Calculator/       # Calculator app
│   ├── Calendar/         # Calendar app
│   ├── Camera2/          # Camera app
│   ├── Contacts/         # Contacts app
│   ├── Dialer/           # Phone dialer
│   └── ...
├── services/              # Package services
│   ├── Telephony/        # Telephony service
│   └── ...
├── providers/             # Content providers
│   ├── ContactsProvider/
│   ├── CalendarProvider/
│   ├── MediaProvider/
│   └── ...
└── inputmethods/          # Input method editors
```

### device/

Device-specific configurations and makefiles. This is where OEMs and device maintainers add their device-specific code.

```
device/
├── generic/              # Generic device configs
└── [manufacturer]/       # Manufacturer-specific
    └── [device]/         # Device-specific files
        ├── device.mk     # Device makefile
        ├── BoardConfig.mk # Board configuration
        ├── overlay/      # Resource overlays
        └── ...
```

### external/

Third-party open-source projects used by Android.

```
external/
├── chromium-webview/     # WebView implementation
├── sqlite/               # SQLite database
├── openssl/              # OpenSSL crypto library
├── zlib/                 # Compression library
├── libpng/               # PNG library
├── freetype/             # Font rendering
└── ... (hundreds more)
```

### build/

The Android build system files.

```
build/
├── make/                 # Make-based build system (legacy)
├── soong/                # Soong build system (current)
│   ├── android/          # Android-specific build logic
│   ├── cc/               # C/C++ build rules
│   ├── java/             # Java build rules
│   └── ...
├── blueprint/            # Blueprint meta-build system
└── tools/                # Build-related tools
```

## Understanding the Build System

Android's build system has evolved significantly over the years. Understanding this evolution helps you work with both legacy and modern code.

### Build System Evolution

1. **Make-based system** (Android 1.0 - 7.x): Used `Android.mk` files with GNU Make
2. **Soong/Blueprint** (Android 8.0+): Uses `Android.bp` files with a custom build system
3. **Bazel** (future): Google is working on migrating to Bazel for even faster builds

Most modern AOSP code uses `Android.bp`, but you'll still encounter `Android.mk` in older modules and some third-party code.

### Android.bp Format

`Android.bp` files use a declarative, JSON-like syntax. Here's a typical example:

```go
// Builds a Java library
java_library {
    name: "my-framework-lib",
    srcs: ["src/**/*.java"],
    sdk_version: "system_current",
    libs: ["framework"],
    static_libs: ["some-static-lib"],
}

// Builds a native shared library
cc_library_shared {
    name: "libmyhal",
    srcs: ["MyHal.cpp"],
    shared_libs: [
        "liblog",
        "libutils",
        "libhidlbase",
    ],
    vendor: true,
}

// Builds an Android app
android_app {
    name: "MySystemApp",
    srcs: ["src/**/*.java"],
    resource_dirs: ["res"],
    platform_apis: true,
    certificate: "platform",
    privileged: true,
}
```

### Android.mk Format (Legacy)

While being phased out, you'll still encounter `Android.mk` files:

```makefile
LOCAL_PATH := $(call my-dir)

include $(CLEAR_VARS)
LOCAL_MODULE := my-framework-lib
LOCAL_SRC_FILES := $(call all-java-files-under, src)
LOCAL_SDK_VERSION := system_current
LOCAL_JAVA_LIBRARIES := framework
LOCAL_STATIC_JAVA_LIBRARIES := some-static-lib
include $(BUILD_JAVA_LIBRARY)
```

### Module Types

Understanding module types is crucial for navigating the build system:

#### Java Modules
- `java_library`: Regular Java library
- `java_library_static`: Static Java library (embedded in dependents)
- `android_app`: Android application (APK)
- `android_library`: Android library with resources
- `java_system_modules`: System modules for the Java module system

#### Native Modules
- `cc_library`: Native library (builds both static and shared)
- `cc_library_shared`: Shared library (.so)
- `cc_library_static`: Static library (.a)
- `cc_binary`: Native executable
- `cc_defaults`: Defaults shared across modules

#### Other Modules
- `prebuilt_etc`: Prebuilt files installed to /system/etc or similar
- `sh_binary`: Shell scripts
- `filegroup`: Group of files (not built, used as input to other modules)

### Build Configuration Files

Several files control how the build system operates:

**device.mk**: Device-specific build configuration
```makefile
PRODUCT_PACKAGES += \
    MyCustomApp \
    MySystemService

PRODUCT_PROPERTY_OVERRIDES += \
    ro.my.custom.property=value

PRODUCT_COPY_FILES += \
    device/manufacturer/device/config.xml:system/etc/config.xml
```

**BoardConfig.mk**: Board-specific hardware configuration
```makefile
TARGET_ARCH := arm64
TARGET_CPU_ABI := arm64-v8a
BOARD_KERNEL_CMDLINE := console=ttyMSM0,115200,n8
BOARD_BOOTIMAGE_PARTITION_SIZE := 67108864
```

## The Repo Tool and Manifest

AOSP uses the `repo` tool to manage its multiple git repositories. This is necessary because AOSP is not a single monolithic repository but rather hundreds of separate git repositories.

### Understanding repo

When you run `repo init` and `repo sync`, you're:
1. Downloading a manifest XML file that lists all repositories
2. Cloning/syncing all those repositories to specific directories
3. Checking out specific branches/tags for each repository

### The Manifest File

The manifest file (`.repo/manifests/default.xml`) defines the repository structure:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <remote name="aosp"
          fetch="https://android.googlesource.com/" />
  
  <default revision="refs/tags/android-14.0.0_r1"
           remote="aosp"
           sync-j="4" />
  
  <project path="build/make" name="platform/build" />
  <project path="build/soong" name="platform/build/soong" />
  <project path="frameworks/base" name="platform/frameworks/base" />
  <project path="system/core" name="platform/system/core" />
  <!-- hundreds more projects -->
</manifest>
```

### Common repo Commands

```bash
# Initialize repo for a specific branch
repo init -u https://android.googlesource.com/platform/manifest -b android-14.0.0_r1

# Sync all repositories
repo sync -j$(nproc)

# Start a new branch across all repositories
repo start my-feature --all

# Upload changes for review (if using Gerrit)
repo upload

# Check status across all repositories
repo status

# Show differences across all repositories
repo diff

# Abandon a branch
repo abandon my-feature
```

## Navigating the AOSP Codebase

### Finding Code

With millions of lines of code, finding what you need is a critical skill:

#### Using grep
```bash
# Find all uses of a specific API
grep -r "PackageManager" frameworks/base/services

# Find method definitions
grep -rn "public.*installPackage" frameworks/base

# Find XML resource definitions
find packages/apps -name "*.xml" -exec grep -l "my_layout" {} \;
```

#### Using Android Code Search

Google provides an online code search tool: https://cs.android.com

This is incredibly useful for:
- Cross-referencing
- Finding usage examples
- Understanding API evolution across versions
- Searching with regex

#### Using cscope or ctags

For efficient local code navigation:

```bash
# Generate cscope database
cd aosp/
find . -name "*.java" -o -name "*.cpp" -o -name "*.h" > cscope.files
cscope -b -q -k

# Or use ctags
ctags -R frameworks/base/
```

### Understanding Code Organization Patterns

AOSP follows several organizational patterns:

1. **Interface/Implementation Split**: Public APIs are defined in `frameworks/base/core/java/android/*` while implementations are often in `com.android.internal.*` or `frameworks/base/services/`

2. **JNI Boundaries**: Java code in `frameworks/base/core/java/` often has corresponding native implementation in `frameworks/base/core/jni/`

3. **AIDL Definitions**: Inter-process interfaces are defined in `.aidl` files, typically alongside the implementing service

4. **Resource Overlays**: Device-specific resources override framework resources through the overlay mechanism

## Build Artifacts and Output

Understanding where build outputs go is essential for testing and debugging.

### Output Directory Structure

After building, the `out/` directory contains all build artifacts:

```
out/
├── target/
│   ├── common/           # Target-independent outputs
│   └── product/
│       └── [device]/     # Device-specific outputs
│           ├── system/   # System partition contents
│           ├── vendor/   # Vendor partition contents
│           ├── obj/      # Object files (.o)
│           ├── symbols/  # Debug symbols
│           └── [device]-img-*.zip # Flashable images
└── host/                 # Host tools and binaries
```

### Important Output Locations

- **System apps**: `out/target/product/[device]/system/app/`
- **System priv-apps**: `out/target/product/[device]/system/priv-app/`
- **Native libraries**: `out/target/product/[device]/system/lib64/`
- **Framework JARs**: `out/target/product/[device]/system/framework/`
- **Images**: `out/target/product/[device]/*.img` (system.img, vendor.img, boot.img, etc.)

## System Partitions

Modern Android devices use multiple partitions, each serving a specific purpose:

### Key Partitions

**system**: Core Android OS, framework, and basic apps
- Mount point: `/system`
- Read-only at runtime
- Contains framework, system apps, libraries

**vendor**: Vendor/OEM-specific code and HAL implementations
- Mount point: `/vendor`
- Isolates vendor code from system
- Critical for Treble architecture

**product**: Additional OEM apps and customizations
- Mount point: `/product`
- Optional partition for OEM differentiation

**odm**: Original Design Manufacturer customizations
- Mount point: `/odm`
- Device-specific configurations

**boot**: Kernel and ramdisk
- Not mounted as filesystem
- Contains kernel and initial ramdisk

**data**: User data and apps
- Mount point: `/data`
- Read-write, persists across boots
- Contains user apps, data, system data

**recovery**: Recovery system
- Alternative boot environment for system updates and recovery

### Project Treble and Partitions

Project Treble (Android 8.0+) introduced a strict separation between:
- **System partition**: Google-provided Android framework
- **Vendor partition**: SOC vendor and OEM code

This separation allows:
- Framework updates without vendor code changes
- Better testing and certification
- Faster OS updates

The VINTF (Vendor Interface) enforces compatibility between system and vendor partitions.

## Source Code Organization by Layer

Android's architecture can be thought of in layers, and the source code is organized accordingly:

### Application Layer
- **Location**: `packages/apps/`
- **Purpose**: User-facing applications
- **Examples**: Settings, Dialer, Camera

### Application Framework Layer
- **Location**: `frameworks/base/core/java/android/`
- **Purpose**: APIs available to applications
- **Examples**: Activity, ContentProvider, PackageManager

### System Services Layer
- **Location**: `frameworks/base/services/`
- **Purpose**: Background services implementing framework functionality
- **Examples**: ActivityManagerService, WindowManagerService

### Native Framework Layer
- **Location**: `frameworks/native/`
- **Purpose**: Native-level framework services
- **Examples**: SurfaceFlinger, AudioFlinger

### HAL Layer
- **Location**: `hardware/interfaces/`, `hardware/libhardware/`
- **Purpose**: Hardware abstraction
- **Examples**: Camera HAL, Audio HAL, Sensor HAL

### Kernel Layer
- **Location**: `kernel/`
- **Purpose**: Linux kernel with Android-specific patches
- **Examples**: Binder driver, ashmem, lowmemorykiller

## Practical Navigation Exercise

Let's trace a common operation—installing an app—through the codebase:

### 1. Application Layer
User taps "Install" in an app store → handled by an installer app in `packages/apps/`

### 2. Framework API
App calls `PackageManager.installPackage()` → defined in `frameworks/base/core/java/android/content/pm/PackageManager.java`

### 3. PackageManagerService
Call routed to `PackageManagerService` → implemented in `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`

### 4. Native Package Parser
APK parsed using native code → JNI in `frameworks/base/core/jni/`, native code in `frameworks/base/libs/androidfw/`

### 5. File System Operations
APK copied to `/data/app/` → file operations through kernel

### 6. SELinux Checks
Security policies enforced → policies in `system/sepolicy/`

This exercise demonstrates how understanding the source structure helps you trace execution flow through the system.

## Key Takeaways

1. **AOSP is modular**: The source tree is organized into logical components (frameworks, system, packages, etc.)

2. **Multiple build systems coexist**: Modern Android uses Soong (Android.bp) but legacy Android.mk files remain

3. **Repo manages complexity**: The repo tool orchestrates hundreds of git repositories

4. **Partitions enforce separation**: System/vendor/product partitions separate concerns and enable Treble

5. **Code navigation is essential**: Learning to efficiently find and trace code is a core AOSP developer skill

6. **Layers are physical**: The architectural layers map directly to source tree organization

## Next Steps

In Chapter 2, we'll set up your development environment and walk through building AOSP from source, working with the build system, and deploying your changes to a device. Understanding the project structure from this chapter will make the build process much more intuitive.

## Quick Reference

### Essential Directories Cheat Sheet
```
frameworks/base/         → Framework APIs and system services
system/core/            → Core system utilities (init, adb, etc.)
packages/apps/          → Standard Android apps
hardware/interfaces/    → HAL interface definitions
build/soong/            → Build system
device/                 → Device-specific configurations
external/               → Third-party dependencies
```

### Common Code Patterns
```
Service Definition:      frameworks/base/core/java/android/*/I*.aidl
Service Implementation:  frameworks/base/services/core/java/com/android/server/*Service.java
Public API:             frameworks/base/core/java/android/*
Internal API:           frameworks/base/core/java/com/android/internal/*
JNI Implementation:     frameworks/base/core/jni/android_*
Native Service:         frameworks/native/services/*
```
