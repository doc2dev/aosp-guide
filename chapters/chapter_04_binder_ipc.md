# Chapter 4: Binder IPC

## Contents

- [Introduction](#introduction)
- [Why Binder? The IPC Problem](#why-binder-the-ipc-problem)
- [Binder Architecture Overview](#binder-architecture-overview)
- [Binder Communication Model](#binder-communication-model)
- [The Binder Kernel Driver](#the-binder-kernel-driver)
- [ServiceManager: The Name Service](#servicemanager-the-name-service)
- [AIDL: Interface Definition Language](#aidl-interface-definition-language)
- [Creating a Custom System Service](#creating-a-custom-system-service)
- [Binder Security and Permissions](#binder-security-and-permissions)
- [Death Recipients: Handling Client/Server Death](#death-recipients-handling-clientserver-death)
- [Native Binder (C++)](#native-binder-c)
- [Debugging Binder](#debugging-binder)
- [Performance Considerations](#performance-considerations)
- [HIDL: HAL Interface Definition Language](#hidl-hal-interface-definition-language)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)
- [Quick Reference](#quick-reference)

## Introduction

Binder is Android's primary Inter-Process Communication (IPC) mechanism. It's the foundation that enables apps to communicate with system services, allows system services to communicate with each other, and provides the infrastructure for Android's component model (Activities, Services, ContentProviders, BroadcastReceivers).

Understanding Binder is essential for AOSP development because:
- Almost every system service uses Binder for communication
- Creating custom system services requires understanding Binder
- Debugging system-level issues often involves tracing Binder transactions
- Performance tuning requires understanding Binder overhead

This chapter provides a deep dive into Binder's architecture, the AIDL interface definition language, how to create custom system services, and practical examples of extending Android's IPC infrastructure.

## Why Binder? The IPC Problem

Before diving into Binder's details, let's understand why Android needs a specialized IPC mechanism.

### Traditional Linux IPC Limitations

Linux provides several IPC mechanisms:
- **Pipes/FIFOs**: Unidirectional byte streams
- **Sockets**: Network-oriented, requires protocol implementation
- **Message Queues**: Limited size, no RPC semantics
- **Shared Memory**: Requires complex synchronization
- **Signals**: Primitive, no data transfer

None of these provide what Android needs:
1. **Object-oriented RPC**: Method calls across processes
2. **Synchronous and asynchronous calls**: Both blocking and non-blocking
3. **Security**: Per-process credentials and SELinux integration
4. **Reference counting**: Automatic object lifecycle management
5. **Death notifications**: Know when remote process dies
6. **Thread pool management**: Automatic thread handling for service requests

### Binder's Advantages

Binder was designed specifically for Android to provide:

**Performance:**
- Single copy between processes (vs. two copies for traditional IPC)
- Efficient for small messages (most Android IPC is small)
- Thread pool management reduces context switches

**Security:**
- Automatic credential passing (UID, PID, SELinux context)
- Per-transaction permission checking
- Isolated per-process communication channels

**Object Model:**
- Interface-based programming (like COM or CORBA)
- Reference counting for automatic cleanup
- Death recipients for handling service crashes

**Integration:**
- Deep integration with Android's runtime (ART)
- Built into the framework and kernel
- Seamless Java/Native interoperability

## Binder Architecture Overview

Binder operates across multiple layers:

```
┌─────────────────────────────────────────┐
│         Application / Service           │
│  (Java or Native - High Level APIs)    │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│         Binder Framework Layer          │
│    (libbinder, AIDL generated code)    │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────┴───────────────────────┐
│          Kernel Binder Driver           │
│        (/dev/binder, /dev/hwbinder)     │
└─────────────────────────────────────────┘
```

### Key Components

**1. Kernel Driver:**
- Character device: `/dev/binder`
- Handles transaction routing
- Manages memory mapping
- Enforces security policies

**2. Native Library (libbinder):**
- Location: `frameworks/native/libs/binder/`
- C++ abstractions over kernel interface
- Provides `IBinder`, `BBinder`, `BpBinder` classes
- Thread pool management

**3. AIDL (Android Interface Definition Language):**
- Tool: `system/tools/aidl/`
- Defines service interfaces
- Generates proxy/stub code
- Supports both Java and C++

**4. ServiceManager:**
- Central registry for system services
- Maps service names to Binder references
- Location: `frameworks/native/cmds/servicemanager/`

## Binder Communication Model

### Proxies and Stubs

Binder uses a proxy-stub pattern:

```
┌────────────────┐         ┌────────────────┐
│  Client        │         │  Server        │
│  Process       │         │  Process       │
│                │         │                │
│  ┌──────────┐ │         │ ┌──────────┐   │
│  │  Proxy   │ │ Binder  │ │  Stub    │   │
│  │ (BpXxx)  │◄├─────────┤►│ (BnXxx)  │   │
│  └────┬─────┘ │  Driver │ └────┬─────┘   │
│       │       │         │      │         │
│       ▼       │         │      ▼         │
│   App Code    │         │  Service Code  │
└────────────────┘         └────────────────┘
```

**Proxy (BpXxx - Binder Proxy):**
- Lives in client process
- Implements interface methods
- Marshals arguments into Parcel
- Sends transaction to kernel
- Unmarshals return values

**Stub (BnXxx - Binder Native):**
- Lives in server process
- Receives transaction from kernel
- Unmarshals arguments
- Calls actual implementation
- Marshals return values

### Transaction Flow

A typical Binder transaction:

1. **Client calls proxy method**
   ```cpp
   service->doSomething(arg1, arg2);
   ```

2. **Proxy marshals data**
   ```cpp
   Parcel data, reply;
   data.writeInt32(arg1);
   data.writeString16(arg2);
   ```

3. **Proxy sends transaction**
   ```cpp
   remote()->transact(TRANSACTION_doSomething, data, &reply);
   ```

4. **Kernel routes to server**
   - Copies data from client to server
   - Wakes up server thread

5. **Stub receives transaction**
   ```cpp
   onTransact(code, data, reply, flags);
   ```

6. **Stub unmarshals and calls implementation**
   ```cpp
   int32_t arg1 = data.readInt32();
   String16 arg2 = data.readString16();
   result = doSomething(arg1, arg2);
   ```

7. **Stub marshals reply**
   ```cpp
   reply->writeInt32(result);
   ```

8. **Kernel returns to client**
   - Copies reply data
   - Wakes up client thread

9. **Proxy unmarshals reply**
   ```cpp
   int32_t result = reply.readInt32();
   return result;
   ```

## The Binder Kernel Driver

### Device Files

Binder driver exposes multiple device nodes:

- `/dev/binder`: Framework IPC (apps ↔ system services)
- `/dev/hwbinder`: HAL IPC (framework ↔ vendor HALs)
- `/dev/vndbinder`: Vendor IPC (vendor ↔ vendor)

This separation is part of Project Treble to isolate vendor and system code.

### Driver Operations

The driver is implemented as a character device with `ioctl` interface:

**Location:** `drivers/android/binder.c` (in kernel)

**Key ioctl commands:**

```c
BINDER_WRITE_READ       // Main operation: send/receive data
BINDER_SET_MAX_THREADS  // Configure thread pool
BINDER_VERSION          // Get driver version
BINDER_SET_CONTEXT_MGR  // Become context manager (ServiceManager)
```

### Memory Management

Binder uses a clever memory mapping scheme:

1. **Kernel allocates buffer** in server process
2. **Client writes to its own buffer**
3. **Kernel copies to server's mapped buffer**
4. **Server reads from its buffer** (no copy needed)

Result: Only one copy operation (vs. two for socket-based IPC)

**Buffer limits:**
- Per-process: 1MB (default, configurable)
- Per-transaction: ~1MB
- Small transactions are most efficient

### Binder Threads

Each process using Binder maintains a thread pool:

**Main thread:** The thread that calls `ProcessState::self()->startThreadPool()`

**Spawned threads:** Driver signals when more threads are needed

**Thread states:**
- Free: Ready to handle requests
- Transaction: Processing a request
- Waiting: Blocked on synchronous call

Code location: `frameworks/native/libs/binder/IPCThreadState.cpp`

## ServiceManager: The Name Service

ServiceManager is Binder's central registry. All system services register with it.

### ServiceManager Responsibilities

1. **Service registration**: Services call `addService(name, binder)`
2. **Service lookup**: Clients call `getService(name)` 
3. **Service listing**: Enumerate all registered services
4. **Permission checking**: Enforce who can register services

### ServiceManager as Context Manager

ServiceManager is special—it's assigned Binder handle 0, known to all processes:

```cpp
// Get ServiceManager reference (always handle 0)
sp<IServiceManager> sm = defaultServiceManager();

// Register a service
sm->addService(String16("my_service"), myBinder);

// Look up a service
sp<IBinder> binder = sm->getService(String16("activity"));
```

### ServiceManager Implementation

**Location:** `frameworks/native/cmds/servicemanager/`

**Key files:**
- `main.cpp`: Entry point
- `ServiceManager.cpp`: Service registry implementation
- `Access.cpp`: Permission checking

**Starting ServiceManager:**

Defined in init.rc:
```rc
service servicemanager /system/bin/servicemanager
    class core animation
    user system
    group system readproc
    critical
    onrestart restart healthd
    onrestart restart zygote
    onrestart restart audioserver
    onrestart restart media
    onrestart restart surfaceflinger
    onrestart restart inputflinger
    onrestart restart drm
    onrestart restart cameraserver
```

Note: ServiceManager is `critical`—if it dies, the system reboots.

### Viewing Registered Services

```bash
# List all services
adb shell service list

# Output example:
# 0    phone: [com.android.internal.telephony.ITelephony]
# 1    isms: [com.android.internal.telephony.ISms]
# 2    activity: [android.app.IActivityManager]
# ...

# Check specific service
adb shell service check activity

# Call service method (limited functionality)
adb shell service call phone 1 i32 0
```

## AIDL: Interface Definition Language

AIDL is how you define Binder interfaces. The AIDL compiler generates proxy and stub code.

### AIDL Syntax

**Basic structure:**

```java
// IMyService.aidl
package com.example;

interface IMyService {
    int add(int a, int b);
    String getText();
    void setText(String text);
}
```

**Supported types:**

**Primitives:**
- `int`, `long`, `short`, `byte`, `char`
- `boolean`
- `float`, `double`

**Strings:**
- `String`
- `CharSequence`

**Lists and Maps:**
- `List` (becomes `ArrayList`)
- `Map` (becomes `HashMap`)

**Parcelable objects:**
- Custom classes implementing `Parcelable`
- Other AIDL-defined interfaces

**File descriptors:**
- `ParcelFileDescriptor`

### Directionality

Parameters can have directionality modifiers:

```java
interface IMyService {
    void readData(out byte[] data);      // Server fills data
    void writeData(in byte[] data);      // Client sends data
    void updateData(inout byte[] data);  // Both directions
}
```

- `in`: Parameter is input only (default for primitives)
- `out`: Parameter is output only
- `inout`: Parameter is both input and output

### AIDL Compilation

AIDL compiler generates Java or C++ code:

**Java:**
```bash
# Compiler location
aidl

# Generates:
# - IMyService.java (interface)
# - Stub (abstract class)
# - Proxy (private class inside Stub)
```

**Build system integration:**

In `Android.bp`:
```go
java_library {
    name: "my-service-interface",
    srcs: ["src/**/*.java", "src/**/*.aidl"],
    aidl: {
        local_include_dirs: ["src"],
    },
}
```

### Generated Code Structure (Java)

For `IMyService.aidl`, the compiler generates:

```java
public interface IMyService extends android.os.IInterface {
    
    // Stub: Server-side implementation
    public static abstract class Stub extends android.os.Binder 
            implements IMyService {
        
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }
        
        public static IMyService asInterface(android.os.IBinder obj) {
            // Returns proxy or local implementation
        }
        
        @Override
        public boolean onTransact(int code, Parcel data, 
                                  Parcel reply, int flags) {
            // Dispatch to implementation methods
        }
        
        // Proxy: Client-side stub
        private static class Proxy implements IMyService {
            private android.os.IBinder mRemote;
            
            @Override
            public int add(int a, int b) throws RemoteException {
                // Marshal, transact, unmarshal
            }
        }
    }
    
    // Interface methods
    public int add(int a, int b) throws android.os.RemoteException;
    public String getText() throws android.os.RemoteException;
    public void setText(String text) throws android.os.RemoteException;
}
```

## Creating a Custom System Service

Let's create a complete custom system service using Binder.

### Example: ColorManager Service

We'll create a service that manages a system-wide color scheme.

### Step 1: Define the AIDL Interface

**File:** `frameworks/base/core/java/android/app/IColorManager.aidl`

```java
package android.app;

/** {@hide} */
interface IColorManager {
    /**
     * Set the current system color scheme.
     * @param color The color in ARGB format
     */
    void setSystemColor(int color);
    
    /**
     * Get the current system color scheme.
     * @return The color in ARGB format
     */
    int getSystemColor();
    
    /**
     * Register a callback for color changes.
     * @param callback The callback to register
     */
    void registerCallback(IColorChangeCallback callback);
    
    /**
     * Unregister a callback.
     * @param callback The callback to unregister
     */
    void unregisterCallback(IColorChangeCallback callback);
}
```

**File:** `frameworks/base/core/java/android/app/IColorChangeCallback.aidl`

```java
package android.app;

/** {@hide} */
oneway interface IColorChangeCallback {
    void onColorChanged(int newColor);
}
```

Note: `oneway` means the call is asynchronous (doesn't block waiting for return).

### Step 2: Implement the Service

**File:** `frameworks/base/services/core/java/com/android/server/ColorManagerService.java`

```java
package com.android.server;

import android.app.IColorManager;
import android.app.IColorChangeCallback;
import android.content.Context;
import android.os.Binder;
import android.os.RemoteCallbackList;
import android.os.RemoteException;
import android.util.Slog;

public class ColorManagerService extends IColorManager.Stub {
    private static final String TAG = "ColorManagerService";
    
    private final Context mContext;
    private final Object mLock = new Object();
    
    // Current color value
    private int mCurrentColor = 0xFF000000; // Default: black
    
    // List of registered callbacks
    private final RemoteCallbackList<IColorChangeCallback> mCallbacks =
            new RemoteCallbackList<>();
    
    public ColorManagerService(Context context) {
        mContext = context;
        Slog.i(TAG, "ColorManagerService started");
    }
    
    @Override
    public void setSystemColor(int color) {
        // Check permission
        mContext.enforceCallingOrSelfPermission(
                android.Manifest.permission.CHANGE_CONFIGURATION,
                "setSystemColor");
        
        Slog.i(TAG, "Setting system color to: " + Integer.toHexString(color));
        
        synchronized (mLock) {
            if (mCurrentColor == color) {
                return; // No change
            }
            
            mCurrentColor = color;
            
            // Persist the value
            android.provider.Settings.System.putInt(
                    mContext.getContentResolver(),
                    "system_color",
                    color);
        }
        
        // Notify callbacks
        notifyColorChanged(color);
    }
    
    @Override
    public int getSystemColor() {
        synchronized (mLock) {
            Slog.d(TAG, "Getting system color: " + Integer.toHexString(mCurrentColor));
            return mCurrentColor;
        }
    }
    
    @Override
    public void registerCallback(IColorChangeCallback callback) {
        if (callback == null) {
            throw new IllegalArgumentException("callback must not be null");
        }
        
        Slog.i(TAG, "Registering callback from uid: " + Binder.getCallingUid());
        
        synchronized (mLock) {
            mCallbacks.register(callback);
            
            // Send current color to new callback
            try {
                callback.onColorChanged(mCurrentColor);
            } catch (RemoteException e) {
                Slog.w(TAG, "Failed to send initial color", e);
            }
        }
    }
    
    @Override
    public void unregisterCallback(IColorChangeCallback callback) {
        if (callback == null) {
            return;
        }
        
        Slog.i(TAG, "Unregistering callback from uid: " + Binder.getCallingUid());
        
        synchronized (mLock) {
            mCallbacks.unregister(callback);
        }
    }
    
    private void notifyColorChanged(int newColor) {
        // Begin broadcast
        int i = mCallbacks.beginBroadcast();
        
        while (i > 0) {
            i--;
            try {
                mCallbacks.getBroadcastItem(i).onColorChanged(newColor);
            } catch (RemoteException e) {
                // The RemoteCallbackList will take care of removing
                // the dead object for us.
                Slog.w(TAG, "Failed to notify callback", e);
            }
        }
        
        // End broadcast
        mCallbacks.finishBroadcast();
    }
    
    /**
     * Called when the service should shut down.
     */
    public void shutdown() {
        Slog.i(TAG, "Shutting down ColorManagerService");
        mCallbacks.kill();
    }
}
```

### Step 3: Register Service with SystemServer

**File:** `frameworks/base/services/java/com/android/server/SystemServer.java`

```java
// In startOtherServices() method

// Start ColorManagerService
traceBeginAndSlog("StartColorManagerService");
mSystemServiceManager.startService(ColorManagerService.class);
ServiceManager.addService("color", 
        new ColorManagerService(mSystemContext));
traceEnd();
```

### Step 4: Create Client API

**File:** `frameworks/base/core/java/android/app/ColorManager.java`

```java
package android.app;

import android.annotation.SystemService;
import android.content.Context;
import android.os.IBinder;
import android.os.RemoteException;
import android.os.ServiceManager;
import android.util.Log;

/**
 * Manager for system-wide color scheme.
 * 
 * @hide
 */
@SystemService(Context.COLOR_SERVICE)
public class ColorManager {
    private static final String TAG = "ColorManager";
    
    private final Context mContext;
    private final IColorManager mService;
    
    /**
     * Callback interface for color changes.
     */
    public interface ColorChangeListener {
        void onColorChanged(int newColor);
    }
    
    /**
     * @hide
     */
    public ColorManager(Context context, IColorManager service) {
        mContext = context;
        mService = service;
    }
    
    /**
     * Set the system color scheme.
     * 
     * Requires {@link android.Manifest.permission#CHANGE_CONFIGURATION}
     * 
     * @param color The color in ARGB format
     */
    public void setSystemColor(int color) {
        try {
            mService.setSystemColor(color);
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    
    /**
     * Get the current system color.
     * 
     * @return The color in ARGB format
     */
    public int getSystemColor() {
        try {
            return mService.getSystemColor();
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    
    /**
     * Register a listener for color changes.
     * 
     * @param listener The listener to register
     */
    public void registerColorChangeListener(ColorChangeListener listener) {
        if (listener == null) {
            throw new IllegalArgumentException("listener must not be null");
        }
        
        try {
            mService.registerCallback(new IColorChangeCallback.Stub() {
                @Override
                public void onColorChanged(int newColor) {
                    listener.onColorChanged(newColor);
                }
            });
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
    
    /**
     * Unregister a listener.
     * 
     * @param listener The listener to unregister
     */
    public void unregisterColorChangeListener(ColorChangeListener listener) {
        // Note: In production code, you'd need to track the callback
        // mapping to properly unregister
        Log.w(TAG, "unregisterColorChangeListener not fully implemented");
    }
}
```

### Step 5: Register Manager in Context

**File:** `frameworks/base/core/java/android/app/SystemServiceRegistry.java`

```java
// In static block

registerService(Context.COLOR_SERVICE, ColorManager.class,
        new CachedServiceFetcher<ColorManager>() {
    @Override
    public ColorManager createService(ContextImpl ctx) {
        IBinder b = ServiceManager.getService("color");
        IColorManager service = IColorManager.Stub.asInterface(b);
        return new ColorManager(ctx, service);
    }
});
```

**File:** `frameworks/base/core/java/android/content/Context.java`

```java
// Add constant
public static final String COLOR_SERVICE = "color";
```

### Step 6: Build and Deploy

```bash
# Build framework
m framework services

# Push to device
adb root
adb remount
adb push $ANDROID_PRODUCT_OUT/system/framework/framework.jar /system/framework/
adb push $ANDROID_PRODUCT_OUT/system/framework/services.jar /system/framework/

# Restart system
adb reboot
```

### Step 7: Use the Service

**In an app:**

```java
// Get the ColorManager
ColorManager colorManager = (ColorManager) getSystemService(Context.COLOR_SERVICE);

// Set color
colorManager.setSystemColor(0xFFFF0000); // Red

// Get color
int color = colorManager.getSystemColor();

// Listen for changes
colorManager.registerColorChangeListener(newColor -> {
    Log.i(TAG, "Color changed to: " + Integer.toHexString(newColor));
    // Update UI
});
```

**From command line:**

```bash
# List services (verify it's registered)
adb shell service list | grep color

# Call methods directly (requires understanding transaction codes)
adb shell service call color 1 i32 0xFFFF0000  # setSystemColor
adb shell service call color 2                  # getSystemColor
```

## Binder Security and Permissions

### Caller Identity

Binder automatically provides caller credentials:

```java
// Get calling UID
int uid = Binder.getCallingUid();

// Get calling PID
int pid = Binder.getCallingPid();

// Get calling user ID
int userId = UserHandle.getUserId(uid);

// Check if caller is system
boolean isSystem = uid == Process.SYSTEM_UID;
```

### Permission Checking

```java
// In service implementation
@Override
public void privilegedOperation() {
    mContext.enforceCallingPermission(
            android.Manifest.permission.SOME_PERMISSION,
            "Need SOME_PERMISSION to call privilegedOperation");
    
    // Implementation...
}

// Or check without throwing
if (mContext.checkCallingPermission(
        android.Manifest.permission.SOME_PERMISSION) 
        != PackageManager.PERMISSION_GRANTED) {
    throw new SecurityException("Permission denied");
}
```

### SELinux Integration

Binder transactions are subject to SELinux policy:

```
# Example policy allowing ActivityManagerService to call ColorManagerService
allow system_server color_service:binder call;

# Allow apps to find the service
allow appdomain color_service:service_manager find;

# Allow service to add itself to ServiceManager
allow color_service servicemanager:binder call;
allow color_service service_manager_type:service_manager add;
```

### Identity Token Management

Long-running operations should clear caller identity:

```java
@Override
public void longOperation() {
    // Clear and save caller identity
    final long token = Binder.clearCallingIdentity();
    
    try {
        // Long-running work
        // This runs with service's identity, not caller's
        doLongWork();
    } finally {
        // Restore caller identity
        Binder.restoreCallingIdentity(token);
    }
}
```

## Death Recipients: Handling Client/Server Death

### What are Death Recipients?

Death recipients allow you to be notified when a remote process dies. This is crucial for cleanup.

### Registering a Death Recipient

**In Java:**

```java
IBinder.DeathRecipient deathRecipient = new IBinder.DeathRecipient() {
    @Override
    public void binderDied() {
        Log.w(TAG, "Service died!");
        // Cleanup and maybe reconnect
        mService = null;
        reconnect();
    }
};

// Link to death
service.asBinder().linkToDeath(deathRecipient, 0);

// Unlink when done
service.asBinder().unlinkToDeath(deathRecipient, 0);
```

**In service (tracking clients):**

```java
private class ClientDeathRecipient implements IBinder.DeathRecipient {
    private final IMyCallback mCallback;
    
    ClientDeathRecipient(IMyCallback callback) {
        mCallback = callback;
    }
    
    @Override
    public void binderDied() {
        Slog.w(TAG, "Client died, cleaning up");
        synchronized (mLock) {
            mCallbacks.unregister(mCallback);
        }
    }
}

@Override
public void registerCallback(IMyCallback callback) {
    ClientDeathRecipient recipient = new ClientDeathRecipient(callback);
    try {
        callback.asBinder().linkToDeath(recipient, 0);
    } catch (RemoteException e) {
        Slog.e(TAG, "Client already dead", e);
        return;
    }
    
    mCallbacks.register(callback);
}
```

### RemoteCallbackList

Framework provides `RemoteCallbackList` which handles death recipients automatically:

```java
private final RemoteCallbackList<IMyCallback> mCallbacks = 
        new RemoteCallbackList<>();

// Register - automatic death tracking
mCallbacks.register(callback);

// Broadcast to all
int i = mCallbacks.beginBroadcast();
while (i > 0) {
    i--;
    try {
        mCallbacks.getBroadcastItem(i).onEvent();
    } catch (RemoteException e) {
        // Client dead, automatically removed
    }
}
mCallbacks.finishBroadcast();
```

## Native Binder (C++)

While most AOSP development uses Java Binder, understanding native Binder is valuable for HALs and performance-critical services.

### Native Binder Classes

**Key classes in `frameworks/native/libs/binder/`:**

- `IBinder`: Base interface for all Binder objects
- `BBinder`: Base for objects that implement interfaces (server-side)
- `BpBinder`: Proxy to remote object (client-side)
- `Parcel`: Container for serialized data
- `ProcessState`: Per-process Binder state
- `IPCThreadState`: Per-thread Binder state

### Native Interface Definition

**File:** `IMyNativeService.h`

```cpp
#ifndef ANDROID_IMYNATIVESERVICE_H
#define ANDROID_IMYNATIVESERVICE_H

#include <binder/IInterface.h>
#include <binder/Parcel.h>

namespace android {

class IMyNativeService : public IInterface {
public:
    DECLARE_META_INTERFACE(MyNativeService)
    
    virtual status_t doSomething(int32_t value) = 0;
    virtual status_t getData(String16* outData) = 0;
};

// Transaction codes
enum {
    DO_SOMETHING = IBinder::FIRST_CALL_TRANSACTION,
    GET_DATA,
};

}; // namespace android

#endif // ANDROID_IMYNATIVESERVICE_H
```

### Native Proxy Implementation

**File:** `IMyNativeService.cpp`

```cpp
#include "IMyNativeService.h"

namespace android {

// Implement descriptor
IMPLEMENT_META_INTERFACE(MyNativeService, "android.os.IMyNativeService")

// Proxy (client-side)
class BpMyNativeService : public BpInterface<IMyNativeService> {
public:
    explicit BpMyNativeService(const sp<IBinder>& impl)
        : BpInterface<IMyNativeService>(impl) {
    }
    
    virtual status_t doSomething(int32_t value) {
        Parcel data, reply;
        data.writeInterfaceToken(IMyNativeService::getInterfaceDescriptor());
        data.writeInt32(value);
        
        status_t status = remote()->transact(DO_SOMETHING, data, &reply);
        if (status != OK) {
            return status;
        }
        
        return reply.readInt32();
    }
    
    virtual status_t getData(String16* outData) {
        Parcel data, reply;
        data.writeInterfaceToken(IMyNativeService::getInterfaceDescriptor());
        
        status_t status = remote()->transact(GET_DATA, data, &reply);
        if (status != OK) {
            return status;
        }
        
        *outData = reply.readString16();
        return OK;
    }
};

IMPLEMENT_META_INTERFACE(MyNativeService, "android.os.IMyNativeService")

// Stub (server-side)
status_t BnMyNativeService::onTransact(
        uint32_t code, const Parcel& data, Parcel* reply, uint32_t flags) {
    
    switch (code) {
        case DO_SOMETHING: {
            CHECK_INTERFACE(IMyNativeService, data, reply);
            int32_t value = data.readInt32();
            status_t result = doSomething(value);
            reply->writeInt32(result);
            return OK;
        }
        
        case GET_DATA: {
            CHECK_INTERFACE(IMyNativeService, data, reply);
            String16 outData;
            status_t result = getData(&outData);
            reply->writeString16(outData);
            return result;
        }
        
        default:
            return BBinder::onTransact(code, data, reply, flags);
    }
}

}; // namespace android
```

### Native Service Implementation

**File:** `MyNativeService.h`

```cpp
#ifndef ANDROID_MYNATIVESERVICE_H
#define ANDROID_MYNATIVESERVICE_H

#include "IMyNativeService.h"
#include <binder/BinderService.h>

namespace android {

class MyNativeService : 
        public BinderService<MyNativeService>,
        public BnMyNativeService {
public:
    static const char* getServiceName() { return "mynative"; }
    
    MyNativeService();
    virtual ~MyNativeService();
    
    // IMyNativeService implementation
    virtual status_t doSomething(int32_t value) override;
    virtual status_t getData(String16* outData) override;
    
private:
    int32_t mData;
};

}; // namespace android

#endif // ANDROID_MYNATIVESERVICE_H
```

**File:** `MyNativeService.cpp`

```cpp
#include "MyNativeService.h"
#include <binder/IPCThreadState.h>
#include <binder/IServiceManager.h>
#include <utils/Log.h>

namespace android {

MyNativeService::MyNativeService() : mData(0) {
    ALOGI("MyNativeService created");
}

MyNativeService::~MyNativeService() {
    ALOGI("MyNativeService destroyed");
}

status_t MyNativeService::doSomething(int32_t value) {
    ALOGI("doSomething called with value: %d", value);
    
    // Check caller permissions
    uid_t uid = IPCThreadState::self()->getCallingUid();
    ALOGI("Called by UID: %d", uid);
    
    mData = value;
    return OK;
}

status_t MyNativeService::getData(String16* outData) {
    ALOGI("getData called");
    
    *outData = String16("Data value: ") + String16(std::to_string(mData).c_str());
    return OK;
}

}; // namespace android
```

### Native Service Main

**File:** `main.cpp`

```cpp
#include "MyNativeService.h"
#include <binder/ProcessState.h>
#include <binder/IPCThreadState.h>
#include <utils/Log.h>

using namespace android;

int main(int argc, char** argv) {
    // Set up Binder
    sp<ProcessState> proc(ProcessState::self());
    sp<IServiceManager> sm = defaultServiceManager();
    
    // Publish service
    MyNativeService::publishService();
    
    ALOGI("MyNativeService is starting");
    
    // Start thread pool
    ProcessState::self()->startThreadPool();
    
    // Join thread pool (block forever)
    IPCThreadState::self()->joinThreadPool();
    
    return 0;
}
```

### Android.bp for Native Service

```go
cc_binary {
    name: "mynativeservice",
    srcs: [
        "main.cpp",
        "MyNativeService.cpp",
        "IMyNativeService.cpp",
    ],
    shared_libs: [
        "libutils",
        "liblog",
        "libbinder",
    ],
    init_rc: ["mynativeservice.rc"],
}
```

**Init script:** `mynativeservice.rc`

```rc
service mynativeservice /system/bin/mynativeservice
    class main
    user system
    group system
```

## Debugging Binder

### Viewing Binder State

```bash
# View Binder stats
adb shell cat /sys/kernel/debug/binder/stats

# View process-specific state
adb shell cat /sys/kernel/debug/binder/proc/<pid>

# View transaction log
adb shell cat /sys/kernel/debug/binder/transaction_log

# View failed transactions
adb shell cat /sys/kernel/debug/binder/failed_transaction_log
```

### Binder Transaction Tracing

Enable in systrace:

```bash
python systrace.py binder_driver -t 10 -o trace.html
```

Or use perfetto:

```bash
adb shell perfetto -o /data/misc/perfetto-traces/trace \
    -t 20s binder_driver binder_lock
```

### Common Binder Errors

**TransactionTooLargeException:**
```
android.os.TransactionTooLargeException: data parcel size 1048576 bytes
```
Solution: Reduce data size or use shared memory (ashmem/file descriptor)

**DeadObjectException:**
```
android.os.DeadObjectException
```
Solution: Implement death recipients and reconnection logic

**SecurityException:**
```
java.lang.SecurityException: Permission denied
```
Solution: Check permissions in manifest and enforce in service

### Binder Leaks

Check for Binder leaks:

```bash
# Dump service
adb shell dumpsys activity services <package>

# Look for:
# - Unreleased references
# - Growing callback lists
# - Services not shutting down
```

## Performance Considerations

### Minimize Transaction Size

```java
// Bad: Sending large data
interface IBadService {
    void sendData(byte[] largeArray);  // Can exceed Binder limit
}

// Good: Use file descriptor
interface IGoodService {
    void sendData(ParcelFileDescriptor fd);
}

// Or shared memory
interface IBetterService {
    void sendData(in MemoryFile memFile);
}
```

### Use Oneway for Fire-and-Forget

```java
// Oneway: Returns immediately, no reply
oneway interface ICallback {
    void onEvent(int eventType);
}

// Regular: Blocks until server processes
interface IService {
    void processRequest();  // Blocks until complete
}
```

### Batch Transactions

```java
// Bad: Multiple transactions
for (int i = 0; i < 1000; i++) {
    service.addItem(i);  // 1000 Binder transactions
}

// Good: Single transaction
service.addItems(items);  // 1 Binder transaction
```

### Avoid Synchronous Calls from Main Thread

```java
// Bad: Blocks UI thread
int result = service.longOperation();

// Good: Async
service.longOperationAsync(new ICallback.Stub() {
    public void onResult(int result) {
        // Handle result on binder thread
        handler.post(() -> updateUI(result));
    }
});
```

## HIDL: HAL Interface Definition Language

HIDL is similar to AIDL but designed for HAL interfaces (framework ↔ vendor).

### Key Differences from AIDL

- Uses `/dev/hwbinder` instead of `/dev/binder`
- Versioned interfaces for stability
- C++ focused (though Java bindings exist)
- Used for Treble compatibility

### Example HIDL Interface

```cpp
// IMyHal.hal
package android.hardware.myhal@1.0;

interface IMyHal {
    /**
     * Initialize the HAL
     */
    init() generates (Result result);
    
    /**
     * Perform operation
     */
    doOperation(int32 value) generates (int32 result);
    
    /**
     * Register callback
     */
    registerCallback(IMyHalCallback callback) generates (Result result);
};
```

HIDL is beyond the scope of this chapter but follows similar patterns to AIDL.

## Key Takeaways

1. **Binder is Android's IPC backbone**: Nearly all system communication uses Binder
2. **Proxy-Stub pattern**: Clean separation between client and server
3. **ServiceManager is central**: All system services register here
4. **AIDL generates code**: Define interfaces, compiler creates proxy/stub
5. **Security is built-in**: Caller identity and permissions automatically enforced
6. **Death recipients handle failures**: Critical for robust services
7. **Performance matters**: Minimize transaction size and frequency
8. **Native and Java coexist**: Choose the right tool for your use case

## Next Steps

In Chapter 5, we'll explore PackageManager and the application lifecycle. You'll learn how Android installs, manages, and verifies applications, and how to customize the package management system.

## Quick Reference

### Java Binder Essentials

```java
// Get service
IBinder binder = ServiceManager.getService("service_name");
IMyService service = IMyService.Stub.asInterface(binder);

// Call method
service.doSomething();

// Implement service
public class MyService extends IMyService.Stub {
    @Override
    public void doSomething() {
        // Check permissions
        mContext.enforceCallingPermission(...);
        
        // Implementation
    }
}

// Register service
ServiceManager.addService("my_service", new MyService());
```

### Debugging Commands

```bash
# List services
adb shell service list

# Call service
adb shell service call <service> <code> [args]

# View Binder state
adb shell cat /sys/kernel/debug/binder/stats

# Trace Binder
adb shell atrace --async_start binder
# ... do operations ...
adb shell atrace --async_stop > trace.txt
```

### Common Patterns

```java
// Caller identity
int uid = Binder.getCallingUid();
int pid = Binder.getCallingPid();

// Clear identity for long operations
long token = Binder.clearCallingIdentity();
try {
    // Long operation
} finally {
    Binder.restoreCallingIdentity(token);
}

// Death recipient
service.asBinder().linkToDeath(recipient, 0);

// Callback list
RemoteCallbackList<ICallback> callbacks = new RemoteCallbackList<>();
callbacks.register(callback);
```
