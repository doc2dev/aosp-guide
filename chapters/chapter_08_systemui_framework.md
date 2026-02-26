# Chapter 8: System UI & Framework UI

## Contents

- [Introduction](#introduction)
- [SystemUI Architecture](#systemui-architecture)
- [Status Bar Implementation](#status-bar-implementation)
- [Quick Settings (QS)](#quick-settings-qs)
- [Notifications](#notifications)
- [Lock Screen (Keyguard)](#lock-screen-keyguard)
- [Practical Example 1: Adding Custom Quick Settings Tile](#practical-example-1-adding-custom-quick-settings-tile)
- [Practical Example 2: Custom Status Bar Icon](#practical-example-2-custom-status-bar-icon)
- [SystemUI Theming](#systemui-theming)
- [Key Takeaways](#key-takeaways)
- [Next Steps](#next-steps)
- [Quick Reference](#quick-reference)

## Introduction

SystemUI is Android's system-level user interface framework that provides the status bar, notification shade, quick settings, lock screen, and other critical UI elements. Unlike application UI, SystemUI runs as a privileged system application with special permissions and direct access to system services.

Understanding SystemUI is crucial for AOSP development because it:
- Provides the primary interface for system notifications and controls
- Manages lock screen and security UI
- Handles navigation gestures and buttons
- Implements system-wide UI patterns and themes
- Serves as a reference for framework UI implementation

This chapter explores SystemUI's architecture, key components, theming system, and provides practical examples of customizing system UI elements.

## SystemUI Architecture

### Overview

SystemUI is a privileged system application located at `frameworks/base/packages/SystemUI/`:

```
SystemUI Components
├── Status Bar
│   ├── Clock
│   ├── Battery indicator
│   ├── Signal indicators
│   └── Notification icons
├── Notification Shade
│   ├── Notifications
│   ├── Quick Settings
│   └── Media controls
├── Lock Screen
│   ├── Clock & date
│   ├── Notifications
│   └── Security (PIN, pattern, biometric)
├── Navigation
│   ├── Navigation bar
│   ├── Gesture navigation
│   └── Recents/Overview
└── System Overlays
    ├── Volume dialog
    ├── Screenshot UI
    └── Power menu
```

### Source Code Structure

```
frameworks/base/packages/SystemUI/
├── src/com/android/systemui/
│   ├── SystemUIApplication.java       # Application entry point
│   ├── SystemUIService.java           # Main service
│   ├── statusbar/                     # Status bar & notifications
│   │   ├── phone/
│   │   │   ├── StatusBar.java         # Phone status bar
│   │   │   ├── NotificationPanelView.java
│   │   │   └── StatusBarWindowView.java
│   │   ├── notification/
│   │   │   ├── NotificationEntryManager.java
│   │   │   ├── NotificationListener.java
│   │   │   └── collection/
│   │   └── policy/
│   ├── qs/                            # Quick Settings
│   │   ├── QSPanel.java
│   │   ├── QSTileHost.java
│   │   ├── tiles/                     # Individual tiles
│   │   │   ├── WifiTile.java
│   │   │   ├── BluetoothTile.java
│   │   │   └── ...
│   │   └── customize/
│   ├── keyguard/                      # Lock screen
│   │   ├── KeyguardViewMediator.java
│   │   ├── KeyguardSecurityContainer.java
│   │   └── KeyguardUpdateMonitor.java
│   ├── volume/                        # Volume dialog
│   │   └── VolumeDialogImpl.java
│   ├── power/                         # Power menu
│   │   └── PowerUI.java
│   ├── recents/                       # Recent apps
│   │   └── OverviewProxyService.java
│   └── util/                          # Utilities
├── res/                               # Resources
│   ├── layout/                        # XML layouts
│   ├── values/                        # Dimensions, colors, strings
│   └── drawable/                      # Icons and graphics
├── plugin/                            # Plugin framework
└── shared/                            # Shared code
```

### SystemUI Startup

SystemUI is started by SystemServer during boot:

```java
// frameworks/base/services/java/com/android/server/SystemServer.java

private void startOtherServices() {
    // ...
    
    traceBeginAndSlog("StartSystemUI");
    try {
        startSystemUi(context);
    } catch (Throwable e) {
        reportWtf("starting System UI", e);
    }
    traceEnd();
}

private static void startSystemUi(Context context) {
    Intent intent = new Intent();
    intent.setComponent(new ComponentName("com.android.systemui",
            "com.android.systemui.SystemUIService"));
    intent.addFlags(Intent.FLAG_RECEIVER_BOOT_UPGRADE);
    context.startServiceAsUser(intent, UserHandle.SYSTEM);
}
```

**SystemUIApplication initialization:**

```java
// SystemUIApplication.java

public class SystemUIApplication extends Application {
    
    private SystemUI[] mServices;
    private boolean mServicesStarted;
    
    @Override
    public void onCreate() {
        super.onCreate();
        
        // Set default uncaught exception handler
        setDefaultUncaughtExceptionHandler();
        
        // Initialize Dagger dependency injection
        mRootComponent = SystemUIFactory.getInstance()
            .getRootComponent()
            .getSysUIComponent()
            .build();
        
        // Start services
        startServicesIfNeeded();
    }
    
    private void startServicesIfNeeded() {
        String[] names = getResources().getStringArray(
            R.array.config_systemUIServiceComponents);
        
        mServices = new SystemUI[names.length];
        
        for (int i = 0; i < names.length; i++) {
            String className = names[i];
            
            try {
                // Instantiate service
                SystemUI obj = (SystemUI) Class.forName(className)
                    .newInstance();
                
                mServices[i] = obj;
                obj.mContext = this;
                obj.mComponents = mRootComponent;
                
                // Start the service
                obj.start();
                
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
        
        mServicesStarted = true;
    }
}
```

**config_systemUIServiceComponents.xml:**

```xml
<resources>
    <string-array name="config_systemUIServiceComponents">
        <item>com.android.systemui.util.NotificationChannels</item>
        <item>com.android.systemui.keyguard.KeyguardViewMediator</item>
        <item>com.android.systemui.recents.Recents</item>
        <item>com.android.systemui.volume.VolumeUI</item>
        <item>com.android.systemui.statusbar.phone.StatusBar</item>
        <item>com.android.systemui.usb.StorageNotification</item>
        <item>com.android.systemui.power.PowerUI</item>
        <item>com.android.systemui.media.RingtonePlayer</item>
        <item>com.android.systemui.keyboard.KeyboardUI</item>
        <item>com.android.systemui.shortcut.ShortcutKeyDispatcher</item>
        <item>com.android.systemui.LatencyTester</item>
        <item>com.android.systemui.globalactions.GlobalActionsComponent</item>
        <item>com.android.systemui.SliceBroadcastRelayHandler</item>
    </string-array>
</resources>
```

## Status Bar Implementation

### StatusBar Class

The main status bar implementation:

```java
// StatusBar.java

public class StatusBar extends SystemUI implements 
        DemoMode, ActivityStarter {
    
    // Views
    private StatusBarWindowView mStatusBarWindow;
    private PhoneStatusBarView mStatusBarView;
    private NotificationPanelViewController mNotificationPanel;
    
    // State
    private int mState = StatusBarState.SHADE;
    private boolean mBouncerShowing;
    private boolean mExpandedVisible;
    
    // Controllers
    private final NotificationEntryManager mEntryManager;
    private final KeyguardStateController mKeyguardStateController;
    private final CommandQueue mCommandQueue;
    
    @Override
    public void start() {
        // Create window
        createAndAddWindows();
        
        // Register with command queue
        mCommandQueue.addCallback(this);
        
        // Set up notification listener
        mEntryManager.attach(mNotificationListener);
        
        // Initialize status bar icons
        createStatusBarIcons();
        
        // Set up system info (clock, battery, etc.)
        updateSystemInfo();
    }
    
    private void createAndAddWindows() {
        // Inflate status bar window
        mStatusBarWindow = (StatusBarWindowView) LayoutInflater
            .from(mContext)
            .inflate(R.layout.super_status_bar, null);
        
        // Add to window manager
        WindowManager.LayoutParams lp = new WindowManager.LayoutParams(
            ViewGroup.LayoutParams.MATCH_PARENT,
            ViewGroup.LayoutParams.MATCH_PARENT,
            WindowManager.LayoutParams.TYPE_STATUS_BAR,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |
                WindowManager.LayoutParams.FLAG_SPLIT_TOUCH,
            PixelFormat.TRANSLUCENT);
        
        lp.gravity = Gravity.TOP;
        lp.setTitle("StatusBar");
        lp.packageName = mContext.getPackageName();
        
        mWindowManager.addView(mStatusBarWindow, lp);
        
        // Find status bar view
        mStatusBarView = mStatusBarWindow.findViewById(R.id.status_bar);
    }
    
    public void animateExpandNotificationsPanel() {
        if (mState == StatusBarState.KEYGUARD) {
            return; // Can't expand on keyguard
        }
        
        // Animate panel expansion
        mNotificationPanel.expand(true /* animate */);
    }
    
    public void animateCollapsePanels() {
        // Collapse notification panel
        mNotificationPanel.collapse(true /* animate */, 
            false /* delayed */, 1.0f /* speedUpFactor */);
    }
}
```

### Status Bar Layout

**super_status_bar.xml:**

```xml
<com.android.systemui.statusbar.phone.StatusBarWindowView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:fitsSystemWindows="true">

    <!-- Status bar -->
    <com.android.systemui.statusbar.phone.PhoneStatusBarView
        android:id="@+id/status_bar"
        android:layout_width="match_parent"
        android:layout_height="@dimen/status_bar_height"
        android:background="@drawable/system_bar_background">
        
        <!-- Left side -->
        <LinearLayout
            android:id="@+id/status_bar_contents"
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:orientation="horizontal">
            
            <!-- Clock -->
            <com.android.systemui.statusbar.policy.Clock
                android:id="@+id/clock"
                android:layout_width="wrap_content"
                android:layout_height="match_parent"
                android:gravity="center_vertical"
                android:paddingStart="@dimen/status_bar_clock_starting_padding"
                android:textAppearance="@style/TextAppearance.StatusBar.Clock" />
            
            <!-- Notification icons -->
            <com.android.systemui.statusbar.phone.NotificationIconContainer
                android:id="@+id/notificationIcons"
                android:layout_width="0dp"
                android:layout_height="match_parent"
                android:layout_weight="1" />
            
            <!-- System icons (battery, signal, etc.) -->
            <LinearLayout
                android:id="@+id/system_icons"
                android:layout_width="wrap_content"
                android:layout_height="match_parent"
                android:gravity="center_vertical"
                android:orientation="horizontal">
                
                <!-- Battery -->
                <com.android.systemui.battery.BatteryMeterView
                    android:id="@+id/battery"
                    android:layout_width="wrap_content"
                    android:layout_height="match_parent" />
                
            </LinearLayout>
        </LinearLayout>
    </com.android.systemui.statusbar.phone.PhoneStatusBarView>
    
    <!-- Notification panel (pulls down from status bar) -->
    <com.android.systemui.statusbar.phone.NotificationPanelView
        android:id="@+id/notification_panel"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:visibility="gone"
        systemui:layout_maxHeight="@dimen/notification_panel_max_height">
        
        <!-- Quick Settings -->
        <include layout="@layout/qs_panel" />
        
        <!-- Notifications -->
        <com.android.systemui.statusbar.notification.stack.NotificationStackScrollLayout
            android:id="@+id/notification_stack_scroller"
            android:layout_width="match_parent"
            android:layout_height="match_parent" />
        
    </com.android.systemui.statusbar.phone.NotificationPanelView>
    
</com.android.systemui.statusbar.phone.StatusBarWindowView>
```

### Clock Implementation

Custom view for displaying time:

```java
// Clock.java

public class Clock extends TextView implements 
        BatteryController.BatteryStateChangeCallback {
    
    private boolean mAttached;
    private Calendar mCalendar;
    private String mClockFormatString;
    private SimpleDateFormat mClockFormat;
    private Locale mLocale;
    
    private BroadcastReceiver mIntentReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            String action = intent.getAction();
            if (Intent.ACTION_TIME_TICK.equals(action)
                    || Intent.ACTION_TIME_CHANGED.equals(action)
                    || Intent.ACTION_TIMEZONE_CHANGED.equals(action)
                    || Intent.ACTION_LOCALE_CHANGED.equals(action)) {
                updateClock();
            }
        }
    };
    
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        
        if (!mAttached) {
            mAttached = true;
            
            // Register for time changes
            IntentFilter filter = new IntentFilter();
            filter.addAction(Intent.ACTION_TIME_TICK);
            filter.addAction(Intent.ACTION_TIME_CHANGED);
            filter.addAction(Intent.ACTION_TIMEZONE_CHANGED);
            filter.addAction(Intent.ACTION_LOCALE_CHANGED);
            
            getContext().registerReceiver(mIntentReceiver, filter);
            
            // Update immediately
            updateClock();
        }
    }
    
    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        if (mAttached) {
            getContext().unregisterReceiver(mIntentReceiver);
            mAttached = false;
        }
    }
    
    private final void updateClock() {
        if (mCalendar == null) {
            mCalendar = Calendar.getInstance(TimeZone.getDefault());
        }
        
        mCalendar.setTimeInMillis(System.currentTimeMillis());
        
        // Get format
        if (mClockFormat == null || !mLocale.equals(Locale.getDefault())) {
            mLocale = Locale.getDefault();
            mClockFormatString = Settings.System.getString(
                getContext().getContentResolver(),
                Settings.System.TIME_12_24);
            
            if (mClockFormatString == null || "12".equals(mClockFormatString)) {
                mClockFormat = new SimpleDateFormat("h:mm a", mLocale);
            } else {
                mClockFormat = new SimpleDateFormat("HH:mm", mLocale);
            }
        }
        
        // Format and set text
        String text = mClockFormat.format(mCalendar.getTime());
        setText(text);
    }
}
```

## Quick Settings (QS)

### QS Architecture

Quick Settings provides toggles for common system settings:

```
QS Architecture
├── QSPanel
│   ├── QuickQSPanel (top 4-6 tiles)
│   └── QSTileLayout (full grid)
├── QSTileHost
│   ├── Manages tile lifecycle
│   └── Creates tile instances
└── QSTile implementations
    ├── WifiTile
    ├── BluetoothTile
    ├── FlashlightTile
    └── ... (many more)
```

### QSTile Base Class

```java
// QSTile.java

public abstract class QSTile<TState extends State> {
    
    protected final QSHost mHost;
    protected final Context mContext;
    protected final Handler mHandler;
    
    private TState mState;
    private TState mTmpState;
    
    public QSTile(QSHost host) {
        mHost = host;
        mContext = host.getContext();
        mHandler = new Handler(Looper.getMainLooper());
        
        // Create initial state
        mState = newTileState();
        mTmpState = newTileState();
    }
    
    /**
     * Create the state object for this tile.
     */
    public abstract TState newTileState();
    
    /**
     * Called when user clicks the tile.
     */
    protected abstract void handleClick();
    
    /**
     * Called when user long-clicks the tile.
     */
    protected void handleLongClick() {
        // Default: launch detail view
        showDetail(true);
    }
    
    /**
     * Update the tile state.
     */
    protected abstract void handleUpdateState(TState state, Object arg);
    
    /**
     * Get the intent to launch when tile is clicked.
     * Returning null means handle click in handleClick().
     */
    public Intent getLongClickIntent() {
        return null;
    }
    
    /**
     * Refresh the tile.
     */
    public void refreshState() {
        handleUpdateState(mTmpState, null);
        
        if (!mTmpState.equals(mState)) {
            // State changed, update UI
            mState.copyFrom(mTmpState);
            handleStateChanged();
        }
    }
    
    private void handleStateChanged() {
        mHandler.post(() -> {
            // Notify listeners
            for (Callback callback : mCallbacks) {
                callback.onStateChanged(mState);
            }
        });
    }
    
    /**
     * Called when tile is added.
     */
    public void setListening(Object client, boolean listening) {
        if (listening) {
            handleSetListening(listening);
        }
    }
    
    protected void handleSetListening(boolean listening) {
        // Override to register/unregister listeners
    }
    
    public static class State {
        public String label;
        public CharSequence secondaryLabel;
        public Icon icon;
        public int state = Tile.STATE_ACTIVE;
        
        public boolean equals(Object other) {
            if (other == null || other.getClass() != getClass()) {
                return false;
            }
            State o = (State) other;
            return Objects.equals(label, o.label)
                && Objects.equals(secondaryLabel, o.secondaryLabel)
                && Objects.equals(icon, o.icon)
                && state == o.state;
        }
    }
    
    public interface Callback {
        void onStateChanged(State state);
    }
}
```

### Example: Flashlight Tile

```java
// FlashlightTile.java

public class FlashlightTile extends QSTile<BooleanState> {
    
    private final FlashlightController mFlashlightController;
    
    public FlashlightTile(QSHost host) {
        super(host);
        mFlashlightController = host.getFlashlightController();
    }
    
    @Override
    public BooleanState newTileState() {
        return new BooleanState();
    }
    
    @Override
    protected void handleClick() {
        // Toggle flashlight
        boolean newState = !mState.value;
        mFlashlightController.setFlashlight(newState);
        refreshState(newState);
    }
    
    @Override
    protected void handleUpdateState(BooleanState state, Object arg) {
        // Check if flashlight is available
        state.value = mFlashlightController.isEnabled();
        state.label = mContext.getString(R.string.quick_settings_flashlight_label);
        
        if (!mFlashlightController.isAvailable()) {
            state.icon = ResourceIcon.get(R.drawable.ic_signal_flashlight_disable);
            state.state = Tile.STATE_UNAVAILABLE;
            state.secondaryLabel = mContext.getString(
                R.string.quick_settings_flashlight_unavailable);
        } else {
            state.icon = ResourceIcon.get(
                state.value ? R.drawable.ic_signal_flashlight_enable
                           : R.drawable.ic_signal_flashlight_disable);
            state.state = state.value ? Tile.STATE_ACTIVE : Tile.STATE_INACTIVE;
            state.secondaryLabel = null;
        }
    }
    
    @Override
    protected void handleSetListening(boolean listening) {
        if (listening) {
            mFlashlightController.addCallback(mCallback);
        } else {
            mFlashlashController.removeCallback(mCallback);
        }
    }
    
    private final FlashlightController.FlashlightListener mCallback = 
            new FlashlightController.FlashlightListener() {
        @Override
        public void onFlashlightChanged(boolean enabled) {
            refreshState(enabled);
        }
        
        @Override
        public void onFlashlightError() {
            refreshState();
        }
        
        @Override
        public void onFlashlightAvailabilityChanged(boolean available) {
            refreshState();
        }
    };
}
```

## Notifications

### Notification Pipeline

```
Notification Flow
├── App posts notification
│   ↓
├── NotificationManagerService receives
│   ↓
├── NotificationListener (in SystemUI) receives
│   ↓
├── NotificationEntryManager processes
│   ↓
├── NotificationData stores
│   ↓
└── NotificationStackScrollLayout displays
```

### NotificationListener

SystemUI listens for notifications:

```java
// NotificationListener.java

public class NotificationListener extends NotificationListenerService {
    
    @Override
    public void onNotificationPosted(StatusBarNotification sbn) {
        // New notification posted
        mEntryManager.addNotification(sbn);
    }
    
    @Override
    public void onNotificationRemoved(StatusBarNotification sbn) {
        // Notification removed
        mEntryManager.removeNotification(sbn.getKey());
    }
    
    @Override
    public void onNotificationRankingUpdate(RankingMap rankingMap) {
        // Ranking changed (order, importance, etc.)
        mEntryManager.updateNotificationRanking(rankingMap);
    }
}
```

### NotificationEntry

Represents a single notification:

```java
// NotificationEntry.java

public final class NotificationEntry {
    public final String key;
    public StatusBarNotification sbn;
    public NotificationChannel channel;
    
    // Views
    public ExpandableNotificationRow row;
    
    // State
    public boolean isClearable;
    public boolean isTopLevelChild;
    public int targetSdk;
    
    // Ranking
    public int importance = NotificationManager.IMPORTANCE_DEFAULT;
    public CharSequence rankingExplanation;
    
    // User interaction
    public long lastFullScreenIntentLaunchTime;
    public long lastAudibleAlertTime;
    
    // Grouping
    public NotificationEntry parent;
    public List<NotificationEntry> children;
    
    public boolean isHighPriority() {
        return importance >= NotificationManager.IMPORTANCE_DEFAULT;
    }
    
    public boolean isAmbient() {
        return importance < NotificationManager.IMPORTANCE_DEFAULT;
    }
}
```

## Lock Screen (Keyguard)

### Keyguard Architecture

```
Keyguard Components
├── KeyguardViewMediator
│   └── Mediates between system and keyguard UI
├── KeyguardViewController
│   └── Manages keyguard view hierarchy
├── KeyguardSecurityContainer
│   ├── PIN entry
│   ├── Pattern entry
│   ├── Password entry
│   └── Biometric authentication
└── KeyguardUpdateMonitor
    └── Monitors system state (battery, SIM, etc.)
```

### KeyguardViewMediator

Main keyguard controller:

```java
// KeyguardViewMediator.java

public class KeyguardViewMediator extends SystemUI {
    
    private static final int KEYGUARD_TIMEOUT = 10000;  // 10 seconds
    
    private boolean mShowing;
    private boolean mOccluded;
    private boolean mDeviceInteractive;
    
    private KeyguardViewController mKeyguardViewController;
    private final KeyguardUpdateMonitor mUpdateMonitor;
    
    @Override
    public void start() {
        // Setup keyguard
        setupLocked();
        
        // Register callbacks
        mUpdateMonitor.registerCallback(mUpdateCallback);
        
        // Handle initial state
        doKeyguardLocked(null);
    }
    
    private void doKeyguardLocked(Bundle options) {
        // Don't show if disabled
        if (!mEnabled) {
            return;
        }
        
        // Show keyguard
        mShowing = true;
        mKeyguardViewController.show(options);
        
        // Update states
        adjustStatusBarLocked();
        
        // Set timeout
        mHandler.sendEmptyMessageDelayed(KEYGUARD_TIMEOUT_MSG, 
            KEYGUARD_TIMEOUT);
    }
    
    /**
     * Handle user authentication success
     */
    private void handleKeyguardDone() {
        // Unlock
        mShowing = false;
        mKeyguardViewController.hide(0, 0);
        
        // Notify system
        mUpdateMonitor.clearBiometricRecognized();
        
        // Show launcher
        ActivityManager.getService().resumeAppSwitches();
    }
    
    /**
     * Called when device goes to sleep
     */
    public void onStartedGoingToSleep(int why) {
        // Show keyguard
        mHandler.post(() -> {
            doKeyguardLocked(null);
        });
    }
    
    /**
     * Called when device wakes up
     */
    public void onStartedWakingUp() {
        // Update display state
        mKeyguardViewController.onStartedWakingUp();
    }
    
    private final KeyguardUpdateMonitorCallback mUpdateCallback = 
            new KeyguardUpdateMonitorCallback() {
        
        @Override
        public void onUserSwitching(int userId) {
            // Reset keyguard for new user
            resetStateLocked();
        }
        
        @Override
        public void onBiometricAuthenticated(int userId, 
                BiometricSourceType biometricSourceType) {
            // Biometric auth succeeded
            if (mShowing) {
                handleKeyguardDone();
            }
        }
        
        @Override
        public void onSimStateChanged(int subId, int slotId, 
                TelephonyManager.SimState simState) {
            // SIM state changed
            if (simState == TelephonyManager.SIM_STATE_ABSENT) {
                // Show SIM missing notification
            }
        }
    };
}
```

## Practical Example 1: Adding Custom Quick Settings Tile

Let's create a custom QS tile that controls a system feature.

### Step 1: Create Custom Tile Class

**File:** `frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tiles/CustomFeatureTile.java`

```java
package com.android.systemui.qs.tiles;

import android.content.Intent;
import android.provider.Settings;
import android.service.quicksettings.Tile;

import com.android.internal.logging.MetricsLogger;
import com.android.internal.logging.nano.MetricsProto.MetricsEvent;
import com.android.systemui.R;
import com.android.systemui.plugins.qs.QSTile.BooleanState;
import com.android.systemui.qs.QSHost;
import com.android.systemui.qs.tileimpl.QSTileImpl;

/**
 * Custom feature toggle tile
 */
public class CustomFeatureTile extends QSTileImpl<BooleanState> {
    
    private static final String SETTING_KEY = "custom_feature_enabled";
    
    private final Icon mIcon = ResourceIcon.get(
        R.drawable.ic_qs_custom_feature);
    
    public CustomFeatureTile(QSHost host) {
        super(host);
    }
    
    @Override
    public BooleanState newTileState() {
        BooleanState state = new BooleanState();
        state.handlesLongClick = true;
        return state;
    }
    
    @Override
    protected void handleClick() {
        // Toggle setting
        boolean newState = !mState.value;
        setEnabled(newState);
        refreshState(newState);
    }
    
    @Override
    protected void handleLongClick() {
        // Open settings page
        Intent intent = new Intent(Settings.ACTION_SETTINGS);
        intent.putExtra("extra_fragment_arg_key", "custom_feature");
        mHost.startActivityDismissingKeyguard(intent);
    }
    
    @Override
    protected void handleUpdateState(BooleanState state, Object arg) {
        // Get current state
        final boolean enabled = isEnabled();
        
        if (state.slash == null) {
            state.slash = new SlashState();
        }
        
        state.value = enabled;
        state.label = mContext.getString(R.string.quick_settings_custom_feature_label);
        state.icon = mIcon;
        state.slash.isSlashed = !enabled;
        
        if (enabled) {
            state.state = Tile.STATE_ACTIVE;
            state.secondaryLabel = mContext.getString(
                R.string.quick_settings_custom_feature_on);
        } else {
            state.state = Tile.STATE_INACTIVE;
            state.secondaryLabel = mContext.getString(
                R.string.quick_settings_custom_feature_off);
        }
        
        state.contentDescription = state.label;
        state.expandedAccessibilityClassName = Switch.class.getName();
    }
    
    @Override
    public int getMetricsCategory() {
        return MetricsEvent.QS_CUSTOM_FEATURE;
    }
    
    @Override
    public Intent getLongClickIntent() {
        return new Intent(Settings.ACTION_SETTINGS)
            .putExtra("extra_fragment_arg_key", "custom_feature");
    }
    
    @Override
    public CharSequence getTileLabel() {
        return mContext.getString(R.string.quick_settings_custom_feature_label);
    }
    
    private boolean isEnabled() {
        return Settings.System.getInt(mContext.getContentResolver(),
            SETTING_KEY, 0) != 0;
    }
    
    private void setEnabled(boolean enabled) {
        Settings.System.putInt(mContext.getContentResolver(),
            SETTING_KEY, enabled ? 1 : 0);
        
        // Notify system of change
        Intent intent = new Intent("com.android.systemui.CUSTOM_FEATURE_CHANGED");
        intent.putExtra("enabled", enabled);
        mContext.sendBroadcast(intent);
    }
}
```

### Step 2: Register Tile

**File:** `frameworks/base/packages/SystemUI/src/com/android/systemui/qs/tileimpl/QSFactoryImpl.java`

```java
public class QSFactoryImpl implements QSFactory {
    
    @Override
    public QSTile createTile(String tileSpec) {
        // ... existing tiles ...
        
        switch (tileSpec) {
            // ... existing cases ...
            
            case "custom_feature":
                return new CustomFeatureTile(mHost);
                
            default:
                return null;
        }
    }
}
```

### Step 3: Add Tile to Default Set

**File:** `frameworks/base/packages/SystemUI/res/values/config.xml`

```xml
<resources>
    <!-- Quick Settings tiles -->
    <string name="quick_settings_tiles_default" translatable="false">
        wifi,bt,dnd,flashlight,rotation,battery,cell,airplane,cast,custom_feature
    </string>
</resources>
```

### Step 4: Add Resources

**File:** `frameworks/base/packages/SystemUI/res/values/strings.xml`

```xml
<resources>
    <!-- Custom Feature QS Tile -->
    <string name="quick_settings_custom_feature_label">Custom Feature</string>
    <string name="quick_settings_custom_feature_on">On</string>
    <string name="quick_settings_custom_feature_off">Off</string>
</resources>
```

**File:** `frameworks/base/packages/SystemUI/res/drawable/ic_qs_custom_feature.xml`

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="24dp"
    android:height="24dp"
    android:viewportWidth="24"
    android:viewportHeight="24">
    <path
        android:fillColor="@android:color/white"
        android:pathData="M12,2C6.48,2 2,6.48 2,12s4.48,10 10,10 10,-4.48 10,-10S17.52,2 12,2zM13,17h-2v-6h2v6zM13,9h-2L11,7h2v2z"/>
</vector>
```

### Step 5: Build and Test

```bash
# Build SystemUI
m SystemUI

# Push to device
adb root && adb remount
adb push $ANDROID_PRODUCT_OUT/system/priv-app/SystemUI/SystemUI.apk \
    /system/priv-app/SystemUI/

# Restart SystemUI
adb shell killall com.android.systemui

# Or reboot
adb reboot

# Add tile via adb (for testing)
adb shell settings put secure sysui_qs_tiles \
    "wifi,bt,dnd,flashlight,rotation,battery,cell,airplane,cast,custom_feature"
```

## Practical Example 2: Custom Status Bar Icon

Let's add a custom status bar icon that shows system state.

### Step 1: Create Custom Status Bar Icon

**File:** `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/CustomStatusBarIcon.java`

```java
package com.android.systemui.statusbar.phone;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.IntentFilter;
import android.graphics.drawable.Drawable;
import android.util.AttributeSet;
import android.widget.ImageView;

import com.android.systemui.R;

/**
 * Custom status bar icon that shows when feature is active
 */
public class CustomStatusBarIcon extends ImageView {
    
    private boolean mAttached;
    private boolean mFeatureActive;
    
    public CustomStatusBarIcon(Context context, AttributeSet attrs) {
        super(context, attrs);
        setImageResource(R.drawable.stat_sys_custom_feature);
    }
    
    @Override
    protected void onAttachedToWindow() {
        super.onAttachedToWindow();
        
        if (!mAttached) {
            mAttached = true;
            
            // Register for feature state changes
            IntentFilter filter = new IntentFilter();
            filter.addAction("com.android.systemui.CUSTOM_FEATURE_CHANGED");
            getContext().registerReceiver(mReceiver, filter);
            
            // Update initial state
            updateState();
        }
    }
    
    @Override
    protected void onDetachedFromWindow() {
        super.onDetachedFromWindow();
        if (mAttached) {
            getContext().unregisterReceiver(mReceiver);
            mAttached = false;
        }
    }
    
    private void updateState() {
        // Check if feature is active
        mFeatureActive = isFeatureActive();
        
        // Update visibility and appearance
        setVisibility(mFeatureActive ? VISIBLE : GONE);
        
        if (mFeatureActive) {
            // Pulse animation when active
            animate()
                .alpha(0.3f)
                .setDuration(500)
                .withEndAction(() -> {
                    animate().alpha(1.0f).setDuration(500).start();
                })
                .start();
        }
    }
    
    private boolean isFeatureActive() {
        return Settings.System.getInt(getContext().getContentResolver(),
            "custom_feature_enabled", 0) != 0;
    }
    
    private final BroadcastReceiver mReceiver = new BroadcastReceiver() {
        @Override
        public void onReceive(Context context, Intent intent) {
            if ("com.android.systemui.CUSTOM_FEATURE_CHANGED".equals(
                    intent.getAction())) {
                updateState();
            }
        }
    };
}
```

### Step 2: Add to Status Bar Layout

**File:** `frameworks/base/packages/SystemUI/res/layout/status_bar.xml`

```xml
<com.android.systemui.statusbar.phone.PhoneStatusBarView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:systemui="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="@dimen/status_bar_height">
    
    <LinearLayout
        android:id="@+id/status_bar_contents"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="horizontal">
        
        <!-- ... existing elements ... -->
        
        <!-- Custom icon (before battery) -->
        <com.android.systemui.statusbar.phone.CustomStatusBarIcon
            android:id="@+id/custom_feature_icon"
            android:layout_width="@dimen/status_bar_icon_size"
            android:layout_height="match_parent"
            android:paddingStart="2dp"
            android:paddingEnd="2dp"
            android:visibility="gone" />
        
        <!-- ... rest of layout ... -->
        
    </LinearLayout>
</com.android.systemui.statusbar.phone.PhoneStatusBarView>
```

### Step 3: Create Icon Drawable

**File:** `frameworks/base/packages/SystemUI/res/drawable/stat_sys_custom_feature.xml`

```xml
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="18dp"
    android:height="18dp"
    android:viewportWidth="24"
    android:viewportHeight="24"
    android:tint="?android:attr/colorControlNormal">
    <path
        android:fillColor="@android:color/white"
        android:pathData="M12,2l-5.5,9h11L12,2zM12,22c3.31,0 6,-2.69 6,-6s-2.69,-6 -6,-6 -6,2.69 -6,6 2.69,6 6,6z"/>
</vector>
```

### Step 4: Initialize in StatusBar

**File:** `frameworks/base/packages/SystemUI/src/com/android/systemui/statusbar/phone/StatusBar.java`

```java
public class StatusBar extends SystemUI {
    
    private CustomStatusBarIcon mCustomFeatureIcon;
    
    @Override
    public void start() {
        // ... existing initialization ...
        
        createAndAddWindows();
        
        // Find custom icon
        mCustomFeatureIcon = mStatusBarWindow.findViewById(
            R.id.custom_feature_icon);
        
        // ... rest of initialization ...
    }
}
```

## SystemUI Theming

### Theme Resources

SystemUI supports theming through resource overlays:

**colors.xml:**
```xml
<resources>
    <!-- Status bar background -->
    <color name="system_bar_background_opaque">@android:color/black</color>
    <color name="system_bar_background_semi_transparent">#66000000</color>
    
    <!-- Quick settings -->
    <color name="qs_background_dark">#FF1A1A1A</color>
    <color name="qs_tile_background">#FF2A2A2A</color>
    
    <!-- Notification panel -->
    <color name="notification_panel_solid_background">#FF000000</color>
</resources>
```

**styles.xml:**
```xml
<resources>
    <style name="TextAppearance.StatusBar.Clock" parent="@android:style/TextAppearance.StatusBar.Icon">
        <item name="android:textSize">@dimen/status_bar_clock_size</item>
        <item name="android:fontFamily">sans-serif</item>
        <item name="android:textColor">?android:attr/textColorPrimary</item>
    </style>
</resources>
```

### Runtime Theme Changes

SystemUI responds to theme changes:

```java
// ThemeAccentUtils.java

public class ThemeAccentUtils implements Dumpable {
    
    private final UiModeManager mUiModeManager;
    private final OverlayManager mOverlayManager;
    
    public void setAccentColor(int color) {
        // Apply accent color
        String overlayPackage = getOverlayPackageForColor(color);
        
        try {
            mOverlayManager.setEnabled(overlayPackage, true, 
                UserHandle.USER_CURRENT);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to enable overlay", e);
        }
    }
    
    public void setNightMode(boolean enabled) {
        mUiModeManager.setNightMode(enabled ? 
            UiModeManager.MODE_NIGHT_YES : UiModeManager.MODE_NIGHT_NO);
    }
}
```

## Key Takeaways

1. **SystemUI is a privileged system app**: Runs with system permissions and direct service access
2. **Modular architecture**: Separate components for status bar, QS, lock screen, etc.
3. **QS tiles are extensible**: Easy to add custom toggles
4. **Notifications flow through listener**: SystemUI receives and displays all notifications
5. **Keyguard mediates lock screen**: Controls security and display state
6. **Theming uses resource overlays**: Runtime theme changes without recompilation
7. **Custom UI requires system build**: Can't add SystemUI features via regular apps

## Next Steps

In Chapter 9, we'll explore the Hardware Abstraction Layer (HAL). You'll learn how Android interfaces with hardware, how to write HAL implementations, and how to integrate custom hardware into the Android stack.

## Quick Reference

### SystemUI Build Commands

```bash
# Build SystemUI
m SystemUI

# Push to device
adb root && adb remount
adb push $ANDROID_PRODUCT_OUT/system/priv-app/SystemUI/SystemUI.apk \
    /system/priv-app/SystemUI/

# Restart SystemUI
adb shell killall com.android.systemui

# Or via am
adb shell am crash com.android.systemui
```

### SystemUI Debugging

```bash
# Dump SystemUI state
adb shell dumpsys activity service SystemUIService

# Dump status bar
adb shell dumpsys statusbar

# Dump notification state
adb shell dumpsys notification

# View SystemUI logs
adb logcat -s SystemUI:V
```

### QS Tile Management

```bash
# Get current tiles
adb shell settings get secure sysui_qs_tiles

# Set tiles
adb shell settings put secure sysui_qs_tiles \
    "wifi,bt,dnd,flashlight,rotation,battery"

# Reset to default
adb shell settings delete secure sysui_qs_tiles
```

### Common SystemUI Customizations

```java
// Status bar height
<dimen name="status_bar_height">24dp</dimen>

// QS tile size
<dimen name="qs_tile_height">88dp</dimen>

// Notification icon size
<dimen name="status_bar_icon_size">17dp</dimen>

// Clock format (in StatusBar.java)
Settings.System.TIME_12_24

// Show/hide elements
View.setVisibility(View.VISIBLE / View.GONE)
```
