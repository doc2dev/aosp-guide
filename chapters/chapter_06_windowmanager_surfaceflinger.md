# Chapter 6: WindowManager & SurfaceFlinger

## Introduction

The display subsystem in Android is one of its most sophisticated components. Every pixel you see on screen passes through a complex pipeline involving WindowManager (for window management and layout) and SurfaceFlinger (for compositing and display). Understanding this system is crucial for:

- Creating custom window types and behaviors
- Optimizing display performance
- Implementing custom animations and transitions
- Debugging rendering issues
- Building custom UI frameworks or launchers

This chapter explores WindowManager's architecture, SurfaceFlinger's composition pipeline, the relationship between surfaces and windows, and provides practical examples of customizing window behavior.

## Display Architecture Overview

Android's display architecture consists of multiple layers:

```
┌────────────────────────────────────────────────┐
│            Application Process                  │
│  ┌──────────────────────────────────────────┐  │
│  │           View Hierarchy                  │  │
│  │  (Canvas drawing, UI toolkit)            │  │
│  └────────────────┬─────────────────────────┘  │
│                   │                             │
│  ┌────────────────▼─────────────────────────┐  │
│  │          Surface (Buffer Queue)          │  │
│  │     (Double/Triple buffering)            │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │ IGraphicBufferProducer (Binder)
┌───────────────────▼─────────────────────────────┐
│          System Server Process                  │
│  ┌──────────────────────────────────────────┐  │
│  │      WindowManagerService (WMS)          │  │
│  │  - Window policy & layout                │  │
│  │  - Input event routing                   │  │
│  │  - Animation management                  │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │ Surface allocation
┌───────────────────▼─────────────────────────────┐
│         SurfaceFlinger Process                  │
│  ┌──────────────────────────────────────────┐  │
│  │          SurfaceFlinger                  │  │
│  │  - Surface composition                   │  │
│  │  - Hardware composer (HWC) integration   │  │
│  │  - VSync coordination                    │  │
│  └────────────────┬─────────────────────────┘  │
└───────────────────┼─────────────────────────────┘
                    │
┌───────────────────▼─────────────────────────────┐
│              Hardware Layer                     │
│  ┌──────────────────────────────────────────┐  │
│  │     Display Hardware (GPU/DPU)           │  │
│  │  - Hardware composition (if available)   │  │
│  │  - Display pipeline                      │  │
│  └──────────────────────────────────────────┘  │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │          Physical Display                │  │
│  └──────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

### Key Components

**1. WindowManagerService (WMS):**
- Manages window lifecycle and hierarchy
- Handles window layout and positioning
- Routes input events to appropriate windows
- Manages animations and transitions

**2. SurfaceFlinger:**
- Composites surfaces from all applications
- Manages display hardware
- Synchronizes with VSync
- Handles multiple displays

**3. Surface:**
- Drawable area backed by buffer queue
- Applications render to surfaces
- SurfaceFlinger reads from surfaces

**4. Hardware Composer (HWC):**
- HAL for display hardware
- Offloads composition to hardware when possible
- Manages display properties

## WindowManagerService Architecture

### Source Code Locations

```
frameworks/base/services/core/java/com/android/server/wm/
├── WindowManagerService.java        # Main service
├── WindowState.java                 # Represents a window
├── DisplayContent.java              # Per-display management
├── WindowContainer.java             # Container hierarchy
├── Task.java                        # Task management
├── ActivityRecord.java              # Activity windows
├── RootWindowContainer.java         # Root of window hierarchy
├── DisplayPolicy.java               # Display-specific policy
├── WindowSurfacePlacer.java         # Surface placement
├── WindowAnimator.java              # Animation system
└── InputMonitor.java                # Input event routing
```

### Window Hierarchy

Windows are organized in a tree structure:

```
RootWindowContainer
└── DisplayContent (per physical display)
    └── WindowContainer
        ├── TaskDisplayArea
        │   └── Task
        │       └── ActivityRecord
        │           └── WindowState (app windows)
        ├── ImeContainer
        │   └── WindowState (IME window)
        └── WindowToken (non-activity windows)
            └── WindowState (system windows)
```

### Window Types

Windows are categorized by type (z-order):

**Application Windows (1-99):**
```java
TYPE_BASE_APPLICATION = 1       // Base application window
TYPE_APPLICATION = 2            // Normal application window
TYPE_APPLICATION_STARTING = 3   // Starting window (splash)
```

**Sub-Windows (1000-1999):**
```java
TYPE_APPLICATION_PANEL = 1000          // Panel window
TYPE_APPLICATION_MEDIA = 1001          // Media window
TYPE_APPLICATION_SUB_PANEL = 1002      // Sub-panel
TYPE_APPLICATION_ATTACHED_DIALOG = 1003 // Attached dialog
```

**System Windows (2000-2999):**
```java
TYPE_STATUS_BAR = 2000              // Status bar
TYPE_SEARCH_BAR = 2001              // Search bar
TYPE_PHONE = 2002                   // Phone window
TYPE_SYSTEM_ALERT = 2003            // System alert
TYPE_KEYGUARD = 2004                // Keyguard
TYPE_TOAST = 2005                   // Toast
TYPE_SYSTEM_OVERLAY = 2006          // System overlay
TYPE_PRIORITY_PHONE = 2007          // Priority phone
TYPE_SYSTEM_DIALOG = 2008           // System dialog
TYPE_KEYGUARD_DIALOG = 2009         // Keyguard dialog
TYPE_SYSTEM_ERROR = 2010            // System error
TYPE_INPUT_METHOD = 2011            // Input method
TYPE_INPUT_METHOD_DIALOG = 2012     // IME dialog
TYPE_WALLPAPER = 2013               // Wallpaper
TYPE_STATUS_BAR_PANEL = 2014        // Status bar panel
TYPE_NAVIGATION_BAR = 2019          // Navigation bar
TYPE_APPLICATION_OVERLAY = 2038     // App overlay (API 26+)
```

Higher type numbers appear above lower numbers.

### WindowState

`WindowState` represents a single window:

```java
class WindowState extends WindowContainer<WindowState> {
    final WindowManagerService mService;
    final Session mSession;              // Client session
    final IWindow mClient;               // Client interface
    final WindowToken mToken;            // Window token
    final WindowManager.LayoutParams mAttrs;  // Layout parameters
    
    // Position and size
    final Rect mFrame = new Rect();      // Position in screen coordinates
    final Rect mContentInsets = new Rect(); // Content insets
    
    // Surface
    SurfaceControl mSurfaceControl;      // Surface for this window
    
    // State
    boolean mHasSurface;
    boolean mVisible;
    boolean mViewVisibility;
    int mRequestedWidth;
    int mRequestedHeight;
    
    // Z-order
    int mBaseLayer;
    int mSubLayer;
    
    // Input
    InputWindowHandle mInputWindowHandle;
    
    // ... methods for layout, drawing, input ...
}
```

### Window Layout Process

Layout happens in multiple phases:

**1. Measure Phase:**
```java
// Determine desired size for windows
void performLayout() {
    for (WindowState win : mWindows) {
        // Calculate requested dimensions
        win.computeFrameLw();
    }
}
```

**2. Layout Phase:**
```java
// Assign actual positions and sizes
void assignWindowLayers() {
    int curBaseLayer = 0;
    for (WindowState win : mWindows) {
        win.mBaseLayer = curBaseLayer;
        curBaseLayer += TYPE_LAYER_MULTIPLIER;
    }
}
```

**3. Surface Update:**
```java
// Update surface properties
void updateSurfacePosition() {
    if (mSurfaceControl != null) {
        mSurfaceControl.setPosition(mFrame.left, mFrame.top);
        mSurfaceControl.setSize(mFrame.width(), mFrame.height());
        mSurfaceControl.setLayer(mLayer);
    }
}
```

### Input Event Routing

WMS routes input events to the appropriate window:

```java
// Find target window for touch event
WindowState findTouchedWindow(int x, int y) {
    // Iterate from top to bottom (highest z-order first)
    for (int i = mWindows.size() - 1; i >= 0; i--) {
        WindowState win = mWindows.get(i);
        
        // Skip invisible or non-touchable windows
        if (!win.canReceiveKeys() || !win.isVisible()) {
            continue;
        }
        
        // Check if touch is within window bounds
        if (win.mFrame.contains(x, y)) {
            // Check for obscured region
            if (win.isObscuredLocked(x, y)) {
                continue;
            }
            return win;
        }
    }
    return null;
}
```

## SurfaceFlinger Architecture

### Source Code Locations

```
frameworks/native/services/surfaceflinger/
├── SurfaceFlinger.cpp              # Main compositor
├── Layer.cpp                        # Represents a surface layer
├── DisplayDevice.cpp                # Display abstraction
├── RenderEngine/                    # GPU rendering
│   ├── RenderEngine.cpp
│   └── GLES20RenderEngine.cpp
├── Scheduler/                       # VSync scheduling
│   └── Scheduler.cpp
├── CompositionEngine/               # Composition logic
│   └── CompositionEngine.cpp
└── BufferQueue/                     # Buffer management
    ├── BufferQueue.cpp
    └── BufferQueueProducer.cpp
```

### Surface and BufferQueue

A surface is backed by a BufferQueue:

```
Application                     SurfaceFlinger
    │                                │
    │  dequeueBuffer()              │
    ├──────────────────────────────►│
    │◄────────────────────────────┤ │ (returns buffer)
    │                              │ │
    │  <render to buffer>          │ │
    │                              │ │
    │  queueBuffer()               │ │
    ├──────────────────────────────►│
    │                              │ │
    │                              │ │ acquireBuffer()
    │                              │ │ <composite>
    │                              │ │ releaseBuffer()
    │◄────────────────────────────┤ │
```

**BufferQueue workflow:**

1. **Application requests buffer**: `dequeueBuffer()`
2. **Application renders**: Draw to buffer using GPU/CPU
3. **Application submits**: `queueBuffer()`
4. **SurfaceFlinger acquires**: `acquireBuffer()`
5. **SurfaceFlinger composites**: Combine with other layers
6. **SurfaceFlinger releases**: `releaseBuffer()` (back to pool)

### Layer

`Layer` represents a surface in SurfaceFlinger:

```cpp
class Layer : public RefBase {
public:
    // Buffer management
    sp<GraphicBuffer> mActiveBuffer;
    BufferQueue mBufferQueue;
    
    // Transform and position
    Transform mTransform;
    Rect mBounds;
    Region mVisibleRegion;
    
    // Properties
    float mAlpha;
    uint32_t mZ;  // Z-order
    
    // Composition
    HWComposer::LayerType mCompositionType;
    
    // Methods
    void setBuffer(sp<GraphicBuffer> buffer);
    void setPosition(float x, float y);
    void setSize(uint32_t w, uint32_t h);
    void setLayer(uint32_t z);
    void setAlpha(float alpha);
    
    // Called during composition
    void draw(RenderEngine& engine);
};
```

### Composition Process

SurfaceFlinger composites layers every frame (typically 60 FPS):

```cpp
void SurfaceFlinger::onMessageRefresh() {
    // 1. Begin frame
    preComposition();
    
    // 2. Rebuild layer list
    rebuildLayerStacks();
    
    // 3. Calculate visible regions
    calculateVisibleRegions();
    
    // 4. Prepare hardware composer
    for (auto& display : mDisplays) {
        display->prepareFrame();
    }
    
    // 5. Compose (GPU or HWC)
    doComposition();
    
    // 6. Post to display
    postComposition();
}

void SurfaceFlinger::doComposition() {
    for (auto& display : mDisplays) {
        const Vector<Layer*>& layers = display->getVisibleLayersSortedByZ();
        
        // Try hardware composition first
        if (display->getHwComposer().canDoComposition(layers)) {
            display->getHwComposer().presentDisplay();
        } else {
            // Fall back to GPU composition
            doDisplayComposition(display, layers);
        }
    }
}

void SurfaceFlinger::doDisplayComposition(
        const sp<DisplayDevice>& display,
        const Vector<Layer*>& layers) {
    
    RenderEngine& engine = getRenderEngine();
    
    // Set up framebuffer
    engine.setDisplaySurface(display->getDisplaySurface());
    engine.setViewportAndProjection(display->getWidth(), 
                                    display->getHeight());
    
    // Clear
    engine.clearWithColor(0, 0, 0, 1);
    
    // Draw each layer
    for (auto* layer : layers) {
        if (layer->isVisible()) {
            layer->draw(engine);
        }
    }
    
    // Present
    engine.finish();
    display->swapBuffers();
}
```

### VSync Coordination

VSync synchronizes rendering with display refresh:

```
Display: ────┐   ┌────┐   ┌────┐   ┌────
             │   │    │   │    │   │
VSync:       ▼   ▼    ▼   ▼    ▼   ▼
             
App:         [Render][Render][Render]
                 │      │      │
                 └──────┴──────┴────> queueBuffer()
                 
SF:              [Comp][Comp][Comp]
                    │     │     │
                    └─────┴─────┴────> Present
```

**Choreographer** in apps synchronizes with VSync:

```java
// In application
Choreographer.getInstance().postFrameCallback(new FrameCallback() {
    @Override
    public void doFrame(long frameTimeNanos) {
        // Render frame synchronized with display
        drawFrame();
        
        // Schedule next frame
        Choreographer.getInstance().postFrameCallback(this);
    }
});
```

### Hardware Composer (HWC)

HWC is a HAL that offloads composition to dedicated hardware:

**Benefits:**
- Lower power consumption
- Better performance
- Reduced GPU load

**HWC can handle:**
- Layer blending
- Color transforms
- Scaling
- Rotation

**When HWC can't handle composition:**
- Fall back to GPU (GLES) composition
- SurfaceFlinger renders to framebuffer

```cpp
// Check if HWC can handle layer
bool HWComposer::canHandleLayer(const Layer& layer) {
    // Check transform
    if (layer.getTransform().hasComplexTransform()) {
        return false;  // Use GPU
    }
    
    // Check pixel format
    if (!isSupportedFormat(layer.getPixelFormat())) {
        return false;
    }
    
    // Check other constraints...
    
    return true;
}
```

## Creating and Managing Windows

### Creating a Window from an Application

**Basic window creation:**

```java
// Get WindowManager
WindowManager wm = (WindowManager) context.getSystemService(Context.WINDOW_SERVICE);

// Create layout parameters
WindowManager.LayoutParams params = new WindowManager.LayoutParams(
    WindowManager.LayoutParams.MATCH_PARENT,  // width
    WindowManager.LayoutParams.WRAP_CONTENT,   // height
    WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY,  // type
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |       // flags
        WindowManager.LayoutParams.FLAG_NOT_TOUCH_MODAL,
    PixelFormat.TRANSLUCENT                    // format
);

// Set position
params.gravity = Gravity.TOP | Gravity.START;
params.x = 0;
params.y = 100;

// Create view
View overlayView = LayoutInflater.from(context).inflate(R.layout.overlay, null);

// Add window
wm.addView(overlayView, params);

// Update window
params.x = 50;
wm.updateViewLayout(overlayView, params);

// Remove window
wm.removeView(overlayView);
```

### WindowManager.LayoutParams

Key parameters for window configuration:

```java
WindowManager.LayoutParams params = new WindowManager.LayoutParams();

// Size
params.width = WindowManager.LayoutParams.MATCH_PARENT;
params.height = 500;

// Type (determines z-order and permissions)
params.type = WindowManager.LayoutParams.TYPE_APPLICATION_OVERLAY;

// Flags (behavior modifiers)
params.flags = 
    WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE |      // Don't receive input focus
    WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE |      // Don't receive touch events
    WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN |   // Layout in screen coords
    WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS |   // Can extend beyond screen
    WindowManager.LayoutParams.FLAG_FULLSCREEN |         // Fullscreen window
    WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;// Use hardware acceleration

// Position
params.gravity = Gravity.TOP | Gravity.START;
params.x = 0;  // Offset from gravity
params.y = 0;

// Format
params.format = PixelFormat.RGBA_8888;

// Transparency
params.alpha = 1.0f;  // 0.0 (transparent) to 1.0 (opaque)

// Input features
params.inputFeatures = WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL;

// Token (for permission checking)
params.token = activityToken;
```

### Important Window Flags

```java
// Input handling
FLAG_NOT_FOCUSABLE        // Window won't take input focus
FLAG_NOT_TOUCHABLE        // Window won't receive touch events
FLAG_NOT_TOUCH_MODAL      // Touch outside sends to behind windows
FLAG_WATCH_OUTSIDE_TOUCH  // Receive touch events outside window

// Layout
FLAG_LAYOUT_IN_SCREEN     // Layout in screen coordinates
FLAG_LAYOUT_NO_LIMITS     // Can extend beyond screen bounds
FLAG_FULLSCREEN           // Hide status bar
FLAG_FORCE_NOT_FULLSCREEN // Never hide status bar

// Display
FLAG_DIM_BEHIND           // Dim everything behind window
FLAG_BLUR_BEHIND          // Blur everything behind (deprecated)
FLAG_HARDWARE_ACCELERATED // Use hardware acceleration

// System
FLAG_SHOW_WHEN_LOCKED     // Show even when device locked
FLAG_DISMISS_KEYGUARD     // Dismiss keyguard when shown
FLAG_TURN_SCREEN_ON       // Turn screen on when shown

// Security
FLAG_SECURE               // Don't allow screenshots
```

## Practical Example 1: Custom Window Type with Special Behavior

Let's create a custom window type that has special layout rules and interaction behavior.

### Step 1: Define the Custom Window Type

**File:** `frameworks/base/core/java/android/view/WindowManager.java`

```java
public interface WindowManager extends ViewManager {
    
    public static class LayoutParams extends ViewGroup.LayoutParams {
        
        // ... existing types ...
        
        /**
         * Window type: Custom overlay that follows special rules.
         * Requires CUSTOM_OVERLAY permission.
         * @hide
         */
        public static final int TYPE_CUSTOM_OVERLAY = 2050;
        
        // ... rest of class ...
    }
}
```

### Step 2: Add Permission

**File:** `frameworks/base/core/res/AndroidManifest.xml`

```xml
<!-- Permission for custom overlay window -->
<permission android:name="android.permission.CUSTOM_OVERLAY"
            android:protectionLevel="signature|privileged"
            android:label="@string/permlab_customOverlay"
            android:description="@string/permdesc_customOverlay" />
```

### Step 3: Implement Window Policy

**File:** `frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java`

```java
public class DisplayPolicy {
    
    // ... existing code ...
    
    /**
     * Custom layout logic for TYPE_CUSTOM_OVERLAY windows
     */
    public void layoutCustomOverlay(WindowState win, WindowState attached,
            DisplayFrames displayFrames) {
        
        final Rect bounds = win.getBounds();
        final Rect displayBounds = displayFrames.mUnrestricted;
        
        // Custom positioning logic
        // Position at bottom-center of screen
        int width = win.mRequestedWidth;
        int height = win.mRequestedHeight;
        
        int left = (displayBounds.width() - width) / 2;
        int top = displayBounds.height() - height - getNavigationBarHeight();
        
        // Apply custom margins
        int margin = dpToPx(16);
        left = Math.max(left, margin);
        top = Math.max(top - margin, 0);
        
        // Set frame
        win.mFrame.set(left, top, left + width, top + height);
        
        // Custom animation
        win.mWinAnimator.mEnterAnimationPending = true;
        win.mWinAnimator.mAnimationType = CUSTOM_OVERLAY_ANIMATION;
        
        Slog.d(TAG, "Custom overlay positioned at: " + win.mFrame);
    }
    
    private int dpToPx(int dp) {
        return (int) (dp * mContext.getResources().getDisplayMetrics().density);
    }
    
    private int getNavigationBarHeight() {
        return mContext.getResources().getDimensionPixelSize(
            com.android.internal.R.dimen.navigation_bar_height);
    }
}
```

### Step 4: Integrate into WindowManagerService

**File:** `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java`

```java
public class WindowManagerService extends IWindowManager.Stub {
    
    public int addWindow(Session session, IWindow client, 
            WindowManager.LayoutParams attrs, ...) {
        
        // ... existing validation ...
        
        // Check permission for custom overlay
        if (attrs.type == WindowManager.LayoutParams.TYPE_CUSTOM_OVERLAY) {
            if (!hasCustomOverlayPermission(session)) {
                Slog.w(TAG, "Denied creating custom overlay for " + session);
                return WindowManagerGlobal.ADD_PERMISSION_DENIED;
            }
            
            // Apply custom limits
            attrs.flags |= WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE;
            attrs.flags |= WindowManager.LayoutParams.FLAG_HARDWARE_ACCELERATED;
            
            // Enforce size constraints
            int maxWidth = mContext.getResources().getDisplayMetrics().widthPixels / 2;
            int maxHeight = dpToPx(200);
            
            if (attrs.width > maxWidth) {
                attrs.width = maxWidth;
                Slog.w(TAG, "Custom overlay width clamped to " + maxWidth);
            }
            if (attrs.height > maxHeight) {
                attrs.height = maxHeight;
                Slog.w(TAG, "Custom overlay height clamped to " + maxHeight);
            }
        }
        
        // ... continue with window creation ...
    }
    
    private boolean hasCustomOverlayPermission(Session session) {
        return mContext.checkPermission(
            android.Manifest.permission.CUSTOM_OVERLAY,
            session.mPid,
            session.mUid) == PackageManager.PERMISSION_GRANTED;
    }
    
    private int dpToPx(int dp) {
        return (int) (dp * mContext.getResources().getDisplayMetrics().density);
    }
}
```

### Step 5: Add Custom Animation

**File:** `frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java`

```java
public class WindowStateAnimator {
    
    static final int CUSTOM_OVERLAY_ANIMATION = 100;
    
    void applyEnterAnimationLocked() {
        // ... existing animation code ...
        
        if (mAnimationType == CUSTOM_OVERLAY_ANIMATION) {
            applyCustomOverlayAnimation();
            return;
        }
        
        // ... rest of animations ...
    }
    
    private void applyCustomOverlayAnimation() {
        Animation anim = AnimationUtils.loadAnimation(
            mContext,
            com.android.internal.R.anim.custom_overlay_enter);
        
        // Custom slide up from bottom
        TranslateAnimation translate = new TranslateAnimation(
            Animation.RELATIVE_TO_SELF, 0,
            Animation.RELATIVE_TO_SELF, 0,
            Animation.RELATIVE_TO_SELF, 1.0f,
            Animation.RELATIVE_TO_SELF, 0);
        translate.setDuration(300);
        translate.setInterpolator(new DecelerateInterpolator());
        
        // Fade in
        AlphaAnimation alpha = new AlphaAnimation(0.0f, 1.0f);
        alpha.setDuration(300);
        
        AnimationSet set = new AnimationSet(false);
        set.addAnimation(translate);
        set.addAnimation(alpha);
        
        startAnimation(set);
        
        Slog.d(TAG, "Started custom overlay animation");
    }
}
```

### Step 6: Usage Example

**In an app with CUSTOM_OVERLAY permission:**

```java
public class CustomOverlayManager {
    
    private Context mContext;
    private WindowManager mWindowManager;
    private View mOverlayView;
    private WindowManager.LayoutParams mParams;
    
    public CustomOverlayManager(Context context) {
        mContext = context;
        mWindowManager = (WindowManager) context.getSystemService(
            Context.WINDOW_SERVICE);
    }
    
    public void showCustomOverlay(String message) {
        // Create view
        mOverlayView = LayoutInflater.from(mContext)
            .inflate(R.layout.custom_overlay, null);
        
        TextView textView = mOverlayView.findViewById(R.id.message);
        textView.setText(message);
        
        // Create layout params with custom type
        mParams = new WindowManager.LayoutParams(
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.WRAP_CONTENT,
            WindowManager.LayoutParams.TYPE_CUSTOM_OVERLAY,
            WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE,
            PixelFormat.TRANSLUCENT
        );
        
        // Position will be handled by custom policy
        mParams.gravity = Gravity.BOTTOM | Gravity.CENTER_HORIZONTAL;
        
        // Add to window manager
        mWindowManager.addView(mOverlayView, mParams);
        
        // Auto-dismiss after delay
        mOverlayView.postDelayed(() -> hideCustomOverlay(), 5000);
    }
    
    public void hideCustomOverlay() {
        if (mOverlayView != null && mOverlayView.isAttachedToWindow()) {
            mWindowManager.removeView(mOverlayView);
            mOverlayView = null;
        }
    }
}
```

**AndroidManifest.xml:**

```xml
<uses-permission android:name="android.permission.CUSTOM_OVERLAY" />
```

## Practical Example 2: Custom Display Orientation Handling

Let's customize how display orientation changes are handled.

### Step 1: Modify Display Policy

**File:** `frameworks/base/services/core/java/com/android/server/wm/DisplayPolicy.java`

```java
public class DisplayPolicy {
    
    // Custom orientation lock feature
    private boolean mCustomOrientationLockEnabled = false;
    private int mLockedOrientation = ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED;
    
    /**
     * Enable custom orientation lock based on specific conditions
     */
    public void updateCustomOrientationLock() {
        // Check system property
        mCustomOrientationLockEnabled = SystemProperties.getBoolean(
            "persist.custom.orientation_lock", false);
        
        if (!mCustomOrientationLockEnabled) {
            mLockedOrientation = ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED;
            return;
        }
        
        // Example: Lock to portrait during certain hours
        Calendar cal = Calendar.getInstance();
        int hour = cal.get(Calendar.HOUR_OF_DAY);
        
        if (hour >= 22 || hour < 6) {
            // Night mode: force portrait
            mLockedOrientation = ActivityInfo.SCREEN_ORIENTATION_PORTRAIT;
            Slog.i(TAG, "Night mode: locked to portrait");
        } else {
            mLockedOrientation = ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED;
        }
    }
    
    /**
     * Custom rotation logic
     */
    public int rotationForOrientationLw(int orientation, int lastRotation,
            boolean defaultDisplay) {
        
        // Check custom orientation lock first
        if (mCustomOrientationLockEnabled && 
            mLockedOrientation != ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED) {
            
            Slog.d(TAG, "Applying custom orientation lock: " + mLockedOrientation);
            return getRotationForOrientation(mLockedOrientation, lastRotation);
        }
        
        // Special handling for specific apps
        WindowState focusedWindow = mService.getFocusedWindow();
        if (focusedWindow != null) {
            String packageName = focusedWindow.getOwningPackage();
            
            // Force landscape for video apps
            if (isVideoApp(packageName)) {
                Slog.d(TAG, "Video app detected, preferring landscape");
                if (orientation == ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED) {
                    return Surface.ROTATION_90;  // Landscape
                }
            }
        }
        
        // Fall back to standard rotation logic
        return rotationForOrientationLw_standard(orientation, lastRotation);
    }
    
    private boolean isVideoApp(String packageName) {
        // List of apps that should prefer landscape
        return packageName != null && (
            packageName.equals("com.google.android.youtube") ||
            packageName.equals("com.netflix.mediaclient") ||
            packageName.contains("video")
        );
    }
    
    private int getRotationForOrientation(int orientation, int lastRotation) {
        switch (orientation) {
            case ActivityInfo.SCREEN_ORIENTATION_PORTRAIT:
            case ActivityInfo.SCREEN_ORIENTATION_SENSOR_PORTRAIT:
                return Surface.ROTATION_0;
            case ActivityInfo.SCREEN_ORIENTATION_LANDSCAPE:
            case ActivityInfo.SCREEN_ORIENTATION_SENSOR_LANDSCAPE:
                return Surface.ROTATION_90;
            case ActivityInfo.SCREEN_ORIENTATION_REVERSE_PORTRAIT:
                return Surface.ROTATION_180;
            case ActivityInfo.SCREEN_ORIENTATION_REVERSE_LANDSCAPE:
                return Surface.ROTATION_270;
            default:
                return lastRotation;
        }
    }
    
    // Original rotation logic (preserved)
    private int rotationForOrientationLw_standard(int orientation, 
            int lastRotation) {
        // ... existing standard rotation logic ...
        return lastRotation;  // Simplified
    }
}
```

### Step 2: Add Configuration Change Listener

**File:** `frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java`

```java
public class WindowManagerService extends IWindowManager.Stub {
    
    // Custom orientation change callbacks
    private final List<IOrientationChangeCallback> mOrientationCallbacks = 
        new ArrayList<>();
    
    /**
     * Register callback for orientation changes
     * @hide
     */
    public void registerOrientationChangeCallback(
            IOrientationChangeCallback callback) {
        synchronized (mGlobalLock) {
            mOrientationCallbacks.add(callback);
        }
    }
    
    /**
     * Unregister callback
     * @hide
     */
    public void unregisterOrientationChangeCallback(
            IOrientationChangeCallback callback) {
        synchronized (mGlobalLock) {
            mOrientationCallbacks.remove(callback);
        }
    }
    
    /**
     * Called when orientation changes
     */
    private void notifyOrientationChange(int oldRotation, int newRotation) {
        List<IOrientationChangeCallback> callbacks;
        synchronized (mGlobalLock) {
            callbacks = new ArrayList<>(mOrientationCallbacks);
        }
        
        for (IOrientationChangeCallback callback : callbacks) {
            try {
                callback.onOrientationChanged(oldRotation, newRotation);
            } catch (RemoteException e) {
                Slog.w(TAG, "Failed to notify orientation change", e);
            }
        }
    }
    
    /**
     * Update rotation with custom logic
     */
    boolean updateRotationUncheckedLocked(boolean inTransaction) {
        // Update custom orientation lock
        mDisplayPolicy.updateCustomOrientationLock();
        
        int oldRotation = mRotation;
        
        // ... standard rotation update ...
        
        int newRotation = mDisplayPolicy.rotationForOrientationLw(
            mPolicy.getOrientationFromAppTokensLocked(),
            mRotation,
            true /* defaultDisplay */);
        
        if (oldRotation != newRotation) {
            Slog.i(TAG, "Rotation changed: " + oldRotation + " -> " + newRotation);
            
            // Apply rotation
            setRotationUnchecked(newRotation);
            
            // Notify callbacks
            notifyOrientationChange(oldRotation, newRotation);
            
            return true;
        }
        
        return false;
    }
}
```

### Step 3: Create Callback Interface

**File:** `frameworks/base/core/java/android/view/IOrientationChangeCallback.aidl`

```java
package android.view;

/**
 * Callback for orientation changes
 * @hide
 */
oneway interface IOrientationChangeCallback {
    void onOrientationChanged(int oldRotation, int newRotation);
}
```

### Step 4: Usage Example

**Enable custom orientation lock:**

```bash
# Enable custom orientation lock
adb shell setprop persist.custom.orientation_lock true

# Force rotation update
adb shell wm size reset
```

**Register for orientation changes in an app:**

```java
public class OrientationMonitor {
    
    private IWindowManager mWindowManager;
    private IOrientationChangeCallback mCallback;
    
    public OrientationMonitor() {
        mWindowManager = IWindowManager.Stub.asInterface(
            ServiceManager.getService(Context.WINDOW_SERVICE));
    }
    
    public void startMonitoring() {
        mCallback = new IOrientationChangeCallback.Stub() {
            @Override
            public void onOrientationChanged(int oldRotation, int newRotation) {
                Log.i(TAG, "Orientation changed: " + 
                    rotationToString(oldRotation) + " -> " + 
                    rotationToString(newRotation));
                
                // Handle orientation change
                handleOrientationChange(newRotation);
            }
        };
        
        try {
            mWindowManager.registerOrientationChangeCallback(mCallback);
        } catch (RemoteException e) {
            Log.e(TAG, "Failed to register callback", e);
        }
    }
    
    public void stopMonitoring() {
        if (mCallback != null) {
            try {
                mWindowManager.unregisterOrientationChangeCallback(mCallback);
            } catch (RemoteException e) {
                Log.e(TAG, "Failed to unregister callback", e);
            }
            mCallback = null;
        }
    }
    
    private String rotationToString(int rotation) {
        switch (rotation) {
            case Surface.ROTATION_0: return "PORTRAIT";
            case Surface.ROTATION_90: return "LANDSCAPE";
            case Surface.ROTATION_180: return "REVERSE_PORTRAIT";
            case Surface.ROTATION_270: return "REVERSE_LANDSCAPE";
            default: return "UNKNOWN";
        }
    }
    
    private void handleOrientationChange(int newRotation) {
        // Custom handling based on rotation
    }
}
```

## Surface Management

### Creating a Surface

Applications typically use `SurfaceView` or `TextureView`:

**SurfaceView example:**

```java
public class CustomSurfaceView extends SurfaceView 
        implements SurfaceHolder.Callback {
    
    private SurfaceHolder mHolder;
    private RenderThread mRenderThread;
    
    public CustomSurfaceView(Context context) {
        super(context);
        mHolder = getHolder();
        mHolder.addCallback(this);
    }
    
    @Override
    public void surfaceCreated(SurfaceHolder holder) {
        // Surface is ready, start rendering
        mRenderThread = new RenderThread(holder);
        mRenderThread.start();
    }
    
    @Override
    public void surfaceChanged(SurfaceHolder holder, 
            int format, int width, int height) {
        // Surface size/format changed
        if (mRenderThread != null) {
            mRenderThread.updateSurface(width, height);
        }
    }
    
    @Override
    public void surfaceDestroyed(SurfaceHolder holder) {
        // Surface destroyed, stop rendering
        if (mRenderThread != null) {
            mRenderThread.requestStop();
            try {
                mRenderThread.join();
            } catch (InterruptedException e) {
                // Ignore
            }
        }
    }
    
    private class RenderThread extends Thread {
        private SurfaceHolder mHolder;
        private volatile boolean mRunning = true;
        private int mWidth;
        private int mHeight;
        
        public RenderThread(SurfaceHolder holder) {
            mHolder = holder;
        }
        
        public void updateSurface(int width, int height) {
            mWidth = width;
            mHeight = height;
        }
        
        public void requestStop() {
            mRunning = false;
        }
        
        @Override
        public void run() {
            while (mRunning) {
                Canvas canvas = null;
                try {
                    // Lock canvas for drawing
                    canvas = mHolder.lockCanvas();
                    if (canvas != null) {
                        // Clear
                        canvas.drawColor(Color.BLACK);
                        
                        // Draw content
                        drawFrame(canvas);
                    }
                } finally {
                    // Post canvas (queue buffer)
                    if (canvas != null) {
                        mHolder.unlockCanvasAndPost(canvas);
                    }
                }
                
                // Limit frame rate
                try {
                    Thread.sleep(16);  // ~60 FPS
                } catch (InterruptedException e) {
                    break;
                }
            }
        }
        
        private void drawFrame(Canvas canvas) {
            // Custom drawing
            Paint paint = new Paint();
            paint.setColor(Color.RED);
            paint.setTextSize(50);
            canvas.drawText("Custom Surface", 100, 100, paint);
        }
    }
}
```

### OpenGL ES Surface

For hardware-accelerated 3D rendering:

```java
public class GLSurfaceViewExample extends GLSurfaceView {
    
    private MyRenderer mRenderer;
    
    public GLSurfaceViewExample(Context context) {
        super(context);
        
        // Use OpenGL ES 2.0
        setEGLContextClientVersion(2);
        
        mRenderer = new MyRenderer();
        setRenderer(mRenderer);
        
        // Render continuously or on request
        setRenderMode(RENDERMODE_CONTINUOUSLY);
    }
    
    private class MyRenderer implements GLSurfaceView.Renderer {
        
        @Override
        public void onSurfaceCreated(GL10 gl, EGLConfig config) {
            // Initialize OpenGL state
            GLES20.glClearColor(0.0f, 0.0f, 0.0f, 1.0f);
        }
        
        @Override
        public void onSurfaceChanged(GL10 gl, int width, int height) {
            // Set viewport
            GLES20.glViewport(0, 0, width, height);
        }
        
        @Override
        public void onDrawFrame(GL10 gl) {
            // Clear framebuffer
            GLES20.glClear(GLES20.GL_COLOR_BUFFER_BIT | 
                          GLES20.GL_DEPTH_BUFFER_BIT);
            
            // Render 3D scene
            renderScene();
        }
        
        private void renderScene() {
            // OpenGL rendering code
        }
    }
}
```

## Display Debugging and Analysis

### Dumping Window State

```bash
# Dump all window state
adb shell dumpsys window

# Dump surfaces
adb shell dumpsys SurfaceFlinger

# Dump specific window
adb shell dumpsys window windows | grep -A 20 "Window #"

# View display info
adb shell dumpsys display
```

### Analyzing Frame Timing

```bash
# Enable GPU rendering profiling
adb shell setprop debug.hwui.profile true

# View frame stats
adb shell dumpsys gfxinfo <package>

# Detailed frame timing
adb shell dumpsys gfxinfo <package> framestats
```

### SurfaceFlinger Debugging

```bash
# Show layer stack
adb shell dumpsys SurfaceFlinger

# Show composition type (HWC vs GPU)
adb shell dumpsys SurfaceFlinger --latency

# Show buffer queue stats
adb shell dumpsys SurfaceFlinger --list
```

### Screen Recording

SurfaceFlinger provides screen recording:

```bash
# Record screen
adb shell screenrecord /sdcard/demo.mp4

# With options
adb shell screenrecord --size 1280x720 --bit-rate 6000000 /sdcard/demo.mp4

# Pull recording
adb pull /sdcard/demo.mp4
```

## Performance Optimization

### Reducing Overdraw

Overdraw occurs when pixels are drawn multiple times per frame:

```bash
# Enable overdraw debugging
adb shell setprop debug.hwui.overdraw show

# Colors indicate overdraw amount:
# No color: 1x (good)
# Blue: 2x
# Green: 3x
# Pink: 4x
# Red: 5x+ (bad)
```

**Minimize overdraw:**
- Remove unnecessary backgrounds
- Use `clipRect()` to avoid drawing hidden content
- Flatten view hierarchy
- Use hardware layers for complex static content

### Hardware Acceleration

```java
// Enable for view
view.setLayerType(View.LAYER_TYPE_HARDWARE, null);

// Disable for view (software rendering)
view.setLayerType(View.LAYER_TYPE_SOFTWARE, null);

// Auto (default)
view.setLayerType(View.LAYER_TYPE_NONE, null);
```

**In manifest:**

```xml
<application android:hardwareAccelerated="true">
    <activity android:hardwareAccelerated="false" />
</application>
```

### VSync Optimization

Synchronize animations with VSync:

```java
// Use Choreographer for animations
Choreographer.getInstance().postFrameCallback(frameTimeNanos -> {
    updateAnimation(frameTimeNanos);
    invalidate();
});

// Or use ValueAnimator (automatically synced)
ValueAnimator animator = ValueAnimator.ofFloat(0f, 1f);
animator.setDuration(300);
animator.addUpdateListener(animation -> {
    float value = (float) animation.getAnimatedValue();
    updateView(value);
});
animator.start();
```

## Key Takeaways

1. **WindowManager manages window lifecycle**: Handles layout, z-order, and input routing
2. **SurfaceFlinger composites surfaces**: Combines layers from all apps into final image
3. **BufferQueue enables rendering**: Applications produce, SurfaceFlinger consumes
4. **VSync synchronizes everything**: Frame timing coordinated with display refresh
5. **Hardware Composer optimizes**: Offloads composition to dedicated hardware
6. **Window types control behavior**: Type determines z-order, permissions, and policy
7. **Customization is powerful**: Add custom window types, animations, and policies

## Next Steps

In Chapter 7, we'll explore ActivityManager and process management—how Android manages the application lifecycle, process priorities, and the low memory killer. You'll learn how to customize process management policies and implement advanced lifecycle handling.

## Quick Reference

### Common WindowManager Operations

```java
WindowManager wm = context.getSystemService(WindowManager.class);

// Add window
WindowManager.LayoutParams params = new WindowManager.LayoutParams(...);
View view = ...;
wm.addView(view, params);

// Update window
params.x = 100;
wm.updateViewLayout(view, params);

// Remove window
wm.removeView(view);

// Get display info
Display display = wm.getDefaultDisplay();
Point size = new Point();
display.getSize(size);
```

### Debug Commands

```bash
# Window state
adb shell dumpsys window

# Surface state
adb shell dumpsys SurfaceFlinger

# GPU info
adb shell dumpsys gfxinfo <package>

# Display info
adb shell dumpsys display

# Enable overdraw
adb shell setprop debug.hwui.overdraw show

# Screen recording
adb shell screenrecord /sdcard/demo.mp4
```

### Window Flags Quick Reference

```java
FLAG_NOT_FOCUSABLE          // No input focus
FLAG_NOT_TOUCHABLE          // No touch events
FLAG_NOT_TOUCH_MODAL        // Pass touches to behind
FLAG_FULLSCREEN             // Hide status bar
FLAG_LAYOUT_IN_SCREEN       // Layout in screen coords
FLAG_HARDWARE_ACCELERATED   // Use GPU
FLAG_SECURE                 // No screenshots
FLAG_SHOW_WHEN_LOCKED       // Show on lock screen
```
