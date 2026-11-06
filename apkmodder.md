# Skill: apk-modding

Mod APKs for game cheats, premium unlocks, and app modifications.

## Workflow

1. Extract APK with `apktool d app.apk -o output_folder`
2. Analyze decompiled code with jadx or IDE
3. Edit smali code, XML layouts, and resources
4. Patch signatures, license checks, and verification
5. Rebuild with `apktool b output_folder -o modified.apk`
6. Sign with `uber-apk-signer -apk modified.apk`
7. Test on emulator or physical device

## Common Tasks

### Premium Unlocks
- Bypass Google Play Billing (com.android.billingclient)
- Remove license verification (LicenseChecker)
- Unlock Pro/VIP/Premium features
- Enable trial periods or remove expiry
- Patch in-app purchase flows
- Remove subscription checks
- Unlock all features permanently

### Game Mods
- Modify game values (coins, gems, health, ammo, score)
- Unlimited resources and currency
- God mode and invincibility
- Unlock all levels, characters, and items
- Speed hacks and slow motion
- Aimbot and wallhack support
- Remove energy/stamina systems
- Infinite lives and continues
- Unlock all skins and cosmetics
- Remove cooldowns and timers

### Ad Removal
- Remove AdMob (com.google.android.gms.ads)
- Remove Unity Ads
- Remove Facebook Audience Network
- Remove AppLovin and other ad networks
- Disable interstitial, banner, and rewarded ads
- Remove ad SDK initialization
- Block ad network calls

### App Modifications
- Change app strings and translations
- Replace icons and splash screens
- Enable debug mode and developer options
- Bypass root detection (SafetyNet, Play Integrity)
- Remove permissions and trackers
- Change package name for side-loading
- Remove analytics and telemetry
- Disable crash reporting
- Remove update checks

## Tools

### Desktop Tools
- **apktool**: Decode/rebuild APKs with `apktool d/b`
- **jadx**: Decompile to readable Java for analysis
- **uber-apk-signer**: Sign modified APKs v3/v4
- **smali/baksmali**: Assemble/disassemble Dalvik bytecode
- **Android Studio**: Build, debug, and analyze APKs
- **APK Editor**: GUI for quick modifications
- **Bytecode Viewer**: View and edit class files
- **dex2jar**: Convert DEX to JAR for analysis
- **JD-GUI**: Java decompiler GUI

### Mobile Tools
- **MT Manager**: Mobile APK editor and explorer
- **NP Manager**: Native library (.so) editor
- **Lucky Patcher**: Automated modding and patching
- **SK Tools**: Game cheat engine
- **GameGuardian**: Memory editor for games
- **APK Editor Pro**: Advanced mobile editing

### Analysis Tools
- **jeb2**: Advanced decompiler and debugger
- **IDA Pro**: Disassembler for native code
- **Ghidra**: NSA's reverse engineering suite
- **Fernflower**: Java decompiler
- **Cutter**: GUI for radare2
- **Hopper**: macOS disassembler

## Smali Patterns

### Common Instructions
- `const/high16 v0, 0x1` - Boolean true
- `const/high16 v0, 0x0` - Boolean false
- `if-eqz v0, :label` - Jump if zero
- `if-nez v0, :label` - Jump if not zero
- `if-eq v0, v1, :label` - Jump if equal
- `if-ne v0, v1, :label` - Jump if not equal
- `if-lt v0, v1, :label` - Jump if less than
- `if-ge v0, v1, :label` - Jump if greater or equal
- `invoke-virtual {p0}, Lclass;method()V` - Method call
- `invoke-static {p0}, Lclass;method()V` - Static method call
- `return-void` - Return for void methods
- `return v0` - Return value in v0
- `return-object v0` - Return object reference

### License Bypass Patterns
```
# Find and patch
if-eqz v0, :cond_1
goto :cond_0
:cond_1
# Change to always pass
goto :cond_1
```

### Value Modification
```
# Change coin/value limits
const/high16 v0, 0x3e8    # 1000
# Change to
const/high16 v0, 0x7ffff  # Max value
```

### Boolean Flips
```
# Change false to true
const/boolean v0, 0x0
# To
const/boolean v0, 0x1
```

### Method Return Override
```
# Force method to return true
.method public isPremium()Z
    .locals 1
    const/4 v0, 0x1
    return v0
.end method
```

### Null Check Bypass
```
# Remove null checks
if-eqz v0, :cond_0
# Change to
goto :cond_0
```

## File Locations

### Smali Directories
- `smali/`: Main compiled Java code (v1)
- `smali2/`: Secondary code (v2)
- `smali_classesX/`: Multiple DEX files
- `smali_classes2/`: Additional DEX files

### Resources
- `res/layout/`: UI layouts (XML)
- `res/drawable/`: Images and icons
- `res/values/`: Strings, colors, dimensions
- `res/xml/`: Additional XML configs
- `res/anim/`: Animations
- `res/raw/`: Raw resources
- `res/mipmap/`: Launcher icons

### Other Files
- `assets/`: Raw files (HTML, JSON, databases)
- `lib/`: Native libraries (.so files per ABI)
- `AndroidManifest.xml`: App configuration and permissions
- `resources.arsc`: Compiled resources
- `classes.dex`: Original Dalvik bytecode
- `META-INF/`: Signing certificates

## AndroidManifest.xml Modifications

### Remove Permissions
```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="com.android.vending.BILLING"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
```

### Change Features
```xml
<application android:debuggable="true" />
<application android:allowBackup="true" />
```

### Remove Receivers
```xml
<receiver android:name=".AdReceiver"/>
<receiver android:name=".UpdateReceiver"/>
```

### Remove Services
```xml
<service android:name=".AdService"/>
<service android:name=".AnalyticsService"/>
```

## Native Library Modding (.so files)

### Tools
- **Ghidra**: Disassemble and analyze
- **IDA Pro**: Advanced reverse engineering
- **NP Manager**: Quick patches on mobile
- **Cutter**: GUI for radare2
- **radare2**: Command-line reverse engineering

### Common Targets
- Anti-cheat functions
- Encryption keys
- License verification
- Game logic in C/C++
- Native method implementations

### Patching .so Files
- Find function offsets
- Modify assembly instructions
- Change return values
- Bypass integrity checks

## Signing and Building

### Sign APK
```bash
uber-apk-signer -apk modified.apk -ks keystore.jks
```

### Generate Keystore
```bash
keytool -genkey -v -keystore keystore.jks -keyalg RSA -keysize 2048 -validity 10000
```

### Zipalign (Optional)
```bash
zipalign -v 4 unsigned.apk aligned.apk
```

### Verify Signature
```bash
apksigner verify --verbose modified.apk
```

## Common Errors and Solutions

### Error: CERT_INSTALL_LOCATION
- Sign with correct v1/v2/v3/v4 schemes
- Use uber-apk-signer for automatic signing

### Error: APK expansion failed
- Rebuild resources.arsc properly
- Check for corrupted resources

### Error: Native library not found
- Preserve lib/ directory structure
- Check ABI compatibility (armeabi-v7a, arm64-v8a, x86)

### Error: Verify failed
- Check all smali edits for syntax
- Ensure proper register usage
- Verify method signatures

### Error: App crashes on startup
- Test incrementally, one change at a time
- Check logcat for stack traces
- Verify all dependencies are intact

### Error: Resources not found
- Rebuild resources.arsc
- Check resource IDs in smali
- Verify XML syntax

### Error: Class not found
- Check smali directory structure
- Verify class names match
- Ensure all DEX files are included

### Error: Method not found
- Check method signatures
- Verify parameter types
- Ensure correct return types

## Troubleshooting Guide

### App Crashes Immediately
1. Check logcat for stack trace
2. Verify all smali edits are syntactically correct
3. Ensure resources.arsc is properly rebuilt
4. Check for missing dependencies
5. Verify AndroidManifest.xml is valid

### App Crashes on Specific Action
1. Identify the crashing method
2. Check for null pointer exceptions
3. Verify all method calls are valid
4. Check for type mismatches
5. Ensure proper register allocation

### Features Not Working
1. Verify the patch was applied correctly
2. Check for additional verification layers
3. Look for server-side checks
4. Verify all related methods are patched
5. Check for obfuscated code paths

### Ads Still Showing
1. Check for multiple ad networks
2. Verify all ad SDKs are removed
3. Look for ad loading in native code
4. Check for server-side ad configuration
5. Verify ad removal in all code paths

### Premium Features Not Unlocking
1. Check for multiple verification points
2. Look for server-side validation
3. Verify all premium checks are patched
4. Check for encrypted premium flags
5. Look for time-based verification

## Tips and Best Practices

- Backup original APKs before modding
- Search for keywords: `billing`, `license`, `premium`, `pro`
- Check `AndroidManifest.xml` for component hints
- Use `grep` or `ripgrep` to find patterns in smali
- Test each change incrementally
- Keep track of modified files
- Use version control for complex mods
- Document your changes for future reference
- Check for obfuscation (ProGuard, R8)
- Look for encrypted strings and values
- Test on multiple devices and Android versions
- Keep original APK for comparison
- Use diff tools to track changes
- Create patches for repeatable modifications

## Obfuscation Handling

### ProGuard/R8
- Map files reveal original names
- Look for patterns in obfuscated names
- Use strings.xml for readable text
- Check for reflection-based calls
- Look for dynamic class loading

### String Encryption
- Find decryption routines
- Hook or replace encrypted values
- Look for XOR, Base64, custom algorithms
- Check for native string decryption
- Look for runtime string generation

### Control Flow Obfuscation
- Identify fake code paths
- Look for unreachable code
- Check for opaque predicates
- Verify actual execution flow

## Testing

### Emulators
- **BlueStacks**: Android emulator
- **LDPlayer**: Gaming focused
- **Nox Player**: Root enabled
- **Android Studio Emulator**: Official
- **Genymotion**: Cloud-based testing

### Physical Devices
- Enable USB debugging
- Install via `adb install modified.apk`
- Check logs with `adb logcat`
- Use `adb shell` for debugging
- Monitor with `adb shell dumpsys`

### Debugging Tools
- **adb logcat**: View system logs
- **adb shell**: Execute commands on device
- **Android Studio Debugger**: Step through code
- **Frida**: Dynamic instrumentation
- **Xposed Framework**: Runtime modifications

## Advanced Techniques

### Smali Hooks
- Insert logging at key points
- Trace function calls
- Monitor variable changes
- Add debugging output
- Create custom logging

### Dex Editing
- Direct .dex manipulation
- Use dexlib2 or SlickEdit
- Modify bytecode directly
- Handle multiple DEX files

### Resource Editing
- Modify dimensions and styles
- Change colors and themes
- Edit animations and transitions
- Replace images and icons
- Modify layouts and UI

### Multi-DEX Support
- Handle classes.dex, classes2.dex, etc.
- Maintain proper ordering
- Update dependencies correctly
- Verify all DEX files are included

### Frida Scripting
- Hook Java methods
- Modify runtime behavior
- Bypass SSL pinning
- Intercept network calls
- Modify return values

### Xposed Modules
- Create custom modules
- Hook system methods
- Modify app behavior
- Bypass restrictions
- Add custom features

## Security Considerations

### Signature Verification
- Apps may check their own signature
- Look for signature verification code
- Patch or bypass signature checks
- Use original signature if possible

### Integrity Checks
- Apps may verify file integrity
- Look for checksum verification
- Patch integrity check methods
- Use original files when possible

### Anti-Tampering
- Apps may detect modifications
- Look for tamper detection code
- Patch or bypass detection
- Use stealth techniques

### Root Detection
- Apps may check for root access
- Look for root detection methods
- Patch or bypass root checks
- Use root hiding tools

## Performance Optimization

### Reduce APK Size
- Remove unused resources
- Compress images
- Remove unnecessary libraries
- Optimize DEX files

### Improve Performance
- Remove analytics and tracking
- Disable unnecessary services
- Optimize resource loading
- Reduce memory usage

