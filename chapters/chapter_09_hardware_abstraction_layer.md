# Chapter 9: Hardware Abstraction Layer (HAL)

## Introduction

The Hardware Abstraction Layer (HAL) is Android's interface between the framework and device hardware. It allows Android to remain hardware-agnostic while enabling device manufacturers to implement hardware-specific functionality without exposing proprietary code to the open-source framework.

Understanding HAL is essential for AOSP development because it:
- Enables hardware integration without modifying the framework
- Provides a stable interface for vendor implementations
- Supports Project Treble's system/vendor separation
- Allows proprietary drivers while keeping Android open source
- Defines the contract between Android and device hardware

This chapter explores HAL architecture, HIDL (Hardware Interface Definition Language), AIDL HALs, and provides practical examples of implementing custom HAL modules.

## HAL Evolution

### Legacy HAL (Pre-Android 8.0)

Original HAL used simple C structures:

```c
// hardware/libhardware/include/hardware/hardware.h

typedef struct hw_module_t {
    uint32_t tag;
    uint16_t module_api_version;
    uint16_t hal_api_version;
    const char *id;
    const char *name;
    const char *author;
    struct hw_module_methods_t* methods;
    void* dso;
    uint32_t reserved[32];
} hw_module_t;

typedef struct hw_device_t {
    uint32_t tag;
    uint32_t version;
    struct hw_module_t* module;
    uint32_t reserved[12];
    int (*close)(struct hw_device_t* device);
} hw_device_t;
```

**Problems with legacy HAL:**
- No versioning support
- Unstable ABI between Android versions
- Vendors must recompile for each Android version
- No clear separation of vendor/system code

### HIDL (Android 8.0+)

Hardware Interface Definition Language provides:
- **Versioned interfaces**: `@1.0`, `@1.1`, etc.
- **Binary stability**: No recompilation needed
- **Process isolation**: HALs run in separate processes
- **Binder communication**: Uses `/dev/hwbinder`

### AIDL HAL (Android 11+)

AIDL (Android Interface Definition Language) for HALs:
- **Simpler syntax**: Reuses standard AIDL
- **Better performance**: Reduced overhead
- **Stability guarantees**: Like HIDL but cleaner
- **Migration path**: From HIDL to AIDL

## HAL Architecture Overview

```
┌────────────────────────────────────────────────┐
│         Android Framework (System)              │
│  ┌──────────────────────────────────────────┐  │
│  │  Framework Services                      │  │
│  │  (AudioService, CameraService, etc.)     │  │
│  └────────────────┬─────────────────────────┘  │
│                   │ Java/JNI                    │
│  ┌────────────────▼─────────────────────────┐  │
│  │  Native Framework Libraries              │  │
│  │  (libaudioclient, libcamera_client)      │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │ HIDL/AIDL (Binder)
┌───────────────────▼─────────────────────────────┐
│         HAL Implementation (Vendor)             │
│  ┌──────────────────────────────────────────┐  │
│  │  HAL Service Process                     │  │
│  │  android.hardware.audio@7.0-service      │  │
│  └────────────────┬─────────────────────────┘  │
│                   │                             │
│  ┌────────────────▼─────────────────────────┐  │
│  │  Vendor Implementation                   │  │
│  │  (Proprietary drivers, vendor libs)      │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              Kernel Drivers                     │
│  (Audio driver, camera driver, etc.)           │
└─────────────────────────────────────────────────┘
```

### Key Concepts

**Passthrough vs. Binderized:**
- **Passthrough**: HAL runs in calling process (legacy compatibility)
- **Binderized**: HAL runs in separate process (preferred, better isolation)

**Same-process vs. Different-process:**
- **Same-process**: Framework and HAL in same process (passthrough)
- **Different-process**: Framework and HAL in different processes (binderized)

**Treble Compliance:**
- System partition: Android framework
- Vendor partition: HAL implementations
- Clear interface boundary via VINTF

## HIDL Fundamentals

### HIDL Interface Definition

**Example: Custom sensor HAL**

**File:** `hardware/interfaces/mysensor/1.0/IMySensor.hal`

```hidl
package android.hardware.mysensor@1.0;

/**
 * Interface for custom sensor HAL
 */
interface IMySensor {
    /**
     * Initialize the sensor
     * @return result OK if successful
     */
    initialize() generates (Result result);
    
    /**
     * Read sensor value
     * @return result OK if successful
     * @return value The sensor reading
     */
    readValue() generates (Result result, int32_t value);
    
    /**
     * Configure sensor parameters
     * @param config Configuration parameters
     * @return result OK if successful
     */
    configure(SensorConfig config) generates (Result result);
    
    /**
     * Register callback for sensor events
     * @param callback Callback interface
     * @return result OK if successful
     */
    registerCallback(IMySensorCallback callback) generates (Result result);
};

/**
 * Callback interface for sensor events
 */
interface IMySensorCallback {
    /**
     * Called when sensor value changes
     * @param value New sensor value
     */
    oneway onValueChanged(int32_t value);
    
    /**
     * Called when sensor error occurs
     * @param error Error code
     */
    oneway onError(ErrorCode error);
};

/**
 * Sensor configuration
 */
struct SensorConfig {
    /** Sampling rate in Hz */
    uint32_t samplingRate;
    
    /** Sensitivity level (0-100) */
    uint8_t sensitivity;
    
    /** Enable/disable filtering */
    bool enableFiltering;
};

/**
 * Result codes
 */
enum Result : int32_t {
    OK = 0,
    ERROR_INVALID_ARGUMENTS = 1,
    ERROR_NOT_INITIALIZED = 2,
    ERROR_DEVICE_FAILURE = 3,
    ERROR_NOT_SUPPORTED = 4,
};

/**
 * Error codes for callbacks
 */
enum ErrorCode : int32_t {
    HARDWARE_FAILURE = 1,
    CALIBRATION_ERROR = 2,
    COMMUNICATION_ERROR = 3,
};
```

### HIDL Types

**Basic types:**
```hidl
bool                // Boolean
int8_t, int16_t, int32_t, int64_t    // Signed integers
uint8_t, uint16_t, uint32_t, uint64_t // Unsigned integers
float, double       // Floating point
string              // String
handle              // File descriptor
```

**Complex types:**
```hidl
vec<T>              // Vector (dynamic array)
array<T, N>         // Fixed-size array
struct              // Structure
union               // Union
enum                // Enumeration
bitfield            // Bit field
```

**Special types:**
```hidl
oneway              // Asynchronous call (no return value)
generates           // Return values
```

### Compiling HIDL

**Android.bp for HIDL interface:**

```go
hidl_interface {
    name: "android.hardware.mysensor@1.0",
    root: "android.hardware",
    srcs: [
        "IMySensor.hal",
        "IMySensorCallback.hal",
        "types.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: true,
    gen_java_constants: true,
}
```

**Build generates:**
- C++ headers and implementation stubs
- Java interfaces (if gen_java: true)
- Adapter code for passthrough/binderized modes

### HIDL Implementation

**Server-side implementation:**

**File:** `hardware/interfaces/mysensor/1.0/default/MySensor.h`

```cpp
#ifndef ANDROID_HARDWARE_MYSENSOR_V1_0_MYSENSOR_H
#define ANDROID_HARDWARE_MYSENSOR_V1_0_MYSENSOR_H

#include <android/hardware/mysensor/1.0/IMySensor.h>
#include <hidl/MQDescriptor.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace mysensor {
namespace V1_0 {
namespace implementation {

using ::android::hardware::hidl_array;
using ::android::hardware::hidl_memory;
using ::android::hardware::hidl_string;
using ::android::hardware::hidl_vec;
using ::android::hardware::Return;
using ::android::hardware::Void;
using ::android::sp;

struct MySensor : public IMySensor {
    // Methods from ::android::hardware::mysensor::V1_0::IMySensor follow.
    
    Return<Result> initialize() override;
    
    Return<void> readValue(readValue_cb _hidl_cb) override;
    
    Return<Result> configure(const SensorConfig& config) override;
    
    Return<Result> registerCallback(
        const sp<IMySensorCallback>& callback) override;
    
private:
    sp<IMySensorCallback> mCallback;
    SensorConfig mConfig;
    bool mInitialized;
    int mDeviceFd;  // File descriptor to device
    
    // Helper methods
    bool openDevice();
    void closeDevice();
    int readHardwareValue();
};

}  // namespace implementation
}  // namespace V1_0
}  // namespace mysensor
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_MYSENSOR_V1_0_MYSENSOR_H
```

**File:** `hardware/interfaces/mysensor/1.0/default/MySensor.cpp`

```cpp
#include "MySensor.h"
#include <log/log.h>
#include <fcntl.h>
#include <unistd.h>

namespace android {
namespace hardware {
namespace mysensor {
namespace V1_0 {
namespace implementation {

Return<Result> MySensor::initialize() {
    ALOGD("MySensor::initialize");
    
    if (mInitialized) {
        ALOGW("Already initialized");
        return Result::OK;
    }
    
    if (!openDevice()) {
        ALOGE("Failed to open device");
        return Result::ERROR_DEVICE_FAILURE;
    }
    
    mInitialized = true;
    return Result::OK;
}

Return<void> MySensor::readValue(readValue_cb _hidl_cb) {
    ALOGV("MySensor::readValue");
    
    if (!mInitialized) {
        _hidl_cb(Result::ERROR_NOT_INITIALIZED, 0);
        return Void();
    }
    
    int value = readHardwareValue();
    if (value < 0) {
        ALOGE("Failed to read hardware value");
        _hidl_cb(Result::ERROR_DEVICE_FAILURE, 0);
        return Void();
    }
    
    _hidl_cb(Result::OK, value);
    return Void();
}

Return<Result> MySensor::configure(const SensorConfig& config) {
    ALOGD("MySensor::configure: rate=%u, sensitivity=%u, filtering=%d",
          config.samplingRate, config.sensitivity, config.enableFiltering);
    
    if (!mInitialized) {
        return Result::ERROR_NOT_INITIALIZED;
    }
    
    // Validate configuration
    if (config.samplingRate > 1000 || config.sensitivity > 100) {
        ALOGE("Invalid configuration parameters");
        return Result::ERROR_INVALID_ARGUMENTS;
    }
    
    // Apply configuration to hardware
    // ... hardware-specific code ...
    
    mConfig = config;
    return Result::OK;
}

Return<Result> MySensor::registerCallback(
        const sp<IMySensorCallback>& callback) {
    ALOGD("MySensor::registerCallback");
    
    if (callback == nullptr) {
        return Result::ERROR_INVALID_ARGUMENTS;
    }
    
    mCallback = callback;
    return Result::OK;
}

bool MySensor::openDevice() {
    // Open device node
    mDeviceFd = open("/dev/mysensor", O_RDWR);
    if (mDeviceFd < 0) {
        ALOGE("Failed to open /dev/mysensor: %s", strerror(errno));
        return false;
    }
    
    ALOGI("Opened device: fd=%d", mDeviceFd);
    return true;
}

void MySensor::closeDevice() {
    if (mDeviceFd >= 0) {
        close(mDeviceFd);
        mDeviceFd = -1;
    }
}

int MySensor::readHardwareValue() {
    if (mDeviceFd < 0) {
        return -1;
    }
    
    int value;
    ssize_t ret = read(mDeviceFd, &value, sizeof(value));
    if (ret != sizeof(value)) {
        ALOGE("Failed to read from device: %s", strerror(errno));
        return -1;
    }
    
    return value;
}

}  // namespace implementation
}  // namespace V1_0
}  // namespace mysensor
}  // namespace hardware
}  // namespace android
```

### HAL Service Entry Point

**File:** `hardware/interfaces/mysensor/1.0/default/service.cpp`

```cpp
#include <android/hardware/mysensor/1.0/IMySensor.h>
#include <hidl/LegacySupport.h>
#include "MySensor.h"

using android::hardware::mysensor::V1_0::IMySensor;
using android::hardware::mysensor::V1_0::implementation::MySensor;
using android::hardware::defaultPassthroughServiceImplementation;
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;
using android::sp;
using android::status_t;
using android::OK;

int main() {
    android::sp<IMySensor> service = new MySensor();
    
    configureRpcThreadpool(1, true /* callerWillJoin */);
    
    status_t status = service->registerAsService();
    if (status != OK) {
        ALOGE("Cannot register MySensor HAL service");
        return 1;
    }
    
    ALOGI("MySensor HAL service is ready");
    joinRpcThreadpool();
    
    // Should never get here
    return 1;
}
```

### Android.bp for HAL Service

```go
cc_binary {
    name: "android.hardware.mysensor@1.0-service",
    relative_install_path: "hw",
    vendor: true,
    init_rc: ["android.hardware.mysensor@1.0-service.rc"],
    srcs: [
        "service.cpp",
        "MySensor.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "liblog",
        "libutils",
        "android.hardware.mysensor@1.0",
    ],
}
```

### Init RC File

**File:** `android.hardware.mysensor@1.0-service.rc`

```rc
service vendor.mysensor-1-0 /vendor/bin/hw/android.hardware.mysensor@1.0-service
    class hal
    user system
    group system
```

## Client-Side Usage

### Framework Integration

**Java framework service:**

```java
// frameworks/base/services/core/java/com/android/server/MySensorService.java

package com.android.server;

import android.hardware.mysensor.V1_0.IMySensor;
import android.hardware.mysensor.V1_0.IMySensorCallback;
import android.hardware.mysensor.V1_0.Result;
import android.hardware.mysensor.V1_0.SensorConfig;
import android.os.RemoteException;
import android.util.Slog;

public class MySensorService extends SystemService {
    private static final String TAG = "MySensorService";
    
    private IMySensor mHal;
    private final Object mLock = new Object();
    
    public MySensorService(Context context) {
        super(context);
    }
    
    @Override
    public void onStart() {
        // Get HAL service
        try {
            mHal = IMySensor.getService();
            if (mHal == null) {
                Slog.e(TAG, "Failed to get MySensor HAL");
                return;
            }
            
            // Initialize HAL
            int result = mHal.initialize();
            if (result != Result.OK) {
                Slog.e(TAG, "Failed to initialize HAL: " + result);
                return;
            }
            
            // Register callback
            mHal.registerCallback(mCallback);
            
            Slog.i(TAG, "MySensor HAL initialized successfully");
            
        } catch (RemoteException e) {
            Slog.e(TAG, "Failed to initialize MySensor HAL", e);
        }
        
        // Publish service
        publishBinderService("mysensor", new BinderService());
    }
    
    public int readSensorValue() {
        synchronized (mLock) {
            if (mHal == null) {
                Slog.w(TAG, "HAL not initialized");
                return -1;
            }
            
            try {
                // Use lambda for callback
                final int[] value = {-1};
                mHal.readValue((result, val) -> {
                    if (result == Result.OK) {
                        value[0] = val;
                    }
                });
                return value[0];
            } catch (RemoteException e) {
                Slog.e(TAG, "Failed to read sensor value", e);
                return -1;
            }
        }
    }
    
    public boolean configureSensor(int samplingRate, int sensitivity, 
            boolean enableFiltering) {
        synchronized (mLock) {
            if (mHal == null) {
                return false;
            }
            
            try {
                SensorConfig config = new SensorConfig();
                config.samplingRate = samplingRate;
                config.sensitivity = (byte) sensitivity;
                config.enableFiltering = enableFiltering;
                
                int result = mHal.configure(config);
                return result == Result.OK;
            } catch (RemoteException e) {
                Slog.e(TAG, "Failed to configure sensor", e);
                return false;
            }
        }
    }
    
    private final IMySensorCallback mCallback = new IMySensorCallback.Stub() {
        @Override
        public void onValueChanged(int value) {
            Slog.d(TAG, "Sensor value changed: " + value);
            // Handle value change
            handleSensorValueChanged(value);
        }
        
        @Override
        public void onError(int error) {
            Slog.e(TAG, "Sensor error: " + error);
            // Handle error
            handleSensorError(error);
        }
    };
    
    private void handleSensorValueChanged(int value) {
        // Notify listeners, update state, etc.
    }
    
    private void handleSensorError(int error) {
        // Handle hardware error
    }
    
    private final class BinderService extends IMySensorManager.Stub {
        @Override
        public int getSensorValue() {
            // Permission check
            mContext.enforceCallingPermission(
                android.Manifest.permission.ACCESS_MY_SENSOR,
                "getSensorValue");
            
            return readSensorValue();
        }
        
        @Override
        public boolean configure(int samplingRate, int sensitivity, 
                boolean filtering) {
            mContext.enforceCallingPermission(
                android.Manifest.permission.CONFIGURE_MY_SENSOR,
                "configure");
            
            return configureSensor(samplingRate, sensitivity, filtering);
        }
    }
}
```

## AIDL HAL (Modern Approach)

### AIDL HAL Interface

AIDL HALs use standard AIDL syntax with stability annotations:

**File:** `hardware/interfaces/mysensor/aidl/android/hardware/mysensor/IMySensor.aidl`

```aidl
package android.hardware.mysensor;

import android.hardware.mysensor.IMySensorCallback;
import android.hardware.mysensor.SensorConfig;

@VintfStability
interface IMySensor {
    /**
     * Initialize the sensor
     */
    void initialize();
    
    /**
     * Read sensor value
     * @return The sensor reading
     */
    int readValue();
    
    /**
     * Configure sensor parameters
     * @param config Configuration parameters
     */
    void configure(in SensorConfig config);
    
    /**
     * Register callback for sensor events
     * @param callback Callback interface
     */
    void registerCallback(in IMySensorCallback callback);
}
```

**File:** `hardware/interfaces/mysensor/aidl/android/hardware/mysensor/IMySensorCallback.aidl`

```aidl
package android.hardware.mysensor;

@VintfStability
oneway interface IMySensorCallback {
    /**
     * Called when sensor value changes
     * @param value New sensor value
     */
    void onValueChanged(int value);
    
    /**
     * Called when sensor error occurs
     * @param error Error code
     */
    void onError(int error);
}
```

**File:** `hardware/interfaces/mysensor/aidl/android/hardware/mysensor/SensorConfig.aidl`

```aidl
package android.hardware.mysensor;

@VintfStability
parcelable SensorConfig {
    /** Sampling rate in Hz */
    int samplingRate;
    
    /** Sensitivity level (0-100) */
    byte sensitivity;
    
    /** Enable/disable filtering */
    boolean enableFiltering;
}
```

### AIDL HAL Implementation

**File:** `hardware/interfaces/mysensor/aidl/default/MySensor.h`

```cpp
#pragma once

#include <aidl/android/hardware/mysensor/BnMySensor.h>
#include <aidl/android/hardware/mysensor/IMySensorCallback.h>

namespace aidl {
namespace android {
namespace hardware {
namespace mysensor {

class MySensor : public BnMySensor {
public:
    MySensor();
    ~MySensor();
    
    // Methods from IMySensor
    ::ndk::ScopedAStatus initialize() override;
    
    ::ndk::ScopedAStatus readValue(int32_t* _aidl_return) override;
    
    ::ndk::ScopedAStatus configure(const SensorConfig& config) override;
    
    ::ndk::ScopedAStatus registerCallback(
        const std::shared_ptr<IMySensorCallback>& callback) override;
        
private:
    std::shared_ptr<IMySensorCallback> mCallback;
    SensorConfig mConfig;
    bool mInitialized;
    int mDeviceFd;
    
    bool openDevice();
    void closeDevice();
    int readHardwareValue();
};

}  // namespace mysensor
}  // namespace hardware
}  // namespace android
}  // namespace aidl
```

**AIDL Android.bp:**

```go
aidl_interface {
    name: "android.hardware.mysensor",
    vendor_available: true,
    srcs: ["android/hardware/mysensor/*.aidl"],
    stability: "vintf",
    backend: {
        cpp: {
            enabled: true,
        },
        java: {
            sdk_version: "module_current",
        },
        ndk: {
            enabled: true,
        },
    },
    versions: ["1"],
}
```

## Practical Example 1: Simple LED HAL

Let's create a complete HAL for controlling an LED.

### Step 1: Define HIDL Interface

**File:** `hardware/interfaces/led/1.0/ILed.hal`

```hidl
package android.hardware.led@1.0;

interface ILed {
    /**
     * Turn LED on or off
     * @param on true to turn on, false to turn off
     * @return success true if successful
     */
    setEnabled(bool on) generates (bool success);
    
    /**
     * Set LED brightness
     * @param brightness 0-255
     * @return success true if successful
     */
    setBrightness(uint8_t brightness) generates (bool success);
    
    /**
     * Set LED color (if RGB LED)
     * @param color RGB color value
     * @return success true if successful
     */
    setColor(uint32_t color) generates (bool success);
    
    /**
     * Blink the LED
     * @param onMs milliseconds on
     * @param offMs milliseconds off
     * @return success true if successful
     */
    blink(uint32_t onMs, uint32_t offMs) generates (bool success);
};
```

### Step 2: Implement HAL

**File:** `hardware/interfaces/led/1.0/default/Led.h`

```cpp
#ifndef ANDROID_HARDWARE_LED_V1_0_LED_H
#define ANDROID_HARDWARE_LED_V1_0_LED_H

#include <android/hardware/led/1.0/ILed.h>
#include <hidl/Status.h>

namespace android {
namespace hardware {
namespace led {
namespace V1_0 {
namespace implementation {

using ::android::hardware::Return;

struct Led : public ILed {
    Led();
    ~Led();
    
    // Methods from ::android::hardware::led::V1_0::ILed
    Return<bool> setEnabled(bool on) override;
    Return<bool> setBrightness(uint8_t brightness) override;
    Return<bool> setColor(uint32_t color) override;
    Return<bool> blink(uint32_t onMs, uint32_t offMs) override;
    
private:
    int mBrightnessFile;
    int mTriggerFile;
    int mDelayOnFile;
    int mDelayOffFile;
    
    bool openSysfs();
    void closeSysfs();
    bool writeSysfs(int fd, const char* value);
};

}  // namespace implementation
}  // namespace V1_0
}  // namespace led
}  // namespace hardware
}  // namespace android

#endif  // ANDROID_HARDWARE_LED_V1_0_LED_H
```

**File:** `hardware/interfaces/led/1.0/default/Led.cpp`

```cpp
#include "Led.h"
#include <log/log.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

#define LED_SYSFS_BASE "/sys/class/leds/system_led"

namespace android {
namespace hardware {
namespace led {
namespace V1_0 {
namespace implementation {

Led::Led() : mBrightnessFile(-1), mTriggerFile(-1),
             mDelayOnFile(-1), mDelayOffFile(-1) {
    openSysfs();
}

Led::~Led() {
    closeSysfs();
}

bool Led::openSysfs() {
    mBrightnessFile = open(LED_SYSFS_BASE "/brightness", O_WRONLY);
    if (mBrightnessFile < 0) {
        ALOGE("Failed to open brightness file");
        return false;
    }
    
    mTriggerFile = open(LED_SYSFS_BASE "/trigger", O_WRONLY);
    mDelayOnFile = open(LED_SYSFS_BASE "/delay_on", O_WRONLY);
    mDelayOffFile = open(LED_SYSFS_BASE "/delay_off", O_WRONLY);
    
    return true;
}

void Led::closeSysfs() {
    if (mBrightnessFile >= 0) close(mBrightnessFile);
    if (mTriggerFile >= 0) close(mTriggerFile);
    if (mDelayOnFile >= 0) close(mDelayOnFile);
    if (mDelayOffFile >= 0) close(mDelayOffFile);
}

bool Led::writeSysfs(int fd, const char* value) {
    if (fd < 0) return false;
    
    ssize_t len = strlen(value);
    ssize_t ret = write(fd, value, len);
    
    return ret == len;
}

Return<bool> Led::setEnabled(bool on) {
    ALOGD("Led::setEnabled(%d)", on);
    
    char buf[16];
    snprintf(buf, sizeof(buf), "%d", on ? 255 : 0);
    
    return writeSysfs(mBrightnessFile, buf);
}

Return<bool> Led::setBrightness(uint8_t brightness) {
    ALOGD("Led::setBrightness(%u)", brightness);
    
    char buf[16];
    snprintf(buf, sizeof(buf), "%u", brightness);
    
    return writeSysfs(mBrightnessFile, buf);
}

Return<bool> Led::setColor(uint32_t color) {
    ALOGD("Led::setColor(0x%08x)", color);
    
    // For simple LED, just use brightness from color
    uint8_t r = (color >> 16) & 0xFF;
    uint8_t g = (color >> 8) & 0xFF;
    uint8_t b = color & 0xFF;
    
    // Average RGB for brightness
    uint8_t brightness = (r + g + b) / 3;
    
    return setBrightness(brightness);
}

Return<bool> Led::blink(uint32_t onMs, uint32_t offMs) {
    ALOGD("Led::blink(%u, %u)", onMs, offMs);
    
    // Set trigger to timer
    if (!writeSysfs(mTriggerFile, "timer")) {
        ALOGE("Failed to set timer trigger");
        return false;
    }
    
    // Set delays
    char buf[16];
    
    snprintf(buf, sizeof(buf), "%u", onMs);
    if (!writeSysfs(mDelayOnFile, buf)) {
        ALOGE("Failed to set delay_on");
        return false;
    }
    
    snprintf(buf, sizeof(buf), "%u", offMs);
    if (!writeSysfs(mDelayOffFile, buf)) {
        ALOGE("Failed to set delay_off");
        return false;
    }
    
    return true;
}

}  // namespace implementation
}  // namespace V1_0
}  // namespace led
}  // namespace hardware
}  // namespace android
```

### Step 3: Service Entry Point

**File:** `hardware/interfaces/led/1.0/default/service.cpp`

```cpp
#include <android/hardware/led/1.0/ILed.h>
#include <hidl/LegacySupport.h>
#include "Led.h"

using android::hardware::led::V1_0::ILed;
using android::hardware::led::V1_0::implementation::Led;
using android::hardware::configureRpcThreadpool;
using android::hardware::joinRpcThreadpool;
using android::sp;
using android::status_t;
using android::OK;

int main() {
    sp<ILed> service = new Led();
    
    configureRpcThreadpool(1, true);
    
    status_t status = service->registerAsService();
    if (status != OK) {
        ALOGE("Cannot register LED HAL service");
        return 1;
    }
    
    ALOGI("LED HAL service is ready");
    joinRpcThreadpool();
    
    return 1;
}
```

### Step 4: Build Configuration

**File:** `hardware/interfaces/led/1.0/Android.bp`

```go
hidl_interface {
    name: "android.hardware.led@1.0",
    root: "android.hardware",
    srcs: [
        "ILed.hal",
    ],
    interfaces: [
        "android.hidl.base@1.0",
    ],
    gen_java: false,
}
```

**File:** `hardware/interfaces/led/1.0/default/Android.bp`

```go
cc_binary {
    name: "android.hardware.led@1.0-service",
    relative_install_path: "hw",
    vendor: true,
    init_rc: ["android.hardware.led@1.0-service.rc"],
    srcs: [
        "service.cpp",
        "Led.cpp",
    ],
    shared_libs: [
        "libhidlbase",
        "liblog",
        "libutils",
        "android.hardware.led@1.0",
    ],
}
```

### Step 5: Framework Integration

**Java manager class:**

```java
// frameworks/base/core/java/android/hardware/LedManager.java

package android.hardware;

import android.annotation.SystemService;
import android.content.Context;
import android.os.RemoteException;
import android.util.Log;

@SystemService(Context.LED_SERVICE)
public class LedManager {
    private static final String TAG = "LedManager";
    
    private android.hardware.led.V1_0.ILed mService;
    
    public LedManager(Context context) {
        try {
            mService = android.hardware.led.V1_0.ILed.getService();
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to get LED HAL service", e);
        }
    }
    
    /**
     * Turn LED on or off
     */
    public boolean setEnabled(boolean on) {
        if (mService == null) return false;
        
        try {
            return mService.setEnabled(on);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to set LED enabled", e);
            return false;
        }
    }
    
    /**
     * Set LED brightness
     * @param brightness 0-255
     */
    public boolean setBrightness(int brightness) {
        if (mService == null) return false;
        
        try {
            return mService.setBrightness((byte) brightness);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to set LED brightness", e);
            return false;
        }
    }
    
    /**
     * Blink the LED
     */
    public boolean blink(int onMs, int offMs) {
        if (mService == null) return false;
        
        try {
            return mService.blink(onMs, offMs);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to blink LED", e);
            return false;
        }
    }
}
```

### Step 6: Usage from App

```java
// In an app
LedManager ledManager = getSystemService(LedManager.class);

// Turn on
ledManager.setEnabled(true);

// Set brightness
ledManager.setBrightness(128);

// Blink
ledManager.blink(500, 500);  // 500ms on, 500ms off
```

## Practical Example 2: Extending Existing HAL

Let's extend the Camera HAL with custom functionality.

### Step 1: Extend Interface (New Version)

**File:** `hardware/interfaces/camera/device/4.0/ICameraDevice.hal`

```hidl
package android.hardware.camera.device@4.0;

import @3.2::ICameraDevice;

/**
 * Extended camera device interface with custom features
 */
interface ICameraDevice extends @3.2::ICameraDevice {
    /**
     * Custom: Enable advanced night mode
     * @param enabled true to enable night mode
     * @return status OK if successful
     */
    setNightMode(bool enabled) generates (Status status);
    
    /**
     * Custom: Set HDR level
     * @param level 0-100
     * @return status OK if successful
     */
    setHdrLevel(uint8_t level) generates (Status status);
};
```

### Step 2: Implementation

**Update implementation to include new methods while maintaining backwards compatibility with older versions.**

This allows devices to support both the old and new interfaces.

## VINTF (Vendor Interface)

### Manifest and Compatibility Matrix

VINTF ensures framework and vendor compatibility:

**device.xml (vendor manifest):**
```xml
<manifest version="2.0" type="device">
    <hal format="hidl">
        <name>android.hardware.led</name>
        <transport>hwbinder</transport>
        <version>1.0</version>
        <interface>
            <name>ILed</name>
            <instance>default</instance>
        </interface>
    </hal>
</manifest>
```

**framework_compatibility_matrix.xml:**
```xml
<compatibility-matrix version="1.0" type="framework">
    <hal format="hidl" optional="false">
        <name>android.hardware.led</name>
        <version>1.0</version>
        <interface>
            <name>ILed</name>
            <instance>default</instance>
        </interface>
    </hal>
</compatibility-matrix>
```

## Key Takeaways

1. **HAL enables hardware abstraction**: Framework remains hardware-agnostic
2. **HIDL provides stability**: Versioned interfaces, binary compatibility
3. **AIDL is the future**: Simpler syntax, better performance
4. **Treble separates system/vendor**: Clear boundary, independent updates
5. **Binderized is preferred**: Better isolation, security
6. **VINTF enforces compatibility**: Ensures framework/vendor work together
7. **Custom HALs are powerful**: Add device-specific functionality

## Next Steps

In Chapter 10, we'll explore Power Management and Battery systems. You'll learn how Android manages power states, optimizes battery life, and how to implement custom power policies.

## Quick Reference

### HAL Commands

```bash
# List HAL services
adb shell lshal

# Check if HAL is running
adb shell ps -A | grep "hardware"

# Dump HAL service
adb shell lshal debug android.hardware.led@1.0::ILed/default

# Test HAL from command line (if supported)
adb shell service call led 1

# View VINTF manifest
adb shell cat /vendor/etc/vintf/manifest.xml

# Check VINTF compatibility
adb shell vintf check
```

### Build HAL

```bash
# Build specific HAL
m android.hardware.led@1.0-service

# Build all HALs
m android.hardware

# Update HIDL hash
hidl-gen -L hash -r android.hardware:hardware/interfaces \
    android.hardware.led@1.0 >> current.txt
```

### Common Locations

```
hardware/interfaces/          # AOSP HAL interfaces
vendor/[manufacturer]/        # Vendor implementations
/vendor/etc/vintf/            # VINTF manifests
/dev/hwbinder                 # HwBinder device node
```
