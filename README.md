# Aviatrix
Application de prédication des Jeux comme Aviator, JetX, SoccerX...
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.aviatorapkdownloader">

    <uses-permission android:name="android.permission.INTERNET"/>
    <!-- DownloadManager uses external storage app-specific location; no WRITE_EXTERNAL_STORAGE required for modern APIs,
         but if you target very old SDKs you may need it. -->

    <application
        android:label="Aviator APK Downloader"
        android:icon="@mipmap/ic_launcher"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:allowBackup="true"
        android:supportsRtl="true">
        <activity android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>

        <!-- FileProvider to share APK file URI with installer -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>
    </application>
</manifest><?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <!-- allow sharing files in app-specific external downloads dir -->
    <external-path name="external_files" path="Android/data/com.example.aviatorapkdownloader/files/Download/"/>
    <!-- alternate: use <external-files-path> for app-specific external files -->
    <external-files-path name="app_files" path="Download/"/>
</paths><?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="vertical"
    android:padding="20dp"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <TextView
        android:text="Aviator APK Downloader"
        android:textSize="20sp"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>

    <EditText
        android:id="@+id/urlEdit"
        android:hint="Enter APK URL (https://...)"
        android:inputType="textUri"
        android:layout_marginTop="16dp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

    <Button
        android:id="@+id/downloadBtn"
        android:text="Download APK"
        android:layout_marginTop="16dp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

    <TextView
        android:id="@+id/statusText"
        android:text=""
        android:layout_marginTop="12dp"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>
</LinearLayout>package com.example.aviatorapkdownloader

import android.app.DownloadManager
import android.content.BroadcastReceiver
import android.content.Context
import android.content.Intent
import android.content.IntentFilter
import android.database.Cursor
import android.net.Uri
import android.os.Bundle
import android.os.Environment
import android.widget.Button
import android.widget.EditText
import android.widget.TextView
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.FileProvider
import java.io.File

class MainActivity : AppCompatActivity() {

    private var downloadId: Long = -1L
    private lateinit var dm: DownloadManager
    private lateinit var receiver: BroadcastReceiver

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val urlEdit: EditText = findViewById(R.id.urlEdit)
        val downloadBtn: Button = findViewById(R.id.downloadBtn)
        val statusText: TextView = findViewById(R.id.statusText)

        dm = getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager

        downloadBtn.setOnClickListener {
            val url = urlEdit.text.toString().trim()
            if (url.isEmpty()) {
                Toast.makeText(this, "Enter APK URL", Toast.LENGTH_SHORT).show()
                return@setOnClickListener
            }
            try {
                statusText.text = "Starting download..."
                startDownload(url)
            } catch (e: Exception) {
                e.printStackTrace()
                Toast.makeText(this, "Failed to start download: ${e.message}", Toast.LENGTH_LONG).show()
                statusText.text = "Error starting download"
            }
        }

        // receiver to catch completed downloads
        receiver = object : BroadcastReceiver() {
            override fun onReceive(context: Context?, intent: Intent?) {
                val id = intent?.getLongExtra(DownloadManager.EXTRA_DOWNLOAD_ID, -1L) ?: return
                if (id == downloadId) {
                    // query status
                    val q = DownloadManager.Query().setFilterById(id)
                    val c: Cursor? = dm.query(q)
                    if (c != null && c.moveToFirst()) {
                        val status = c.getInt(c.getColumnIndexOrThrow(DownloadManager.COLUMN_STATUS))
                        val uriString = c.getString(c.getColumnIndexOrThrow(DownloadManager.COLUMN_LOCAL_URI))
                        if (status == DownloadManager.STATUS_SUCCESSFUL) {
                            statusText.text = "Download complete"
                            Toast.makeText(this@MainActivity, "Download complete", Toast.LENGTH_SHORT).show()
                            // uriString is like "file://..." or "content://..."
                            // We'll create a File object from the app's download dir used below
                            installApkFromFileUri(uriString)
                        } else {
                            statusText.text = "Download failed (status $status)"
                            Toast.makeText(this@MainActivity, "Download failed", Toast.LENGTH_LONG).show()
                        }
                    }
                    c?.close()
                }
            }
        }
        registerReceiver(receiver, IntentFilter(DownloadManager.ACTION_DOWNLOAD_COMPLETE))
    }

    override fun onDestroy() {
        super.onDestroy()
        unregisterReceiver(receiver)
    }

    private fun startDownload(apkUrl: String) {
        // Name file with timestamp to avoid collisions
        val fileName = "aviator_${System.currentTimeMillis()}.apk"

        // Place into app-specific external Download directory
        // getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS) -> /storage/.../Android/data/your.package/files/Download
        val destDir = getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
        if (destDir == null) {
            Toast.makeText(this, "External storage not available", Toast.LENGTH_LONG).show()
            return
        }

        // Remove any pre-existing same-name file
        val outFile = File(destDir, fileName)
        if (outFile.exists()) outFile.delete()

        val request = DownloadManager.Request(Uri.parse(apkUrl)).apply {
            setTitle("Downloading APK")
            setDescription(fileName)
            setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE)
            // Save to app-specific external files dir (no special permission required on modern Android)
            setDestinationUri(Uri.fromFile(outFile))
            allowScanningByMediaScanner()
        }

        downloadId = dm.enqueue(request)
    }

    private fun installApkFromFileUri(localUriString: String?) {
        if (localUriString == null) {
            Toast.makeText(this, "No local URI for downloaded file", Toast.LENGTH_SHORT).show()
            return
        }

        // localUriString might start with "file://" - convert to File then to FileProvider URI
        val fileUri = Uri.parse(localUriString)
        val apkFile = if (fileUri.scheme == "file") {
            File(fileUri.path!!)
        } else {
            // try to locate last file in Download dir (fallback)
            val dir = getExternalFilesDir(Environment.DIRECTORY_DOWNLOADS)
            dir?.listFiles()?.maxByOrNull { it.lastModified() }
        }

        if (apkFile == null || !apkFile.exists()) {
            Toast.makeText(this, "Downloaded APK file not found", Toast.LENGTH_LONG).show()
            return
        }

        // Build content Uri using FileProvider
        val contentUri: Uri = FileProvider.getUriForFile(
            this,
            "${applicationContext.packageName}.fileprovider",
            apkFile
        )

        val installIntent = Intent(Intent.ACTION_VIEW).apply {
            setDataAndType(contentUri, "application/vnd.android.package-archive")
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
            addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        }

        // Important: user must enable "Install unknown apps" for this app in Settings
        try {
            startActivity(installIntent)
        } catch (e: Exception) {
            e.printStackTrace()
            Toast.makeText(this,
                "Cannot start installer. Make sure 'Install unknown apps' is enabled for this app.",
                Toast.LENGTH_LONG).show()
        }
    }
}plugins {
    id 'com.android.application'
    id 'org.jetbrains.kotlin.android'
}

android {
    namespace 'com.example.aviatorapkdownloader'
    compileSdk 34

    defaultConfig {
        applicationId "com.example.aviatorapkdownloader"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.9.0"
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.core:core-ktx:1.12.0'
}
