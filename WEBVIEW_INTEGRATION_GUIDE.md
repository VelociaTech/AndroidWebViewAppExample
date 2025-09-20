# WebView Integration Guide for Velocia Webapp

This guide provides complete instructions for integrating the Velocia webapp (`webapp.dev.velocia.io`) into your Android native application with full camera and file upload functionality.

## Overview

Your WebView must support:
- Camera access from web application
- File upload functionality (camera + gallery)
- JavaScript execution
- Secure file sharing

## Step 1: Android Manifest Permissions

Add the following permissions to your `AndroidManifest.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <!-- Required permissions -->
    <uses-permission android:name="android.permission.CAMERA" />
    <uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
    <uses-permission android:name="android.permission.INTERNET" />

    <!-- Optional: Camera feature (not required) -->
    <uses-feature android:name="android.hardware.camera" android:required="false" />

    <application>
        <!-- Your existing application configuration -->

        <!-- FileProvider for secure file sharing -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="YOUR_PACKAGE_NAME.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>
</manifest>
```

**Important:** Replace `YOUR_PACKAGE_NAME` with your actual application package name.

## Step 2: FileProvider Configuration

Create `res/xml/file_paths.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path name="my_images" path="." />
    <external-path name="external_files" path="." />
</paths>
```

## Step 3: WebView Layout

Add WebView to your layout (example using ConstraintLayout):

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <WebView
        android:id="@+id/webview"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

## Step 4: Activity Implementation

### Required Imports

```kotlin
import android.Manifest
import android.app.Activity
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Bundle
import android.provider.MediaStore
import android.webkit.*
import android.widget.Toast
import androidx.activity.result.ActivityResultLauncher
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import androidx.core.content.FileProvider
import java.io.File
import java.io.IOException
import java.text.SimpleDateFormat
import java.util.*
```

### Activity Class Structure

```kotlin
class YourActivity : AppCompatActivity() {

    private lateinit var webView: WebView
    private var filePathCallback: ValueCallback<Array<Uri>>? = null
    private var cameraPhotoPath: String? = null
    private var pendingPermissionRequest: PermissionRequest? = null

    // Permission launchers
    private val cameraPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            pendingPermissionRequest?.let { request ->
                request.grant(arrayOf(PermissionRequest.RESOURCE_VIDEO_CAPTURE))
                pendingPermissionRequest = null
            } ?: openFileChooser()
        } else {
            pendingPermissionRequest?.deny()
            pendingPermissionRequest = null
            Toast.makeText(this, "Camera permission is required for this feature", Toast.LENGTH_SHORT).show()
        }
    }

    private val fileChooserLauncher = registerForActivityResult(
        ActivityResultContracts.StartActivityForResult()
    ) { result ->
        if (filePathCallback == null) return@registerForActivityResult

        val results = if (result.resultCode == Activity.RESULT_OK) {
            if (result.data != null) {
                arrayOf(Uri.parse(result.data!!.dataString))
            } else {
                cameraPhotoPath?.let { arrayOf(Uri.fromFile(File(it))) }
            }
        } else null

        filePathCallback!!.onReceiveValue(results)
        filePathCallback = null
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.your_layout)

        setupWebView()
    }

    // ... (see Step 5 for setupWebView implementation)
}
```

## Step 5: WebView Configuration

Add this method to your Activity:

```kotlin
private fun setupWebView() {
    webView = findViewById(R.id.webview)

    // Essential WebView settings
    webView.settings.apply {
        javaScriptEnabled = true
        domStorageEnabled = true
        allowFileAccess = true
        allowContentAccess = true
        allowFileAccessFromFileURLs = true
        allowUniversalAccessFromFileURLs = true
        mediaPlaybackRequiresUserGesture = false
        mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
        cacheMode = WebSettings.LOAD_DEFAULT
        setGeolocationEnabled(true)
    }

    // WebChromeClient for file upload and camera permissions
    webView.webChromeClient = object : WebChromeClient() {
        // Handle file chooser (for file upload)
        override fun onShowFileChooser(
            webView: WebView,
            filePathCallback: ValueCallback<Array<Uri>>,
            fileChooserParams: FileChooserParams
        ): Boolean {
            this@YourActivity.filePathCallback = filePathCallback

            if (ContextCompat.checkSelfPermission(this@YourActivity, Manifest.permission.CAMERA)
                != PackageManager.PERMISSION_GRANTED) {
                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
            } else {
                openFileChooser()
            }
            return true
        }

        // Handle web camera permissions
        override fun onPermissionRequest(request: PermissionRequest) {
            runOnUiThread {
                val resources = request.resources
                for (resource in resources) {
                    when (resource) {
                        PermissionRequest.RESOURCE_VIDEO_CAPTURE -> {
                            if (ContextCompat.checkSelfPermission(this@YourActivity, Manifest.permission.CAMERA)
                                == PackageManager.PERMISSION_GRANTED) {
                                request.grant(arrayOf(PermissionRequest.RESOURCE_VIDEO_CAPTURE))
                            } else {
                                pendingPermissionRequest = request
                                cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                            }
                            return@runOnUiThread
                        }
                    }
                }
                request.grant(resources)
            }
        }
    }

    webView.webViewClient = WebViewClient()

    // Load the Velocia webapp
    webView.loadUrl("https://webapp.dev.velocia.io")
}
```

## Step 6: File Chooser Implementation

Add these helper methods to your Activity:

```kotlin
private fun openFileChooser() {
    val takePictureIntent = Intent(MediaStore.ACTION_IMAGE_CAPTURE)
    var photoFile: File? = null
    try {
        photoFile = createImageFile()
    } catch (ex: IOException) {
        Toast.makeText(this, "Error creating image file", Toast.LENGTH_SHORT).show()
    }

    if (photoFile != null) {
        cameraPhotoPath = photoFile.absolutePath
        val photoURI: Uri = FileProvider.getUriForFile(
            this,
            "YOUR_PACKAGE_NAME.fileprovider", // Replace with your package name
            photoFile
        )
        takePictureIntent.putExtra(MediaStore.EXTRA_OUTPUT, photoURI)
    }

    val galleryIntent = Intent(Intent.ACTION_GET_CONTENT)
    galleryIntent.type = "image/*"

    val chooserIntent = Intent.createChooser(galleryIntent, "Select Image")
    chooserIntent.putExtra(Intent.EXTRA_INITIAL_INTENTS, arrayOf(takePictureIntent))

    fileChooserLauncher.launch(chooserIntent)
}

@Throws(IOException::class)
private fun createImageFile(): File {
    val timeStamp: String = SimpleDateFormat("yyyyMMdd_HHmmss", Locale.getDefault()).format(Date())
    val storageDir: File = getExternalFilesDir(null)!!
    return File.createTempFile(
        "JPEG_${timeStamp}_",
        ".jpg",
        storageDir
    )
}

override fun onBackPressed() {
    if (webView.canGoBack()) {
        webView.goBack()
    } else {
        super.onBackPressed()
    }
}
```

## Step 7: Testing

After implementation, test the following:

1. **WebView loads the webapp** at `https://webapp.dev.velocia.io`
2. **Camera permission request** - When webapp requests camera access, native permission dialog appears
3. **File upload functionality** - When webapp triggers file upload, shows camera/gallery chooser
4. **Image capture** - Camera captures and returns image to webapp
5. **Gallery selection** - Gallery selection works and returns image to webapp

## Important Notes

### Security Considerations
- All file access is handled securely through FileProvider
- Permissions are requested only when needed
- Camera access requires explicit user permission

### Package Name Updates
Remember to replace `YOUR_PACKAGE_NAME` with your actual package name in:
- `AndroidManifest.xml` (FileProvider authority)
- `openFileChooser()` method (FileProvider.getUriForFile call)

### Minimum Requirements
- Android API level 21+ (Android 5.0)
- Camera hardware (optional - will gracefully handle absence)

### Troubleshooting
- If camera permission is denied automatically, ensure `onPermissionRequest` is properly implemented
- If file upload doesn't work, verify FileProvider configuration and authorities
- If images don't appear in webapp, check file URI generation and permissions

## Support

This configuration has been tested with the Velocia webapp and provides full functionality for camera access and file uploads within a WebView environment.