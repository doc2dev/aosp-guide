# Chapter 14: OTA Updates & System Partitions

## Introduction

Over-The-Air (OTA) updates and partition management are critical for maintaining Android devices in production. Understanding these systems enables you to:
- Deploy system updates remotely
- Manage A/B seamless updates
- Configure partition layouts
- Implement recovery mechanisms
- Ensure update security and integrity
- Handle rollback scenarios

This chapter explores Android's partition architecture, OTA update mechanisms, A/B updates, recovery system, and practical implementation.

## Android Partition Architecture

### Standard Partition Layout

```
┌─────────────────────────────────────┐
│         Boot Partition               │
│  - Kernel                           │
│  - Ramdisk                          │
│  - Device tree                      │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         System Partition             │
│  - Android framework                │
│  - System apps                      │
│  - Libraries                        │
│  - Read-only (mounted at /)         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Vendor Partition             │
│  - Vendor-specific code             │
│  - HAL implementations              │
│  - Firmware                         │
│  - Device-specific libs             │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Product Partition            │
│  - Product-specific customizations  │
│  - OEM apps                         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Data Partition               │
│  - User data                        │
│  - App data                         │
│  - Read-write                       │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Recovery Partition           │
│  - Recovery kernel                  │
│  - Recovery UI                      │
│  - Update installer                 │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Cache Partition              │
│  - Temporary files                  │
│  - OTA packages (legacy)            │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Misc Partition               │
│  - Bootloader messages              │
│  - Recovery commands                │
└─────────────────────────────────────┘
```

### Project Treble Partitions

**Treble separates system and vendor:**

```
System Partition (Google)
  - Framework
  - System apps
  - Platform libraries

Vendor Partition (OEM)
  - Device-specific code
  - HALs
  - Kernel modules
  - Firmware

Product Partition (OEM)
  - Customizations
  - OEM apps
```

### Viewing Partitions

```bash
# List all partitions
adb shell ls -l /dev/block/by-name/

# Example output:
# boot -> /dev/block/sda1
# system -> /dev/block/sda2
# vendor -> /dev/block/sda3
# userdata -> /dev/block/sda4

# View partition info
adb shell df -h

# Mount points
adb shell mount | grep /dev/block
```

### Partition Configuration

**File:** `device/manufacturer/product/BoardConfig.mk`

```makefile
# Partition sizes (in bytes)
BOARD_BOOTIMAGE_PARTITION_SIZE := 67108864          # 64MB
BOARD_SYSTEMIMAGE_PARTITION_SIZE := 3221225472      # 3GB
BOARD_VENDORIMAGE_PARTITION_SIZE := 805306368       # 768MB
BOARD_PRODUCTIMAGE_PARTITION_SIZE := 268435456      # 256MB
BOARD_USERDATAIMAGE_PARTITION_SIZE := 10737418240   # 10GB

# Partition filesystem types
TARGET_USERIMAGES_USE_EXT4 := true
BOARD_SYSTEMIMAGE_FILE_SYSTEM_TYPE := ext4
BOARD_VENDORIMAGE_FILE_SYSTEM_TYPE := ext4
BOARD_PRODUCTIMAGE_FILE_SYSTEM_TYPE := ext4

# Dynamic partitions (for A/B)
BOARD_SUPER_PARTITION_SIZE := 9126805504
BOARD_SUPER_PARTITION_GROUPS := google_dynamic_partitions
BOARD_GOOGLE_DYNAMIC_PARTITIONS_SIZE := 9122611200
BOARD_GOOGLE_DYNAMIC_PARTITIONS_PARTITION_LIST := system vendor product
```

## A/B Seamless Updates

### A/B Update Architecture

```
┌─────────────────────────────────────┐
│         Slot A (Active)              │
│  - boot_a                           │
│  - system_a                         │
│  - vendor_a                         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Slot B (Inactive)            │
│  - boot_b                           │
│  - system_b                         │
│  - vendor_b                         │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│      Shared (Not duplicated)         │
│  - userdata                         │
│  - misc                             │
└─────────────────────────────────────┘
```

**Update process:**
1. Device running from Slot A
2. Download update package
3. Install update to Slot B (in background)
4. Mark Slot B as bootable
5. Reboot into Slot B
6. If successful, mark B as good
7. If failed, rollback to Slot A

### Benefits of A/B Updates

- **No downtime**: Updates install in background
- **Fast recovery**: Automatic rollback on failure
- **No recovery partition**: Updates from running system
- **Smaller updates**: Only changed data

### Configuring A/B Updates

**File:** `device/manufacturer/product/BoardConfig.mk`

```makefile
# Enable A/B updates
AB_OTA_UPDATER := true

# A/B partitions
AB_OTA_PARTITIONS := \
    boot \
    system \
    vendor \
    product \
    vbmeta

# Postinstall script (runs after OTA)
AB_OTA_POSTINSTALL_CONFIG += \
    RUN_POSTINSTALL_system=true \
    POSTINSTALL_PATH_system=system/bin/otapreopt_script \
    FILESYSTEM_TYPE_system=ext4 \
    POSTINSTALL_OPTIONAL_system=true
```

**File:** `device/manufacturer/product/device.mk`

```makefile
# A/B packages
PRODUCT_PACKAGES += \
    update_engine \
    update_engine_client \
    update_verifier \
    bootctrl.$(TARGET_BOARD_PLATFORM) \
    android.hardware.boot@1.1-impl-qti \
    android.hardware.boot@1.1-impl-qti.recovery \
    android.hardware.boot@1.1-service

# A/B scripts
PRODUCT_PACKAGES += \
    otapreopt_script \
    cppreopts.sh \
    update_engine_sideload
```

## OTA Package Structure

### OTA Package Contents

```
ota_package.zip
├── META-INF/
│   └── com/
│       └── android/
│           ├── metadata           # Update metadata
│           ├── otacert            # Signing certificate
│           └── metadata.pb        # Protobuf metadata
├── payload.bin                    # Binary patch data
├── payload_properties.txt         # Payload properties
└── care_map.pb                    # Partition info
```

### Metadata File

```
ota-type=AB
ota-required-cache=0
ota-streaming-property-files=payload.bin,payload_properties.txt,metadata
post-build=manufacturer/product/build_id:13/ABC123/1234567:user/release-keys
post-timestamp=1234567890
pre-build=manufacturer/product/build_id:13/ABC122/1234566:user/release-keys
pre-device=product_name
```

### Payload Format

The payload.bin contains:
- **Delta updates**: Binary diffs between versions
- **Full images**: Complete partition images
- **Compressed data**: zlib/bzip2/xz compression

## Building OTA Packages

### Full OTA Package

```bash
# Build target files
m target-files-package

# Build OTA package
./build/tools/releasetools/ota_from_target_files \
    -k build/target/product/security/testkey \
    out/target/product/mydevice/obj/PACKAGING/target_files_intermediates/mydevice-target_files.zip \
    ota_update.zip
```

### Incremental OTA Package

```bash
# Build incremental from old to new
./build/tools/releasetools/ota_from_target_files \
    -k build/target/product/security/testkey \
    -i old_target_files.zip \
    new_target_files.zip \
    incremental_ota.zip
```

### OTA Package Options

```bash
# A/B OTA
./build/tools/releasetools/ota_from_target_files \
    -k security/testkey \
    --block \
    target_files.zip \
    ota_ab.zip

# Full OTA (forces full image)
./build/tools/releasetools/ota_from_target_files \
    -k security/testkey \
    --block \
    --full_radio \
    target_files.zip \
    ota_full.zip

# Partial OTA (specific partitions)
./build/tools/releasetools/ota_from_target_files \
    -k security/testkey \
    --block \
    --partial system,vendor \
    target_files.zip \
    ota_partial.zip
```

## Update Engine

### Update Engine Architecture

```
┌─────────────────────────────────────┐
│      Update Engine Client            │
│  (System app or service)            │
└──────────────┬──────────────────────┘
               │ Binder IPC
┌──────────────▼──────────────────────┐
│      update_engine Service           │
│  - Download manager                 │
│  - Payload verification             │
│  - Payload application              │
│  - Rollback management              │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│      boot_control HAL                │
│  - Slot management                  │
│  - Bootloader communication         │
└─────────────────────────────────────┘
```

### Triggering Updates

**From adb:**

```bash
# Push OTA package
adb push ota_update.zip /data/ota_package/

# Apply update
adb shell su 0 update_engine_client \
    --payload=file:///data/ota_package/payload.bin \
    --update \
    --headers="FILE_HASH=hash_value
FILE_SIZE=file_size
METADATA_HASH=metadata_hash
METADATA_SIZE=metadata_size"
```

**From code:**

```java
UpdateEngine engine = new UpdateEngine();

UpdateEngineCallback callback = new UpdateEngineCallback() {
    @Override
    public void onStatusUpdate(int status, float percent) {
        Log.i(TAG, "Update status: " + status + ", progress: " + percent);
    }
    
    @Override
    public void onPayloadApplicationComplete(int errorCode) {
        if (errorCode == UpdateEngine.ErrorCodeConstants.SUCCESS) {
            Log.i(TAG, "Update successful, rebooting...");
            // Reboot to apply update
            PowerManager pm = context.getSystemService(PowerManager.class);
            pm.reboot(null);
        } else {
            Log.e(TAG, "Update failed: " + errorCode);
        }
    }
};

engine.bind(callback);

// Apply update
String[] headers = {
    "FILE_HASH=" + fileHash,
    "FILE_SIZE=" + fileSize,
    "METADATA_HASH=" + metadataHash,
    "METADATA_SIZE=" + metadataSize
};

engine.applyPayload(
    "file:///data/ota_package/payload.bin",
    0,  // payload offset
    fileSize,
    headers
);
```

### Update Status Codes

```java
// UpdateEngine status
IDLE = 0
CHECKING_FOR_UPDATE = 1
UPDATE_AVAILABLE = 2
DOWNLOADING = 3
VERIFYING = 4
FINALIZING = 5
UPDATED_NEED_REBOOT = 6
REPORTING_ERROR_EVENT = 7
ATTEMPTING_ROLLBACK = 8
DISABLED = 9
```

## Recovery System

### Recovery Mode

Recovery mode provides:
- System updates (non-A/B devices)
- Factory reset
- Partition wiping
- adb sideload

**Booting to recovery:**

```bash
# Via adb
adb reboot recovery

# Via bootloader
adb reboot bootloader
fastboot boot recovery.img

# Programmatically
String command = "--update_package=/data/ota_package/update.zip\n";
RecoverySystem.installPackage(context, new File(packagePath));
```

### Recovery UI

**File:** `bootable/recovery/recovery_ui.cpp`

```cpp
class RecoveryUI {
public:
    // Display update progress
    void SetProgress(float fraction);
    
    // Show message
    void Print(const char* fmt, ...);
    
    // Show menu
    int ShowMenu(const char* const* headers,
                 const char* const* items,
                 int initial_selection,
                 bool menu_only);
    
    // Wait for key
    int WaitKey();
};
```

### Custom Recovery Actions

**File:** `device/manufacturer/product/recovery/recovery_ui.cpp`

```cpp
#include "recovery_ui/device.h"
#include "recovery_ui/ui.h"

class MyDeviceUI : public RecoveryUI {
public:
    MyDeviceUI() {}
    
    void Init() override {
        RecoveryUI::Init();
        // Custom initialization
    }
    
    // Custom menu items
    const char* const* GetMenuHeaders() override {
        static const char* headers[] = {
            "My Device Recovery",
            "Build: 1.0.0",
            nullptr
        };
        return headers;
    }
    
    const char* const* GetMenuItems() override {
        static const char* items[] = {
            "Reboot system now",
            "Apply update from ADB",
            "Wipe data/factory reset",
            "Wipe cache partition",
            "Mount /system",
            "View recovery logs",
            "Custom action",
            nullptr
        };
        return items;
    }
    
    Device::BuiltinAction InvokeMenuItem(int menu_position) override {
        switch (menu_position) {
            case 0: return Device::REBOOT;
            case 1: return Device::APPLY_ADB_SIDELOAD;
            case 2: return Device::WIPE_DATA;
            case 3: return Device::WIPE_CACHE;
            case 6: return PerformCustomAction();
            default: return Device::NO_ACTION;
        }
    }
    
private:
    Device::BuiltinAction PerformCustomAction() {
        Print("Performing custom action...\n");
        // Custom recovery action
        SetProgress(1.0);
        return Device::NO_ACTION;
    }
};
```

## Verified Boot

### dm-verity

Ensures system partition integrity:

```bash
# Check dm-verity status
adb shell cat /proc/mounts | grep dm

# Output:
# /dev/block/dm-0 /system ext4 ro,seclabel,relatime,data=ordered 0 0

# Disable verity (for development)
adb root
adb disable-verity
adb reboot
```

### AVB (Android Verified Boot)

**vbmeta partition** stores verification metadata:

```bash
# View vbmeta info
avbtool info_image --image vbmeta.img

# Output:
# Minimum libavb version:   1.0
# Header Block:             256 bytes
# Authentication Block:     320 bytes
# Auxiliary Block:          832 bytes
# Algorithm:                SHA256_RSA2048
# Rollback Index:           0
# Flags:                    0
# Release String:           'avbtool 1.2.0'
```

### Signing OTA Packages

```bash
# Generate signing key
openssl genrsa -out testkey.pem 2048
openssl req -new -x509 -key testkey.pem -out testkey.x509.pem -days 10000

# Sign OTA package
java -Xmx2048m -jar out/host/linux-x86/framework/signapk.jar \
    -w build/target/product/security/testkey.x509.pem \
    build/target/product/security/testkey.pk8 \
    unsigned_ota.zip \
    signed_ota.zip
```

## Practical Example: Custom OTA Server

### Step 1: OTA Metadata Generation

```python
#!/usr/bin/env python3
import hashlib
import json
import os

def generate_ota_metadata(ota_path):
    """Generate OTA metadata for update server"""
    
    # Calculate file hash
    sha256_hash = hashlib.sha256()
    with open(ota_path, "rb") as f:
        for chunk in iter(lambda: f.read(4096), b""):
            sha256_hash.update(chunk)
    
    file_hash = sha256_hash.hexdigest()
    file_size = os.path.getsize(ota_path)
    
    metadata = {
        "version": "1.0.0",
        "build_id": "ABC123",
        "timestamp": 1234567890,
        "url": "https://update.example.com/ota.zip",
        "file_size": file_size,
        "file_hash": file_hash,
        "min_version": "0.9.0",
        "changelog": "Bug fixes and improvements",
        "mandatory": False
    }
    
    return metadata

# Generate metadata
metadata = generate_ota_metadata("ota_update.zip")
with open("ota_metadata.json", "w") as f:
    json.dump(metadata, f, indent=2)
```

### Step 2: Update Client Service

```java
public class UpdateService extends Service {
    private static final String UPDATE_URL = "https://update.example.com/metadata.json";
    private UpdateEngine mUpdateEngine;
    
    @Override
    public void onCreate() {
        super.onCreate();
        mUpdateEngine = new UpdateEngine();
        mUpdateEngine.bind(mCallback);
    }
    
    public void checkForUpdates() {
        // Fetch metadata from server
        new Thread(() -> {
            try {
                JSONObject metadata = fetchMetadata(UPDATE_URL);
                
                String availableVersion = metadata.getString("version");
                String currentVersion = Build.VERSION.INCREMENTAL;
                
                if (isNewerVersion(availableVersion, currentVersion)) {
                    // Download and apply update
                    downloadAndApplyUpdate(metadata);
                }
            } catch (Exception e) {
                Log.e(TAG, "Update check failed", e);
            }
        }).start();
    }
    
    private JSONObject fetchMetadata(String url) throws Exception {
        HttpURLConnection conn = (HttpURLConnection) new URL(url).openConnection();
        try {
            BufferedReader reader = new BufferedReader(
                new InputStreamReader(conn.getInputStream()));
            StringBuilder response = new StringBuilder();
            String line;
            while ((line = reader.readLine()) != null) {
                response.append(line);
            }
            return new JSONObject(response.toString());
        } finally {
            conn.disconnect();
        }
    }
    
    private void downloadAndApplyUpdate(JSONObject metadata) throws Exception {
        String url = metadata.getString("url");
        long fileSize = metadata.getLong("file_size");
        String expectedHash = metadata.getString("file_hash");
        
        // Download to temporary file
        File tempFile = new File(getCacheDir(), "update.zip");
        downloadFile(url, tempFile);
        
        // Verify hash
        String actualHash = calculateHash(tempFile);
        if (!actualHash.equals(expectedHash)) {
            throw new SecurityException("Hash mismatch");
        }
        
        // Extract payload
        File payloadFile = extractPayload(tempFile);
        
        // Apply update
        String[] headers = buildHeaders(metadata);
        mUpdateEngine.applyPayload(
            "file://" + payloadFile.getAbsolutePath(),
            0,
            fileSize,
            headers
        );
    }
    
    private String[] buildHeaders(JSONObject metadata) throws Exception {
        return new String[] {
            "FILE_HASH=" + metadata.getString("file_hash"),
            "FILE_SIZE=" + metadata.getLong("file_size"),
        };
    }
    
    private final UpdateEngineCallback mCallback = new UpdateEngineCallback() {
        @Override
        public void onStatusUpdate(int status, float percent) {
            // Update UI
            sendBroadcast(new Intent("UPDATE_PROGRESS")
                .putExtra("status", status)
                .putExtra("percent", percent));
        }
        
        @Override
        public void onPayloadApplicationComplete(int errorCode) {
            if (errorCode == ErrorCodeConstants.SUCCESS) {
                // Schedule reboot
                scheduleReboot();
            } else {
                Log.e(TAG, "Update failed: " + errorCode);
            }
        }
    };
    
    private void scheduleReboot() {
        // Give user option to reboot or delay
        Intent intent = new Intent(this, RebootActivity.class);
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
        startActivity(intent);
    }
}
```

### Step 3: Update Scheduler

```java
public class UpdateScheduler {
    private static final long CHECK_INTERVAL = 24 * 60 * 60 * 1000; // 24 hours
    
    public static void scheduleUpdateCheck(Context context) {
        AlarmManager alarmManager = context.getSystemService(AlarmManager.class);
        
        Intent intent = new Intent(context, UpdateService.class);
        intent.setAction("CHECK_FOR_UPDATES");
        
        PendingIntent pendingIntent = PendingIntent.getService(
            context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
        
        // Schedule periodic check
        alarmManager.setRepeating(
            AlarmManager.ELAPSED_REALTIME_WAKEUP,
            SystemClock.elapsedRealtime() + CHECK_INTERVAL,
            CHECK_INTERVAL,
            pendingIntent
        );
    }
}
```

## Rollback Management

### Automatic Rollback

A/B updates support automatic rollback:

```java
// Mark current slot as successful
BootControl bootControl = BootControl.getInstance();
bootControl.markBootSuccessful();

// If not marked successful within N boots, rollback
```

### Manual Rollback

```bash
# Check current slot
adb shell getprop ro.boot.slot_suffix
# Output: _a or _b

# Switch to other slot
adb shell bootctl set-active-boot-slot 1  # Switch to slot B
adb reboot

# Or from bootloader
adb reboot bootloader
fastboot set_active b
fastboot reboot
```

### Rollback Prevention

```makefile
# In BoardConfig.mk
# Prevent rollback to older versions
BOARD_AVB_ROLLBACK_INDEX := 1
```

## OTA Testing

### Testing Workflow

```bash
# 1. Build two versions
m -j$(nproc)
cp $OUT/target-files.zip target-files-v1.zip

# Make changes...

m -j$(nproc)
cp $OUT/target-files.zip target-files-v2.zip

# 2. Generate OTA package
ota_from_target_files -i target-files-v1.zip target-files-v2.zip ota.zip

# 3. Flash first version
fastboot flashall

# 4. Apply OTA
adb push ota.zip /data/
adb shell update_engine_client --update --payload=file:///data/ota.zip

# 5. Verify update
adb wait-for-device
adb shell getprop ro.build.version.incremental

# 6. Test rollback
adb shell bootctl set-active-boot-slot 0
adb reboot
```

### Stress Testing

```bash
# Repeated update cycles
for i in {1..100}; do
    echo "Update cycle $i"
    adb push ota.zip /data/
    adb shell update_engine_client --update --payload=file:///data/ota.zip
    adb wait-for-device
    sleep 60
    adb reboot
    adb wait-for-device
    sleep 30
done
```

## Key Takeaways

1. **A/B updates are superior**: Seamless, safe, with automatic rollback
2. **Partition layout is critical**: Proper sizing and organization
3. **Security is paramount**: Verified boot, signed packages
4. **Testing is essential**: Test updates, rollbacks, and failure scenarios
5. **OTA server needed**: Metadata management, delivery infrastructure
6. **Recovery is backup**: Still needed for factory reset and emergency
7. **Monitor update success**: Track metrics and failure rates

## Quick Reference

### Partition Commands

```bash
# View partitions
adb shell ls /dev/block/by-name/
adb shell cat /proc/partitions
adb shell df -h

# Flash partitions
fastboot flash boot boot.img
fastboot flash system system.img
fastboot flash vendor vendor.img

# Erase partitions
fastboot erase cache
fastboot erase userdata
```

### OTA Commands

```bash
# Build OTA
ota_from_target_files target_files.zip ota.zip

# Incremental OTA
ota_from_target_files -i old.zip new.zip incremental.zip

# Apply OTA (A/B)
update_engine_client --update --payload=file:///data/ota.zip

# Apply OTA (Recovery)
adb sideload ota.zip

# Check update status
update_engine_client --status
```

### Slot Management

```bash
# Current slot
getprop ro.boot.slot_suffix

# List slots
bootctl get-number-slots
bootctl get-current-slot

# Set active slot
bootctl set-active-boot-slot 1

# Mark successful
bootctl mark-boot-successful

# Check if slot marked successful
bootctl is-slot-marked-successful 0
```

## Conclusion

This completes our comprehensive 14-chapter AOSP development guide! You now have complete knowledge of:

- Project structure and build system
- Core system services and IPC
- Display and window management
- Power and security management
- Hardware abstraction layers
- Debugging and performance optimization
- **OTA updates and partition management**

This guide provides everything needed to build, customize, deploy, and maintain production Android systems professionally.

## Summary of All 14 Chapters

1. **AOSP Project Structure** - Understanding the codebase
2. **Development Environment** - Setup and workflow
3. **Init & Boot Process** - System startup
4. **Binder IPC** - Inter-process communication
5. **PackageManager** - App lifecycle
6. **WindowManager & SurfaceFlinger** - Display system
7. **ActivityManager** - Process management
8. **System UI** - User interface framework
9. **Hardware Abstraction Layer** - Hardware integration
10. **Power Management** - Battery optimization
11. **Security & SELinux** - System security
12. **Build System** - Building and configuration
13. **Debugging & Performance** - Development tools
14. **OTA & Partitions** - Updates and deployment

You are now equipped to be a professional AOSP developer! 🎉
