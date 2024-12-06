 Let's fix the camera preview to be truly full screen by modifying the PreviewView settings and layout. Here's the corrected implementation:

<antArtifact identifier="fullscreen-camera-fix" type="application/vnd.ant.code" language="kotlin" title="Fixed Full Screen Camera Implementation">
```xml
<!-- fragment_camera.xml -->
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/black">

    <androidx.camera.view.PreviewView
        android:id="@+id/preview_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_gravity="center" />

    <ai.setkpsw.objectdetection.ui.views.DetectionOverlay
        android:id="@+id/detection_overlay"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>
```

```kotlin
// CameraFragment.kt
@AndroidEntryPoint
class CameraFragment : Fragment() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Make truly fullscreen
        requireActivity().window.apply {
            setFlags(
                WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS or
                WindowManager.LayoutParams.FLAG_FULLSCREEN,
                WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS or
                WindowManager.LayoutParams.FLAG_FULLSCREEN
            )
            
            decorView.systemUiVisibility = (View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                    or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
                    or View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    or View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                    or View.SYSTEM_UI_FLAG_FULLSCREEN
                    or View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY)
        }
    }

    private fun bindCameraUseCases(cameraProvider: ProcessCameraProvider) {
        // Get screen metrics
        val metrics = DisplayMetrics().also { 
            binding.previewView.display.getRealMetrics(it)
        }

        // Set up the preview use case
        val preview = Preview.Builder()
            .setTargetResolution(Size(metrics.widthPixels, metrics.heightPixels))
            .setTargetRotation(binding.previewView.display.rotation)
            .build()

        // Configure the preview view
        binding.previewView.apply {
            implementationMode = PreviewView.ImplementationMode.PERFORMANCE
            scaleType = PreviewView.ScaleType.FILL_CENTER
        }

        preview.setSurfaceProvider(binding.previewView.surfaceProvider)

        // Set up image analysis use case
        imageAnalyzer = ImageAnalysis.Builder()
            .setTargetResolution(Size(metrics.widthPixels, metrics.heightPixels))
            .setTargetRotation(binding.previewView.display.rotation)
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(cameraExecutor) { imageProxy ->
                    if (!isProcessing) {
                        processImage(imageProxy)
                    } else {
                        imageProxy.close()
                    }
                }
            }

        try {
            cameraProvider.unbindAll()
            
            // Select back camera
            val cameraSelector = CameraSelector.Builder()
                .requireLensFacing(CameraSelector.LENS_FACING_BACK)
                .build()

            camera = cameraProvider.bindToLifecycle(
                viewLifecycleOwner,
                cameraSelector,
                preview,
                imageAnalyzer
            )

            // Set initial camera settings
            camera?.cameraControl?.apply {
                setLinearZoom(0f)
                enableTorch(false)
            }
        } catch (e: Exception) {
            Log.e(TAG, "Use case binding failed", e)
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        // Restore window flags
        requireActivity().window.apply {
            clearFlags(
                WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS or
                WindowManager.LayoutParams.FLAG_FULLSCREEN
            )
            decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_VISIBLE
        }
        _binding = null
    }
}
```

```xml
<!-- Add to themes.xml -->
<style name="Theme.YourApp.FullScreen" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="android:windowNoTitle">true</item>
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
    <item name="android:navigationBarColor">@android:color/transparent</item>
    <item name="android:statusBarColor">@android:color/transparent</item>
    <item name="android:windowTranslucentStatus">true</item>
    <item name="android:windowTranslucentNavigation">true</item>
</style>
```

```xml
<!-- Update in AndroidManifest.xml -->
<activity
    android:name=".ui.DetectionActivity"
    android:theme="@style/Theme.YourApp.FullScreen"
    android:screenOrientation="portrait"
    android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.MAIN" />
        <category android:name="android.intent.category.LAUNCHER" />
    </intent-filter>
    
    <!-- Add support for edge-to-edge display -->
    <meta-data
        android:name="android.max_aspect"
        android:value="2.4" />
</activity>
```

Key changes made:
1. Simplified layout to use FrameLayout
2. Using FILL_CENTER scale type for PreviewView
3. Setting target resolution to match screen size
4. Proper fullscreen flags
5. Translucent status and navigation bars
6. Performance implementation mode for PreviewView

This implementation should:
1. Fill the entire screen
2. Handle notch areas correctly
3. Maintain proper aspect ratio
4. Hide system bars
5. Provide better performance

Would you like me to explain any part in more detail or help with additional adjustments?</antArtifact>