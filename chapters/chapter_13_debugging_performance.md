# Chapter 13: Debugging & Performance

## Introduction

Debugging and performance optimization are essential skills for AOSP development. This chapter covers the tools, techniques, and methodologies for identifying and resolving issues in Android systems, as well as optimizing performance for production devices.

## Debugging Tools Overview

### Essential Tools

```
┌─────────────────────────────────────┐
│         Logging & Inspection         │
│  - logcat                           │
│  - dmesg                            │
│  - dumpsys                          │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         System Tracing               │
│  - systrace/perfetto                │
│  - atrace                           │
│  - simpleperf                       │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Crash Analysis               │
│  - tombstones                       │
│  - stack traces                     │
│  - addr2line                        │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│         Memory Analysis              │
│  - meminfo                          │
│  - procrank                         │
│  - dmabuf_dump                      │
└─────────────────────────────────────┘
```

## Logcat Mastery

### Basic Usage

```bash
# View all logs
adb logcat

# Clear and view
adb logcat -c && adb logcat

# Save to file
adb logcat > logs.txt

# View specific buffer
adb logcat -b system
adb logcat -b events
adb logcat -b crash
```

### Filtering

```bash
# Filter by priority (V/D/I/W/E/F)
adb logcat *:W           # Warnings and above
adb logcat *:E           # Errors and above

# Filter by tag
adb logcat ActivityManager:V *:S

# Multiple tags
adb logcat ActivityManager:V PackageManager:D *:S

# Filter by process
adb logcat --pid=1234

# Grep for specific content
adb logcat | grep "my pattern"

# Format options
adb logcat -v time       # Include timestamps
adb logcat -v threadtime # Include thread info
adb logcat -v brief      # Minimal output
```

### Advanced Filtering

```bash
# Regular expressions
adb logcat -e "pattern.*"

# Case insensitive
adb logcat | grep -i "error"

# Highlight matches
adb logcat --color=always | grep --color=always "error"

# Multiple conditions
adb logcat | grep -E "(error|warning|exception)"
```

### Log Levels

```java
// In code
android.util.Log.v(TAG, "Verbose");  // Development only
android.util.Log.d(TAG, "Debug");    // Debug info
android.util.Log.i(TAG, "Info");     // General info
android.util.Log.w(TAG, "Warning");  // Warnings
android.util.Log.e(TAG, "Error");    // Errors
android.util.Log.wtf(TAG, "WTF");    // Critical errors
```

## System Tracing

### Perfetto (Modern)

```bash
# Web UI
https://ui.perfetto.dev

# Capture trace
adb shell perfetto \
    -o /data/misc/perfetto-traces/trace.perfetto-trace \
    -t 20s \
    sched freq idle am wm gfx view binder_driver hal dalvik

# Pull trace
adb pull /data/misc/perfetto-traces/trace.perfetto-trace

# Upload to ui.perfetto.dev
```

**Key categories:**
- `sched`: CPU scheduling
- `freq`: CPU frequency
- `idle`: CPU idle states
- `am`: ActivityManager
- `wm`: WindowManager
- `gfx`: Graphics
- `view`: View system
- `binder_driver`: Binder calls
- `hal`: HAL interactions

### Systrace (Legacy)

```bash
# Capture 10-second trace
systrace.py --time=10 -o trace.html \
    sched freq idle am wm gfx view binder_driver hal dalvik

# Open in Chrome
chrome trace.html
```

### Analyzing Traces

**Look for:**
1. **Frame drops**: Missed VSync deadlines
2. **Long Binder calls**: Blocking operations
3. **CPU bottlenecks**: Runnable but not running
4. **Wake locks**: Preventing sleep
5. **GC pauses**: Memory pressure

## Crash Analysis

### Tombstones

Native crashes generate tombstones:

```bash
# View tombstones
adb shell ls /data/tombstones/

# Pull latest
adb pull /data/tombstones/tombstone_00

# Example tombstone
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'google/...'
Revision: '0'
ABI: 'arm64'
Timestamp: 2024-01-15 10:30:45
pid: 1234, tid: 1234, name: my_process  >>> /system/bin/my_daemon <<<
uid: 1000
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 0x0
    x0  0000000000000000  x1  0000007f8c8e9000
    ...
backtrace:
    #00 pc 00000000000123ab  /system/lib64/libmylib.so (my_function+27)
    #01 pc 00000000000456cd  /system/lib64/libmylib.so (caller+61)
```

### Symbolizing Stack Traces

```bash
# Using stack script
development/scripts/stack < tombstone_00

# Or manually with addr2line
addr2line -e out/target/product/*/symbols/system/lib64/libmylib.so 0x123ab
```

### ANR Traces

Application Not Responding traces:

```bash
# ANR traces location
/data/anr/traces.txt

# Pull and analyze
adb pull /data/anr/traces.txt

# Look for:
# - Main thread blocked
# - Held locks
# - Waiting threads
```

### Java Exception Analysis

```bash
# View crash logs
adb logcat -b crash

# Filter for exceptions
adb logcat | grep -A 20 "AndroidRuntime: FATAL EXCEPTION"

# Example:
FATAL EXCEPTION: main
Process: com.example.app, PID: 1234
java.lang.NullPointerException: Attempt to invoke virtual method...
    at com.example.app.MainActivity.onCreate(MainActivity.java:42)
    at android.app.Activity.performCreate(Activity.java:7136)
```

## Memory Debugging

### Dumpsys Meminfo

```bash
# System memory overview
adb shell dumpsys meminfo

# Specific process
adb shell dumpsys meminfo com.example.app

# Output includes:
# - Dalvik heap
# - Native heap
# - Graphics
# - Private/Shared memory
# - Swap
```

### Procrank

```bash
# Memory usage by process
adb shell procrank

# Output:
#   PID      Vss      Rss      Pss      Uss  cmdline
#  1234   1.2G     500M     300M     250M  system_server
#  5678   800M     200M     150M     120M  com.android.systemui
```

### Showmap

```bash
# Detailed memory mapping
adb shell showmap <pid>

# Shows:
# - Virtual/RSS/PSS per mapping
# - Shared libraries
# - Anonymous memory
# - File mappings
```

### Memory Leaks

**Native leaks (using malloc debug):**

```bash
# Enable malloc debug
adb shell setprop libc.debug.malloc.program my_daemon
adb shell setprop libc.debug.malloc.options backtrace

# Restart daemon
adb shell killall my_daemon

# Dump leaks
adb shell am dumpheap -n <pid> /data/local/tmp/heap.txt
adb pull /data/local/tmp/heap.txt
```

**Java leaks (using hprof):**

```bash
# Dump Java heap
adb shell am dumpheap com.example.app /data/local/tmp/heap.hprof

# Pull and analyze
adb pull /data/local/tmp/heap.hprof
hprof-conv heap.hprof heap-converted.hprof

# Analyze with Android Studio or Eclipse MAT
```

## Performance Profiling

### Simpleperf

CPU profiling for native code:

```bash
# Record profile
adb shell simpleperf record -p <pid> -g --duration 30 -o /data/profile.data

# Pull data
adb pull /data/profile.data

# Generate report
simpleperf report -i profile.data

# Generate flamegraph
simpleperf report -i profile.data -g | \
    ./scripts/inferno.py > flamegraph.html
```

### Method Tracing

Java method profiling:

```java
// Start tracing
Debug.startMethodTracing("myapp");

// Code to profile
doWork();

// Stop tracing
Debug.stopMethodTracing();

// Pull trace
// adb pull /sdcard/Android/data/<package>/files/myapp.trace

// Analyze with Android Studio profiler
```

### GPU Profiling

```bash
# Enable GPU profiling
adb shell setprop debug.hwui.profile true

# View in adb logcat
adb logcat | grep hwui

# Or use systrace/perfetto
```

## Dumpsys Deep Dive

### Activity Manager

```bash
# Full activity dump
adb shell dumpsys activity

# Activities only
adb shell dumpsys activity activities

# Services
adb shell dumpsys activity services

# Processes
adb shell dumpsys activity processes

# Recent tasks
adb shell dumpsys activity recents
```

### Package Manager

```bash
# All packages
adb shell dumpsys package

# Specific package
adb shell dumpsys package com.example.app

# Permissions
adb shell dumpsys package com.example.app | grep permission

# Installers
adb shell dumpsys package installers
```

### Battery Stats

```bash
# Battery statistics
adb shell dumpsys batterystats

# Reset stats
adb shell dumpsys batterystats --reset

# Specific package
adb shell dumpsys batterystats com.example.app

# Generate battery historian report
adb shell dumpsys batterystats > batterystats.txt
```

### Surface Flinger

```bash
# Display information
adb shell dumpsys SurfaceFlinger

# Layer stack
adb shell dumpsys SurfaceFlinger --list

# Frame stats
adb shell dumpsys SurfaceFlinger --latency

# Wide color gamut
adb shell dumpsys SurfaceFlinger --wide-color-gamut
```

## Performance Optimization

### CPU Optimization

**1. Profile first:**
```bash
simpleperf record -p <pid> --duration 30
simpleperf report
```

**2. Identify hot paths:**
- Look for functions consuming most time
- Check for unnecessary loops
- Optimize algorithms

**3. Use threading wisely:**
```java
// Move work off main thread
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.execute(() -> {
    // Heavy work here
});
```

### Memory Optimization

**1. Reduce allocations:**
```java
// Bad: Creates objects in loop
for (int i = 0; i < 1000; i++) {
    String s = "Item " + i;  // New string each iteration
}

// Good: Reuse objects
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.setLength(0);
    sb.append("Item ").append(i);
}
```

**2. Use appropriate data structures:**
```java
// For primitive types
SparseArray<String> map = new SparseArray<>();  // Instead of HashMap<Integer, String>

// For small maps
ArrayMap<String, String> map = new ArrayMap<>();  // Instead of HashMap
```

**3. Avoid memory leaks:**
```java
// Bad: Activity leak via static reference
public class MyActivity extends Activity {
    private static Context sContext;
    
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        sContext = this;  // LEAK!
    }
}

// Good: Use application context
sContext = getApplicationContext();
```

### I/O Optimization

**1. Batch operations:**
```java
// Bad: Multiple small writes
for (String line : lines) {
    fileOutputStream.write(line.getBytes());
}

// Good: Buffer writes
BufferedOutputStream out = new BufferedOutputStream(fileOutputStream);
for (String line : lines) {
    out.write(line.getBytes());
}
out.flush();
```

**2. Use memory-mapped files for large data:**
```java
FileChannel channel = new RandomAccessFile(file, "rw").getChannel();
MappedByteBuffer buffer = channel.map(FileChannel.MapMode.READ_WRITE, 0, size);
```

### Graphics Optimization

**1. Reduce overdraw:**
```bash
# Enable overdraw debugging
adb shell setprop debug.hwui.overdraw show

# Colors indicate overdraw amount
```

**2. Use hardware layers:**
```java
view.setLayerType(View.LAYER_TYPE_HARDWARE, null);
```

**3. Optimize layouts:**
```xml
<!-- Bad: Nested LinearLayouts -->
<LinearLayout>
    <LinearLayout>
        <TextView />
    </LinearLayout>
</LinearLayout>

<!-- Good: ConstraintLayout -->
<ConstraintLayout>
    <TextView />
</ConstraintLayout>
```

## Practical Example: Debug Workflow

### Scenario: App Crashes on Startup

**Step 1: Capture crash log**
```bash
adb logcat -b crash -d > crash.log
```

**Step 2: Identify exception**
```bash
grep "FATAL EXCEPTION" crash.log

# Output:
# FATAL EXCEPTION: main
# java.lang.RuntimeException: Unable to start activity
#   at android.app.ActivityThread.performLaunchActivity
#   Caused by: java.lang.NullPointerException
#   at com.example.app.MainActivity.onCreate(MainActivity.java:42)
```

**Step 3: Check line 42 in MainActivity**
```java
// Line 42 in MainActivity.java
textView.setText(data.getValue());  // data is null!
```

**Step 4: Fix**
```java
if (data != null) {
    textView.setText(data.getValue());
}
```

**Step 5: Verify**
```bash
adb install -r app.apk
adb shell am start -n com.example.app/.MainActivity
adb logcat | grep MainActivity
```

## Practical Example: Performance Issue

### Scenario: Slow List Scrolling

**Step 1: Profile scrolling**
```bash
# Capture trace during scroll
adb shell perfetto \
    -o /data/misc/perfetto-traces/scroll.perfetto-trace \
    -t 10s \
    sched freq gfx view
```

**Step 2: Analyze in Perfetto UI**
- Upload trace to ui.perfetto.dev
- Look for frame drops (>16ms per frame)
- Check for:
  - View inflation in scroll
  - Bitmap decoding
  - Excessive GC

**Step 3: Identify issue**
```
Frame time: 45ms (should be <16ms)
- View.onDraw: 30ms
  - Bitmap decode: 25ms
```

**Step 4: Fix - Cache decoded bitmaps**
```java
// Bad: Decode in onBindViewHolder
public void onBindViewHolder(ViewHolder holder, int position) {
    Bitmap bitmap = BitmapFactory.decodeResource(...);
    holder.imageView.setImageBitmap(bitmap);
}

// Good: Use image loading library
public void onBindViewHolder(ViewHolder holder, int position) {
    Glide.with(context)
        .load(imageUrl)
        .into(holder.imageView);
}
```

**Step 5: Verify improvement**
```bash
# Profile again
adb shell perfetto -o /data/scroll_fixed.perfetto-trace -t 10s gfx view

# Check frame times - should be <16ms
```

## Debug Build Configuration

### Enable Debug Features

**File:** `device/manufacturer/product/BoardConfig.mk`

```makefile
# Enable additional logging
BOARD_KERNEL_CMDLINE += printk.devkmsg=on

# Enable address sanitizer (ASAN)
SANITIZE_TARGET := address

# Enable undefined behavior sanitizer (UBSAN)
SANITIZE_TARGET += undefined
```

**File:** `device/manufacturer/product/device.mk`

```makefile
# Debug packages
PRODUCT_PACKAGES += \
    strace \
    gdbserver \
    tcpdump \
    iotop

# Debug properties
PRODUCT_PROPERTY_OVERRIDES += \
    ro.debuggable=1 \
    persist.sys.usb.config=adb \
    debug.atrace.tags.enableflags=0xfffffff
```

## Advanced Debugging Techniques

### GDB Remote Debugging

```bash
# On device
adb shell gdbserver :5039 --attach <pid>

# On host
adb forward tcp:5039 tcp:5039
gdb out/target/product/*/symbols/system/bin/mydaemon
(gdb) target remote :5039
(gdb) continue
(gdb) backtrace
```

### Kernel Debugging

```bash
# Enable kernel debugging
adb shell "echo 8 > /proc/sys/kernel/printk"

# View kernel log
adb shell dmesg -w

# Specific subsystem (e.g., binder)
adb shell "echo 1 > /sys/kernel/debug/binder/debug_mask"
```

### Binder Debugging

```bash
# Binder statistics
adb shell cat /sys/kernel/debug/binder/stats

# Binder transactions
adb shell cat /sys/kernel/debug/binder/transactions

# Binder transaction log
adb shell cat /sys/kernel/debug/binder/transaction_log
```

## Quick Reference

### Essential Commands

```bash
# Logging
adb logcat
adb logcat -b crash
adb logcat *:E

# System info
adb shell dumpsys
adb shell dumpsys meminfo
adb shell dumpsys cpuinfo

# Traces
adb shell perfetto -o /data/trace -t 10s
adb pull /data/anr/traces.txt
adb pull /data/tombstones/tombstone_*

# Performance
adb shell simpleperf record -p <pid>
adb shell am profile start <package>

# Memory
adb shell dumpsys meminfo <package>
adb shell am dumpheap <package> /data/heap.hprof
```

### Debug Shortcuts

```bash
# Quick crash analysis
adb logcat -b crash -d | grep -A 20 "FATAL EXCEPTION"

# Monitor specific app
adb logcat | grep <package>

# Watch memory
watch -n 1 'adb shell dumpsys meminfo <package> | grep TOTAL'

# Frame stats
adb shell dumpsys gfxinfo <package> framestats
```

## Key Takeaways

1. **Use the right tool**: logcat for logs, perfetto for traces, dumpsys for state
2. **Profile before optimizing**: Measure first, optimize hot paths
3. **Symbolize crashes**: Stack traces are essential for debugging
4. **Monitor memory**: Leaks accumulate over time
5. **Test on real devices**: Performance varies significantly
6. **Automate debugging**: Scripts save time
7. **Learn the tools deeply**: They reveal everything

## Conclusion

This completes our comprehensive AOSP development guide with 13 chapters. You now have the knowledge and tools to build, debug, optimize, and maintain production-quality Android systems.

The guide covers everything from project structure through advanced debugging and performance optimization, providing you with the complete skillset needed for professional AOSP development.
