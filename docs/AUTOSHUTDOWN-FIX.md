# Autoshutdown Persistence Fix

## Problem Statement

The Innioasis Y1 MP3 player has an autoshutdown timer feature with multiple timing options, but the selected setting is only stored in RAM. When the device restarts, the setting reverts to default, requiring the user to reconfigure it every time.

## Investigation Plan

### 1. Extract and Explore the Firmware

**On Windows (PowerShell):**

```powershell
# Run the extraction helper
.\tools\extract-system-windows.ps1
```

This extracts `system.img` from your `staging/rom.zip` to `staging/extracted-system/`.

**In WSL/Linux (required for mounting):**

```bash
# If system.img is sparse, convert it
simg2img staging/extracted-system/system.img staging/extracted-system/system-raw.img

# Mount read-only to explore
sudo mkdir -p /mnt/y1-system
sudo mount -o loop,ro staging/extracted-system/system-raw.img /mnt/y1-system

# Extract the music APK (or settings APK if separate)
cp /mnt/y1-system/app/com.innioasis.y1*.apk staging/
ls /mnt/y1-system/app/  # List all APKs to find settings-related ones
```

### 2. Decompile the APK to Find Autoshutdown Logic

```bash
# Use the project's apktool
cd staging
java -jar ../tools/apktool-2.9.3.jar d com.innioasis.y1_*.apk -o y1-apk-investigation

# Search for autoshutdown-related code
cd y1-apk-investigation
grep -ri "shutdown" smali*/
grep -ri "timer" smali*/ | grep -i "auto\|power\|sleep"
grep -ri "shutdown\|timer" res/values/
grep -ri "shutdown" res/xml/  # Check for preferences XML
```

### 3. Locate the Setting Storage Issue

Look for patterns like:

**In smali code - BAD (only RAM):**
```smali
# This stores to a field but never persists
iput v0, p0, Lcom/innioasis/SomeClass;->mAutoShutdownTimer:I
```

**In smali code - GOOD (persists):**
```smali
# This uses SharedPreferences
invoke-virtual {v0, v1}, Landroid/content/SharedPreferences$Editor;->putInt(Ljava/lang/String;I)Landroid/content/SharedPreferences$Editor;
invoke-virtual {v0}, Landroid/content/SharedPreferences$Editor;->apply()V
```

### 4. Find Where the Setting is Changed

Search for:
- Menu handlers that set the autoshutdown timer
- Preference screens (look in `res/xml/preferences*.xml`)
- Methods that handle timer selection (look for spinners, radio buttons, or list preferences)

### 5. Find Where the Setting is Read on Boot

Search for:
- Application onCreate methods
- Service onCreate/onStartCommand methods
- BroadcastReceivers for BOOT_COMPLETED

## Patch Strategy

Once we identify the location, we'll create a patch following the project's pattern:

### Option A: SharedPreferences Patch (Most Common)

If the code currently stores to a field, we'll intercept the setter and add SharedPreferences persistence:

**File:** `src/patches/patch_y1_apk.py`

Add a new patch section (e.g., "Patch AS1: Autoshutdown Persistence"):

```python
# ============================================================
# Patch AS1: Autoshutdown timer persistence fix
# ============================================================
# Current behavior: autoshutdown timer selection stored only
# in memory (field), lost on restart.
# Fix: intercept the setter, persist to SharedPreferences,
# and restore on Application.onCreate.

print(f"\nPatch AS1: Autoshutdown persistence in music app")

# Find the method that sets the autoshutdown timer
# Replace the setter to also write to SharedPreferences

OLD_AUTOSHUTDOWN_SETTER = """
	# Original code that only sets field
	iput v0, p0, Lcom/innioasis/SettingsClass;->mAutoShutdownTimer:I

	return-void
"""

NEW_AUTOSHUTDOWN_SETTER = """
	# Store to field (original behavior)
	iput v0, p0, Lcom/innioasis/SettingsClass;->mAutoShutdownTimer:I

	# PATCH: Also persist to SharedPreferences
	const-string v1, "settings"

	const/4 v2, 0x0

	invoke-virtual {p0, v1, v2}, Landroid/content/Context;->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;

	move-result-object v1

	invoke-interface {v1}, Landroid/content/SharedPreferences;->edit()Landroid/content/SharedPreferences$Editor;

	move-result-object v1

	const-string v2, "autoShutdownTimer"

	invoke-interface {v1, v2, v0}, Landroid/content/SharedPreferences$Editor;->putInt(Ljava/lang/String;I)Landroid/content/SharedPreferences$Editor;

	invoke-interface {v1}, Landroid/content/SharedPreferences$Editor;->apply()V

	return-void
"""
```

And add restoration on boot in the Application or Service onCreate:

```python
OLD_APP_ONCREATE_TAIL = """
	# Existing tail of onCreate
	return-void
.end method
"""

NEW_APP_ONCREATE_TAIL = """
	# PATCH: Restore autoshutdown timer from SharedPreferences
	const-string v0, "settings"

	const/4 v1, 0x0

	invoke-virtual {p0, v0, v1}, Landroid/content/Context;->getSharedPreferences(Ljava/lang/String;I)Landroid/content/SharedPreferences;

	move-result-object v0

	const-string v1, "autoShutdownTimer"

	const/4 v2, 0x0

	invoke-interface {v0, v1, v2}, Landroid/content/SharedPreferences;->getInt(Ljava/lang/String;I)I

	move-result v0

	# Call the setter to restore the value (triggers any necessary side effects)
	invoke-virtual {p0, v0}, Lcom/innioasis/SettingsClass;->setAutoShutdownTimer(I)V

	return-void
.end method
"""
```

### Option B: Preference Screen Patch

If it's a PreferenceScreen that's not persisting properly, we need to ensure the Preference has:
- `android:persistent="true"` in the XML
- A proper `android:key` attribute

We'd patch `res/xml/preferences.xml` using the `_axml.py` helper.

### Option C: Binary Config File Patch

If the setting is supposed to save to a config file (like the repeat/shuffle pattern in `PappSetFileObserver`), we'd create a similar file-based persistence mechanism.

## Testing the Fix

After creating the patch:

1. **Build the patched APK:**
   ```bash
   cd src/patches
   python3 patch_y1_apk.py ../../staging/com.innioasis.y1_*.apk
   ```

2. **Apply all patches and flash:**
   ```bash
   # In WSL/Linux
   ./apply.bash --all
   ```

3. **Test on device:**
   - Set autoshutdown timer to a non-default value
   - Restart device
   - Verify the setting persists

## Next Steps

1. **Run the extraction script** to get system.img
2. **Mount in WSL** and explore the APK
3. **Report back** the findings:
   - What smali file contains the autoshutdown setter?
   - Is it currently using SharedPreferences or just a field?
   - What's the method signature?
   - Where is it called from (menu, preference screen, etc.)?

Once we have these details, we can create the precise patch following the project's established patterns.

## Example Search Commands

```bash
# In the decompiled APK directory:

# Find string references to "shutdown" or timer-related text
grep -r "shutdown" res/values/strings.xml
grep -r "timer" res/values/strings.xml

# Find the corresponding smali code
grep -r "auto.*shutdown\|shutdown.*timer" smali*/ --include="*.smali"

# Find SharedPreferences usage (to understand the existing pattern)
grep -r "getSharedPreferences" smali*/ --include="*.smali" | head -20

# Find Application or Service classes (likely where settings are initialized)
find smali* -name "*Application.smali" -o -name "*Service.smali"

# Look for preferences XML
find res/xml -name "*.xml" 2>/dev/null
```

## Reference Files in This Project

Study these for the persistence pattern:
- `src/patches/patch_y1_apk.py` - Lines 1439-1530 (PappStateBroadcaster SharedPreferences pattern)
- `src/patches/inject/com/koensayr/y1/papp/PappSetFileObserver.smali` - File-based settings pattern
- `docs/PATCHES.md` - Documentation of all existing patches

