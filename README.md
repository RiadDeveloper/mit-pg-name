# Documentation: Package Name Feature Implementation

This document details the end-to-end implementation of the **Package Name** feature in MIT App Inventor. This feature allows developers to specify a custom Android package name for their applications, providing greater control over application identifiers and distribution.

---

## 1. Overview
The implementation involves changes across several layers of the App Inventor architecture:
- **Shared Constants**: For system-wide property recognition.
- **Client (App Engine)**: To manage the property in the project settings and designer.
- **Server (App Engine)**: To handle serialization and persistence of project settings.
- **Components (Runtime)**: To expose the package name via blocks.
- **Build Server**: To integrate the custom package name into the final `AndroidManifest.xml`.
- **Documentation**: To update the component reference.

---

## 2. File-by-File Breakdown

### A. Shared Configuration
#### `appinventor/appengine/src/com/google/appinventor/shared/settings/SettingsConstants.java`
- **Change**: Added `YOUNG_ANDROID_SETTINGS_PACKAGE_NAME` constant.
- **Code**:
  ```java
  public static final String YOUNG_ANDROID_SETTINGS_PACKAGE_NAME = "PackageName";
  ```

---

### B. Client-Side Designer & Settings
#### `appinventor/appengine/src/com/google/appinventor/client/settings/project/YoungAndroidSettings.java`
- **Change**: Registered the `PackageName` property in the project settings constructor.
- **Code**:
  ```java
  addProperty(new EditableProperty(this,
      SettingsConstants.YOUNG_ANDROID_SETTINGS_PACKAGE_NAME, "",
      EditableProperty.TYPE_INVISIBLE));
  ```

#### `appinventor/appengine/src/com/google/appinventor/client/editor/simple/components/MockForm.java`
- **Change**: 
  - Defined `PROPERTY_NAME_PACKAGE_NAME`.
  - Updated `isPropertyVisible` to handle visibility logic.
  - Implemented `onPropertyChange` to sync with project settings.
  - Updated `getProperties` to load the value on non-Screen1 forms.
  - Added `setPackageNameProperty` helper.
- **Code**:
  ```java
  private void setPackageNameProperty(String packageName) {
    if (editor.isScreen1()) {
      editor.getProjectEditor().changeProjectSettingsProperty(
          SettingsConstants.PROJECT_YOUNG_ANDROID_SETTINGS,
          SettingsConstants.YOUNG_ANDROID_SETTINGS_PACKAGE_NAME, packageName);
    }
  }

  @Override
  public EditableProperties getProperties() {
    if (!editor.isScreen1()) {
      properties.changePropertyValue(SettingsConstants.YOUNG_ANDROID_SETTINGS_PACKAGE_NAME,
          editor.getProjectEditor().getProjectSettingsProperty(
              SettingsConstants.PROJECT_YOUNG_ANDROID_SETTINGS,
              SettingsConstants.YOUNG_ANDROID_SETTINGS_PACKAGE_NAME));
    }
    return properties;
  }
  ```

---

### C. Server-Side Persistence
#### `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidSettingsBuilder.java`
- **Change**: 
  - Added `packageName` field.
  - Updated constructors to read from `Settings` and `Properties`.
  - Updated `build()` to include the package name in JSON output.
  - Updated `toProperties()` to include the package name in `.properties` output.
  - Added missing static import to fix build errors.
- **Code**:
  ```java
  public String getPackageName() {
    return packageName;
  }

  public String toProperties() {
    Properties result = new Properties();
    // ...
    String pkgNameToWrite = packageName;
    if (pkgNameToWrite == null || pkgNameToWrite.isEmpty()) {
      if (qualifiedFormName != null && qualifiedFormName.contains(".")) {
        pkgNameToWrite = qualifiedFormName.substring(0, qualifiedFormName.lastIndexOf('.'));
      }
    }
    if (pkgNameToWrite != null && !pkgNameToWrite.isEmpty()) {
      result.put("packagename", pkgNameToWrite);
    }
    // ...
  }
  ```

---

### D. Build Server Logic
#### `appinventor/buildserver/src/com/google/appinventor/buildserver/Project.java`
- **Change**: 
  - Defined `PACKAGENAME_TAG = "packagename"`.
  - Added `getPackageName()` and `setPackageName(String packageName)` methods.
- **Code**:
  ```java
  public String getPackageName() {
    return packageName;
  }

  public void setPackageName(String packageName) {
    this.packageName = packageName;
  }
  ```

#### `appinventor/buildserver/src/com/google/appinventor/buildserver/tasks/android/CreateManifest.java`
- **Change**: 
  - Extracted `packageName` from the project settings.
  - Defined `mainactivityformClassName` using `Signatures.getPackageName(mainClass)`.
  - Updated the `<manifest>` tag to use the custom `packageName`.
  - Fully qualified the `mainactivity` name in the manifest (e.g., `android:name="com.example.app.Screen1"`) to ensure the app starts correctly regardless of the package name.
- **Code**:
  ```java
  String packageName = project.getPackageName();
  // ...
  out.write("  package=\"" + packageName + "\"\n");
  // ...
  out.write("    <activity android:name=\"" + mainactivityformClassName + "\"\n");
  ```

---

### E. Component Runtime & Versioning
#### `appinventor/components/src/com/google/appinventor/components/runtime/Form.java`
- **Change**: 
  - Added `PackageName(String packageName)` setter with `@DesignerProperty` and `@SimpleProperty`.
  - Added `PackageName()` getter to return the runtime package name.
  - Fixed an initial category error (switched from `APPEARANCE` to `GENERAL` to ensure consistency).
- **Code**:
  ```java
  @DesignerProperty(editorType = PropertyTypeConstants.PROPERTY_TYPE_STRING,
      defaultValue = "")
  @SimpleProperty(userVisible = false,
      description = "This is the package name of the installed application in the phone.",
      category = PropertyCategory.GENERAL)
  public void PackageName(String packageName) {
    // We don't actually need to do anything.
  }
  ```

#### `appinventor/components/src/com/google/appinventor/common/YaVersion.java`
- **Change**: Incremented `FORM_COMPONENT_VERSION` to **32**.
- **Code**:
  ```java
  public static final int FORM_COMPONENT_VERSION = 32;
  ```

#### `appinventor/appengine/src/com/google/appinventor/client/youngandroid/YoungAndroidFormUpgrader.java`
- **Change**: Added upgrade logic for `FORM_COMPONENT_VERSION 32` to handle existing projects.
- **Code**:
  ```java
  if (version < 32) {
    // Add PackageName property if missing
    properties.put(SettingsConstants.YOUNG_ANDROID_SETTINGS_PACKAGE_NAME, "");
    version = 32;
  }
  ```

---

### F. Projects & Documentation
#### `appinventor/aiplayapp/youngandroidproject/project.properties`
- **Change**: Added the default package name for the AI Companion project for testing.
  ```properties
  packagename=com.riad.aicompanion3
  ```

#### `appinventor/docs/markdown/reference/components/userinterface.md`
- **Change**: Added documentation for the `PackageName` property under the `Screen` component.

---


---

## 4. Advanced Persistence & Project Lifecycle
This section details the logic implemented to ensure the package name is correctly handled during project creation, copying, and updates.

### A. Robust Project Creation & Initial State
To ensure every project starts with a valid package name, changes were made to the project creation parameters and initial file generation.

- **File**: `appinventor/appengine/src/com/google/appinventor/shared/rpc/project/youngandroid/NewYoungAndroidProjectParameters.java`
  - **Change**: Updated `getAndroidPackageName()` to return `packageName` if `androidPackageName` is empty.
  - **Code**:
    ```java
    public String getAndroidPackageName() {
      if (androidPackageName == null || androidPackageName.isEmpty()) {
        return packageName;
      }
      return androidPackageName;
    }
    ```
- **File**: `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidProjectService.java`
  - **Method**: `newProject(...)`
    - **Logic**: Explicitly sets the package name in `YoungAndroidSettingsBuilder` using the parameters provided during project creation.
    - **Code**:
      ```java
      YoungAndroidSettingsBuilder builder = new YoungAndroidSettingsBuilder()
          .setProjectName(projectName)
          .setQualifiedFormName(qualifiedFormName)
          .setPackageName(youngAndroidParams.getAndroidPackageName());
      ```
  - **Method**: `getInitialFormPropertiesFileContents(...)`
    - **Logic**: Modified the JSON template for the initial `Screen1.scm` to include the `PackageName` property. This ensures the value is persisted at the component level from the moment the project is born.
    - **Code**:
      ```java
      return "#|\n$JSON\n" +
          "{\"authURL\":[],\"YaVersion\":\"" + YaVersion.YOUNG_ANDROID_VERSION + "\",\"Source\":\"Form\"," +
          "\"Properties\":{\"$Name\":\"" + formName + "\",\"$Type\":\"Form\"," +
          "\"$Version\":\"" + YaVersion.FORM_COMPONENT_VERSION + "\",\"Uuid\":\"" + 0 + "\"," +
          "\"Title\":\"" + formName + "\",\"AppName\":\"" + appName + "\",\"PackageName\":\"" + packageName + "\"}}\n|#";
      ```

### B. Prevention of Data Loss (Server-Side Gating)
A safeguard was added to `YoungAndroidProjectService` to prevent the package name from being wiped out by client-side updates that might not include the property.

- **File**: `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidProjectService.java`
  - **Method**: `storeProjectSettings(...)`
    - **Change**: Checks if the incoming package name is empty and, if so, preserves the existing value from the project properties.
    - **Code**:
      ```java
      if (newProperties.getPackageName().isEmpty()) {
        newProperties.setPackageName(properties.getProperty("packagename", ""));
      }
      ```

### C. Project Copying & Persistence
- **File**: `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidProjectService.java`
  - **Method**: `copyProject(...)`
    - **Logic**: Extracts the `packagename` from the source project's properties and applies it to the clone via `YoungAndroidSettingsBuilder`.

### D. Settings Builder Support
- **File**: `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidSettingsBuilder.java`
  - **Changes**:
    - Added `getPackageName()` getter method.
    - Updated `toProperties()` with fallback logic to derive the package name from `qualifiedFormName` if it is missing during the export process.

---

## 5. Build Summary & Final Verification
The project maintains a stable state with **BUILD SUCCESSFUL** status. Recent fixes include:
1.  **Getter Addition**: Added `getPackageName()` to `YoungAndroidSettingsBuilder` to support persistence logic.
2.  **Syntax Correction**: Fixed a missing semicolon in `YoungAndroidProjectService.storeProjectSettings`.
3.  **Property Gating**: Verified that the package name is correctly gated and preserved during various project lifecycle events.
