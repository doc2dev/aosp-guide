# Chapter 11: Security & SELinux

## Introduction

Security is fundamental to Android's architecture. SELinux (Security-Enhanced Linux) provides mandatory access control, enforcing security policies at the kernel level. Understanding Android's security model and SELinux is essential for:

- Implementing secure system services
- Debugging permission denials
- Writing SELinux policies for custom components
- Understanding Android's security architecture
- Ensuring production-ready AOSP builds

This chapter explores Android's security model, SELinux architecture, policy writing, and provides practical examples of implementing secure components.

## Android Security Model

### Security Layers

Android security operates at multiple layers:

```
┌────────────────────────────────────────────────┐
│         Application Security                    │
│  - App sandbox (UID isolation)                 │
│  - Permissions                                  │
│  - Code signing                                 │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│         Framework Security                      │
│  - Permission checking                         │
│  - Binder UID/PID verification                 │
│  - Capability checks                           │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│         SELinux (Kernel Level)                 │
│  - Mandatory Access Control (MAC)              │
│  - Type enforcement                            │
│  - Process isolation                           │
└────────────────┬───────────────────────────────┘
                 │
┌────────────────▼───────────────────────────────┐
│         Kernel Security                        │
│  - Process isolation                           │
│  - Memory protection                           │
│  - Secure boot                                 │
└─────────────────────────────────────────────────┘
```

### Application Sandbox

Each app runs in its own sandbox:

```
UID 10000: com.example.app1
  - Process: app1
  - Data: /data/data/com.example.app1
  - Cannot access other app data

UID 10001: com.example.app2
  - Process: app2
  - Data: /data/data/com.example.app2
  - Cannot access app1 data
```

**Key principles:**
- Each app gets unique UID (Linux user ID)
- UID determines file access permissions
- Apps isolated by Linux kernel
- SELinux provides additional enforcement

## SELinux Fundamentals

### What is SELinux?

SELinux provides Mandatory Access Control (MAC):
- **Discretionary Access Control (DAC)**: Traditional Linux permissions (owner decides)
- **Mandatory Access Control (MAC)**: System-wide policy (enforced regardless of owner)

### SELinux Modes

```bash
# Check SELinux mode
adb shell getenforce

# Enforcing: Denials are blocked
# Permissive: Denials are logged but allowed
# Disabled: SELinux not active
```

**Set mode (temporary):**
```bash
adb shell setenforce 0  # Permissive
adb shell setenforce 1  # Enforcing
```

### SELinux Contexts

Everything has a context (label):

**Format:** `user:role:type:level`

Example: `u:r:system_server:s0`
- `u`: User (usually just `u`)
- `r`: Role (usually just `r`)
- `type`: The security type (most important)
- `s0`: MLS/MCS level (usually `s0`)

**View contexts:**

```bash
# Process contexts
adb shell ps -Z

# Output example:
# u:r:init:s0           root      1     0     ...  /init
# u:r:system_server:s0  system    456   1     ...  system_server
# u:r:zygote:s0         root      789   1     ...  zygote

# File contexts
adb shell ls -Z /system/bin/

# Output example:
# u:object_r:system_file:s0  app_process
# u:object_r:shell_exec:s0   sh
# u:object_r:toolbox_exec:s0 toybox
```

### SELinux Policy Structure

```
system/sepolicy/
├── private/              # Private (platform) policy
│   ├── domain.te        # Base domain rules
│   ├── init.te          # Init process policy
│   ├── system_server.te # System server policy
│   ├── app.te           # App domain policy
│   ├── file_contexts    # File labeling
│   ├── service_contexts # Service labeling
│   └── ...
├── public/               # Public API for vendor
│   ├── domain.te
│   ├── attributes       # Policy attributes
│   └── ...
├── vendor/               # Vendor-specific policy
└── prebuilts/            # Precompiled components
```

## SELinux Policy Language

### Allow Rules

Basic syntax: `allow source target:class perms;`

```
# Allow system_server to read system_file
allow system_server system_file:file read;

# Allow init to write to kmsg device
allow init kmsg_device:chr_file write;

# Allow app to connect to network
allow untrusted_app node:tcp_socket node_bind;
```

**Multiple permissions:**
```
allow system_server system_file:file { read open getattr };
```

**Common permission sets (macros):**
```
allow system_server system_file:file r_file_perms;
# Expands to: { read open getattr }
```

### Types and Attributes

**Type declarations:**
```
# Define a new type
type my_service, domain;
type my_service_exec, exec_type, file_type;

# Type with attributes
type my_daemon, domain, mlstrustedsubject;
```

**Attributes group related types:**
```
# All domains (processes)
attribute domain;

# All file types
attribute file_type;

# All executables
attribute exec_type;
```

**Attribute rules apply to all types with that attribute:**
```
# All domains can read system_file
allow domain system_file:file r_file_perms;
```

### Type Transitions

Automatic domain transitions when executing files:

```
# When init executes a file labeled my_service_exec,
# transition to my_service domain
type_transition init my_service_exec:process my_service;

# Shorthand using macros
init_daemon_domain(my_service)
```

### Macros

Common macros simplify policy writing:

```
# Allow domain to execute shell commands
allow_shell_domain(my_service)

# Allow file read/write/execute
rw_file_perms(my_service, my_data_file)

# Allow Binder communication
binder_call(my_service, system_server)

# Allow setting properties
set_prop(my_service, my_prop)
```

### Neverallow Rules

Enforced at compile time - violations cause build errors:

```
# Apps must never have these capabilities
neverallow appdomain self:capability { sys_admin sys_module };

# Apps cannot access sensitive files
neverallow appdomain kernel_debugfs:file read;

# Only init can write to system files
neverallow { domain -init } system_file:file write;
```

## SELinux Policy Files

### domain.te

Base policy for all domains:

```
# system/sepolicy/private/domain.te

# All domains are allowed basic operations
allow domain self:process { sigchld sigkill sigstop };
allow domain self:fd use;
allow domain proc:file r_file_perms;
allow domain sysfs:dir search;

# All domains can read system files
allow domain system_file:dir r_dir_perms;
allow domain system_file:file r_file_perms;
allow domain system_file:lnk_file r_file_perms;

# Logging
allow domain kernel:system syslog_read;
```

### app.te

Policy for application domains:

```
# system/sepolicy/private/app.te

# Defines base app domain
type appdomain;

# Apps can read/write their own data
allow appdomain app_data_file:dir create_dir_perms;
allow appdomain app_data_file:file create_file_perms;

# Apps can access app ops service
allow appdomain app_api_service:service_manager find;

# Apps can use Binder
binder_use(appdomain)
binder_call(appdomain, system_server)

# Prevent apps from accessing sensitive data
neverallow appdomain kernel_debugfs:file read;
neverallow appdomain init:binder call;
```

### file_contexts

Labels files in the filesystem:

```
# system/sepolicy/private/file_contexts

# System files
/system(/.*)?              u:object_r:system_file:s0
/system/bin/init           u:object_r:init_exec:s0
/system/bin/app_process.*  u:object_r:zygote_exec:s0

# Vendor files
/vendor(/.*)?              u:object_r:vendor_file:s0
/vendor/bin/hw(/.*)?       u:object_r:vendor_hal_file:s0

# Data directories
/data/data(/.*)?           u:object_r:app_data_file:s0
/data/system(/.*)?         u:object_r:system_data_file:s0

# Device nodes
/dev/binder                u:object_r:binder_device:s0
/dev/kmsg                  u:object_r:kmsg_device:s0
```

### service_contexts

Labels system services:

```
# system/sepolicy/private/service_contexts

activity                    u:object_r:activity_service:s0
package                     u:object_r:package_service:s0
window                      u:object_r:window_service:s0
power                       u:object_r:power_service:s0
```

### property_contexts

Labels system properties:

```
# system/sepolicy/private/property_contexts

net.rmnet               u:object_r:net_radio_prop:s0
ro.build.               u:object_r:build_prop:s0
persist.sys.locale      u:object_r:system_prop:s0
debug.                  u:object_r:debug_prop:s0
```

## Writing SELinux Policy

### Example: Custom System Service

Let's write policy for a custom service.

### Step 1: Define Service Files

**Service executable:** `/system/bin/my_daemon`
**Service name:** `my_service`
**Data directory:** `/data/system/my_service`

### Step 2: Create Type Definitions

**File:** `device/manufacturer/sepolicy/my_daemon.te`

```
# Define my_daemon domain
type my_daemon, domain;
type my_daemon_exec, exec_type, file_type;

# Define data file type
type my_daemon_data_file, file_type, data_file_type;

# Init launches my_daemon
init_daemon_domain(my_daemon)
```

### Step 3: Define Basic Permissions

```
# my_daemon.te (continued)

# Allow self operations
allow my_daemon self:capability { dac_override setuid setgid };
allow my_daemon self:process { sigchld sigkill sigstop signull };

# Allow reading system files
allow my_daemon system_file:dir r_dir_perms;
allow my_daemon system_file:file r_file_perms;

# Allow logging
allow my_daemon kernel:system syslog_read;
unix_socket_connect(my_daemon, logdr, logd)
```

### Step 4: Data Directory Access

```
# Allow creating and accessing data directory
allow my_daemon my_daemon_data_file:dir create_dir_perms;
allow my_daemon my_daemon_data_file:file create_file_perms;

# Allow system_server to setup data dir
allow system_server my_daemon_data_file:dir create_dir_perms;
```

### Step 5: Binder Communication

```
# Use Binder
binder_use(my_daemon)

# Allow registering as a service
add_service(my_daemon, my_service)

# Allow clients to call us
binder_call(system_server, my_daemon)
binder_call(appdomain, my_daemon)
```

### Step 6: Property Access

```
# Get system properties
get_prop(my_daemon, system_prop)

# Set custom properties
set_prop(my_daemon, my_daemon_prop)
```

### Step 7: Hardware Access (if needed)

```
# Access a device node
allow my_daemon my_device:chr_file rw_file_perms;

# Access sysfs
allow my_daemon sysfs:dir r_dir_perms;
allow my_daemon sysfs_my_device:file rw_file_perms;
```

### Step 8: File Contexts

**File:** `device/manufacturer/sepolicy/file_contexts`

```
# Executable
/system/bin/my_daemon       u:object_r:my_daemon_exec:s0

# Data directory
/data/system/my_service(/.*)?  u:object_r:my_daemon_data_file:s0

# Device node (if custom)
/dev/my_device              u:object_r:my_device:s0
```

### Step 9: Service Contexts

**File:** `device/manufacturer/sepolicy/service_contexts`

```
my_service    u:object_r:my_service:s0
```

### Step 10: Property Contexts

**File:** `device/manufacturer/sepolicy/property_contexts`

```
my.daemon.    u:object_r:my_daemon_prop:s0
```

## Debugging SELinux Denials

### Viewing Denials

```bash
# View all denials
adb logcat -b events | grep avc

# View denials in kernel log
adb shell dmesg | grep avc

# Example denial:
# avc: denied { read } for pid=1234 comm="my_daemon" 
# name="config.xml" dev="dm-0" ino=123456 
# scontext=u:r:my_daemon:s0 
# tcontext=u:object_r:system_file:s0 
# tclass=file permissive=0
```

### Understanding Denials

Denial format:
- `denied { read }` - Operation denied
- `pid=1234` - Process ID
- `comm="my_daemon"` - Process name
- `name="config.xml"` - Target file
- `scontext=u:r:my_daemon:s0` - Source context
- `tcontext=u:object_r:system_file:s0` - Target context
- `tclass=file` - Object class
- `permissive=0` - Enforcing mode

### Generating Policy from Denials

**Using audit2allow:**

```bash
# Collect denials
adb shell dmesg | grep avc > denials.txt

# Generate policy suggestions
audit2allow -i denials.txt

# Output:
#============= my_daemon ==============
allow my_daemon system_file:file read;

# Generate module
audit2allow -i denials.txt -M my_daemon_policy
```

**Warning:** Don't blindly apply audit2allow suggestions. Understand what you're allowing and ensure it's appropriate.

### Common Denial Patterns

**File access denied:**
```
avc: denied { read } for ... tclass=file
→ allow source target:file read;
```

**Directory access:**
```
avc: denied { search } for ... tclass=dir
→ allow source target:dir search;
```

**Property access:**
```
avc: denied { set } for ... tclass=property_service
→ set_prop(source, property_type)
```

**Binder call:**
```
avc: denied { call } for ... tclass=binder
→ binder_call(source, target)
```

## Practical Example 1: Secure Custom Service

Let's implement a complete secure service with proper SELinux policy.

### Step 1: Service Implementation

**File:** `device/mycompany/mydevice/secure_service/SecureService.cpp`

```cpp
#include <binder/IPCThreadState.h>
#include <binder/IServiceManager.h>
#include <private/android_filesystem_config.h>

class SecureService : public BnSecureService {
public:
    status_t doPrivilegedOperation() {
        // Check caller UID
        uid_t uid = IPCThreadState::self()->getCallingUid();
        
        // Only system UID allowed
        if (uid != AID_SYSTEM) {
            ALOGE("Unauthorized access attempt from UID %d", uid);
            return PERMISSION_DENIED;
        }
        
        // Perform privileged operation
        return performOperation();
    }
    
private:
    status_t performOperation() {
        // Implementation
        return OK;
    }
};
```

### Step 2: SELinux Policy

**File:** `device/mycompany/mydevice/sepolicy/secure_service.te`

```
# Define domain
type secure_service, domain;
type secure_service_exec, exec_type, file_type;

# Init launches service
init_daemon_domain(secure_service)

# Basic permissions
allow secure_service self:capability { setuid setgid };

# Binder usage
binder_use(secure_service)
add_service(secure_service, secure_service)

# Only system_server can call us
binder_call(system_server, secure_service)

# Explicitly deny app access
neverallow appdomain secure_service:binder call;

# Read system files
allow secure_service system_file:dir r_dir_perms;
allow secure_service system_file:file r_file_perms;

# Access privileged resource
allow secure_service secure_device:chr_file rw_file_perms;

# Logging
allow secure_service kernel:system syslog_read;
```

### Step 3: Additional Protection

**Require signature permission in framework:**

```java
// Define permission
<permission android:name="android.permission.ACCESS_SECURE_SERVICE"
            android:protectionLevel="signature" />

// Check in service
public void doOperation() {
    mContext.enforceCallingPermission(
        android.Manifest.permission.ACCESS_SECURE_SERVICE,
        "Requires ACCESS_SECURE_SERVICE permission");
    
    // Proceed with operation
}
```

## Practical Example 2: HAL Service Policy

SELinux policy for a HAL service.

### Step 1: HAL Policy Template

**File:** `device/mycompany/mydevice/sepolicy/hal_myhal.te`

```
# Define HAL domain
type hal_myhal, domain;
hal_server_domain(hal_myhal, hal_myhal)

type hal_myhal_exec, exec_type, vendor_file_type, file_type;

# Init launches HAL
init_daemon_domain(hal_myhal)

# HAL can register with hwservicemanager
add_hwservice(hal_myhal, hal_myhal_hwservice)

# Framework can call HAL
binder_call(hal_myhal_client, hal_myhal)

# HAL can access hardware
allow hal_myhal hal_myhal_device:chr_file rw_file_perms;

# Read vendor configs
allow hal_myhal vendor_configs_file:dir r_dir_perms;
allow hal_myhal vendor_configs_file:file r_file_perms;

# Set HAL properties
set_prop(hal_myhal, hal_myhal_prop)

# Logging
allow hal_myhal kernel:system syslog_read;
```

### Step 2: Client Policy

```
# Define client attribute
attribute hal_myhal_client;

# System server is a client
typeattribute system_server hal_myhal_client;

# Clients can find HAL service
allow hal_myhal_client hal_myhal_hwservice:hwservice_manager find;

# Clients can call HAL
binder_call(hal_myhal_client, hal_myhal)
```

### Step 3: File Contexts

```
# HAL executable
/vendor/bin/hw/android\.hardware\.myhal@1\.0-service  u:object_r:hal_myhal_exec:s0

# HAL device
/dev/myhal                                            u:object_r:hal_myhal_device:s0
```

### Step 4: Service Contexts

```
android.hardware.myhal::IMyHal    u:object_r:hal_myhal_hwservice:s0
```

## SELinux Best Practices

### 1. Principle of Least Privilege

Only grant necessary permissions:

```
# Bad: Too permissive
allow my_service domain:process sigkill;

# Good: Specific permission
allow my_service self:process sigkill;
```

### 2. Use Macros

Leverage existing macros:

```
# Instead of:
allow my_service binder_device:chr_file rw_file_perms;
allow my_service hwbinder_device:chr_file rw_file_perms;
allow my_service hwservicemanager:binder call;
# ...many more lines...

# Use:
binder_use(my_service)
```

### 3. Avoid Overly Broad Rules

```
# Bad: Allows access to ALL files
allow my_service file_type:file read;

# Good: Specific file types
allow my_service system_file:file read;
allow my_service my_config_file:file read;
```

### 4. Document Policy

```
# My daemon needs to read sensor data from sysfs
# Bug: #12345
allow my_daemon sysfs_sensors:file r_file_perms;
```

### 5. Test in Permissive Mode First

```bash
# Set service domain to permissive
adb shell su 0 setenforce 0

# Or per-domain (requires sepolicy change)
permissive my_daemon;
```

### 6. Use Neverallow Assertions

Enforce security requirements:

```
# Ensure apps never get system capabilities
neverallow appdomain self:capability sys_admin;

# Ensure only init can write system files
neverallow { domain -init } system_file:file write;
```

## Common SELinux Issues

### Issue 1: Service Won't Start

**Symptom:** Service crashes immediately after start

**Debug:**
```bash
adb logcat -b events | grep avc
```

**Solution:** Check for denials and add necessary permissions

### Issue 2: Binder Call Denied

**Denial:**
```
avc: denied { call } for scontext=u:r:client:s0 
tcontext=u:r:server:s0 tclass=binder
```

**Solution:**
```
binder_call(client, server)
```

### Issue 3: File Access Denied

**Denial:**
```
avc: denied { read } for name="config.xml" 
scontext=u:r:my_service:s0 tcontext=u:object_r:system_file:s0
```

**Solution:**
```
allow my_service system_file:file read;
```

### Issue 4: Property Access Denied

**Denial:**
```
avc: denied { set } for property=my.prop 
scontext=u:r:my_service:s0 tcontext=u:object_r:default_prop:s0
```

**Solution:**
1. Define property type in property_contexts
2. Add set_prop rule

```
# property_contexts
my.prop    u:object_r:my_prop:s0

# my_service.te
set_prop(my_service, my_prop)
```

## Verified Boot and dm-verity

### Verified Boot

Ensures system partition hasn't been tampered with:

```
Bootloader
  ↓ (verifies)
Boot Image (kernel + ramdisk)
  ↓ (verifies)
System Partition (via dm-verity)
  ↓ (verifies)
Vendor Partition (via dm-verity)
```

### dm-verity

Device-mapper target that verifies block integrity:

```bash
# Check dm-verity status
adb shell cat /proc/mounts | grep dm

# Example output:
# /dev/block/dm-0 /system ext4 ro,seclabel,relatime,data=ordered 0 0
```

**Disable verification (for development):**
```bash
adb root
adb disable-verity
adb reboot
```

## Key Takeaways

1. **SELinux provides mandatory access control**: Enforced at kernel level
2. **Context labels everything**: Processes, files, services, properties
3. **Policy defines allowed operations**: Source, target, class, permissions
4. **Denial debugging is iterative**: View denials, understand, add policy
5. **Use macros and patterns**: Don't reinvent the wheel
6. **Principle of least privilege**: Only grant necessary permissions
7. **Test thoroughly**: Both functionality and security
8. **Neverallow enforces security**: Compile-time verification

## Quick Reference

### SELinux Commands

```bash
# Check mode
adb shell getenforce

# Set mode (temporary)
adb shell setenforce 0  # Permissive
adb shell setenforce 1  # Enforcing

# View contexts
adb shell ps -Z          # Processes
adb shell ls -Z /path    # Files
adb shell service list   # Services

# View denials
adb logcat -b events | grep avc
adb shell dmesg | grep avc

# Check file context
adb shell ls -Z /system/bin/app_process
```

### Policy Syntax Quick Reference

```
# Basic rule
allow source target:class permission;

# Multiple permissions
allow source target:class { perm1 perm2 };

# Type declaration
type mytype;
type mytype, attribute1, attribute2;

# Type transition
type_transition source target:class result;

# File labeling
/path/to/file    u:object_r:type:s0

# Macros
binder_use(domain)
binder_call(from, to)
set_prop(domain, property_type)
```

### Common Permission Sets

```
r_file_perms     = { read open getattr }
w_file_perms     = { write append }
rw_file_perms    = { r_file_perms w_file_perms }
create_file_perms = { rw_file_perms create unlink }
r_dir_perms      = { read search open getattr }
create_dir_perms = { r_dir_perms create rmdir }
```

## Conclusion

This completes our comprehensive AOSP development guide with 11 chapters covering all critical aspects of Android system development. You now have deep knowledge of Android's security architecture and SELinux policy system, enabling you to build secure, production-ready AOSP components.

The guide covers everything from project structure and build environment through system services, HAL implementation, power management, and security—providing you with the foundation to become a proficient AOSP developer.
