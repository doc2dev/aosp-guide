# Chapter 10: Power Management & Battery

## Contents

- [Introduction](#introduction)
- [Power Management Architecture](#power-management-architecture)
- [PowerManagerService](#powermanagerservice)
- [Wakelock System](#wakelock-system)
- [Battery Monitoring](#battery-monitoring)
- [Doze Mode and App Standby](#doze-mode-and-app-standby)
- [Display Power Management](#display-power-management)
- [Practical Example 1: Custom Power Profile](#practical-example-1-custom-power-profile)
- [Practical Example 2: Wakelock Monitor](#practical-example-2-wakelock-monitor)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)
- [Quick Reference](#quick-reference)

## Introduction

Power management is critical for mobile devices. Android's power management system controls device power states, manages wakelocks, optimizes battery consumption, and provides battery statistics. Understanding this system is essential for:

- Optimizing device battery life
- Implementing custom power policies
- Debugging battery drain issues
- Managing device sleep states
- Controlling CPU and display power

This chapter explores Android's power management architecture, battery monitoring, wakelock system, doze mode, and provides practical examples of implementing custom power policies.

## Power Management Architecture

### High-Level Overview

```
┌────────────────────────────────────────────────┐
│         Application Layer                       │
│  ┌──────────────────────────────────────────┐  │
│  │  PowerManager API                        │  │
│  │  (Wakelocks, screen control)             │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │ Binder
┌───────────────────▼─────────────────────────────┐
│         System Server                           │
│  ┌──────────────────────────────────────────┐  │
│  │  PowerManagerService                     │  │
│  │  - Wakelock management                   │  │
│  │  - Display power                         │  │
│  │  - Power hints                           │  │
│  └────────┬─────────────────────────────────┘  │
│           │                                     │
│  ┌────────▼─────────────────────────────────┐  │
│  │  BatteryService                          │  │
│  │  - Battery stats                         │  │
│  │  - Charging state                        │  │
│  └────────┬─────────────────────────────────┘  │
│           │                                     │
│  ┌────────▼─────────────────────────────────┐  │
│  │  BatteryStatsService                     │  │
│  │  - Usage tracking                        │  │
│  │  - History recording                     │  │
│  └──────────────────────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │ HAL
┌───────────────────▼─────────────────────────────┐
│         Power HAL                               │
│  ┌──────────────────────────────────────────┐  │
│  │  android.hardware.power                  │  │
│  │  - Power hints                           │  │
│  │  - Power modes                           │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│         Kernel                                  │
│  - CPUFreq governor                            │
│  - CPU hotplug                                 │
│  - Suspend/Resume                              │
│  - Wakeup sources                              │
└─────────────────────────────────────────────────┘
```

### Source Code Locations

```
frameworks/base/services/core/java/com/android/server/power/
├── PowerManagerService.java          # Main power service
├── Notifier.java                      # Power notifications
├── PowerGroup.java                    # Power group management
├── DisplayPowerController.java        # Display power control
├── WakeLock.java                      # Wakelock implementation
└── batterysaver/                      # Battery saver mode

frameworks/base/services/core/java/com/android/server/
├── BatteryService.java                # Battery monitoring
└── am/BatteryStatsService.java        # Battery statistics

frameworks/base/core/java/android/os/
├── PowerManager.java                  # Public API
├── BatteryManager.java                # Battery info API
└── PowerManagerInternal.java          # Internal interface

hardware/interfaces/power/
├── 1.0/                              # Power HAL 1.0 (HIDL)
└── aidl/                             # Power HAL (AIDL)
```

## PowerManagerService

### Core Responsibilities

PowerManagerService (PMS) manages all power-related operations:

1. **Wakelock management**: Track and enforce wakelocks
2. **Display power**: Control screen on/off, brightness
3. **System power states**: Awake, dozing, sleeping
4. **Power hints**: Notify HAL of power state changes
5. **Battery saver**: Low power mode
6. **Dream (screensaver)**: Start/stop dreams

### Power States

```java
// PowerManagerService.java

// Display states
static final int DISPLAY_STATE_OFF = 0;
static final int DISPLAY_STATE_DOZE = 1;
static final int DISPLAY_STATE_ON = 2;
static final int DISPLAY_STATE_DOZE_SUSPEND = 3;
static final int DISPLAY_STATE_ON_SUSPEND = 4;

// Wakefulness states
static final int WAKEFULNESS_ASLEEP = 0;    // Device sleeping
static final int WAKEFULNESS_AWAKE = 1;     // Device awake
static final int WAKEFULNESS_DREAMING = 2;  // Screensaver active
static final int WAKEFULNESS_DOZING = 3;    // Doze mode
```

### Power State Transitions

```
User Activity → AWAKE
      ↓
User Inactive (timeout) → DOZING/DREAMING
      ↓
Further Timeout → ASLEEP
      ↓
Motion/Notification → DOZING (Ambient Display)
      ↓
User Interaction → AWAKE
```

### PMS Initialization

```java
// PowerManagerService.java

public PowerManagerService(Context context) {
    super(context);
    
    mContext = context;
    mHandlerThread = new ServiceThread(TAG,
            Process.THREAD_PRIORITY_DISPLAY, false);
    mHandlerThread.start();
    mHandler = new PowerManagerHandler(mHandlerThread.getLooper());
    
    synchronized (mLock) {
        // Initialize wakefulness
        mWakefulness = WAKEFULNESS_AWAKE;
        mWakefulnessChanging = false;
        mIsPowered = false;
        mPlugType = BatteryManager.BATTERY_PLUGGED_NONE;
        mBatteryLevel = 100;
        
        // Create wake locks
        mWakeLocks = new ArrayList<>();
        
        // Initialize display power controller
        mDisplayPowerController = new DisplayPowerController(
                mContext, mHandler.getLooper(), this);
    }
}

@Override
public void onStart() {
    // Publish service
    publishBinderService(Context.POWER_SERVICE, new BinderService());
    publishLocalService(PowerManagerInternal.class, new LocalService());
    
    // Register for broadcasts
    IntentFilter filter = new IntentFilter();
    filter.addAction(Intent.ACTION_BATTERY_CHANGED);
    filter.addAction(Intent.ACTION_SCREEN_OFF);
    filter.addAction(Intent.ACTION_SCREEN_ON);
    mContext.registerReceiver(mReceiver, filter);
}
```

### User Activity Handling

```java
// PowerManagerService.java

/**
 * Called when user activity detected
 */
private void userActivityNoUpdateLocked(long eventTime, int event, 
        int flags, int uid) {
    
    if (eventTime < mLastSleepTime || eventTime < mLastWakeTime
            || !mBootCompleted || !mSystemReady) {
        return;
    }
    
    // Record user activity time
    mLastUserActivityTime = eventTime;
    mLastUserActivityTimeNoChangeLights = eventTime;
    
    // Reset sleep timeout
    if (mWakefulness == WAKEFULNESS_ASLEEP
            || mWakefulness == WAKEFULNESS_DOZING
            || (flags & PowerManager.USER_ACTIVITY_FLAG_INDIRECT) != 0) {
        return;
    }
    
    // Update power state
    if ((flags & PowerManager.USER_ACTIVITY_FLAG_NO_CHANGE_LIGHTS) != 0) {
        if (eventTime > mLastUserActivityTimeNoChangeLights) {
            mLastUserActivityTimeNoChangeLights = eventTime;
            mDirty |= DIRTY_USER_ACTIVITY;
        }
    } else {
        if (eventTime > mLastUserActivityTime) {
            mLastUserActivityTime = eventTime;
            mDirty |= DIRTY_USER_ACTIVITY;
        }
    }
    
    // Wake up if needed
    if (mWakefulness == WAKEFULNESS_DREAMING) {
        wakeUpNoUpdateLocked(eventTime, "android.server.power:USER_ACTIVITY",
                uid, mContext.getOpPackageName(), uid);
    }
}

/**
 * Called to update power state
 */
private void updatePowerStateLocked() {
    if (!mSystemReady || mDirty == 0) {
        return;
    }
    
    // Update wakefulness
    updateWakefulnessLocked(mDirty);
    
    // Update display
    updateDisplayPowerStateLocked(mDirty);
    
    // Update dreams
    updateDreamLocked(mDirty, false);
    
    // Notify policy changes
    finishWakefulnessChangeIfNeededLocked();
    
    // Send power state notifications
    sendPowerStateNotifications();
    
    mDirty = 0;
}
```

### Going to Sleep

```java
// PowerManagerService.java

/**
 * Request device go to sleep
 */
private void goToSleepNoUpdateLocked(long eventTime, int reason, 
        int flags, int uid) {
    
    if (eventTime < mLastWakeTime
            || mWakefulness == WAKEFULNESS_ASLEEP
            || !mBootCompleted || !mSystemReady) {
        return;
    }
    
    Slog.i(TAG, "Going to sleep due to " + 
        PowerManager.sleepReasonToString(reason) +
        " (uid " + uid + ")...");
    
    mLastSleepTime = eventTime;
    mSandmanSummoned = true;
    setWakefulnessLocked(WAKEFULNESS_DOZING, reason);
    
    // Notify listeners
    mNotifier.onGoToSleepStarted(reason);
    
    // Update state
    updatePowerStateLocked();
}

private void setWakefulnessLocked(int wakefulness, int reason) {
    if (mWakefulness != wakefulness) {
        mWakefulness = wakefulness;
        mWakefulnessChanging = true;
        mDirty |= DIRTY_WAKEFULNESS;
        
        // Notify power HAL
        if (mPowerHal != null) {
            mPowerHal.setMode(
                wakefulness == WAKEFULNESS_AWAKE ? 
                    Mode.INTERACTIVE : Mode.LOW_POWER,
                true);
        }
    }
}
```

## Wakelock System

### Wakelock Types

```java
// PowerManager.java

public static final int PARTIAL_WAKE_LOCK = 0x00000001;
// CPU stays on, screen can turn off

public static final int SCREEN_DIM_WAKE_LOCK = 0x00000006;
// Screen at dim brightness (deprecated)

public static final int SCREEN_BRIGHT_WAKE_LOCK = 0x0000000a;
// Screen at full brightness (deprecated)

public static final int FULL_WAKE_LOCK = 0x0000001a;
// Screen and keyboard at full brightness (deprecated)

public static final int PROXIMITY_SCREEN_OFF_WAKE_LOCK = 0x00000020;
// Screen off when proximity sensor detects object

public static final int DOZE_WAKE_LOCK = 0x00000040;
// Keep CPU running in doze mode

public static final int DRAW_WAKE_LOCK = 0x00000080;
// Ensures screen updates are drawn
```

### Wakelock Flags

```java
public static final int ACQUIRE_CAUSES_WAKEUP = 0x10000000;
// Wake device when acquiring lock

public static final int ON_AFTER_RELEASE = 0x20000000;
// Keep screen on briefly after release
```

### Acquiring Wakelocks

**From application:**

```java
PowerManager pm = (PowerManager) getSystemService(Context.POWER_SERVICE);
PowerManager.WakeLock wakeLock = pm.newWakeLock(
    PowerManager.PARTIAL_WAKE_LOCK,
    "MyApp::MyWakeLock");

// Acquire wakelock
wakeLock.acquire();

// Do work...

// Release wakelock
wakeLock.release();

// Or acquire with timeout
wakeLock.acquire(10 * 60 * 1000);  // 10 minutes
```

**In PowerManagerService:**

```java
// PowerManagerService.java

private final class WakeLock implements IBinder.DeathRecipient {
    public final IBinder mLock;
    public int mFlags;
    public String mTag;
    public final int mOwnerUid;
    public final int mOwnerPid;
    public final WorkSource mWorkSource;
    public long mAcquireTime;
    public boolean mNotifiedAcquired;
    
    // ...
}

private void acquireWakeLockInternal(IBinder lock, int flags, String tag,
        String packageName, WorkSource ws, String historyTag, int uid, 
        int pid) {
    
    synchronized (mLock) {
        // Find or create wake lock
        WakeLock wakeLock = findWakeLockLocked(lock);
        if (wakeLock == null) {
            wakeLock = new WakeLock(lock, flags, tag, packageName, 
                ws, historyTag, uid, pid);
            
            try {
                lock.linkToDeath(wakeLock, 0);
            } catch (RemoteException e) {
                throw new IllegalArgumentException("Wake lock is already dead.");
            }
            
            mWakeLocks.add(wakeLock);
        } else {
            // Update existing wake lock
            updateWakeLockWorkSource(wakeLock, ws);
        }
        
        // Record acquire time
        wakeLock.mAcquireTime = SystemClock.uptimeMillis();
        
        // Apply wake lock
        applyWakeLockFlagsOnAcquireLocked(wakeLock);
        
        mDirty |= DIRTY_WAKE_LOCKS;
        updatePowerStateLocked();
    }
}

private void applyWakeLockFlagsOnAcquireLocked(WakeLock wakeLock) {
    if ((wakeLock.mFlags & PowerManager.ACQUIRE_CAUSES_WAKEUP) != 0
            && isScreenLock(wakeLock)) {
        // Wake up device
        wakeUpNoUpdateLocked(SystemClock.uptimeMillis(),
                "android.server.power:WAKELOCK",
                wakeLock.mOwnerUid,
                wakeLock.mPackageName,
                wakeLock.mOwnerUid);
    }
}
```

### Releasing Wakelocks

```java
// PowerManagerService.java

private void releaseWakeLockInternal(IBinder lock, int flags) {
    synchronized (mLock) {
        WakeLock wakeLock = findWakeLockLocked(lock);
        if (wakeLock == null) {
            return;
        }
        
        // Remove wake lock
        mWakeLocks.remove(wakeLock);
        wakeLock.mLock.unlinkToDeath(wakeLock, 0);
        
        // Apply ON_AFTER_RELEASE flag
        if ((wakeLock.mFlags & PowerManager.ON_AFTER_RELEASE) != 0
                && isScreenLock(wakeLock)) {
            userActivityNoUpdateLocked(SystemClock.uptimeMillis(),
                    PowerManager.USER_ACTIVITY_EVENT_OTHER,
                    0, wakeLock.mOwnerUid);
        }
        
        mDirty |= DIRTY_WAKE_LOCKS;
        updatePowerStateLocked();
    }
}
```

### Wakelock Timeout Enforcement

```java
// PowerManagerService.java

private void checkForLongWakeLocks() {
    synchronized (mLock) {
        final long now = SystemClock.uptimeMillis();
        
        for (int i = mWakeLocks.size() - 1; i >= 0; i--) {
            WakeLock wakeLock = mWakeLocks.get(i);
            
            if ((wakeLock.mFlags & PowerManager.PARTIAL_WAKE_LOCK) != 0) {
                long duration = now - wakeLock.mAcquireTime;
                
                // Warn about long-held wakelocks
                if (duration > 60 * 60 * 1000) {  // 1 hour
                    Slog.w(TAG, "Long wake lock held: " + wakeLock.mTag +
                        " for " + (duration / 1000) + " seconds");
                }
            }
        }
    }
}
```

## Battery Monitoring

### BatteryService

Monitors battery state and notifies system:

```java
// BatteryService.java

public final class BatteryService extends SystemService {
    
    private final Object mLock = new Object();
    
    // Battery properties
    private BatteryProperties mBatteryProps;
    private boolean mCharging;
    private int mBatteryLevel;
    private int mBatteryStatus;
    private int mBatteryHealth;
    private boolean mBatteryPresent;
    private int mPlugType;
    
    @Override
    public void onStart() {
        // Register for battery property changes
        registerHealthHalCallback();
        
        // Publish service
        publishBinderService("battery", new BinderService());
        publishLocalService(BatteryManagerInternal.class, new LocalService());
    }
    
    private void registerHealthHalCallback() {
        // Register with Health HAL
        IHealth healthService = IHealth.getService();
        if (healthService != null) {
            healthService.registerCallback(mHealthHalCallback);
        }
    }
    
    private final IHealthInfoCallback mHealthHalCallback = 
            new IHealthInfoCallback.Stub() {
        @Override
        public void healthInfoChanged(HealthInfo info) {
            // Update battery properties
            updateBatteryProperties(info);
        }
    };
    
    private void updateBatteryProperties(HealthInfo info) {
        synchronized (mLock) {
            // Update cached properties
            mBatteryLevel = info.batteryLevel;
            mBatteryStatus = info.batteryStatus;
            mBatteryHealth = info.batteryHealth;
            mBatteryPresent = info.batteryPresent;
            mCharging = info.batteryStatus == 
                BatteryManager.BATTERY_STATUS_CHARGING;
            
            // Determine plug type
            if (info.chargerAcOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_AC;
            } else if (info.chargerUsbOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_USB;
            } else if (info.chargerWirelessOnline) {
                mPlugType = BatteryManager.BATTERY_PLUGGED_WIRELESS;
            } else {
                mPlugType = BatteryManager.BATTERY_PLUGGED_NONE;
            }
            
            // Send battery changed intent
            sendBatteryChangedIntent();
        }
    }
    
    private void sendBatteryChangedIntent() {
        Intent intent = new Intent(Intent.ACTION_BATTERY_CHANGED);
        intent.addFlags(Intent.FLAG_RECEIVER_REGISTERED_ONLY);
        
        intent.putExtra(BatteryManager.EXTRA_STATUS, mBatteryStatus);
        intent.putExtra(BatteryManager.EXTRA_HEALTH, mBatteryHealth);
        intent.putExtra(BatteryManager.EXTRA_PRESENT, mBatteryPresent);
        intent.putExtra(BatteryManager.EXTRA_LEVEL, mBatteryLevel);
        intent.putExtra(BatteryManager.EXTRA_SCALE, 100);
        intent.putExtra(BatteryManager.EXTRA_PLUGGED, mPlugType);
        intent.putExtra(BatteryManager.EXTRA_VOLTAGE, mBatteryProps.batteryVoltage);
        intent.putExtra(BatteryManager.EXTRA_TEMPERATURE, mBatteryProps.batteryTemperature);
        intent.putExtra(BatteryManager.EXTRA_TECHNOLOGY, mBatteryProps.batteryTechnology);
        
        mContext.sendBroadcastAsUser(intent, UserHandle.ALL);
    }
}
```

### BatteryStatsService

Tracks battery usage per-app:

```java
// BatteryStatsService.java

public final class BatteryStatsService extends IBatteryStats.Stub {
    
    private BatteryStatsImpl mStats;
    
    /**
     * Note wakelock acquired
     */
    public void noteStartWakelock(int uid, int pid, String name, 
            String historyName, int type, boolean unimportant) {
        synchronized (mStats) {
            mStats.noteStartWakeLocked(uid, pid, name, historyName, type,
                    unimportant, SystemClock.elapsedRealtime(), 
                    SystemClock.uptimeMillis());
        }
    }
    
    /**
     * Note wakelock released
     */
    public void noteStopWakelock(int uid, int pid, String name, 
            String historyName, int type) {
        synchronized (mStats) {
            mStats.noteStopWakeLocked(uid, pid, name, historyName, type,
                    SystemClock.elapsedRealtime(), SystemClock.uptimeMillis());
        }
    }
    
    /**
     * Note screen state changed
     */
    public void noteScreenState(int state) {
        synchronized (mStats) {
            mStats.noteScreenStateLocked(state);
        }
    }
    
    /**
     * Note screen brightness changed
     */
    public void noteScreenBrightness(int brightness) {
        synchronized (mStats) {
            mStats.noteScreenBrightnessLocked(brightness);
        }
    }
    
    /**
     * Get battery statistics
     */
    public byte[] getStatistics() {
        Parcel out = Parcel.obtain();
        synchronized (mStats) {
            mStats.writeToParcel(out, 0);
        }
        byte[] data = out.marshall();
        out.recycle();
        return data;
    }
}
```

## Doze Mode and App Standby

### Doze Mode

Doze reduces battery consumption when device is idle:

```java
// DeviceIdleController.java

public class DeviceIdleController extends SystemService {
    
    // Doze states
    static final int STATE_ACTIVE = 0;
    static final int STATE_INACTIVE = 1;
    static final int STATE_IDLE_PENDING = 2;
    static final int STATE_SENSING = 3;
    static final int STATE_LOCATING = 4;
    static final int STATE_IDLE = 5;
    static final int STATE_IDLE_MAINTENANCE = 6;
    
    private int mState;
    
    /**
     * Enter doze mode
     */
    private void becomeIdleLocked(String reason) {
        if (mState == STATE_ACTIVE || mState == STATE_INACTIVE) {
            mState = STATE_IDLE_PENDING;
            scheduleAlarmLocked(mConstants.IDLE_AFTER_INACTIVE_TIMEOUT, false);
        }
    }
    
    /**
     * Step to next doze state
     */
    private void stepIdleStateLocked(String reason) {
        switch (mState) {
            case STATE_IDLE_PENDING:
                // Start sensing
                mState = STATE_SENSING;
                startSensing();
                scheduleAlarmLocked(mConstants.SENSING_TIMEOUT, false);
                break;
                
            case STATE_SENSING:
                // Check if device moved
                if (mMotionDetected) {
                    // Exit doze
                    becomeActiveLocked("motion", Process.SYSTEM_UID);
                } else {
                    // Continue to idle
                    mState = STATE_IDLE;
                    enterDeepIdleMode();
                    scheduleAlarmLocked(mConstants.IDLE_TIMEOUT, true);
                }
                break;
                
            case STATE_IDLE:
                // Enter maintenance window
                mState = STATE_IDLE_MAINTENANCE;
                exitDeepIdleMode();
                scheduleAlarmLocked(mConstants.MAINTENANCE_WINDOW, true);
                break;
                
            case STATE_IDLE_MAINTENANCE:
                // Back to idle
                mState = STATE_IDLE;
                enterDeepIdleMode();
                scheduleAlarmLocked(mConstants.IDLE_TIMEOUT, true);
                break;
        }
    }
    
    private void enterDeepIdleMode() {
        Slog.i(TAG, "Entering deep idle mode");
        
        // Disable network
        mNetworkPolicyManager.setDeviceIdleMode(true);
        
        // Restrict apps
        mAlarmManager.setDeviceIdleUserWhitelist(mPowerSaveWhitelistUserAppIds);
        
        // Notify listeners
        mGoingIdleWakeLock.acquire();
        mHandler.sendEmptyMessage(MSG_BROADCAST_IDLE_MODE_CHANGED);
    }
    
    private void exitDeepIdleMode() {
        Slog.i(TAG, "Exiting deep idle mode");
        
        // Re-enable network
        mNetworkPolicyManager.setDeviceIdleMode(false);
        
        // Release restrictions
        mAlarmManager.setDeviceIdleUserWhitelist(new int[0]);
        
        // Notify listeners
        mHandler.sendEmptyMessage(MSG_BROADCAST_IDLE_MODE_CHANGED);
    }
}
```

### App Standby

Restricts apps that haven't been used recently:

```java
// AppStandbyController.java

public class AppStandbyController {
    
    // Standby buckets
    static final int STANDBY_BUCKET_ACTIVE = 10;
    static final int STANDBY_BUCKET_WORKING_SET = 20;
    static final int STANDBY_BUCKET_FREQUENT = 30;
    static final int STANDBY_BUCKET_RARE = 40;
    static final int STANDBY_BUCKET_RESTRICTED = 45;
    static final int STANDBY_BUCKET_NEVER = 50;
    
    /**
     * Update app standby bucket based on usage
     */
    private void updateAppStandbyBucket(String packageName, int userId,
            int bucket, int reason) {
        
        synchronized (mAppIdleLock) {
            AppIdleHistory.AppUsageHistory app = 
                mAppIdleHistory.getAppUsageHistory(packageName, userId);
            
            if (app.currentBucket != bucket) {
                app.currentBucket = bucket;
                app.bucketingReason = reason;
                
                // Notify listeners
                mHandler.obtainMessage(MSG_APP_IDLE_STATE_CHANGED,
                        userId, bucket, packageName).sendToTarget();
                
                // Apply restrictions
                if (bucket >= STANDBY_BUCKET_RARE) {
                    mAlarmManager.setAppStandbyBucket(packageName, 
                        userId, bucket);
                    mJobScheduler.setAppStandbyBucket(packageName,
                        userId, bucket);
                }
            }
        }
    }
}
```

## Display Power Management

### DisplayPowerController

Controls display brightness and power:

```java
// DisplayPowerController.java

public final class DisplayPowerController {
    
    private int mPowerState = Display.STATE_UNKNOWN;
    private float mBrightness = PowerManager.BRIGHTNESS_INVALID_FLOAT;
    private boolean mAutoBrightnessEnabled;
    
    /**
     * Update display power state
     */
    private void updatePowerState() {
        // Determine target state
        final int state = mPowerRequest.policy == 
            DisplayPowerRequest.POLICY_OFF ? Display.STATE_OFF :
            mPowerRequest.policy == DisplayPowerRequest.POLICY_DOZE ?
                Display.STATE_DOZE : Display.STATE_ON;
        
        // Determine target brightness
        float brightness = mPowerRequest.screenBrightness;
        
        if (mAutoBrightnessEnabled) {
            // Use automatic brightness
            brightness = mAutomaticBrightnessController
                .getAutomaticScreenBrightness();
        }
        
        // Apply brightness
        if (mPowerState != state || mBrightness != brightness) {
            mPowerState = state;
            mBrightness = brightness;
            
            // Update display
            setDisplayState(state);
            setDisplayBrightness(brightness);
        }
    }
    
    private void setDisplayState(int state) {
        // Control display via HAL
        if (mDisplayDevice != null) {
            mDisplayDevice.requestDisplayStateLocked(state);
        }
    }
    
    private void setDisplayBrightness(float brightness) {
        // Set backlight brightness
        if (mBacklight != null) {
            mBacklight.setBrightness(brightness);
        }
    }
}
```

### Automatic Brightness

```java
// AutomaticBrightnessController.java

public class AutomaticBrightnessController {
    
    private final LightSensor mLightSensor;
    private float mAmbientLux;
    private float mScreenBrightness;
    
    /**
     * Update automatic brightness based on ambient light
     */
    private void updateAutoBrightness() {
        // Get current lux reading
        mAmbientLux = mLightSensor.getLux();
        
        // Calculate screen brightness
        mScreenBrightness = calculateBrightness(mAmbientLux);
        
        // Apply with smoothing
        animateToScreenBrightness(mScreenBrightness);
    }
    
    private float calculateBrightness(float lux) {
        // Use brightness curve
        // Low light: lower brightness
        // Bright light: higher brightness
        
        if (lux < 10) {
            return 0.1f;  // 10% in dark
        } else if (lux < 100) {
            return 0.3f;  // 30% in dim light
        } else if (lux < 1000) {
            return 0.6f;  // 60% in normal light
        } else {
            return 1.0f;  // 100% in bright light
        }
    }
}
```

## Practical Example 1: Custom Power Profile

Let's implement a custom power management profile system.

### Step 1: Define Power Profiles

```java
// PowerProfile.java

package com.android.server.power;

public class PowerProfile {
    
    public static final int PROFILE_POWER_SAVE = 0;
    public static final int PROFILE_BALANCED = 1;
    public static final int PROFILE_PERFORMANCE = 2;
    
    public final int id;
    public final String name;
    
    // CPU settings
    public final int maxCpuFreq;        // MHz
    public final int minCpuFreq;        // MHz
    public final String cpuGovernor;
    
    // Display settings
    public final int maxBrightness;     // 0-255
    public final int screenTimeout;     // milliseconds
    
    // Network settings
    public final boolean restrictBackground;
    public final int syncInterval;      // minutes
    
    // System settings
    public final boolean vibrateEnabled;
    public final int animationScale;    // percentage
    
    public PowerProfile(int id, String name) {
        this.id = id;
        this.name = name;
        
        switch (id) {
            case PROFILE_POWER_SAVE:
                maxCpuFreq = 1400;
                minCpuFreq = 300;
                cpuGovernor = "powersave";
                maxBrightness = 128;
                screenTimeout = 15000;  // 15 seconds
                restrictBackground = true;
                syncInterval = 60;       // 1 hour
                vibrateEnabled = false;
                animationScale = 50;     // 50%
                break;
                
            case PROFILE_BALANCED:
                maxCpuFreq = 1800;
                minCpuFreq = 300;
                cpuGovernor = "interactive";
                maxBrightness = 255;
                screenTimeout = 30000;   // 30 seconds
                restrictBackground = false;
                syncInterval = 15;        // 15 minutes
                vibrateEnabled = true;
                animationScale = 100;     // 100%
                break;
                
            case PROFILE_PERFORMANCE:
                maxCpuFreq = 2400;
                minCpuFreq = 600;
                cpuGovernor = "performance";
                maxBrightness = 255;
                screenTimeout = 60000;    // 1 minute
                restrictBackground = false;
                syncInterval = 5;          // 5 minutes
                vibrateEnabled = true;
                animationScale = 100;      // 100%
                break;
                
            default:
                throw new IllegalArgumentException("Unknown profile: " + id);
        }
    }
}
```

### Step 2: Profile Manager

```java
// PowerProfileManager.java

package com.android.server.power;

public class PowerProfileManager {
    
    private static final String TAG = "PowerProfileManager";
    
    private final Context mContext;
    private final PowerManagerService mPowerManager;
    private PowerProfile mCurrentProfile;
    
    public PowerProfileManager(Context context, 
            PowerManagerService powerManager) {
        mContext = context;
        mPowerManager = powerManager;
        
        // Load saved profile
        int profileId = Settings.System.getInt(
            mContext.getContentResolver(),
            "power_profile", PowerProfile.PROFILE_BALANCED);
        
        mCurrentProfile = new PowerProfile(profileId, "");
        applyProfile(mCurrentProfile);
    }
    
    /**
     * Set active power profile
     */
    public void setProfile(int profileId) {
        Slog.i(TAG, "Switching to profile: " + profileId);
        
        mCurrentProfile = new PowerProfile(profileId, "");
        applyProfile(mCurrentProfile);
        
        // Save preference
        Settings.System.putInt(mContext.getContentResolver(),
            "power_profile", profileId);
        
        // Notify listeners
        Intent intent = new Intent("android.power.PROFILE_CHANGED");
        intent.putExtra("profile", profileId);
        mContext.sendBroadcast(intent);
    }
    
    /**
     * Apply profile settings to system
     */
    private void applyProfile(PowerProfile profile) {
        Slog.d(TAG, "Applying power profile: " + profile.name);
        
        // Apply CPU settings
        applyCpuSettings(profile);
        
        // Apply display settings
        applyDisplaySettings(profile);
        
        // Apply network settings
        applyNetworkSettings(profile);
        
        // Apply system settings
        applySystemSettings(profile);
    }
    
    private void applyCpuSettings(PowerProfile profile) {
        try {
            // Set CPU governor
            writeFile("/sys/devices/system/cpu/cpu0/cpufreq/scaling_governor",
                profile.cpuGovernor);
            
            // Set max frequency
            writeFile("/sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq",
                String.valueOf(profile.maxCpuFreq * 1000));
            
            // Set min frequency
            writeFile("/sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq",
                String.valueOf(profile.minCpuFreq * 1000));
            
            Slog.d(TAG, "CPU settings applied: " + profile.cpuGovernor);
        } catch (IOException e) {
            Slog.e(TAG, "Failed to apply CPU settings", e);
        }
    }
    
    private void applyDisplaySettings(PowerProfile profile) {
        // Set max brightness
        Settings.System.putInt(mContext.getContentResolver(),
            Settings.System.SCREEN_BRIGHTNESS, profile.maxBrightness);
        
        // Set screen timeout
        Settings.System.putInt(mContext.getContentResolver(),
            Settings.System.SCREEN_OFF_TIMEOUT, profile.screenTimeout);
        
        Slog.d(TAG, "Display settings applied");
    }
    
    private void applyNetworkSettings(PowerProfile profile) {
        // Restrict background data
        NetworkPolicyManager npm = NetworkPolicyManager.from(mContext);
        npm.setRestrictBackground(profile.restrictBackground);
        
        // Update sync settings
        ContentResolver.setMasterSyncAutomatically(true);
        // Note: Per-account sync intervals require AccountManager
        
        Slog.d(TAG, "Network settings applied");
    }
    
    private void applySystemSettings(PowerProfile profile) {
        // Vibration
        Settings.System.putInt(mContext.getContentResolver(),
            Settings.System.HAPTIC_FEEDBACK_ENABLED,
            profile.vibrateEnabled ? 1 : 0);
        
        // Animation scale
        float scale = profile.animationScale / 100.0f;
        Settings.Global.putFloat(mContext.getContentResolver(),
            Settings.Global.WINDOW_ANIMATION_SCALE, scale);
        Settings.Global.putFloat(mContext.getContentResolver(),
            Settings.Global.TRANSITION_ANIMATION_SCALE, scale);
        
        Slog.d(TAG, "System settings applied");
    }
    
    private void writeFile(String path, String value) throws IOException {
        FileOutputStream fos = new FileOutputStream(path);
        try {
            fos.write(value.getBytes());
        } finally {
            fos.close();
        }
    }
    
    /**
     * Get current profile
     */
    public PowerProfile getCurrentProfile() {
        return mCurrentProfile;
    }
}
```

### Step 3: Integrate with PowerManagerService

```java
// In PowerManagerService.java

private PowerProfileManager mProfileManager;

@Override
public void onStart() {
    // ... existing initialization ...
    
    // Initialize profile manager
    mProfileManager = new PowerProfileManager(mContext, this);
}

/**
 * Set power profile
 * @hide
 */
public void setPowerProfile(int profileId) {
    mContext.enforceCallingOrSelfPermission(
        android.Manifest.permission.DEVICE_POWER,
        "setPowerProfile");
    
    mProfileManager.setProfile(profileId);
}
```

### Step 4: Expose API

```java
// PowerManager.java

/**
 * Power profile constants
 */
public static final int POWER_PROFILE_POWER_SAVE = 0;
public static final int POWER_PROFILE_BALANCED = 1;
public static final int POWER_PROFILE_PERFORMANCE = 2;

/**
 * Set power profile
 * Requires DEVICE_POWER permission
 */
@RequiresPermission(android.Manifest.permission.DEVICE_POWER)
public void setPowerProfile(int profile) {
    try {
        mService.setPowerProfile(profile);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### Step 5: Usage

```bash
# Set power profile via settings
adb shell settings put system power_profile 0  # Power save
adb shell settings put system power_profile 1  # Balanced
adb shell settings put system power_profile 2  # Performance

# Verify CPU settings
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
adb shell cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```

## Practical Example 2: Wakelock Monitor

Let's create a wakelock monitoring and enforcement system.

### Step 1: Wakelock Monitor

```java
// WakelockMonitor.java

package com.android.server.power;

public class WakelockMonitor {
    
    private static final String TAG = "WakelockMonitor";
    private static final long WARNING_THRESHOLD = 30 * 60 * 1000;  // 30 minutes
    private static final long FORCE_RELEASE_THRESHOLD = 60 * 60 * 1000;  // 1 hour
    
    private final Context mContext;
    private final Handler mHandler;
    private final Map<String, WakelockInfo> mWakelocks = new HashMap<>();
    
    private static class WakelockInfo {
        String tag;
        int uid;
        long acquireTime;
        boolean warned;
        
        long getDuration() {
            return SystemClock.uptimeMillis() - acquireTime;
        }
    }
    
    public WakelockMonitor(Context context, Handler handler) {
        mContext = context;
        mHandler = handler;
        
        // Start monitoring
        mHandler.postDelayed(mMonitorRunnable, 60000);  // Check every minute
    }
    
    /**
     * Record wakelock acquisition
     */
    public void onWakelockAcquired(String tag, int uid) {
        synchronized (mWakelocks) {
            WakelockInfo info = new WakelockInfo();
            info.tag = tag;
            info.uid = uid;
            info.acquireTime = SystemClock.uptimeMillis();
            info.warned = false;
            
            mWakelocks.put(tag, info);
            
            Slog.d(TAG, "Wakelock acquired: " + tag + " (uid=" + uid + ")");
        }
    }
    
    /**
     * Record wakelock release
     */
    public void onWakelockReleased(String tag) {
        synchronized (mWakelocks) {
            WakelockInfo info = mWakelocks.remove(tag);
            if (info != null) {
                long duration = info.getDuration();
                Slog.d(TAG, "Wakelock released: " + tag + 
                    " (held for " + (duration / 1000) + "s)");
            }
        }
    }
    
    /**
     * Monitor runnable - checks for long-held wakelocks
     */
    private final Runnable mMonitorRunnable = new Runnable() {
        @Override
        public void run() {
            synchronized (mWakelocks) {
                for (WakelockInfo info : mWakelocks.values()) {
                    long duration = info.getDuration();
                    
                    // Force release if held too long
                    if (duration > FORCE_RELEASE_THRESHOLD) {
                        Slog.w(TAG, "Force releasing wakelock: " + info.tag +
                            " (held for " + (duration / 1000) + "s)");
                        
                        forceReleaseWakelock(info);
                        
                    // Warn if approaching threshold
                    } else if (duration > WARNING_THRESHOLD && !info.warned) {
                        Slog.w(TAG, "Long wakelock detected: " + info.tag +
                            " (held for " + (duration / 1000) + "s)");
                        
                        notifyLongWakelock(info);
                        info.warned = true;
                    }
                }
            }
            
            // Schedule next check
            mHandler.postDelayed(this, 60000);
        }
    };
    
    private void forceReleaseWakelock(WakelockInfo info) {
        // Kill the app holding the wakelock
        ActivityManager am = mContext.getSystemService(ActivityManager.class);
        am.forceStopPackage(getPackageForUid(info.uid));
        
        // Log event
        EventLog.writeEvent(EventLogTags.POWER_WAKELOCK_FORCE_RELEASE,
            info.tag, info.uid, info.getDuration());
    }
    
    private void notifyLongWakelock(WakelockInfo info) {
        // Show notification
        NotificationManager nm = mContext.getSystemService(
            NotificationManager.class);
        
        Notification notification = new Notification.Builder(
                mContext, SystemNotificationChannels.ALERTS)
            .setSmallIcon(com.android.internal.R.drawable.stat_sys_warning)
            .setContentTitle("Battery drain detected")
            .setContentText(getPackageForUid(info.uid) + 
                " is preventing device sleep")
            .build();
        
        nm.notify(info.tag, 1000, notification);
    }
    
    private String getPackageForUid(int uid) {
        PackageManager pm = mContext.getPackageManager();
        String[] packages = pm.getPackagesForUid(uid);
        return (packages != null && packages.length > 0) ? 
            packages[0] : "unknown";
    }
    
    /**
     * Get statistics for all wakelocks
     */
    public List<WakelockInfo> getWakelockStats() {
        synchronized (mWakelocks) {
            return new ArrayList<>(mWakelocks.values());
        }
    }
}
```

### Step 2: Usage

```java
// In PowerManagerService.java

private WakelockMonitor mWakelockMonitor;

@Override
public void onStart() {
    // ... existing initialization ...
    
    mWakelockMonitor = new WakelockMonitor(mContext, mHandler);
}

private void acquireWakeLockInternal(...) {
    // ... existing code ...
    
    // Notify monitor
    mWakelockMonitor.onWakelockAcquired(tag, uid);
}

private void releaseWakeLockInternal(...) {
    // ... existing code ...
    
    // Notify monitor
    mWakelockMonitor.onWakelockReleased(wakeLock.mTag);
}
```

## Key Takeaways

1. **PowerManagerService controls power**: Manages wakelocks, display, system states
2. **Wakelocks prevent sleep**: Apps use to keep CPU/screen active
3. **Battery monitoring is continuous**: BatteryService tracks state, BatteryStatsService tracks usage
4. **Doze mode saves battery**: Deep idle reduces background activity
5. **Display is major power consumer**: Auto-brightness and timeout are critical
6. **Custom policies enable optimization**: Profile systems allow user control
7. **Monitoring prevents abuse**: Track and enforce wakelock limits

## Next Steps

This concludes our core chapters. We've covered 10 comprehensive chapters on AOSP development. The remaining chapters (11-14) cover Security & SELinux, Build System, Debugging, and OTA Updates.

For your continued AOSP development journey, you now have deep knowledge of:
- Project structure and build environment
- Boot process and system initialization
- IPC mechanisms and service architecture
- Package and activity management
- Display and window management
- System UI customization
- Hardware abstraction
- Power management and optimization

## Quick Reference

### Power Management Commands

```bash
# View wakelocks
adb shell dumpsys power

# View battery stats
adb shell dumpsys batterystats

# Reset battery stats
adb shell dumpsys batterystats --reset

# View battery info
adb shell dumpsys battery

# Simulate battery level
adb shell dumpsys battery set level 50

# View doze state
adb shell dumpsys deviceidle

# Force doze mode (testing)
adb shell dumpsys deviceidle force-idle

# Exit doze mode
adb shell dumpsys deviceidle unforce

# Wakelock history
adb shell dumpsys batterystats | grep -A 20 "Wake lock"
```

### Power Settings

```bash
# Screen timeout
adb shell settings put system screen_off_timeout 30000

# Auto brightness
adb shell settings put system screen_brightness_mode 1

# Manual brightness
adb shell settings put system screen_brightness 128

# Battery saver
adb shell settings put global low_power 1
```
