# Sonnet Messages: ONE-FILE BUILD.
# Put this file in your GitHub repository at   .github/workflows/build.yml   and commit.
# GitHub then writes the whole app from the text below, compiles it, and gives you the APK + Play Store bundle.
name: Build Sonnet Messages
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      KEYSTORE_B64: ${{ secrets.KEYSTORE_B64 }}
      KEYSTORE_PASSWORD: ${{ secrets.KEYSTORE_PASSWORD }}
      KEY_ALIAS: ${{ secrets.KEY_ALIAS }}
      KEY_PASSWORD: ${{ secrets.KEY_PASSWORD }}
    steps:
      - uses: actions/setup-java@v4
        with: { distribution: temurin, java-version: 17 }
      - uses: gradle/actions/setup-gradle@v4
        with: { gradle-version: "8.13" }
      - name: Make sure the Android 16 SDK is installed
        run: |
          yes | "$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager" --licenses > /dev/null || true
          "$ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager" "platforms;android-36" "build-tools;36.0.0" > /dev/null || true
      - name: Write the app source files
        run: |
          set -e
          mkdir -p proj && cd proj
          true
          cat > 'settings.gradle.kts' <<'__EOF__'
          pluginManagement { repositories { google(); mavenCentral(); gradlePluginPortal() } }
          dependencyResolutionManagement { repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS); repositories { google(); mavenCentral() } }
          rootProject.name = "Messages"
          include(":app")
          __EOF__
          true
          cat > 'build.gradle.kts' <<'__EOF__'
          plugins {
              id("com.android.application") version "8.11.1" apply false
              id("org.jetbrains.kotlin.android") version "2.1.21" apply false
              id("org.jetbrains.kotlin.plugin.compose") version "2.1.21" apply false
          }
          __EOF__
          true
          cat > 'gradle.properties' <<'__EOF__'
          org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
          android.useAndroidX=true
          kotlin.code.style=official
          __EOF__
          mkdir -p "app"
          cat > 'app/build.gradle.kts' <<'__EOF__'
          import java.util.Properties

          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
              id("org.jetbrains.kotlin.plugin.compose")
          }

          // Signing secrets come from environment variables (GitHub build) or from a local keystore.properties file (Android Studio).
          val keystoreProps = Properties().apply {
              val f = rootProject.file("keystore.properties")
              if (f.exists()) f.inputStream().use { load(it) }
          }
          fun secret(env: String, key: String): String? = System.getenv(env) ?: keystoreProps.getProperty(key)

          android {
              namespace = "com.sonnetmessages.app"
              compileSdk = 36

              defaultConfig {
                  applicationId = "com.sonnetmessages.app"   // your unique app id; it can never change after the first upload
                  minSdk = 26
                  targetSdk = 36                            // Google Play requires 36 for new apps and updates (since 31 Aug 2026)
                  versionCode = 1                           // raise by 1 for every upload
                  versionName = "1.0"
              }

              // Release signing: filled from environment variables (used by the GitHub build) so no secrets live in the code.
              signingConfigs {
                  create("release") {
                      val path = secret("KEYSTORE_PATH", "storeFile")
                      if (path != null) {
                          storeFile = file(path)
                          storePassword = secret("KEYSTORE_PASSWORD", "storePassword")
                          keyAlias = secret("KEY_ALIAS", "keyAlias")
                          keyPassword = secret("KEY_PASSWORD", "keyPassword")
                      }
                  }
              }
              buildTypes {
                  release {
                      isMinifyEnabled = false
                      if (secret("KEYSTORE_PATH", "storeFile") != null) signingConfig = signingConfigs.getByName("release")
                  }
              }
              compileOptions { sourceCompatibility = JavaVersion.VERSION_17; targetCompatibility = JavaVersion.VERSION_17 }
              kotlinOptions { jvmTarget = "17" }
              buildFeatures { compose = true }
              lint {
                  checkReleaseBuilds = false   // do not let style warnings block the release bundle
                  abortOnError = false
              }
          }

          dependencies {
              implementation(platform("androidx.compose:compose-bom:2025.06.01"))
              implementation("androidx.compose.ui:ui")
              implementation("androidx.compose.material3:material3")
              implementation("androidx.compose.material:material-icons-core")
              implementation("androidx.activity:activity-compose:1.10.1")
              implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.9.1")
              implementation("androidx.core:core-ktx:1.16.0")
              implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.10.2")
          }
          __EOF__
          mkdir -p "app/src/main"
          cat > 'app/src/main/AndroidManifest.xml' <<'__EOF__'
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">

              <uses-feature android:name="android.hardware.telephony" android:required="false" />

              <uses-permission android:name="android.permission.READ_SMS" />
              <uses-permission android:name="android.permission.SEND_SMS" />
              <uses-permission android:name="android.permission.RECEIVE_SMS" />
              <uses-permission android:name="android.permission.RECEIVE_MMS" />
              <uses-permission android:name="android.permission.RECEIVE_WAP_PUSH" />
              <uses-permission android:name="android.permission.READ_CONTACTS" />
              <uses-permission android:name="android.permission.CALL_PHONE" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.READ_PHONE_STATE" />
              <uses-permission android:name="android.permission.READ_PHONE_NUMBERS" />

              <application
                  android:allowBackup="false"
                  android:enableOnBackInvokedCallback="true"
                  android:icon="@mipmap/ic_launcher"
                  android:label="@string/app_name"
                  android:theme="@style/Theme.Messages">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTop"
                      android:windowSoftInputMode="adjustResize">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                      <!-- Required to be eligible as the default SMS app: handle "send a message to this number" -->
                      <intent-filter>
                          <action android:name="android.intent.action.SEND" />
                          <action android:name="android.intent.action.SENDTO" />
                          <category android:name="android.intent.category.DEFAULT" />
                          <category android:name="android.intent.category.BROWSABLE" />
                          <data android:scheme="sms" />
                          <data android:scheme="smsto" />
                          <data android:scheme="mms" />
                          <data android:scheme="mmsto" />
                      </intent-filter>
                  </activity>

                  <!-- Delivered to the default SMS app only; works with the app closed -->
                  <receiver
                      android:name=".SmsReceiver"
                      android:exported="true"
                      android:permission="android.permission.BROADCAST_SMS">
                      <intent-filter>
                          <action android:name="android.provider.Telephony.SMS_DELIVER" />
                      </intent-filter>
                  </receiver>

                  <receiver
                      android:name=".MmsReceiver"
                      android:exported="true"
                      android:permission="android.permission.BROADCAST_WAP_PUSH">
                      <intent-filter>
                          <action android:name="android.provider.Telephony.WAP_PUSH_DELIVER" />
                          <data android:mimeType="application/vnd.wap.mms-message" />
                      </intent-filter>
                  </receiver>

                  <service
                      android:name=".RespondViaMessageService"
                      android:exported="true"
                      android:permission="android.permission.SEND_RESPOND_VIA_MESSAGE">
                      <intent-filter>
                          <action android:name="android.intent.action.RESPOND_VIA_MESSAGE" />
                          <category android:name="android.intent.category.DEFAULT" />
                          <data android:scheme="sms" />
                          <data android:scheme="smsto" />
                          <data android:scheme="mms" />
                          <data android:scheme="mmsto" />
                      </intent-filter>
                  </service>

                  <receiver android:name=".ReplyReceiver" android:exported="false" />
                  <receiver android:name=".MarkReadReceiver" android:exported="false" />
                  <receiver android:name=".MmsDownloadReceiver" android:exported="false" />
                  <receiver android:name=".MmsStatusReceiver" android:exported="false" />

                  <!-- Lets the phone's MMS service read/write the picture-message files we hand it -->
                  <provider
                      android:name="androidx.core.content.FileProvider"
                      android:authorities="${applicationId}.mmsfiles"
                      android:exported="false"
                      android:grantUriPermissions="true">
                      <meta-data android:name="android.support.FILE_PROVIDER_PATHS" android:resource="@xml/mms_paths" />
                  </provider>
              </application>
          </manifest>
          __EOF__
          mkdir -p "app/src/main/res/values"
          cat > 'app/src/main/res/values/strings.xml' <<'__EOF__'
          <resources><string name="app_name">Sonnet Messages</string></resources>
          __EOF__
          mkdir -p "app/src/main/res/values"
          cat > 'app/src/main/res/values/themes.xml' <<'__EOF__'
          <resources>
              <style name="Theme.Messages" parent="android:Theme.Material.Light.NoActionBar">
                  <item name="android:windowBackground">@android:color/white</item>
              </style>
          </resources>
          __EOF__
          mkdir -p "app/src/main/res/values"
          cat > 'app/src/main/res/values/colors.xml' <<'__EOF__'
          <resources><color name="ic_launcher_background">#4285F4</color></resources>
          __EOF__
          mkdir -p "app/src/main/res/xml"
          cat > 'app/src/main/res/xml/mms_paths.xml' <<'__EOF__'
          <?xml version="1.0" encoding="utf-8"?>
          <paths><cache-path name="mms" path="mms/" /></paths>
          __EOF__
          mkdir -p "app/src/main/res/mipmap-anydpi-v26"
          cat > 'app/src/main/res/mipmap-anydpi-v26/ic_launcher.xml' <<'__EOF__'
          <?xml version="1.0" encoding="utf-8"?>
          <adaptive-icon xmlns:android="http://schemas.android.com/apk/res/android">
              <background android:drawable="@color/ic_launcher_background" />
              <foreground android:drawable="@drawable/ic_launcher_foreground" />
          </adaptive-icon>
          __EOF__
          mkdir -p "app/src/main/res/drawable"
          cat > 'app/src/main/res/drawable/ic_launcher_foreground.xml' <<'__EOF__'
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="108dp" android:height="108dp" android:viewportWidth="108" android:viewportHeight="108">
              <group android:translateX="30" android:translateY="30" android:scaleX="2" android:scaleY="2">
                  <path android:fillColor="#FFFFFF"
                      android:pathData="M4,2h16a2,2 0,0 1,2 2v12a2,2 0,0 1,-2 2h-8l-8,4 3,-4H4a2,2 0,0 1,-2 -2V4a2,2 0,0 1,2 -2z" />
              </group>
          </vector>
          __EOF__
          mkdir -p "app/src/main/res/drawable"
          cat > 'app/src/main/res/drawable/ic_notify.xml' <<'__EOF__'
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="24dp" android:height="24dp" android:viewportWidth="24" android:viewportHeight="24">
              <path android:fillColor="#FFFFFF"
                  android:pathData="M4,2h16a2,2 0,0 1,2 2v12a2,2 0,0 1,-2 2h-8l-8,4 3,-4H4a2,2 0,0 1,-2 -2V4a2,2 0,0 1,2 -2z" />
          </vector>
          __EOF__
          mkdir -p "app/src/main/java/com/sonnetmessages/app"
          cat > 'app/src/main/java/com/sonnetmessages/app/MainActivity.kt' <<'__EOF__'
          package com.sonnetmessages.app

          import android.Manifest
          import android.app.role.RoleManager
          import android.content.Context
          import android.content.Intent
          import android.content.pm.PackageManager
          import android.net.Uri
          import android.os.Build
          import android.provider.Telephony
          import android.widget.Toast
          import androidx.activity.ComponentActivity
          import androidx.activity.compose.BackHandler
          import androidx.activity.compose.rememberLauncherForActivityResult
          import androidx.activity.compose.setContent
          import androidx.activity.result.contract.ActivityResultContracts
          import androidx.activity.enableEdgeToEdge
          import androidx.activity.viewModels
          import android.graphics.BitmapFactory
          import android.telecom.PhoneAccountHandle
          import android.telecom.TelecomManager
          import androidx.activity.result.PickVisualMediaRequest
          import androidx.compose.foundation.Image
          import androidx.compose.ui.graphics.asImageBitmap
          import androidx.compose.ui.graphics.ImageBitmap
          import androidx.compose.ui.unit.Dp
          import kotlinx.coroutines.Dispatchers
          import kotlinx.coroutines.withContext
          import androidx.compose.foundation.background
          import androidx.compose.foundation.clickable
          import androidx.compose.foundation.layout.*
          import androidx.compose.foundation.lazy.LazyColumn
          import androidx.compose.foundation.lazy.items
          import androidx.compose.foundation.lazy.rememberLazyListState
          import androidx.compose.foundation.shape.CircleShape
          import androidx.compose.foundation.shape.RoundedCornerShape
          import androidx.compose.material.icons.Icons
          import androidx.compose.material.icons.automirrored.filled.ArrowBack
          import androidx.compose.material.icons.automirrored.filled.Send
          import androidx.compose.material.icons.filled.Add
          import androidx.compose.material.icons.filled.Call
          import androidx.compose.material.icons.filled.MoreVert
          import androidx.compose.material3.*
          import androidx.compose.runtime.*
          import androidx.compose.ui.Alignment
          import androidx.compose.ui.Modifier
          import androidx.compose.ui.draw.clip
          import androidx.compose.ui.graphics.Color
          import androidx.compose.ui.platform.LocalConfiguration
          import androidx.compose.ui.platform.LocalContext
          import androidx.compose.ui.text.font.FontWeight
          import androidx.compose.ui.text.style.TextOverflow
          import androidx.compose.ui.unit.dp
          import androidx.compose.ui.unit.sp
          import androidx.core.content.ContextCompat
          import java.text.SimpleDateFormat
          import java.util.Date
          import java.util.Locale

          class MainActivity : ComponentActivity() {
              companion object {
                  const val EXTRA_THREAD = "thread_id"
                  const val EXTRA_ADDRESS = "address"
              }

              private val vm: MessagesViewModel by viewModels()
              private var isDefault by mutableStateOf(false)
              private var callNumber by mutableStateOf<String?>(null)

              override fun onCreate(savedInstanceState: android.os.Bundle?) {
                  super.onCreate(savedInstanceState)
                  enableEdgeToEdge()
                  Notifier.ensureChannel(this)
                  handleIntent(intent)
                  setContent { AppTheme { Root() } }
              }

              override fun onNewIntent(intent: Intent) {
                  super.onNewIntent(intent)
                  setIntent(intent)
                  handleIntent(intent)
              }

              override fun onResume() {
                  super.onResume()
                  isDefault = Telephony.Sms.getDefaultSmsPackage(this) == packageName
                  vm.refresh()
              }

              private fun handleIntent(i: Intent?) {
                  i ?: return
                  val tid = i.getLongExtra(EXTRA_THREAD, -1L)
                  val addr = i.getStringExtra(EXTRA_ADDRESS)
                  if (tid > 0 && addr != null) { vm.open(tid, addr); return }
                  val scheme = i.data?.scheme
                  if (scheme in listOf("sms", "smsto", "mms", "mmsto")) {
                      val number = i.data?.schemeSpecificPart?.substringBefore('?')?.let { Uri.decode(it) }
                      if (!number.isNullOrBlank()) vm.openAddress(number)
                  }
              }

              private fun neededPermissions(): Array<String> {
                  val p = mutableListOf(
                      Manifest.permission.READ_SMS, Manifest.permission.SEND_SMS, Manifest.permission.RECEIVE_SMS,
                      Manifest.permission.READ_CONTACTS, Manifest.permission.CALL_PHONE,
                      Manifest.permission.READ_PHONE_STATE, Manifest.permission.READ_PHONE_NUMBERS
                  )
                  if (Build.VERSION.SDK_INT >= 33) p.add(Manifest.permission.POST_NOTIFICATIONS)
                  return p.toTypedArray()
              }

              private fun hasPerms() = neededPermissions().all {
                  ContextCompat.checkSelfPermission(this, it) == PackageManager.PERMISSION_GRANTED
              }

              // ---------------------------------------------------------------- UI

              @Composable
              private fun Root() {
                  val ctx = LocalContext.current
                  val cur by vm.current.collectAsState()
                  val err by vm.error.collectAsState()
                  var perms by remember { mutableStateOf(hasPerms()) }
                  val permLauncher = rememberLauncherForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) {
                      perms = hasPerms(); vm.refresh()
                  }
                  val roleLauncher = rememberLauncherForActivityResult(ActivityResultContracts.StartActivityForResult()) {
                      isDefault = Telephony.Sms.getDefaultSmsPackage(ctx) == ctx.packageName; vm.refresh()
                  }
                  val pref# Sonnet-messages-
Message app 
