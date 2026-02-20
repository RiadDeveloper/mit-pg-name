# Documentation: Package Name Feature Implementation

This document details the end-to-end implementation of the **Package Name** feature in MIT App Inventor. This feature allows developers to specify a custom Android package name for their applications, providing greater control over application identifiers and distribution.

---

## 1. Overview
The implementation involves changes across several layers of the App Inventor architecture:
- **Shared Constants**: For system-wide property recognition.
- **Client (App Engine)**: To manage the property in the project settings and designer.
- **Server (App Engine)**: To handle serialization and persistence of project settings.
- **Components (Runtime)**: To expose the package name via blocks and fix screen switching logic.
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

### C. Server-Side Persistence & Lifecycle
#### `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidSettingsBuilder.java`
- **Change**: 
  - Added `packageName` field and `getPackageName()` getter.
  - Updated `toProperties()` with fallback logic for export.
- **Code**:
  ```java
  public String toProperties() {
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

#### `appinventor/appengine/src/com/google/appinventor/server/project/youngandroid/YoungAndroidProjectService.java`
- **Change**: 
  - Updated `storeProjectSettings` to prevent wiping out existing package names.
  - Updated `newProject` to set the initial package name.
  - Updated `getInitialFormPropertiesFileContents` to include `PackageName` in the initial `.scm`.
- **Code (Gating)**:
  ```java
  if (newProperties.getPackageName().isEmpty()) {
    newProperties.setPackageName(properties.getProperty("packagename", ""));
  }
  ```

---

### D. Build Server Logic
#### `appinventor/buildserver/src/com/google/appinventor/buildserver/Project.java`
- **Change**: Added `packageName` field and getters/setters.

#### `appinventor/buildserver/src/com/google/appinventor/buildserver/tasks/android/CreateManifest.java`
- **Change**: Updated `<manifest>` tag to use the custom package name while qualifying activity names.

---

### E. Component Runtime & Screen Switching
#### `appinventor/components/src/com/google/appinventor/components/runtime/Form.java`
- **Change**:
  - Added `PackageName` designer property.
  - **CRITICAL FIX**: Updated `switchForm`, `switchFormWithStartValue`, and `startNewForm` to support screen switching when a custom package name is used.
- **Code (Screen Switching Fix)**:
  ```java
  public static void switchForm(String nextFormName) {
    if (activeForm != null) {
      String currentClassName = activeForm.getClass().getName();
      String nextClassName = nextFormName;
      if (!currentClassName.startsWith(YaVersion.ACCEPTABLE_COMPANION_PACKAGE)) {
        int lastDot = currentClassName.lastIndexOf('.');
        if (lastDot != -1) {
          nextClassName = currentClassName.substring(0, lastDot + 1) + nextFormName;
        }
      }
      activeForm.startNewForm(nextClassName, null);
    }
  }

  protected void startNewForm(String nextFormName, Object startupValue) {
    Intent activityIntent = new Intent();
    if (getPackageName().startsWith(YaVersion.ACCEPTABLE_COMPANION_PACKAGE)) {
      activityIntent.setClassName(this, getPackageName() + "." + nextFormName);
    } else {
      activityIntent.setClassName(this, nextFormName);
    }
    // ...
    if (nextFormName.contains(".")) {
      this.nextFormName = nextFormName.substring(nextFormName.lastIndexOf('.') + 1);
    } else {
      this.nextFormName = nextFormName;
    }
  }
  ```

#### `appinventor/components/src/com/google/appinventor/common/YaVersion.java`
- **Change**: Incremented `FORM_COMPONENT_VERSION` to **32**.

---

## 3. Build & Verification
- **Build Status**: `BUILD SUCCESSFUL` via Ant.
- **Stability**: Verified that screen switching works correctly in both compiled apps (with custom packages) and the companion.
