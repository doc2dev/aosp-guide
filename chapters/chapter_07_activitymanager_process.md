# Chapter 7: ActivityManager & Process Management

## Contents

- [Introduction](#introduction)
- [ActivityManagerService Architecture](#activitymanagerservice-architecture)
- [Process Lifecycle](#process-lifecycle)
- [Activity Stack Management](#activity-stack-management)
- [Service Management](#service-management)
- [Broadcast Management](#broadcast-management)
- [ANR (Application Not Responding)](#anr-application-not-responding)
- [Practical Example 1: Custom Process Priority Policy](#practical-example-1-custom-process-priority-policy)
- [Practical Example 2: Activity Interceptor](#practical-example-2-activity-interceptor)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)
- [Quick Reference](#quick-reference)

## Introduction

ActivityManagerService (AMS) is the heart of Android's application framework. It manages the lifecycle of all application components (Activities, Services, BroadcastReceivers, ContentProviders), controls process creation and termination, handles task and back stack management, and enforces resource constraints through the Low Memory Killer.

Understanding ActivityManager is essential for AOSP development because it:
- Controls how and when apps launch and die
- Manages the activity back stack and task switching
- Determines process priorities and OOM scores
- Handles broadcast delivery and service binding
- Enforces permissions and security policies

This chapter explores AMS architecture, process lifecycle management, the OOM killer, and provides practical examples of customizing process management policies.

## ActivityManagerService Architecture

### High-Level Overview

ActivityManagerService runs in system_server and manages all application processes:

```
┌────────────────────────────────────────────────┐
│         Application Process (Zygote fork)       │
│  ┌──────────────────────────────────────────┐  │
│  │     Application Components               │  │
│  │  - Activities                            │  │
│  │  - Services                              │  │
│  │  - BroadcastReceivers                    │  │
│  │  - ContentProviders                      │  │
│  └──────────────┬───────────────────────────┘  │
│                 │ Binder IPC                    │
└─────────────────┼───────────────────────────────┘
                  │
┌─────────────────▼───────────────────────────────┐
│           System Server Process                 │
│  ┌──────────────────────────────────────────┐  │
│  │    ActivityManagerService (AMS)          │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │  Process Management                │ │  │
│  │  │  - Start/kill processes            │ │  │
│  │  │  - OOM adjustment                  │ │  │
│  │  │  - Process records                 │ │  │
│  │  └────────────────────────────────────┘ │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │  Activity Stack Management         │ │  │
│  │  │  - Activity records                │ │  │
│  │  │  - Task management                 │ │  │
│  │  │  - Back stack                      │ │  │
│  │  └────────────────────────────────────┘ │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │  Service Management                │ │  │
│  │  │  - Service records                 │ │  │
│  │  │  - Binding management              │ │  │
│  │  │  - Timeouts                        │ │  │
│  │  └────────────────────────────────────┘ │  │
│  │                                          │  │
│  │  ┌────────────────────────────────────┐ │  │
│  │  │  Broadcast Management              │ │  │
│  │  │  - Broadcast queue                 │ │  │
│  │  │  - Receiver dispatch               │ │  │
│  │  │  - Ordered broadcasts              │ │  │
│  │  └────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Source Code Locations

```
frameworks/base/services/core/java/com/android/server/am/
├── ActivityManagerService.java     # Main service (~25,000 lines)
├── ProcessRecord.java               # Process state
├── ActivityRecord.java              # Activity state
├── TaskRecord.java                  # Task (activity stack)
├── ActivityStack.java               # Activity stack management
├── ActivityStackSupervisor.java    # Multi-stack coordination
├── ServiceRecord.java               # Service state
├── BroadcastQueue.java              # Broadcast dispatch
├── BroadcastRecord.java             # Broadcast state
├── ContentProviderRecord.java      # Provider state
├── ProcessList.java                 # Process list management
├── OomAdjuster.java                 # OOM score calculation
├── LowMemDetector.java              # Low memory detection
└── AppErrors.java                   # Crash handling
```

### Key Data Structures

**ProcessRecord:**
```java
class ProcessRecord {
    final ApplicationInfo info;           // App info
    final String processName;             // Process name
    int pid;                              // Process ID
    int uid;                              // User ID
    IApplicationThread thread;            // Binder to app
    
    // OOM adjustment
    int curAdj;                           // Current OOM adj
    int setAdj;                           // Last set OOM adj
    int curSchedGroup;                    // Scheduler group
    
    // Component tracking
    final ArrayList<ActivityRecord> activities;
    final ArraySet<ServiceRecord> services;
    final ArraySet<ReceiverList> receivers;
    final ArraySet<ContentProviderRecord> pubProviders;
    
    // State
    boolean killed;
    boolean persistent;                   // System persistent
    boolean removed;                      // Being removed
    long lastActivityTime;                // Last activity use
    
    // ANR tracking
    boolean notResponding;
    String notRespondingReport;
}
```

**ActivityRecord:**
```java
class ActivityRecord {
    final ActivityInfo info;              // Activity info
    final String packageName;
    final String processName;
    ProcessRecord app;                    // Hosting process
    
    // State
    ActivityState state;                  // Current state
    boolean visible;
    boolean sleeping;
    boolean finishing;
    
    // Task and stack
    TaskRecord task;
    ActivityStack stack;
    
    // Lifecycle
    long createTime;
    long lastVisibleTime;
    
    // Configuration
    Configuration configuration;
}
```

**TaskRecord:**
```java
class TaskRecord {
    final int taskId;                     // Unique task ID
    final ArrayList<ActivityRecord> mActivities;  // Activity stack
    
    Intent intent;                        // Base intent
    String affinity;                      // Task affinity
    int userId;                           // User ID
    
    ActivityRecord rootActivity;          // Bottom activity
    ActivityRecord topActivity;           // Top activity
    
    boolean inRecents;                    // Show in recents?
    long lastActiveTime;
}
```

## Process Lifecycle

### Process Creation

When an app component needs to run and no process exists:

**1. Check for existing process:**
```java
ProcessRecord app = getProcessRecordLocked(processName, appInfo.uid);
if (app == null || app.thread == null) {
    // Need to start new process
    app = startProcessLocked(processName, appInfo, ...);
}
```

**2. Request process from Zygote:**
```java
private ProcessRecord startProcessLocked(String processName, 
        ApplicationInfo info, ...) {
    
    // Create ProcessRecord
    ProcessRecord app = new ProcessRecord(info, processName, uid);
    
    // Start process via Zygote
    final Process.ProcessStartResult startResult = 
        Process.start("android.app.ActivityThread",
                     app.processName,
                     uid, 
                     gids,
                     runtimeFlags,
                     zygoteArgs);
    
    // Store PID
    app.setPid(startResult.pid);
    
    synchronized (mPidsSelfLocked) {
        mPidsSelfLocked.put(startResult.pid, app);
    }
    
    return app;
}
```

**3. App attaches to ActivityManager:**

When the new process starts, it calls `attachApplication()`:

```java
// In new app process (ActivityThread.main)
public static void main(String[] args) {
    // ...
    ActivityThread thread = new ActivityThread();
    thread.attach(false);  // Attach to system
    // ...
}

private void attach(boolean system) {
    final IActivityManager mgr = ActivityManager.getService();
    mgr.attachApplication(mAppThread);  // Binder callback interface
}

// In ActivityManagerService
public final void attachApplication(IApplicationThread thread) {
    synchronized (this) {
        int callingPid = Binder.getCallingPid();
        ProcessRecord app = mPidsSelfLocked.get(callingPid);
        
        // Store binder reference
        app.thread = thread;
        
        // Bind application
        thread.bindApplication(processName, appInfo, ...);
        
        // Launch pending components
        if (mPendingActivities.contains(app)) {
            startActivityLocked(app, ...);
        }
    }
}
```

### Process States and Priorities

Android classifies processes into different importance levels:

**Process States:**
```java
// ActivityManager.ProcessState constants

PROCESS_STATE_PERSISTENT = 0;           // System persistent
PROCESS_STATE_PERSISTENT_UI = 1;        // Persistent with UI
PROCESS_STATE_TOP = 2;                  // Foreground app
PROCESS_STATE_BOUND_TOP = 3;            // Bound by top
PROCESS_STATE_FOREGROUND_SERVICE = 4;   // Foreground service
PROCESS_STATE_BOUND_FOREGROUND_SERVICE = 5;  // Bound by FGS
PROCESS_STATE_IMPORTANT_FOREGROUND = 6; // Important to user
PROCESS_STATE_IMPORTANT_BACKGROUND = 7; // Important but background
PROCESS_STATE_TRANSIENT_BACKGROUND = 8; // Temporary background
PROCESS_STATE_BACKUP = 9;               // Backup operation
PROCESS_STATE_SERVICE = 10;             // Background service
PROCESS_STATE_RECEIVER = 11;            // Broadcast receiver
PROCESS_STATE_TOP_SLEEPING = 12;        // Top but device sleeping
PROCESS_STATE_HEAVY_WEIGHT = 13;        // Heavy-weight process
PROCESS_STATE_HOME = 14;                // Home/launcher
PROCESS_STATE_LAST_ACTIVITY = 15;       // Last used activity
PROCESS_STATE_CACHED_ACTIVITY = 16;     // Cached with activity
PROCESS_STATE_CACHED_ACTIVITY_CLIENT = 17;  // Client of cached
PROCESS_STATE_CACHED_RECENT = 18;       // Recently used
PROCESS_STATE_CACHED_EMPTY = 19;        // Empty cached process
PROCESS_STATE_NONEXISTENT = 20;         // Not running
```

**OOM Adjustment Values:**
```java
// ProcessList constants - lower values = higher priority

NATIVE_ADJ = -1000;                     // Native daemon
SYSTEM_ADJ = -900;                      // System server
PERSISTENT_PROC_ADJ = -800;             // Persistent system apps
PERSISTENT_SERVICE_ADJ = -700;          // Persistent services
FOREGROUND_APP_ADJ = 0;                 // Foreground app
VISIBLE_APP_ADJ = 100;                  // Visible app
PERCEPTIBLE_APP_ADJ = 200;              // Perceptible to user
PERCEPTIBLE_LOW_APP_ADJ = 250;          // Perceptible low priority
BACKUP_APP_ADJ = 300;                   // Backup operation
HEAVY_WEIGHT_APP_ADJ = 400;             // Heavy-weight process
SERVICE_ADJ = 500;                      // Background service
HOME_APP_ADJ = 600;                     // Home/launcher
PREVIOUS_APP_ADJ = 700;                 // Previous app
SERVICE_B_ADJ = 800;                    // B-list service
CACHED_APP_MIN_ADJ = 900;               // Cached process (first)
CACHED_APP_MAX_ADJ = 999;               // Cached process (last)
```

### OOM Adjuster

The OomAdjuster calculates process priorities:

```java
// OomAdjuster.java

final boolean computeOomAdjLocked(ProcessRecord app, 
        int cachedAdj, ProcessRecord TOP_APP, ...) {
    
    if (app == TOP_APP) {
        // This is the foreground app
        app.adjType = "top-activity";
        app.curAdj = FOREGROUND_APP_ADJ;
        app.curProcState = PROCESS_STATE_TOP;
        return true;
    }
    
    int adj = CACHED_APP_MAX_ADJ;
    int procState = PROCESS_STATE_CACHED_EMPTY;
    
    // Check for foreground services
    if (app.hasForegroundServices()) {
        adj = FOREGROUND_APP_ADJ;
        procState = PROCESS_STATE_FOREGROUND_SERVICE;
        app.adjType = "fg-service";
    }
    
    // Check for activities
    if (app.activities.size() > 0) {
        for (int i = 0; i < app.activities.size(); i++) {
            ActivityRecord r = app.activities.get(i);
            
            if (r.visible) {
                adj = Math.min(adj, VISIBLE_APP_ADJ);
                procState = Math.min(procState, 
                    PROCESS_STATE_IMPORTANT_FOREGROUND);
                app.adjType = "vis-activity";
                break;
            }
            
            if (r.state == ActivityState.PAUSING || 
                r.state == ActivityState.PAUSED) {
                adj = Math.min(adj, PERCEPTIBLE_APP_ADJ);
                procState = Math.min(procState,
                    PROCESS_STATE_IMPORTANT_FOREGROUND);
                app.adjType = "pause-activity";
            }
        }
    }
    
    // Check for services
    if (app.services.size() > 0) {
        for (int i = 0; i < app.services.size(); i++) {
            ServiceRecord s = app.services.valueAt(i);
            
            if (s.isForeground) {
                // Already handled above
                continue;
            }
            
            // Background service
            adj = Math.min(adj, SERVICE_ADJ);
            procState = Math.min(procState, PROCESS_STATE_SERVICE);
            app.adjType = "service";
        }
    }
    
    // Apply final values
    app.curAdj = adj;
    app.curProcState = procState;
    
    // Update kernel OOM score
    if (app.curAdj != app.setAdj) {
        ProcessList.setOomAdj(app.pid, app.uid, app.curAdj);
        app.setAdj = app.curAdj;
    }
    
    return true;
}
```

### Low Memory Killer (LMK)

The kernel's LMK kills processes when memory is low:

**OOM score written to `/proc/<pid>/oom_score_adj`:**

```java
// ProcessList.java

public static final void setOomAdj(int pid, int uid, int amt) {
    // Write to /proc/<pid>/oom_score_adj
    // Kernel uses this to select victims
    
    if (amt == UNKNOWN_ADJ) {
        return;
    }
    
    long start = SystemClock.uptimeMillis();
    
    try {
        FileOutputStream fos = new FileOutputStream(
            String.format("/proc/%d/oom_score_adj", pid));
        fos.write(Integer.toString(amt).getBytes());
        fos.close();
    } catch (IOException e) {
        // Process may have died
    }
    
    long now = SystemClock.uptimeMillis();
    if (now - start > 100) {
        Slog.w(TAG, "Slow OOM adj write: " + (now - start) + "ms for pid " + pid);
    }
}
```

**LMK thresholds configured in init.rc:**

```rc
# Set LMK minfree thresholds (in pages, 1 page = 4KB)
write /sys/module/lowmemorykiller/parameters/minfree "18432,23040,27648,32256,36864,46080"

# Corresponds to OOM adj values:
# 0 (FOREGROUND_APP_ADJ)
# 100 (VISIBLE_APP_ADJ)
# 200 (PERCEPTIBLE_APP_ADJ)
# 300 (BACKUP_APP_ADJ)
# 900 (CACHED_APP_MIN_ADJ)
# 906 (CACHED_APP_MAX_ADJ)
```

### Process Death Handling

When a process dies, AMS cleans up:

```java
// ActivityManagerService.java

private final void handleAppDiedLocked(ProcessRecord app,
        boolean restarting, boolean allowRestart) {
    
    int pid = app.pid;
    
    // Clean up activities
    for (int i = app.activities.size() - 1; i >= 0; i--) {
        ActivityRecord r = app.activities.get(i);
        if (!r.finishing) {
            // Finish the activity
            r.stack.finishActivityLocked(r, Activity.RESULT_CANCELED,
                null, "app-died", false);
        }
    }
    
    // Clean up services
    for (int i = app.services.size() - 1; i >= 0; i--) {
        ServiceRecord sr = app.services.valueAt(i);
        
        // Schedule restart if needed
        if (sr.crashCount < 3 && allowRestart) {
            scheduleServiceRestartLocked(sr, true);
        } else {
            // Too many crashes, give up
            bringDownServiceLocked(sr);
        }
    }
    
    // Clean up broadcast receivers
    for (int i = app.receivers.size() - 1; i >= 0; i--) {
        removeReceiverLocked(app.receivers.valueAt(i));
    }
    
    // Clean up content providers
    if (app.pubProviders.size() > 0) {
        for (int i = app.pubProviders.size() - 1; i >= 0; i--) {
            ContentProviderRecord cpr = app.pubProviders.valueAt(i);
            removeDyingProviderLocked(app, cpr, true);
        }
    }
    
    // Remove from process list
    mProcessNames.remove(app.processName, app.uid);
    mPidsSelfLocked.remove(pid);
    
    // Notify death recipients
    app.kill("death", true);
}
```

## Activity Stack Management

### Activity States

Activities transition through several states:

```java
enum ActivityState {
    INITIALIZING,      // Being created
    RESUMED,           // Active and visible
    PAUSING,           // Losing focus
    PAUSED,            // Not visible but alive
    STOPPING,          // Becoming invisible
    STOPPED,           // Invisible but alive
    FINISHING,         // Being destroyed
    DESTROYING,        // Cleanup in progress
    DESTROYED          // Gone
}
```

### Activity Lifecycle Flow

```
User launches app
      ↓
ActivityStackSupervisor.startActivityLocked()
      ↓
Create ActivityRecord
      ↓
Find/create TaskRecord
      ↓
Push to ActivityStack
      ↓
Pause current activity (if any)
      ↓
┌──────────────────────────────────┐
│  Current Activity: RESUMED       │
│         ↓                        │
│  schedulePauseActivity()         │
│         ↓                        │
│  App: onPause()                  │
│         ↓                        │
│  State: PAUSED                   │
└──────────────────────────────────┘
      ↓
┌──────────────────────────────────┐
│  New Activity: INITIALIZING      │
│         ↓                        │
│  scheduleResumeActivity()        │
│         ↓                        │
│  App: onCreate(), onStart()      │
│         ↓                        │
│  App: onResume()                 │
│         ↓                        │
│  State: RESUMED                  │
└──────────────────────────────────┘
      ↓
┌──────────────────────────────────┐
│  Previous Activity: PAUSED       │
│         ↓                        │
│  scheduleStopActivity()          │
│         ↓                        │
│  App: onStop()                   │
│         ↓                        │
│  State: STOPPED                  │
└──────────────────────────────────┘
```

### Task and Back Stack

Activities are organized into tasks:

```
Task #1 (Browser)
├── BrowserActivity (root)
├── SearchActivity
└── ResultsActivity (top)

Task #2 (Email)
├── InboxActivity (root)
└── ComposeActivity (top)

Back press: ResultsActivity → SearchActivity → BrowserActivity → Home
```

**Task affinity controls task assignment:**

```xml
<!-- Activities with same affinity go in same task -->
<activity android:name=".MainActivity"
          android:taskAffinity="com.example.mytask" />

<activity android:name=".DetailActivity"
          android:taskAffinity="com.example.mytask" />

<!-- Different affinity = different task -->
<activity android:name=".SettingsActivity"
          android:taskAffinity="com.example.settings" />
```

### Launch Modes

Control how activities are instantiated:

```xml
<!-- Standard: new instance every time -->
<activity android:launchMode="standard" />

<!-- SingleTop: reuse if already on top -->
<activity android:launchMode="singleTop" />

<!-- SingleTask: one instance per task -->
<activity android:launchMode="singleTask" />

<!-- SingleInstance: one instance, alone in task -->
<activity android:launchMode="singleInstance" />
```

**Implementation in ActivityStarter:**

```java
// ActivityStarter.java

private int startActivityUnchecked(...) {
    
    // Check launch mode
    if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_INSTANCE) {
        // Must be alone in task
        if (task != null && task.getChildCount() > 0) {
            // Create new task
            task = createNewTask(r);
        }
    } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_TASK) {
        // Find existing instance
        ActivityRecord existing = findActivityInHistory(r);
        if (existing != null) {
            // Clear activities above it
            existing.task.performClearTop(existing);
            // Deliver new intent
            existing.deliverNewIntent(r.intent);
            return START_DELIVERED_TO_TOP;
        }
    } else if (r.launchMode == ActivityInfo.LAUNCH_SINGLE_TOP) {
        if (topActivity != null && topActivity.equals(r)) {
            // Reuse top activity
            topActivity.deliverNewIntent(r.intent);
            return START_DELIVERED_TO_TOP;
        }
    }
    
    // Standard launch - create new instance
    addActivityToStack(r, task);
    return START_SUCCESS;
}
```

## Service Management

### Service Lifecycle

Services have a different lifecycle than activities:

```
startService()
      ↓
Create ServiceRecord
      ↓
Find/create hosting process
      ↓
scheduleCreateService()
      ↓
App: onCreate()
      ↓
scheduleServiceArgs()
      ↓
App: onStartCommand()
      ↓
Service running
      ↓
stopService() or stopSelf()
      ↓
scheduleStopService()
      ↓
App: onDestroy()
      ↓
Remove ServiceRecord
```

### Foreground Services

Foreground services have higher priority:

```java
// In service
startForeground(NOTIFICATION_ID, notification);

// In ActivityManagerService
void setServiceForegroundLocked(ComponentName className, 
        IBinder token, int id, Notification notification, int flags) {
    
    ServiceRecord r = findServiceLocked(className, token);
    if (r == null) {
        return;
    }
    
    if (id != 0) {
        // Starting foreground
        r.isForeground = true;
        r.foregroundId = id;
        r.foregroundNoti = notification;
        
        // Post notification
        mAm.mHandler.post(() -> {
            NotificationManager nm = mAm.mContext.getSystemService(
                NotificationManager.class);
            nm.notify(id, notification);
        });
        
        // Update OOM adj
        r.app.hasForegroundServices = true;
        updateOomAdjLocked(r.app);
        
    } else {
        // Stopping foreground
        r.isForeground = false;
        r.foregroundId = 0;
        r.foregroundNoti = null;
        
        r.app.hasForegroundServices = false;
        updateOomAdjLocked(r.app);
    }
}
```

### Service Binding

Bound services connect clients to services:

```java
// Client binds
Intent intent = new Intent(this, MyService.class);
bindService(intent, connection, BIND_AUTO_CREATE);

// In ActivityManagerService
public int bindServiceLocked(IApplicationThread caller,
        IBinder token, Intent service, String resolvedType,
        IServiceConnection connection, int flags, ...) {
    
    // Find or create service
    ServiceRecord s = retrieveServiceLocked(service, ...);
    
    // Create connection record
    ConnectionRecord c = new ConnectionRecord(connection, s);
    
    // Start service if needed
    if (s.app == null || s.app.thread == null) {
        bringUpServiceLocked(s, ...);
    }
    
    // Bind connection
    if (s.app != null && s.app.thread != null) {
        s.app.thread.scheduleBindService(s, service, true);
    }
    
    return 1;
}
```

### Service Timeouts

Services must respond within time limits:

```java
// ActiveServices.java

static final int SERVICE_TIMEOUT = 20 * 1000;  // 20 seconds
static final int SERVICE_BACKGROUND_TIMEOUT = 200 * 1000;  // 200 seconds

void serviceForegroundTimeout(ServiceRecord r) {
    ProcessRecord app = r.app;
    if (app != null && app.isDebugging()) {
        // Ignore while debugging
        return;
    }
    
    // Service didn't call startForeground() in time
    Slog.w(TAG, "Timeout for foreground service: " + r);
    
    // Generate ANR
    mAm.mHandler.post(() -> {
        mAm.mAnrHelper.appNotResponding(app, "Service foreground timeout");
    });
}

void scheduleServiceTimeoutLocked(ProcessRecord proc) {
    if (proc.executingServices.size() == 0) {
        return;
    }
    
    Message msg = mAm.mHandler.obtainMessage(
        ActivityManagerService.SERVICE_TIMEOUT_MSG);
    msg.obj = proc;
    
    mAm.mHandler.sendMessageDelayed(msg,
        proc.execServicesFg ? SERVICE_TIMEOUT : SERVICE_BACKGROUND_TIMEOUT);
}
```

## Broadcast Management

### Broadcast Types

**Normal broadcasts:**
- Delivered to all registered receivers
- No order guarantees
- Asynchronous delivery

**Ordered broadcasts:**
- Delivered in priority order
- Each receiver can pass data to next
- Can abort broadcast

**Sticky broadcasts (deprecated):**
- Last value cached and delivered to new receivers

### Broadcast Queue

```java
// BroadcastQueue.java

final class BroadcastQueue {
    final ArrayList<BroadcastRecord> mParallelBroadcasts = 
        new ArrayList<>();  // Parallel delivery
    final ArrayList<BroadcastRecord> mOrderedBroadcasts = 
        new ArrayList<>();  // Ordered delivery
    
    void scheduleBroadcastsLocked() {
        // Process parallel broadcasts
        while (mParallelBroadcasts.size() > 0) {
            BroadcastRecord r = mParallelBroadcasts.remove(0);
            deliverToRegisteredReceiverLocked(r, ...);
        }
        
        // Process ordered broadcasts
        if (mOrderedBroadcasts.size() > 0) {
            BroadcastRecord r = mOrderedBroadcasts.get(0);
            processNextBroadcast(false);
        }
    }
    
    void deliverToRegisteredReceiverLocked(BroadcastRecord r, 
            BroadcastFilter filter, boolean ordered) {
        
        // Permission checks
        if (!checkPermission(r, filter)) {
            return;
        }
        
        // Deliver to receiver
        try {
            performReceiveLocked(filter.receiverList.app, 
                filter.receiverList.receiver,
                new Intent(r.intent), ...);
        } catch (RemoteException e) {
            // Receiver died
        }
    }
}
```

### Broadcast Timeouts

Receivers must complete within time limits:

```java
// BroadcastQueue.java

static final int BROADCAST_FG_TIMEOUT = 10 * 1000;  // 10 seconds
static final int BROADCAST_BG_TIMEOUT = 60 * 1000;  // 60 seconds

void setBroadcastTimeoutLocked(long timeoutTime) {
    if (!mPendingBroadcastTimeoutMessage) {
        Message msg = mHandler.obtainMessage(BROADCAST_TIMEOUT_MSG, this);
        mHandler.sendMessageAtTime(msg, timeoutTime);
        mPendingBroadcastTimeoutMessage = true;
    }
}

void broadcastTimeoutLocked(boolean fromMsg) {
    if (fromMsg) {
        mPendingBroadcastTimeoutMessage = false;
    }
    
    BroadcastRecord r = mOrderedBroadcasts.get(0);
    if (r == null) {
        return;
    }
    
    // Receiver timed out
    Slog.w(TAG, "Timeout of broadcast " + r);
    
    // Move to next receiver
    r.receiverTime = SystemClock.uptimeMillis();
    finishReceiverLocked(r, r.resultCode, r.resultData, ...);
    
    scheduleBroadcastsLocked();
}
```

## ANR (Application Not Responding)

### ANR Detection

ANRs are detected when:
- Input event not processed within 5 seconds
- Broadcast receiver not completed within 10 seconds (foreground) or 60 seconds (background)
- Service not started within 20 seconds (foreground) or 200 seconds (background)

### Input ANR

```java
// InputDispatcher.cpp (native)

void InputDispatcher::onANRLocked(...) {
    nsecs_t currentTime = now();
    
    // Check input dispatch timeout
    const KeyEntry* keyEntry = mPendingEvent;
    nsecs_t timeoutTime = keyEntry->eventTime + INPUT_DISPATCHING_TIMEOUT;
    
    if (currentTime < timeoutTime) {
        return;  // Not timed out yet
    }
    
    // Find target app
    sp<InputApplicationHandle> applicationHandle = 
        getApplicationHandleLocked(keyEntry);
    
    // Notify ANR
    onAnrLocked(applicationHandle);
}

void InputDispatcher::onAnrLocked(
        sp<InputApplicationHandle> application) {
    
    // Call into ActivityManagerService
    mPolicy->notifyANR(application->getToken(), ...);
}
```

**In ActivityManagerService:**

```java
public void appNotResponding(ProcessRecord app, ActivityRecord activity,
        ActivityRecord parent, boolean aboveSystem, String annotation) {
    
    synchronized (this) {
        // Already showing ANR?
        if (app.notResponding) {
            return;
        }
        app.notResponding = true;
        
        // Collect trace
        EventLog.writeEvent(EventLogTags.AM_ANR, app.userId, app.pid,
            app.processName, app.info.flags, annotation);
        
        // Dump stack traces
        StringBuilder info = new StringBuilder();
        info.append("ANR in ").append(app.processName);
        if (activity != null && activity.shortComponentName != null) {
            info.append(" (").append(activity.shortComponentName).append(")");
        }
        info.append("\n");
        info.append("Reason: ").append(annotation).append("\n");
        
        // Collect all process traces
        File tracesFile = ActivityManagerService.dumpStackTraces(
            true, /* clearTraces */
            null, /* firstPids */
            null, /* lastPids */
            null  /* nativePids */
        );
        
        // Read traces
        String traces = readFileAsString(tracesFile);
        info.append(traces);
        
        app.notRespondingReport = info.toString();
        
        // Show ANR dialog
        Message msg = mUiHandler.obtainMessage(SHOW_NOT_RESPONDING_UI_MSG);
        msg.obj = app;
        mUiHandler.sendMessage(msg);
    }
}
```

## Practical Example 1: Custom Process Priority Policy

Let's implement a custom policy that adjusts process priorities based on user preferences.

### Step 1: Add Custom Priority Configuration

**File:** `frameworks/base/core/java/android/app/ActivityManager.java`

```java
public class ActivityManager {
    
    /**
     * Custom priority boost for specific packages.
     * @hide
     */
    public static final String EXTRA_CUSTOM_PRIORITY = "custom_priority";
    
    /**
     * Priority levels for custom boost
     * @hide
     */
    public static final int PRIORITY_BOOST_NONE = 0;
    public static final int PRIORITY_BOOST_LOW = 1;
    public static final int PRIORITY_BOOST_MEDIUM = 2;
    public static final int PRIORITY_BOOST_HIGH = 3;
}
```

### Step 2: Implement Custom OOM Adjustment

**File:** `frameworks/base/services/core/java/com/android/server/am/OomAdjuster.java`

```java
public class OomAdjuster {
    
    // Map of package name to priority boost level
    private final ArrayMap<String, Integer> mCustomPriorityMap = new ArrayMap<>();
    
    public OomAdjuster(ActivityManagerService service) {
        // ...
        loadCustomPriorityConfig();
    }
    
    private void loadCustomPriorityConfig() {
        // Load from settings or config file
        String config = Settings.Global.getString(
            mService.mContext.getContentResolver(),
            "custom_process_priorities");
        
        if (config != null) {
            // Format: "package1:level1,package2:level2"
            String[] entries = config.split(",");
            for (String entry : entries) {
                String[] parts = entry.split(":");
                if (parts.length == 2) {
                    mCustomPriorityMap.put(parts[0], 
                        Integer.parseInt(parts[1]));
                }
            }
        }
        
        Slog.i(TAG, "Loaded custom priority config: " + mCustomPriorityMap.size() 
            + " entries");
    }
    
    private int applyCustomPriorityBoost(ProcessRecord app, int currentAdj) {
        String packageName = app.info.packageName;
        Integer boost = mCustomPriorityMap.get(packageName);
        
        if (boost == null || boost == ActivityManager.PRIORITY_BOOST_NONE) {
            return currentAdj;
        }
        
        int adjustedAdj = currentAdj;
        
        switch (boost) {
            case ActivityManager.PRIORITY_BOOST_HIGH:
                // Significant boost - treat as visible app
                adjustedAdj = Math.min(adjustedAdj, 
                    ProcessList.VISIBLE_APP_ADJ);
                Slog.d(TAG, "HIGH priority boost for " + packageName);
                break;
                
            case ActivityManager.PRIORITY_BOOST_MEDIUM:
                // Moderate boost - treat as perceptible
                adjustedAdj = Math.min(adjustedAdj, 
                    ProcessList.PERCEPTIBLE_APP_ADJ);
                Slog.d(TAG, "MEDIUM priority boost for " + packageName);
                break;
                
            case ActivityManager.PRIORITY_BOOST_LOW:
                // Small boost - reduce by 100
                adjustedAdj = Math.max(ProcessList.FOREGROUND_APP_ADJ,
                    adjustedAdj - 100);
                Slog.d(TAG, "LOW priority boost for " + packageName);
                break;
        }
        
        return adjustedAdj;
    }
    
    final boolean computeOomAdjLocked(ProcessRecord app, 
            int cachedAdj, ProcessRecord TOP_APP, ...) {
        
        // ... standard OOM adjustment calculation ...
        
        int adj = calculateStandardAdj(app, TOP_APP);
        
        // Apply custom priority boost
        adj = applyCustomPriorityBoost(app, adj);
        
        // Apply final values
        app.curAdj = adj;
        
        if (app.curAdj != app.setAdj) {
            ProcessList.setOomAdj(app.pid, app.uid, app.curAdj);
            app.setAdj = app.curAdj;
        }
        
        return true;
    }
    
    /**
     * Update custom priority for a package
     * @hide
     */
    public void setCustomPriority(String packageName, int priorityLevel) {
        synchronized (mService) {
            if (priorityLevel == ActivityManager.PRIORITY_BOOST_NONE) {
                mCustomPriorityMap.remove(packageName);
            } else {
                mCustomPriorityMap.put(packageName, priorityLevel);
            }
            
            // Save configuration
            saveCustomPriorityConfig();
            
            // Recalculate OOM adj for affected processes
            updateOomAdjForPackage(packageName);
        }
    }
    
    private void saveCustomPriorityConfig() {
        StringBuilder config = new StringBuilder();
        for (int i = 0; i < mCustomPriorityMap.size(); i++) {
            if (i > 0) config.append(",");
            config.append(mCustomPriorityMap.keyAt(i))
                  .append(":")
                  .append(mCustomPriorityMap.valueAt(i));
        }
        
        Settings.Global.putString(
            mService.mContext.getContentResolver(),
            "custom_process_priorities",
            config.toString());
    }
    
    private void updateOomAdjForPackage(String packageName) {
        synchronized (mService.mPidsSelfLocked) {
            for (int i = 0; i < mService.mPidsSelfLocked.size(); i++) {
                ProcessRecord app = mService.mPidsSelfLocked.valueAt(i);
                if (packageName.equals(app.info.packageName)) {
                    computeOomAdjLocked(app, app.curAdj, 
                        mService.mTopApp, true, 0, false);
                }
            }
        }
    }
}
```

### Step 3: Expose API

**File:** `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java`

```java
public class ActivityManagerService extends IActivityManager.Stub {
    
    /**
     * Set custom process priority for a package.
     * Requires MANAGE_ACTIVITY_STACKS permission.
     * @hide
     */
    public void setProcessPriority(String packageName, int priorityLevel) {
        enforceCallingPermission(
            android.Manifest.permission.MANAGE_ACTIVITY_STACKS,
            "setProcessPriority");
        
        synchronized (this) {
            mOomAdjuster.setCustomPriority(packageName, priorityLevel);
        }
    }
    
    /**
     * Get custom process priority for a package.
     * @hide
     */
    public int getProcessPriority(String packageName) {
        // Return priority level
        return mOomAdjuster.getCustomPriority(packageName);
    }
}
```

### Step 4: Command-Line Tool

**File:** `frameworks/base/cmds/am/src/com/android/commands/am/Am.java`

```java
public class Am extends BaseCommand {
    
    private void runSetPriority() {
        String packageName = nextArgRequired();
        String priorityStr = nextArgRequired();
        
        int priority;
        switch (priorityStr.toLowerCase()) {
            case "none":
                priority = ActivityManager.PRIORITY_BOOST_NONE;
                break;
            case "low":
                priority = ActivityManager.PRIORITY_BOOST_LOW;
                break;
            case "medium":
                priority = ActivityManager.PRIORITY_BOOST_MEDIUM;
                break;
            case "high":
                priority = ActivityManager.PRIORITY_BOOST_HIGH;
                break;
            default:
                System.err.println("Unknown priority: " + priorityStr);
                return;
        }
        
        IActivityManager am = ActivityManager.getService();
        am.setProcessPriority(packageName, priority);
        
        System.out.println("Set priority for " + packageName + 
            " to " + priorityStr);
    }
}
```

### Step 5: Usage

```bash
# Set high priority for music player
adb shell am set-priority com.example.musicplayer high

# Set medium priority for messaging app
adb shell am set-priority com.example.messenger medium

# Remove priority boost
adb shell am set-priority com.example.game none

# View current processes and their OOM adj
adb shell dumpsys activity processes
```

## Practical Example 2: Activity Interceptor

Let's add an interceptor that can modify or block activity launches based on custom rules.

### Step 1: Define Interceptor Interface

**File:** `frameworks/base/core/java/android/app/IActivityInterceptor.aidl`

```java
package android.app;

import android.content.Intent;

/**
 * Interface for intercepting activity launches.
 * @hide
 */
interface IActivityInterceptor {
    /**
     * Called before an activity is launched.
     * @return null to allow launch, or replacement intent to redirect
     */
    Intent onActivityLaunch(in Intent intent, String callingPackage, int userId);
}
```

### Step 2: Implement Interceptor in AMS

**File:** `frameworks/base/services/core/java/com/android/server/am/ActivityStarter.java`

```java
public class ActivityStarter {
    
    private IActivityInterceptor mActivityInterceptor;
    
    /**
     * Register activity interceptor
     * @hide
     */
    public void setActivityInterceptor(IActivityInterceptor interceptor) {
        mActivityInterceptor = interceptor;
    }
    
    private int startActivityLocked(...) {
        // ... existing code ...
        
        // Call interceptor if registered
        if (mActivityInterceptor != null) {
            try {
                String callingPackage = r.launchedFromPackage;
                Intent intercepted = mActivityInterceptor.onActivityLaunch(
                    r.intent, callingPackage, userId);
                
                if (intercepted == null) {
                    // Block the launch
                    Slog.i(TAG, "Activity launch blocked by interceptor: " 
                        + r.intent);
                    return ActivityManager.START_CANCELED;
                }
                
                if (intercepted != r.intent) {
                    // Replace intent
                    Slog.i(TAG, "Activity launch redirected by interceptor: " 
                        + r.intent + " -> " + intercepted);
                    r.intent = intercepted;
                    
                    // Re-resolve with new intent
                    resolveActivity(intercepted);
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Activity interceptor failed", e);
            }
        }
        
        // ... continue with launch ...
    }
}
```

### Step 3: Example Interceptor Implementation

**Create a system service that provides interception:**

```java
public class ActivityInterceptorService extends SystemService {
    
    private static final String TAG = "ActivityInterceptorService";
    
    private final ActivityInterceptorImpl mInterceptor = 
        new ActivityInterceptorImpl();
    
    public ActivityInterceptorService(Context context) {
        super(context);
    }
    
    @Override
    public void onStart() {
        // Register with ActivityManager
        IActivityManager am = ActivityManager.getService();
        try {
            am.setActivityInterceptor(mInterceptor);
        } catch (RemoteException e) {
            Slog.e(TAG, "Failed to register interceptor", e);
        }
    }
    
    private class ActivityInterceptorImpl extends IActivityInterceptor.Stub {
        
        @Override
        public Intent onActivityLaunch(Intent intent, String callingPackage,
                int userId) {
            
            // Example 1: Block launches during certain hours
            if (isRestrictedTime()) {
                String action = intent.getAction();
                if (Intent.ACTION_VIEW.equals(action)) {
                    Slog.i(TAG, "Blocked VIEW intent during restricted time");
                    return null;  // Block
                }
            }
            
            // Example 2: Redirect social media apps to productivity app
            if (isSocialMediaApp(intent)) {
                if (isProductivityModeEnabled()) {
                    Slog.i(TAG, "Redirecting social media to productivity app");
                    Intent redirect = new Intent();
                    redirect.setClassName("com.example.productivity",
                        "com.example.productivity.FocusActivity");
                    redirect.putExtra("blocked_app", 
                        intent.getComponent().getPackageName());
                    return redirect;
                }
            }
            
            // Example 3: Log all activity launches
            logActivityLaunch(intent, callingPackage);
            
            // Allow launch
            return intent;
        }
        
        private boolean isRestrictedTime() {
            Calendar cal = Calendar.getInstance();
            int hour = cal.get(Calendar.HOUR_OF_DAY);
            return (hour >= 22 || hour < 6);  // 10 PM to 6 AM
        }
        
        private boolean isSocialMediaApp(Intent intent) {
            ComponentName component = intent.getComponent();
            if (component == null) return false;
            
            String pkg = component.getPackageName();
            return pkg.contains("facebook") || 
                   pkg.contains("twitter") ||
                   pkg.contains("instagram");
        }
        
        private boolean isProductivityModeEnabled() {
            return Settings.System.getInt(
                getContext().getContentResolver(),
                "productivity_mode_enabled", 0) == 1;
        }
        
        private void logActivityLaunch(Intent intent, String callingPackage) {
            Slog.i(TAG, "Activity launch: " + intent.getComponent() +
                " from " + callingPackage);
        }
    }
}
```

### Step 4: Usage

**Enable/disable productivity mode:**

```bash
# Enable productivity mode
adb shell settings put system productivity_mode_enabled 1

# Disable productivity mode
adb shell settings put system productivity_mode_enabled 0

# Try launching social media app - gets redirected
adb shell am start -n com.facebook.katana/.MainActivity
```

## Key Takeaways

1. **AMS manages all application components**: Activities, services, receivers, providers
2. **Process priorities are dynamic**: Based on current component states and user interaction
3. **OOM adjuster prevents out-of-memory**: Calculates priorities, kernel kills lowest
4. **Activity stacks organize navigation**: Tasks, back stack, launch modes
5. **Services have lifecycle and timeouts**: Must respond to start/bind within limits
6. **Broadcasts can be ordered or parallel**: Different delivery guarantees
7. **ANRs detect unresponsive apps**: Input, broadcast, and service timeouts
8. **Customization enables policies**: Priority boosting, launch interception, custom rules

## Next Steps

In Chapter 8, we'll explore System UI and the framework UI layer. You'll learn how SystemUI is structured, how to customize the status bar and quick settings, and how to implement custom UI components at the system level.

## Quick Reference

### Process Management Commands

```bash
# List running processes
adb shell ps -A

# View process details
adb shell dumpsys activity processes

# View OOM adj values
adb shell cat /proc/<pid>/oom_score_adj

# Kill process
adb shell am kill <package>
adb shell am force-stop <package>

# Start activity
adb shell am start -n <package>/<activity>

# Start service
adb shell am startservice <package>/<service>

# Send broadcast
adb shell am broadcast -a <action>
```

### Activity Stack Commands

```bash
# View activity stack
adb shell dumpsys activity activities

# View specific task
adb shell dumpsys activity tasks

# View recents
adb shell dumpsys activity recents

# Move task to front
adb shell am task <task-id> front
```

### Debugging ANRs

```bash
# Trigger ANR (if app frozen)
# ANR traces saved to /data/anr/

# Pull ANR traces
adb pull /data/anr/traces.txt

# View ANR in logcat
adb logcat | grep "ANR in"
```
