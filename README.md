# Android WebView Camera Access Implementation Guide (Minimum requirements for Velocia WebApp)

## Overview
This guide provides step-by-step instructions for implementing camera access and file upload functionality in an Android app that hosts a web application in a WebView. This enables the web application to request and use the device's camera through proper native permission handling.

## What You'll Add to Your Existing App
- Camera and storage permissions
- FileProvider for secure file sharing
- WebView with JavaScript enabled
- Camera permission handling for web content
- File upload support (camera + gallery)

## Prerequisites
- Android Studio Arctic Fox or later
- Android project with Kotlin
- Web application that requires camera access
- Android SDK minimum version 21 (Android 5.0)

## Implementation Steps (If you already have a WebView setup in your native application, please reference the basic configuration below and add any missing parts if necessary)

### Step 1: Update AndroidManifest.xml

Add the following permissions and FileProvider configuration to your AndroidManifest.xml:
```xml
<!-- Add these permissions before <application> tag -->
<uses-permission android:name="android.permission.CAMERA" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.INTERNET" />

<!-- Camera feature (optional) -->
<uses-feature android:name="android.hardware.camera" android:required="false" />

<!-- Add this provider inside your <application> tag -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

### Step 2: Create File Provider Configuration

Create a new XML resource file for FileProvider paths.

**Create file:** `res/xml/file_paths.xml`
```xml
<?xml version="1.0" encoding="utf-8"?>
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <external-files-path name="my_images" path="." />
    <external-path name="external_files" path="." />
</paths>
```

### Step 3: Update Layout File

Create or update your activity layout file (res/layout/activity_main.xml) to include a WebView:
```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

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

### Step 4: Update MainActivity.kt

Update your MainActivity with the following WebView implementation and camera handling:
```kotlin
package com.yourcompany.yourapp

import android.Manifest
import android.app.Activity
import android.content.Intent
import android.content.pm.PackageManager
import android.net.Uri
import android.os.Bundle
import android.provider.MediaStore
import android.webkit.*
import android.widget.Toast
import androidx.activity.OnBackPressedCallback
import androidx.activity.result.contract.ActivityResultContracts
import androidx.appcompat.app.AppCompatActivity
import androidx.core.content.ContextCompat
import androidx.core.content.FileProvider
import java.io.File
import java.io.IOException
import java.text.SimpleDateFormat
import java.util.*

class MainActivity : AppCompatActivity() {

    private lateinit var webView: WebView
    private var filePathCallback: ValueCallback<Array<Uri>>? = null
    private var cameraPhotoPath: String? = null
    private var pendingPermissionRequest: PermissionRequest? = null

    // Camera permission launcher
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

    // File chooser launcher
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
        setContentView(R.layout.activity_main)
        setupWebView()
        setupBackPressHandler()
    }

    private fun setupWebView() {
        webView = findViewById(R.id.webview)

        // Configure WebView settings
        webView.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
            allowFileAccess = true
            allowContentAccess = true
            mediaPlaybackRequiresUserGesture = false
            mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
            cacheMode = WebSettings.LOAD_DEFAULT
            setGeolocationEnabled(true)
        }

        // Set WebChromeClient for handling file uploads and permissions
        webView.webChromeClient = object : WebChromeClient() {

            // Handle file chooser for uploads
            override fun onShowFileChooser(
                webView: WebView,
                filePathCallback: ValueCallback<Array<Uri>>,
                fileChooserParams: FileChooserParams
            ): Boolean {
                this@MainActivity.filePathCallback = filePathCallback

                if (ContextCompat.checkSelfPermission(this@MainActivity, Manifest.permission.CAMERA)
                    != PackageManager.PERMISSION_GRANTED) {
                    cameraPermissionLauncher.launch(Manifest.permission.CAMERA)
                } else {
                    openFileChooser()
                }
                return true
            }

            // Handle permission requests from web content
            override fun onPermissionRequest(request: PermissionRequest) {
                runOnUiThread {
                    val resources = request.resources
                    for (resource in resources) {
                        when (resource) {
                            PermissionRequest.RESOURCE_VIDEO_CAPTURE -> {
                                if (ContextCompat.checkSelfPermission(this@MainActivity, Manifest.permission.CAMERA)
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

        // Load your web application URL
        webView.loadUrl("YOUR_WEBAPP_URL_HERE")
    }

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
                "$packageName.fileprovider",
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

    private fun setupBackPressHandler() {
        onBackPressedDispatcher.addCallback(this, object : OnBackPressedCallback(true) {
            override fun handleOnBackPressed() {
                if (webView.canGoBack()) {
                    webView.goBack()
                } else {
                    finish()
                }
            }
        })
    }
}
```

## Key Components Explained

### 1. WebView Settings
```kotlin
webView.settings.apply {
    javaScriptEnabled = true
    domStorageEnabled = true
    allowFileAccess = true
    // ... other settings
}
```
- **javaScriptEnabled**: Enables JavaScript execution
- **domStorageEnabled**: Enables DOM storage API
- **allowFileAccess**: Allows file access for the WebView
- **allowContentAccess**: Enables content URL access within WebView
- **mediaPlaybackRequiresUserGesture**: Allows media autoplay without user interaction
- **mixedContentMode**: Allows loading HTTP content on HTTPS pages
- **setGeolocationEnabled**: Enables geolocation API

### 2. WebChromeClient
Handles advanced WebView features:
- **onShowFileChooser**: Intercepts file upload requests
- **onPermissionRequest**: Handles permission requests from web content

### 3. Permission Handling
```kotlin
private val cameraPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { isGranted ->
    // Handle permission result
}
```
Uses Android's Activity Result API for modern permission handling.

### 4. FileProvider
Enables secure file sharing between the app and camera:
```kotlin
val photoURI: Uri = FileProvider.getUriForFile(
    this,
    "$packageName.fileprovider",
    photoFile
)
```

## Testing the Implementation

1. **Build and run** your app on a physical Android device
2. **Replace** `YOUR_WEBAPP_URL_HERE` with your actual web application URL
3. **Navigate** to a page that requests camera access or file upload
4. **Verify** that:
   - Permission dialog appears when camera is first requested
   - File chooser shows both camera and gallery options
   - Selected/captured images are properly uploaded to the web app
   - Camera stream works correctly for WebRTC applications

## Common Issues and Solutions

### Issue 1: FileProvider crash on camera launch
**Solution:** Ensure the authorities in FileProvider match `$packageName.fileprovider`

### Issue 2: Camera permission denied automatically
**Solution:** Check that `onPermissionRequest` is properly implemented in WebChromeClient

### Issue 3: File upload not working
**Solution:** Verify `onShowFileChooser` is implemented and returns `true`

### Issue 4: Images not uploading after capture
**Solution:** Ensure file_paths.xml is properly configured

### Issue 5: WebView not loading
**Solution:** Verify INTERNET permission is added to AndroidManifest.xml

## Security Considerations

1. **HTTPS Required**: Camera access via getUserMedia requires HTTPS
2. **Domain Validation**: Consider checking the origin before granting permissions
3. **File Access**: Use FileProvider to securely share files
4. **Permission Descriptions**: Users should understand why permissions are needed

## Sample Permission Handler with Domain Checking

```kotlin
override fun onPermissionRequest(request: PermissionRequest) {
    runOnUiThread {
        // Only allow for trusted domains
        val origin = request.origin.toString()
        if (origin.contains("yourdomain.com")) {
            val resources = request.resources
            for (resource in resources) {
                when (resource) {
                    PermissionRequest.RESOURCE_VIDEO_CAPTURE -> {
                        // Handle camera permission
                        if (ContextCompat.checkSelfPermission(this@MainActivity, Manifest.permission.CAMERA)
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
        } else {
            request.deny()
        }
    }
}
```

## Minimum Requirements

- **Android API Level**: 21+ (Android 5.0 Lollipop)
- **Kotlin Version**: 1.5+
- **AndroidX**: Required for modern Android development
- **Physical Device**: Camera features require a physical device for testing

## Project Structure

```
app/
├── src/
│   ├── main/
│   │   ├── java/com/yourcompany/yourapp/
│   │   │   └── MainActivity.kt
│   │   ├── res/
│   │   │   ├── layout/
│   │   │   │   └── activity_main.xml
│   │   │   └── xml/
│   │   │       └── file_paths.xml
│   │   └── AndroidManifest.xml
```

## Dependencies

Ensure your app's `build.gradle` includes:
```gradle
dependencies {
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    // ... other dependencies
}
```

## Additional Resources

- [Android WebView Documentation](https://developer.android.com/reference/android/webkit/WebView)
- [Android Permissions Best Practices](https://developer.android.com/training/permissions/requesting)
- [FileProvider Documentation](https://developer.android.com/reference/androidx/core/content/FileProvider)
- [WebChromeClient Reference](https://developer.android.com/reference/android/webkit/WebChromeClient)

## Contributing

If you find issues or have improvements to suggest, please:
1. Fork the repository
2. Create a feature branch
3. Submit a pull request with a clear description

## License

This example code is provided under the MIT License. See LICENSE file for details.

## Support

If you encounter any issues implementing this solution, please provide:
1. Error messages or crash logs
2. Android version being tested
3. Device manufacturer and model
4. Web application URL (if possible)
5. Any custom modifications made to the implementation

---

**Note:** Remember to replace `YOUR_WEBAPP_URL_HERE` with your actual web application URL before building the app.
