# Chapter 5: PackageManager & Application Lifecycle

## Introduction

PackageManager is one of Android's most critical system services. It manages the entire lifecycle of applications—from installation and verification through runtime queries and uninstallation. Understanding PackageManager is essential for AOSP development because it:

- Controls what apps can be installed and how
- Manages permissions and security policies
- Provides runtime information about installed packages
- Handles updates and versioning
- Enforces signature verification and integrity checks

This chapter explores PackageManager's architecture, the APK installation process, permission management, and provides practical examples of customizing package management behavior.

## PackageManager Architecture

### High-Level Overview

PackageManager is actually a client-side wrapper around PackageManagerService, which runs in system_server:

```
┌─────────────────────────────────────┐
│     Application Process             │
│                                     │
│  ┌────────────────────────────┐    │
│  │   PackageManager (API)     │    │
│  │   (frameworks/base/core)   │    │
│  └──────────────┬─────────────┘    │
│                 │ Binder IPC       │
└─────────────────┼─────────────────-┘
                  │
┌─────────────────┼──────────────────┐
│  System Server  │                  │
│                 ▼                  │
│  ┌──────────────────────────────┐  │
│  │  PackageManagerService       │  │
│  │  (frameworks/base/services)  │  │
│  └──────────────────────────────┘  │
│                 │                  │
│                 ├─ Package Parser   │
│                 ├─ Permission DB    │
│                 ├─ Package Cache    │
│                 └─ Install/Delete   │
└────────────────────────────────────┘
```

### Key Components

**PackageManager (Client API):**
- Location: `frameworks/base/core/java/android/content/pm/PackageManager.java`
- Abstract class defining package query APIs
- Applications interact through Context.getPackageManager()

**ApplicationPackageManager (Implementation):**
- Location: `frameworks/base/core/java/android/app/ApplicationPackageManager.java`
- Concrete implementation of PackageManager
- Delegates to PackageManagerService via Binder

**IPackageManager (AIDL Interface):**
- Location: `frameworks/base/core/java/android/content/pm/IPackageManager.aidl`
- Binder interface definition
- Defines system-level package operations

**PackageManagerService (Service):**
- Location: `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`
- Core service implementation (~50,000+ lines of code)
- Manages all package-related operations

**PackageParser:**
- Location: `frameworks/base/core/java/android/content/pm/PackageParser.java`
- Parses APK files and AndroidManifest.xml
- Extracts package metadata

**Settings:**
- Location: `frameworks/base/services/core/java/com/android/server/pm/Settings.java`
- Persists package state to `/data/system/packages.xml`
- Manages runtime permissions

### Source Code Locations

```
frameworks/base/
├── core/java/android/content/pm/
│   ├── PackageManager.java          # Public API
│   ├── PackageInfo.java             # Package metadata
│   ├── ApplicationInfo.java         # App metadata
│   ├── ActivityInfo.java            # Activity metadata
│   ├── ServiceInfo.java             # Service metadata
│   ├── IPackageManager.aidl         # Binder interface
│   └── PackageParser.java           # APK parser
│
├── core/java/android/app/
│   └── ApplicationPackageManager.java  # PM implementation
│
└── services/core/java/com/android/server/pm/
    ├── PackageManagerService.java   # Main service
    ├── PackageInstallerService.java # Installation logic
    ├── Settings.java                # Persistence
    ├── permission/                  # Permission management
    │   ├── PermissionManagerService.java
    │   └── PermissionSettings.java
    └── parsing/                     # Modern parsing code
```

## PackageManager Startup

PackageManager is one of the first services started by SystemServer.

### Initialization Sequence

**1. SystemServer starts PackageManagerService:**

```java
// frameworks/base/services/java/com/android/server/SystemServer.java

private void startBootstrapServices() {
    // ...
    
    // Start PackageManagerService
    traceBeginAndSlog("StartPackageManagerService");
    try {
        mPackageManagerService = PackageManagerService.main(
                mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF,
                mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
    } catch (RuntimeException e) {
        Slog.e(TAG, "Failed to start PackageManagerService", e);
    }
    traceEnd();
    
    mPackageManager = mSystemContext.getPackageManager();
    
    // ...
}
```

**2. PackageManagerService.main() initialization:**

```java
public static PackageManagerService main(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    
    // Create service instance
    PackageManagerService m = new PackageManagerService(context, installer,
            factoryTest, onlyCore);
    
    // Register with ServiceManager
    ServiceManager.addService("package", m);
    
    // Register native service
    ServiceManager.addService("package_native",
            new PackageManagerNative(m));
    
    return m;
}
```

**3. PackageManagerService constructor:**

The constructor performs extensive initialization:

```java
public PackageManagerService(Context context, Installer installer,
        boolean factoryTest, boolean onlyCore) {
    
    // Phase 1: Lock and setup
    synchronized (mInstallLock) {
        synchronized (mPackages) {
            
            // Initialize data directories
            mAppInstallDir = new File(dataDir, "app");
            mAppLib32InstallDir = new File(dataDir, "app-lib");
            mDrmAppPrivateInstallDir = new File(dataDir, "app-private");
            
            // Create system directories
            mUserManager = UserManagerService.getInstance();
            
            // Initialize settings
            mSettings = new Settings(mPermissionManager.getPermissionSettings());
            
            // Read packages.xml
            mSettings.readLPw(mUserManager.getUsers(false));
        }
    }
    
    // Phase 2: Scan system packages
    scanDirLI(new File(Environment.getRootDirectory(), "framework"),
            flags, scanFlags);
    scanDirLI(new File(Environment.getRootDirectory(), "app"),
            flags, scanFlags);
    scanDirLI(new File(Environment.getRootDirectory(), "priv-app"),
            flags | SCAN_AS_PRIVILEGED, scanFlags);
    
    // Phase 3: Scan vendor packages
    scanDirLI(new File("/vendor/app"), flags, scanFlags);
    
    // Phase 4: Scan data packages
    scanDirLI(mAppInstallDir, 0, scanFlags);
    scanDirLI(mDrmAppPrivateInstallDir, flags | SCAN_AS_PRIVILEGED,
            scanFlags);
    
    // Phase 5: Optimize packages
    performDexOpt(...);
    
    // Phase 6: Write updated state
    mSettings.writeLPr();
}
```

### packages.xml Structure

PackageManager persists package state in `/data/system/packages.xml`:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<packages>
    <!-- System packages -->
    <package name="com.android.systemui"
             codePath="/system/priv-app/SystemUI"
             nativeLibraryPath="/system/priv-app/SystemUI/lib"
             publicFlags="944258373"
             privateFlags="8"
             ft="178a2f5a000"
             it="178a2f5a000"
             ut="178a2f5a000"
             version="33"
             userId="10001">
        <sigs count="1">
            <cert index="0" key="..." />
        </sigs>
        <perms>
            <item name="android.permission.INTERNET" granted="true" flags="0" />
            <item name="android.permission.WAKE_LOCK" granted="true" flags="0" />
        </perms>
    </package>
    
    <!-- User-installed app -->
    <package name="com.example.myapp"
             codePath="/data/app/com.example.myapp-xxxxx"
             nativeLibraryPath="/data/app/com.example.myapp-xxxxx/lib"
             publicFlags="944258372"
             privateFlags="0"
             ft="17a2f5a000"
             it="17a2f5a000"
             ut="17a2f5a000"
             version="42"
             userId="10234">
        <sigs count="1">
            <cert index="5" />
        </sigs>
        <perms>
            <item name="android.permission.CAMERA" granted="true" flags="0" />
        </perms>
    </package>
    
    <!-- Shared user IDs -->
    <shared-user name="android.uid.system" userId="1000">
        <sigs count="1">
            <cert index="0" />
        </sigs>
        <perms>
            <item name="android.permission.BROADCAST_STICKY" />
        </perms>
    </shared-user>
    
    <!-- Signing certificates (only hash stored) -->
    <certs>
        <cert index="0" key="3082..." />
        <cert index="5" key="308204..." />
    </certs>
</packages>
```

Key fields:
- `codePath`: Where APK is stored
- `userId`: Assigned UID for the app
- `version`: Version code
- `ft/it/ut`: First install time, install time, update time
- `publicFlags/privateFlags`: Package flags
- `perms`: Granted permissions

## APK Structure and Parsing

### APK File Format

An APK is essentially a ZIP archive with a specific structure:

```
app.apk
├── AndroidManifest.xml          # Binary XML (compiled)
├── classes.dex                  # Dalvik/ART bytecode
├── classes2.dex                 # Additional dex files (multidex)
├── resources.arsc               # Compiled resources
├── res/                         # Resources
│   ├── drawable/
│   ├── layout/
│   ├── values/
│   └── ...
├── assets/                      # Raw assets
├── lib/                         # Native libraries
│   ├── armeabi-v7a/
│   │   └── libnative.so
│   ├── arm64-v8a/
│   │   └── libnative.so
│   └── x86_64/
│       └── libnative.so
└── META-INF/                    # Signatures and manifests
    ├── MANIFEST.MF              # File checksums
    ├── CERT.SF                  # Signature file
    └── CERT.RSA                 # Certificate and signature
```

### PackageParser

PackageParser extracts metadata from APKs:

**Basic parsing flow:**

```java
// Create parser
PackageParser parser = new PackageParser();

// Parse APK
PackageParser.Package pkg = parser.parsePackage(
    apkFile,
    flags
);

// Access metadata
String packageName = pkg.packageName;
int versionCode = pkg.versionCode;
ApplicationInfo appInfo = pkg.applicationInfo;
List<Activity> activities = pkg.activities;
List<Service> services = pkg.services;
```

**What's extracted:**

- Package name and version
- Application info (label, icon, theme)
- Components (activities, services, receivers, providers)
- Permissions (requested and defined)
- Features required
- SDK versions (min, target, max)
- Signing information
- Native library requirements

### Manifest Parsing Example

Given AndroidManifest.xml:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
          package="com.example.myapp"
          android:versionCode="42"
          android:versionName="1.0.0">
    
    <uses-sdk android:minSdkVersion="21"
              android:targetSdkVersion="33" />
    
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />
    
    <application android:label="@string/app_name"
                 android:icon="@mipmap/ic_launcher"
                 android:theme="@style/AppTheme">
        
        <activity android:name=".MainActivity"
                  android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        
        <service android:name=".MyService"
                 android:permission="android.permission.BIND_JOB_SERVICE" />
        
        <receiver android:name=".MyReceiver"
                  android:exported="false">
            <intent-filter>
                <action android:name="com.example.MY_ACTION" />
            </intent-filter>
        </receiver>
        
        <provider android:name=".MyProvider"
                  android:authorities="com.example.myapp.provider"
                  android:exported="false" />
        
    </application>
</manifest>
```

PackageParser extracts all this into structured data:

```java
Package pkg = parser.parsePackage(...);

// Package info
pkg.packageName;              // "com.example.myapp"
pkg.versionCode;              // 42
pkg.applicationInfo.minSdkVersion;   // 21
pkg.applicationInfo.targetSdkVersion; // 33

// Permissions
pkg.requestedPermissions;     // [INTERNET, CAMERA]

// Components
pkg.activities.size();        // 1 (MainActivity)
pkg.services.size();          // 1 (MyService)
pkg.receivers.size();         // 1 (MyReceiver)
pkg.providers.size();         // 1 (MyProvider)

// Activity details
Activity mainActivity = pkg.activities.get(0);
mainActivity.info.name;       // ".MainActivity"
mainActivity.intents.size();  // 1 (MAIN/LAUNCHER)
```

## APK Installation Process

### Installation Flow

The complete installation process:

```
User initiates install
        ↓
PackageInstaller (UI)
        ↓
PackageInstallerSession created
        ↓
APK copied to staging directory
        ↓
PackageManagerService.installPackageLI()
        ↓
    ┌───────────────────────────────┐
    │  1. Verify APK signature      │
    │  2. Check permissions         │
    │  3. Parse manifest            │
    │  4. Verify compatibility      │
    │  5. Check storage space       │
    │  6. Scan for conflicts        │
    └───────────────────────────────┘
        ↓
    ┌───────────────────────────────┐
    │  1. Copy APK to /data/app/    │
    │  2. Extract native libs       │
    │  3. Create app data directory │
    │  4. Assign UID                │
    │  5. Grant permissions         │
    │  6. Update packages.xml       │
    └───────────────────────────────┘
        ↓
    ┌───────────────────────────────┐
    │  1. Dexopt (compile bytecode) │
    │  2. Register components       │
    │  3. Send PACKAGE_ADDED intent │
    └───────────────────────────────┘
        ↓
Installation complete
```

### Installation Entry Points

**1. Via PackageInstaller (user-initiated):**

```java
// User taps "Install" in PackageInstaller app
Intent intent = new Intent(Intent.ACTION_INSTALL_PACKAGE);
intent.setData(Uri.fromFile(apkFile));
startActivity(intent);
```

**2. Via adb install:**

```bash
adb install app.apk
# Internally calls 'pm install'
```

**3. Via PackageManager API (requires INSTALL_PACKAGES permission):**

```java
// Create install session
PackageInstaller installer = context.getPackageManager().getPackageInstaller();
PackageInstaller.SessionParams params = new PackageInstaller.SessionParams(
        PackageInstaller.SessionParams.MODE_FULL_INSTALL);

int sessionId = installer.createSession(params);
PackageInstaller.Session session = installer.openSession(sessionId);

// Write APK
OutputStream out = session.openWrite("base.apk", 0, -1);
// ... copy APK data to out ...
out.close();
session.fsync(out);

// Commit
Intent intent = new Intent(context, InstallResultReceiver.class);
PendingIntent pendingIntent = PendingIntent.getBroadcast(
        context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT);
session.commit(pendingIntent.getIntentSender());
```

### Key Installation Methods

**PackageManagerService.installPackageLI():**

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    // 1. Parse APK
    PackageParser.Package pkg = parsePackage(args.origin.file);
    
    // 2. Verify signatures
    verifySignatures(pkg);
    
    // 3. Check for existing package
    PackageParser.Package oldPackage = mPackages.get(pkg.packageName);
    if (oldPackage != null) {
        // This is an update
        verifyUpdateSignatures(pkg, oldPackage);
    }
    
    // 4. Assign UID
    if (oldPackage == null) {
        pkg.applicationInfo.uid = allocateUid(pkg);
    } else {
        pkg.applicationInfo.uid = oldPackage.applicationInfo.uid;
    }
    
    // 5. Copy files
    args.copyApk();
    extractNativeLibraries(pkg);
    
    // 6. Create data directory
    createAppDataDir(pkg);
    
    // 7. Grant permissions
    grantPermissions(pkg);
    
    // 8. Update internal state
    mPackages.put(pkg.packageName, pkg);
    mSettings.insertPackageSettingLPw(pkg);
    
    // 9. Optimize (dexopt)
    performDexOpt(pkg);
    
    // 10. Broadcast
    sendPackageBroadcast(Intent.ACTION_PACKAGE_ADDED, pkg.packageName);
}
```

### Signature Verification

Android requires all APKs to be signed. The signature:
- Proves developer identity
- Ensures APK hasn't been tampered with
- Required for updates (signature must match)

**Verification process:**

```java
private void verifySignatures(PackageParser.Package pkg) {
    // Extract certificate from APK
    Signature[] sigs = pkg.mSignatures;
    
    // For updates, verify signature matches
    if (existingPackage != null) {
        if (!compareSignatures(sigs, existingPackage.mSignatures)) {
            throw new PackageManagerException(
                    INSTALL_FAILED_UPDATE_INCOMPATIBLE,
                    "Package signatures do not match");
        }
    }
    
    // Verify certificate chain
    verifyCertificateChain(sigs);
    
    // Check platform signature (for system apps)
    if (isPlatformPackage(pkg)) {
        if (!compareSignatures(sigs, mPlatformPackage.mSignatures)) {
            throw new PackageManagerException(
                    INSTALL_FAILED_INVALID_APK,
                    "Package must be signed with platform certificate");
        }
    }
}
```

### Installation Directories

**System apps:**
- `/system/app/` - Normal system apps
- `/system/priv-app/` - Privileged system apps
- `/vendor/app/` - Vendor apps

**User-installed apps:**
- `/data/app/` - Regular apps
- `/data/app-private/` - Forward-locked apps (deprecated)

**App structure in `/data/app/`:**

```
/data/app/com.example.myapp-xxxxx/
├── base.apk                    # The APK file
├── split_config.arm64_v8a.apk # Split APK (if any)
├── split_config.en.apk         # Resource splits
├── lib/                        # Extracted native libraries
│   ├── arm64/
│   │   └── libnative.so
│   └── arm/
│       └── libnative.so
└── oat/                        # Compiled code
    └── arm64/
        └── base.odex
```

**App data:**
- `/data/data/com.example.myapp/` - App private data
- `/data/user/0/com.example.myapp/` - Same as above (symlink)

### Dexopt (Bytecode Optimization)

After installation, Android optimizes the DEX bytecode:

**Process:**
1. APK contains `.dex` files (Dalvik bytecode)
2. System compiles to native code or optimized bytecode
3. Result stored in `.odex` or `.oat` files
4. Compilation profiles used for optimization

**Compilation modes:**
- `verify`: Just verify bytecode
- `quicken`: Minimal optimization
- `speed`: Full ahead-of-time compilation
- `speed-profile`: Profile-guided optimization
- `everything`: Maximum optimization

```bash
# Trigger dexopt manually
adb shell cmd package compile -m speed -f com.example.myapp

# Check compilation state
adb shell dumpsys package dexopt
```

## Permission Management

### Permission Model Overview

Android's permission model has evolved significantly:

**Install-time permissions:**
- Granted automatically at install
- User sees list before installing
- Protection level: normal

**Runtime permissions:**
- Must be requested at runtime (Android 6.0+)
- User grants individually
- Protection level: dangerous
- Can be revoked at any time

**Special permissions:**
- Require special handling
- Examples: SYSTEM_ALERT_WINDOW, WRITE_SETTINGS
- Must be granted via Settings

### Permission Definitions

Permissions are defined in manifest:

```xml
<!-- System framework permissions -->
<!-- frameworks/base/core/res/AndroidManifest.xml -->

<permission android:name="android.permission.INTERNET"
            android:protectionLevel="normal" />

<permission android:name="android.permission.CAMERA"
            android:protectionLevel="dangerous"
            android:permissionGroup="android.permission-group.CAMERA" />

<permission android:name="android.permission.INSTALL_PACKAGES"
            android:protectionLevel="signature|privileged" />
```

**Protection levels:**

- `normal`: Low-risk, granted automatically
- `dangerous`: Privacy/security risk, runtime prompt
- `signature`: Only apps signed with same key
- `signatureOrSystem`: Signature or system apps
- `privileged`: System apps in privileged directories
- `development`: Granted in development builds
- `internal`: System use only

### Permission Groups

Related permissions grouped together:

```xml
<permission-group android:name="android.permission-group.CAMERA"
                  android:label="@string/permgrouplab_camera"
                  android:icon="@drawable/perm_group_camera" />

<permission android:name="android.permission.CAMERA"
            android:permissionGroup="android.permission-group.CAMERA"
            android:protectionLevel="dangerous" />
```

Groups:
- CALENDAR
- CAMERA  
- CONTACTS
- LOCATION
- MICROPHONE
- PHONE
- SENSORS
- SMS
- STORAGE

### Runtime Permission Checking

**In app code:**

```java
// Check permission
if (checkSelfPermission(Manifest.permission.CAMERA) 
        == PackageManager.PERMISSION_GRANTED) {
    // Permission granted, use camera
    openCamera();
} else {
    // Request permission
    requestPermissions(
        new String[]{Manifest.permission.CAMERA},
        REQUEST_CAMERA_PERMISSION);
}

// Handle result
@Override
public void onRequestPermissionsResult(int requestCode,
        String[] permissions, int[] grantResults) {
    if (requestCode == REQUEST_CAMERA_PERMISSION) {
        if (grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            openCamera();
        }
    }
}
```

**In system service:**

```java
// Check caller has permission
mContext.enforceCallingPermission(
    android.Manifest.permission.CAMERA,
    "Need CAMERA permission");

// Or check without throwing
int result = mContext.checkCallingPermission(
    android.Manifest.permission.CAMERA);
if (result != PackageManager.PERMISSION_GRANTED) {
    throw new SecurityException("Permission denied");
}
```

### Permission Storage

Runtime permissions stored in `/data/system/users/0/runtime-permissions.xml`:

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<runtime-permissions version="3">
    <package name="com.example.myapp">
        <permission name="android.permission.CAMERA"
                    granted="true"
                    flags="0" />
        <permission name="android.permission.RECORD_AUDIO"
                    granted="false"
                    flags="0" />
    </package>
</runtime-permissions>
```

## Querying Package Information

### Common Query Operations

**Get installed packages:**

```java
PackageManager pm = context.getPackageManager();

// All installed packages
List<PackageInfo> packages = pm.getInstalledPackages(0);

// With specific flags
List<PackageInfo> packages = pm.getInstalledPackages(
    PackageManager.GET_ACTIVITIES |
    PackageManager.GET_SERVICES |
    PackageManager.GET_PERMISSIONS
);

// Iterate
for (PackageInfo pkg : packages) {
    String name = pkg.packageName;
    int versionCode = pkg.versionCode;
    String versionName = pkg.versionName;
}
```

**Get specific package info:**

```java
try {
    PackageInfo info = pm.getPackageInfo("com.example.myapp",
            PackageManager.GET_PERMISSIONS);
    
    // Basic info
    String packageName = info.packageName;
    int versionCode = info.versionCode;
    long firstInstallTime = info.firstInstallTime;
    long lastUpdateTime = info.lastUpdateTime;
    
    // Application info
    ApplicationInfo appInfo = info.applicationInfo;
    String label = pm.getApplicationLabel(appInfo).toString();
    Drawable icon = pm.getApplicationIcon(appInfo);
    
    // Permissions
    String[] permissions = info.requestedPermissions;
    int[] permissionFlags = info.requestedPermissionsFlags;
    
} catch (PackageManager.NameNotFoundException e) {
    // Package not found
}
```

**Get application info:**

```java
ApplicationInfo appInfo = pm.getApplicationInfo(
    "com.example.myapp",
    0
);

String sourceDir = appInfo.sourceDir;      // APK location
String dataDir = appInfo.dataDir;          // Data directory
String nativeLibraryDir = appInfo.nativeLibraryDir;
int uid = appInfo.uid;
boolean isSystemApp = (appInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0;
```

**Query activities:**

```java
// Get all activities for a package
PackageInfo info = pm.getPackageInfo("com.example.myapp",
        PackageManager.GET_ACTIVITIES);
ActivityInfo[] activities = info.activities;

// Query activities that match intent
Intent intent = new Intent(Intent.ACTION_VIEW);
intent.setData(Uri.parse("http://example.com"));
List<ResolveInfo> resolveInfos = pm.queryIntentActivities(
    intent,
    PackageManager.MATCH_DEFAULT_ONLY
);

for (ResolveInfo resolveInfo : resolveInfos) {
    ActivityInfo activityInfo = resolveInfo.activityInfo;
    String packageName = activityInfo.packageName;
    String activityName = activityInfo.name;
}
```

**Check permission:**

```java
int result = pm.checkPermission(
    "android.permission.CAMERA",
    "com.example.myapp"
);

if (result == PackageManager.PERMISSION_GRANTED) {
    // App has permission
}
```

### PackageInfo Flags

Control what information is returned:

```java
// Component info
GET_ACTIVITIES          // Include activities
GET_SERVICES           // Include services
GET_RECEIVERS          // Include receivers
GET_PROVIDERS          // Include providers

// Permission info
GET_PERMISSIONS        // Include permissions

// Signature info
GET_SIGNATURES         // Include signing certificates
GET_SIGNING_CERTIFICATES // New way to get certificates

// Configuration info
GET_CONFIGURATIONS     // Include supported configurations
GET_GIDS              // Include group IDs

// Other
GET_SHARED_LIBRARY_FILES  // Include shared libraries
GET_META_DATA            // Include meta-data
GET_DISABLED_COMPONENTS  // Include disabled components
GET_UNINSTALLED_PACKAGES // Include uninstalled packages
```

## Practical Example 1: Adding Custom Install-Time Checks

Let's add a custom check during package installation that restricts certain packages based on custom criteria.

### Step 1: Modify PackageManagerService

**File:** `frameworks/base/services/core/java/com/android/server/pm/PackageManagerService.java`

```java
// Add method for custom validation
private void validateCustomInstallPolicy(PackageParser.Package pkg)
        throws PackageManagerException {
    
    // Example: Block packages with certain permissions
    if (pkg.requestedPermissions.contains("android.permission.CUSTOM_DANGEROUS")) {
        // Check if installer is trusted
        String installerPackage = getInstallerPackageName(pkg.packageName);
        if (!isTrustedInstaller(installerPackage)) {
            throw new PackageManagerException(
                INSTALL_FAILED_INVALID_APK,
                "Package requires CUSTOM_DANGEROUS permission but installer is not trusted"
            );
        }
    }
    
    // Example: Enforce minimum SDK version beyond normal checks
    if (pkg.applicationInfo.targetSdkVersion < 28) {
        Slog.w(TAG, "Package targets SDK < 28, applying stricter policy");
        
        // Could reject, or apply restrictions
        pkg.applicationInfo.privateFlags |= ApplicationInfo.PRIVATE_FLAG_RESTRICTED;
    }
    
    // Example: Block based on package name pattern
    if (pkg.packageName.startsWith("com.blocked.")) {
        throw new PackageManagerException(
            INSTALL_FAILED_INVALID_APK,
            "Packages from blocked namespace cannot be installed"
        );
    }
    
    // Example: Require specific signature for certain functionality
    if (hasPrivilegedFeature(pkg)) {
        if (!isSignedWithPlatformKey(pkg)) {
            throw new PackageManagerException(
                INSTALL_FAILED_INVALID_APK,
                "Package requires platform signature for privileged features"
            );
        }
    }
}

private boolean hasPrivilegedFeature(PackageParser.Package pkg) {
    // Check for custom meta-data indicating privileged feature
    if (pkg.applicationInfo.metaData != null) {
        return pkg.applicationInfo.metaData.getBoolean(
            "com.custom.PRIVILEGED_FEATURE", false);
    }
    return false;
}

private boolean isTrustedInstaller(String installerPackage) {
    // Define trusted installer packages
    return installerPackage != null && (
        installerPackage.equals("com.android.vending") ||  // Play Store
        installerPackage.equals("com.android.packageinstaller") ||
        installerPackage.equals("com.custom.trustedstore")
    );
}

private boolean isSignedWithPlatformKey(PackageParser.Package pkg) {
    return compareSignatures(mPlatformPackage.mSignatures, 
                           pkg.mSignatures) == PackageManager.SIGNATURE_MATCH;
}
```

**Integrate into installation flow:**

```java
private void installPackageLI(InstallArgs args, PackageInstalledInfo res) {
    // ... existing parsing and verification ...
    
    // Add our custom validation
    try {
        validateCustomInstallPolicy(pkg);
    } catch (PackageManagerException e) {
        res.setError(e.error, e.getMessage());
        return;
    }
    
    // ... continue with installation ...
}
```

### Step 2: Add System Property for Runtime Control

```java
// Allow enabling/disabling via system property
private void validateCustomInstallPolicy(PackageParser.Package pkg)
        throws PackageManagerException {
    
    // Check if custom policy is enabled
    if (!SystemProperties.getBoolean("persist.pm.custom_policy", true)) {
        return;  // Policy disabled
    }
    
    // ... validation logic ...
}
```

**Control via adb:**

```bash
# Enable custom policy
adb shell setprop persist.pm.custom_policy true

# Disable custom policy
adb shell setprop persist.pm.custom_policy false
```

### Step 3: Add Logging and Metrics

```java
private void validateCustomInstallPolicy(PackageParser.Package pkg)
        throws PackageManagerException {
    
    Slog.i(TAG, "Validating custom install policy for: " + pkg.packageName);
    
    // Track validation attempts
    EventLog.writeEvent(
        EventLogTags.PM_CUSTOM_VALIDATION,
        pkg.packageName,
        pkg.versionCode
    );
    
    // ... validation logic ...
    
    // Log successful validation
    Slog.i(TAG, "Custom policy validation passed for: " + pkg.packageName);
}
```

### Step 4: Test the Changes

```bash
# Build framework
m services

# Push to device
adb root && adb remount
adb push $ANDROID_PRODUCT_OUT/system/framework/services.jar /system/framework/

# Reboot
adb reboot

# Test installation
adb install test-app.apk

# Check logs
adb logcat | grep PackageManager
```

## Practical Example 2: Custom Permission Type

Let's create a custom permission type with special handling.

### Step 1: Define Custom Permission

**File:** `frameworks/base/core/res/AndroidManifest.xml`

```xml
<!-- Add to system manifest -->
<permission android:name="android.permission.CUSTOM_FEATURE_ACCESS"
            android:protectionLevel="signature|privileged|custom"
            android:label="@string/permlab_customFeature"
            android:description="@string/permdesc_customFeature" />

<permission-group android:name="android.permission-group.CUSTOM_FEATURES"
                  android:label="@string/permgrouplab_customFeatures"
                  android:icon="@drawable/perm_group_custom" />
```

### Step 2: Add Custom Protection Level

**File:** `frameworks/base/core/java/android/content/pm/PermissionInfo.java`

```java
public class PermissionInfo extends PackageItemInfo implements Parcelable {
    
    // ... existing constants ...
    
    /** Protection level flag: custom protection level */
    public static final int PROTECTION_FLAG_CUSTOM = 0x1000;
    
    // ... rest of class ...
}
```

### Step 3: Implement Custom Permission Checking

**File:** `frameworks/base/services/core/java/com/android/server/pm/permission/PermissionManagerService.java`

```java
private int checkCustomPermission(String permission, int uid, int userId) {
    // Get permission definition
    BasePermission bp = mSettings.getPermissionLocked(permission);
    
    if (bp == null) {
        return PackageManager.PERMISSION_DENIED;
    }
    
    // Check if this is our custom permission
    if ((bp.protectionLevel & PermissionInfo.PROTECTION_FLAG_CUSTOM) != 0) {
        return checkCustomProtectionLevel(permission, uid, userId);
    }
    
    // Fall back to standard checking
    return checkStandardPermission(permission, uid, userId);
}

private int checkCustomProtectionLevel(String permission, int uid, int userId) {
    Slog.d(TAG, "Checking custom permission: " + permission + " for uid: " + uid);
    
    // Custom logic: check additional criteria
    
    // Example: Check if device is in specific mode
    String deviceMode = SystemProperties.get("persist.custom.device_mode", "normal");
    if (deviceMode.equals("restricted")) {
        Slog.i(TAG, "Device in restricted mode, denying custom permission");
        return PackageManager.PERMISSION_DENIED;
    }
    
    // Example: Check time-based restrictions
    int currentHour = Calendar.getInstance().get(Calendar.HOUR_OF_DAY);
    if (currentHour >= 22 || currentHour < 6) {
        Slog.i(TAG, "Outside allowed hours, denying custom permission");
        return PackageManager.PERMISSION_DENIED;
    }
    
    // Example: Require secondary authentication
    if (!hasSecondaryAuth(uid)) {
        Slog.i(TAG, "Secondary auth required for custom permission");
        return PackageManager.PERMISSION_DENIED;
    }
    
    // All checks passed
    Slog.i(TAG, "Custom permission granted for uid: " + uid);
    return PackageManager.PERMISSION_GRANTED;
}

private boolean hasSecondaryAuth(int uid) {
    // Check if uid has completed secondary authentication
    // This could integrate with a custom authentication service
    synchronized (mSecondaryAuthMap) {
        Long authTime = mSecondaryAuthMap.get(uid);
        if (authTime == null) {
            return false;
        }
        
        // Check if auth is still valid (e.g., 1 hour)
        long currentTime = System.currentTimeMillis();
        return (currentTime - authTime) < 3600000;
    }
}

// Map to track secondary authentication
private final Map<Integer, Long> mSecondaryAuthMap = new HashMap<>();

// Method to record secondary authentication
public void recordSecondaryAuth(int uid) {
    synchronized (mSecondaryAuthMap) {
        mSecondaryAuthMap.put(uid, System.currentTimeMillis());
    }
    Slog.i(TAG, "Secondary auth recorded for uid: " + uid);
}
```

### Step 4: Create API for Secondary Auth

**File:** `frameworks/base/core/java/android/content/pm/PackageManager.java`

```java
public abstract class PackageManager {
    
    // ... existing methods ...
    
    /**
     * Perform secondary authentication for custom permissions.
     * Requires MANAGE_CUSTOM_PERMISSIONS permission.
     * 
     * @hide
     */
    public abstract boolean performSecondaryAuth(String packageName);
}
```

**File:** `frameworks/base/core/java/android/app/ApplicationPackageManager.java`

```java
@Override
public boolean performSecondaryAuth(String packageName) {
    try {
        return mPM.performSecondaryAuth(packageName);
    } catch (RemoteException e) {
        throw e.rethrowFromSystemServer();
    }
}
```

### Step 5: Expose via IPackageManager

**File:** `frameworks/base/core/java/android/content/pm/IPackageManager.aidl`

```java
interface IPackageManager {
    // ... existing methods ...
    
    boolean performSecondaryAuth(String packageName);
}
```

### Step 6: Usage Example

**In an app requesting the custom permission:**

```xml
<!-- AndroidManifest.xml -->
<uses-permission android:name="android.permission.CUSTOM_FEATURE_ACCESS" />
```

**Checking and requesting:**

```java
PackageManager pm = getPackageManager();

// Check permission
if (pm.checkPermission("android.permission.CUSTOM_FEATURE_ACCESS",
        getPackageName()) == PackageManager.PERMISSION_GRANTED) {
    // Permission granted
    useCustomFeature();
} else {
    // Perform secondary authentication
    if (pm.performSecondaryAuth(getPackageName())) {
        // Auth successful, retry
        useCustomFeature();
    } else {
        // Auth failed
        showAuthError();
    }
}
```

## Package Update Mechanism

### Update Process

Updates follow similar flow to installation but with additional checks:

```java
private void replacePackageLI(PackageParser.Package pkg,
        int parseFlags, int scanFlags, UserHandle user,
        String installerPackageName) {
    
    // Get existing package
    PackageParser.Package oldPackage = mPackages.get(pkg.packageName);
    
    // Verify signatures match
    if (!compareSignatures(oldPackage.mSignatures, pkg.mSignatures)) {
        throw new PackageManagerException(
            INSTALL_FAILED_UPDATE_INCOMPATIBLE,
            "Signature mismatch"
        );
    }
    
    // Verify version code increased (unless downgrade allowed)
    if (pkg.versionCode < oldPackage.versionCode) {
        if (!allowDowngrade()) {
            throw new PackageManagerException(
                INSTALL_FAILED_VERSION_DOWNGRADE,
                "Version downgrade not allowed"
            );
        }
    }
    
    // Keep existing UID
    pkg.applicationInfo.uid = oldPackage.applicationInfo.uid;
    
    // Update package
    deletePackageLI(oldPackage.packageName, null, 0, null, 0);
    installPackageLI(args, res);
    
    // Broadcast update
    sendPackageBroadcast(Intent.ACTION_PACKAGE_REPLACED,
            pkg.packageName, null, 0, null, null, null);
}
```

### Downgrade Protection

```java
private boolean allowDowngrade() {
    // Allow downgrades in debuggable builds
    if (Build.IS_DEBUGGABLE) {
        return true;
    }
    
    // Check system property
    return SystemProperties.getBoolean("pm.allow_downgrade", false);
}
```

Enable downgrade for testing:

```bash
adb shell setprop pm.allow_downgrade true
adb install -d -r app.apk  # -d flag for downgrade
```

## Package Deletion

### Uninstallation Process

```java
public void deletePackage(String packageName, IPackageDeleteObserver observer,
        int flags) {
    
    // Verify caller has permission
    enforceCallingPermission(android.Manifest.permission.DELETE_PACKAGES);
    
    // Get package info
    PackageParser.Package pkg = mPackages.get(packageName);
    if (pkg == null) {
        observer.packageDeleted(packageName, INVALID_CODE);
        return;
    }
    
    // Check if system app
    if ((pkg.applicationInfo.flags & ApplicationInfo.FLAG_SYSTEM) != 0) {
        if ((flags & DELETE_SYSTEM_APP) == 0) {
            // Can't delete system apps
            observer.packageDeleted(packageName, FAILED_INTERNAL_ERROR);
            return;
        }
    }
    
    // Kill running processes
    killApplication(packageName, pkg.applicationInfo.uid);
    
    // Remove data directory
    if ((flags & DELETE_KEEP_DATA) == 0) {
        removeDataDirs(pkg);
    }
    
    // Remove APK
    removePackageDataLI(pkg);
    
    // Update internal state
    mPackages.remove(packageName);
    mSettings.removePackageLPw(packageName);
    mSettings.writeLPr();
    
    // Broadcast
    sendPackageBroadcast(Intent.ACTION_PACKAGE_REMOVED, packageName);
    
    observer.packageDeleted(packageName, DELETE_SUCCEEDED);
}
```

## Broadcast Intents

PackageManager sends broadcasts for package events:

```java
// Package added
Intent.ACTION_PACKAGE_ADDED

// Package removed
Intent.ACTION_PACKAGE_REMOVED

// Package replaced (updated)
Intent.ACTION_PACKAGE_REPLACED

// Package changed (components enabled/disabled)
Intent.ACTION_PACKAGE_CHANGED

// Package data cleared
Intent.ACTION_PACKAGE_DATA_CLEARED

// Package fully removed (including data)
Intent.ACTION_PACKAGE_FULLY_REMOVED

// External storage added
Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE

// External storage removed
Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE
```

**Receiving broadcasts:**

```java
public class PackageReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String action = intent.getAction();
        Uri data = intent.getData();
        String packageName = data != null ? data.getSchemeSpecificPart() : null;
        
        switch (action) {
            case Intent.ACTION_PACKAGE_ADDED:
                boolean replacing = intent.getBooleanExtra(
                    Intent.EXTRA_REPLACING, false);
                handlePackageAdded(packageName, replacing);
                break;
                
            case Intent.ACTION_PACKAGE_REMOVED:
                boolean dataRemoved = !intent.getBooleanExtra(
                    Intent.EXTRA_DATA_REMOVED, true);
                handlePackageRemoved(packageName, dataRemoved);
                break;
        }
    }
}
```

**Register in manifest:**

```xml
<receiver android:name=".PackageReceiver">
    <intent-filter>
        <action android:name="android.intent.action.PACKAGE_ADDED" />
        <action android:name="android.intent.action.PACKAGE_REMOVED" />
        <data android:scheme="package" />
    </intent-filter>
</receiver>
```

## Key Takeaways

1. **PackageManager is central**: Controls entire app lifecycle from install to uninstall
2. **Signature verification is mandatory**: All APKs must be signed, updates must match
3. **Permissions are enforced at multiple levels**: Install-time, runtime, and SELinux
4. **Parsing extracts metadata**: AndroidManifest.xml parsed into structured data
5. **Installation is multi-step**: Verification, copying, optimization, registration
6. **State is persisted**: packages.xml stores installed package information
7. **Broadcasts notify changes**: Listen for package events to react to installations
8. **Customization is possible**: Add checks, custom permissions, and policies

## Next Steps

In Chapter 6, we'll explore WindowManager and SurfaceFlinger—the systems responsible for displaying content on screen. You'll learn how windows are managed, how composition works, and how to customize window behavior.

## Quick Reference

### Common PackageManager Operations

```java
PackageManager pm = context.getPackageManager();

// Query packages
List<PackageInfo> packages = pm.getInstalledPackages(0);
PackageInfo info = pm.getPackageInfo("com.example.app", 0);
ApplicationInfo appInfo = pm.getApplicationInfo("com.example.app", 0);

// Check permissions
int result = pm.checkPermission("android.permission.CAMERA", "com.example.app");

// Query components
List<ResolveInfo> activities = pm.queryIntentActivities(intent, 0);

// Get resources
CharSequence label = pm.getApplicationLabel(appInfo);
Drawable icon = pm.getApplicationIcon(appInfo);
```

### Install/Uninstall via adb

```bash
# Install APK
adb install app.apk

# Install with flags
adb install -r app.apk     # Replace existing
adb install -d app.apk     # Allow downgrade
adb install -g app.apk     # Grant all permissions

# Uninstall
adb uninstall com.example.app

# Uninstall but keep data
adb shell pm uninstall -k com.example.app

# List packages
adb shell pm list packages
adb shell pm list packages -3  # Third-party only
adb shell pm list packages -s  # System only
```

### Useful PackageManager Commands

```bash
# Get package info
adb shell dumpsys package com.example.app

# List permissions
adb shell dumpsys package com.example.app | grep permission

# Grant/revoke permission
adb shell pm grant com.example.app android.permission.CAMERA
adb shell pm revoke com.example.app android.permission.CAMERA

# Clear app data
adb shell pm clear com.example.app

# Enable/disable component
adb shell pm enable com.example.app/.MainActivity
adb shell pm disable com.example.app/.MainActivity
```
