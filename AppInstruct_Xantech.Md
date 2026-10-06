1. Create the project
File > New > New Project
Choose Empty Views Activity
Set:
Name: Task Tube
Package name: com.xantech.tasktube
Language: Kotlin
Minimum SDK: API 24
Click Finish and wait for Gradle sync.



--------------------------------------------------------------------------------



2. Set up Firebase and OneSignal for this app
In the Firebase console, you can reuse your existing project or create a new one. Either way, add a new Android app with the package name com.xantech.tasktube.
Generate a service account key (Project settings > Service accounts > Generate new private key).
In OneSignal, click New App/Website, name it Task Tube, choose Google Android (FCM), and upload the JSON key.
Pick Native Android > Kotlin/Java and copy the new App ID.



--------------------------------------------------------------------------------



3. Add the OneSignal dependency

In app/build.gradle.kts, add inside dependencies { }:

implementation("com.onesignal:OneSignal:[5.6.1, 5.9.99]")

Click Sync Now.



--------------------------------------------------------------------------------



4.1. Update the manifest

Replace app/src/main/AndroidManifest.xml as follows. If your generated file has extra attributes like android:icon and android:theme, keep those and only add the two highlighted pieces: the permission line and android:name=".App".


<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />
    <uses-permission
        android:name="android.permission.WRITE_EXTERNAL_STORAGE"
        android:maxSdkVersion="28" />

    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <queries>
        <intent>
            <action android:name="android.media.action.IMAGE_CAPTURE" />
        </intent>
    </queries>

    <application
        android:name=".App"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.TaskTube"
        tools:targetApi="31">
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
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

</manifest>

-----------------------------------
4.2. Create the file provider paths

Right-click res/xml > New > File, name it file_paths.xml, and paste:

<?xml version="1.0" encoding="utf-8"?>
<paths>
    <cache-path name="camera" path="camera/" />
</paths>
-----------------------------------

--------------------------------------------------------------------------------



5. Create the layout

Open res/layout/activity_main.xml, switch to Code view, and replace everything with:

<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <WebView
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>



--------------------------------------------------------------------------------



6. Create the Application class

Right-click the com.xantech.tasktube package > New > Kotlin Class/File, name it App, and paste:

package com.xantech.tasktube

import android.app.Application
import com.onesignal.OneSignal
import com.onesignal.debug.LogLevel
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

class App : Application() {

    private val oneSignalAppId = "PASTE-YOUR-ONESIGNAL-APP-ID"

    override fun onCreate() {
        super.onCreate()

        OneSignal.Debug.logLevel = LogLevel.VERBOSE   // remove for release builds
        OneSignal.initWithContext(this, oneSignalAppId)

        CoroutineScope(Dispatchers.IO).launch {
            OneSignal.Notifications.requestPermission(false)
        }
    }
}



--------------------------------------------------------------------------------



7. Replace MainActivity.kt

package com.xantech.tasktube

import android.Manifest
import android.app.DownloadManager
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Bundle
import android.os.Environment
import android.provider.MediaStore
import android.webkit.CookieManager
import android.webkit.PermissionRequest
import android.webkit.URLUtil
import android.webkit.ValueCallback
import android.webkit.WebChromeClient
import android.webkit.WebResourceRequest
import android.webkit.WebView
import android.webkit.WebViewClient
import android.widget.Toast
import androidx.activity.OnBackPressedCallback
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import androidx.core.content.FileProvider
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import java.io.File

class MainActivity : AppCompatActivity() {

    private lateinit var webView: WebView
    private val siteUrl = "https://yourwebsite.com"   // <- change this
    private val baseHost = Uri.parse(siteUrl).host.orEmpty().removePrefix("www.")

    // File chooser state
    private var filePathCallback: ValueCallback<Array<Uri>>? = null
    private var cameraImageUri: Uri? = null

    // Live camera/mic request state
    private var pendingWebRequest: PermissionRequest? = null

    // Camera permission for the "take photo" option in file uploads
    private var cameraPermissionCallback: (() -> Unit)? = null

    private val fileChooserLauncher =
        registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
            val callback = filePathCallback ?: return@registerForActivityResult
            var uris: Array<Uri>? = null
            if (result.resultCode == RESULT_OK) {
                val data = result.data
                val clip = data?.clipData
                uris = when {
                    clip != null -> Array(clip.itemCount) { clip.getItemAt(it).uri }
                    data?.data != null -> arrayOf(data.data!!)
                    cameraImageUri != null -> arrayOf(cameraImageUri!!)
                    else -> null
                }
            }
            callback.onReceiveValue(uris)
            filePathCallback = null
            cameraImageUri = null
        }

    private val mediaPermissionLauncher =
        registerForActivityResult(ActivityResultContracts.RequestMultiplePermissions()) { _ ->
            val request = pendingWebRequest ?: return@registerForActivityResult
            pendingWebRequest = null
            val granted = request.resources.filter {
                (it == PermissionRequest.RESOURCE_VIDEO_CAPTURE && hasPermission(Manifest.permission.CAMERA)) ||
                (it == PermissionRequest.RESOURCE_AUDIO_CAPTURE && hasPermission(Manifest.permission.RECORD_AUDIO))
            }
            if (granted.isEmpty()) request.deny() else request.grant(granted.toTypedArray())
        }

    private val cameraPermissionLauncher =
        registerForActivityResult(ActivityResultContracts.RequestPermission()) { _ ->
            cameraPermissionCallback?.invoke()
            cameraPermissionCallback = null
        }

    private fun hasPermission(permission: String) =
        ContextCompat.checkSelfPermission(this, permission) == PackageManager.PERMISSION_GRANTED

    private fun isOwnHost(host: String?) =
        host != null && (host == baseHost || host.endsWith(".$baseHost"))

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(bars.left, bars.top, bars.right, bars.bottom)
            insets
        }

        webView = findViewById(R.id.webview)
        webView.settings.javaScriptEnabled = true
        webView.settings.domStorageEnabled = true
        webView.settings.mediaPlaybackRequiresUserGesture = false

        webView.webViewClient = object : WebViewClient() {
            override fun shouldOverrideUrlLoading(
                view: WebView,
                request: WebResourceRequest
            ): Boolean {
                val uri = request.url
                val isInternal = uri.scheme in listOf("http", "https") && isOwnHost(uri.host)
                if (isInternal) return false

                try {
                    startActivity(Intent(Intent.ACTION_VIEW, uri))
                } catch (e: Exception) {
                    // No app can handle this link, ignore it
                }
                return true
            }
        }

        webView.webChromeClient = object : WebChromeClient() {

            // Live camera / microphone (getUserMedia)
            override fun onPermissionRequest(request: PermissionRequest) {
                runOnUiThread {
                    if (!isOwnHost(request.origin.host)) {
                        request.deny()
                        return@runOnUiThread
                    }
                    val supported = request.resources.filter {
                        it == PermissionRequest.RESOURCE_VIDEO_CAPTURE ||
                        it == PermissionRequest.RESOURCE_AUDIO_CAPTURE
                    }
                    if (supported.isEmpty()) {
                        request.deny()
                        return@runOnUiThread
                    }
                    val missing = supported.map {
                        if (it == PermissionRequest.RESOURCE_VIDEO_CAPTURE)
                            Manifest.permission.CAMERA else Manifest.permission.RECORD_AUDIO
                    }.filter { !hasPermission(it) }

                    if (missing.isEmpty()) {
                        request.grant(supported.toTypedArray())
                    } else {
                        pendingWebRequest?.deny()
                        pendingWebRequest = request
                        mediaPermissionLauncher.launch(missing.toTypedArray())
                    }
                }
            }

            // <input type="file"> uploads
            override fun onShowFileChooser(
                webView: WebView,
                callback: ValueCallback<Array<Uri>>,
                params: FileChooserParams
            ): Boolean {
                filePathCallback?.onReceiveValue(null)
                filePathCallback = callback

                val offersImages = params.acceptTypes.isEmpty() ||
                        params.acceptTypes.any { it.isBlank() || it.startsWith("image") || it == "*/*" }

                if (offersImages && !hasPermission(Manifest.permission.CAMERA)) {
                    // Ask once, then open the chooser whether or not it was granted
                    cameraPermissionCallback = { launchChooser(params) }
                    cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                } else {
                    launchChooser(params)
                }
                return true
            }
        }

        // File downloads
        webView.setDownloadListener { url, userAgent, contentDisposition, mimeType, _ ->
            try {
                val fileName = URLUtil.guessFileName(url, contentDisposition, mimeType)
                val request = DownloadManager.Request(Uri.parse(url)).apply {
                    setMimeType(mimeType)
                    addRequestHeader("User-Agent", userAgent)
                    CookieManager.getInstance().getCookie(url)?.let { addRequestHeader("Cookie", it) }
                    setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED)
                    setDestinationInExternalPublicDir(Environment.DIRECTORY_DOWNLOADS, fileName)
                }
                (getSystemService(DOWNLOAD_SERVICE) as DownloadManager).enqueue(request)
                Toast.makeText(this, "Downloading $fileName", Toast.LENGTH_SHORT).show()
            } catch (e: Exception) {
                Toast.makeText(this, "Download failed", Toast.LENGTH_SHORT).show()
            }
        }

        if (savedInstanceState == null) {
            webView.loadUrl(siteUrl)
        } else {
            webView.restoreState(savedInstanceState)
        }

        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                if (webView.canGoBack()) webView.goBack() else finish()
            }
        })
    }

    private fun launchChooser(params: WebChromeClient.FileChooserParams) {
        val callback = filePathCallback ?: return

        val chooser = Intent(Intent.ACTION_CHOOSER).apply {
            putExtra(Intent.EXTRA_INTENT, params.createIntent())
            putExtra(Intent.EXTRA_TITLE, "Choose a file")
        }

        val offersImages = params.acceptTypes.isEmpty() ||
                params.acceptTypes.any { it.isBlank() || it.startsWith("image") || it == "*/*" }
        if (offersImages && hasPermission(Manifest.permission.CAMERA)) {
            createCameraIntent()?.let {
                chooser.putExtra(Intent.EXTRA_INITIAL_INTENTS, arrayOf(it))
            }
        }

        try {
            fileChooserLauncher.launch(chooser)
        } catch (e: Exception) {
            callback.onReceiveValue(null)
            filePathCallback = null
        }
    }

    private fun createCameraIntent(): Intent? {
        return try {
            val dir = File(cacheDir, "camera").apply { mkdirs() }
            val file = File.createTempFile("IMG_", ".jpg", dir)
            val uri = FileProvider.getUriForFile(this, "$packageName.fileprovider", file)
            cameraImageUri = uri
            Intent(MediaStore.ACTION_IMAGE_CAPTURE).apply {
                putExtra(MediaStore.EXTRA_OUTPUT, uri)
                addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION)
            }
        } catch (e: Exception) {
            null
        }
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        webView.saveState(outState)
    }
}


--------------------------------------------------------------------------------




8. Set the name and icon
Name: in res/values/strings.xml, confirm <string name="app_name">Task Tube</string>
Icon: right-click res > New > Image Asset, choose your logo under Foreground Layer > Path, then Next > Finish



--------------------------------------------------------------------------------



9. Run and test
Select an emulator with a Google Play system image, or a real phone, and click Run.
Allow the notification permission when prompted.
In OneSignal, check Audience > Subscriptions for your device, then send a test from Messages > New Push.



--------------------------------------------------------------------------------



10. Build the installable file
Test APK: Build > Build Bundle(s) / APK(s) > Build APK(s)
Play Store: Build > Generate Signed App Bundle / APK > Android App Bundle, and create a new keystore for Task Tube. Back it up, because you can't publish updates without it.



