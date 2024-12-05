 I'll help you modify the layout and CameraFragment to achieve a fullscreen camera preview with overlaid detection results:

<antArtifact identifier="fullscreen-camera" type="application/vnd.ant.code" language="kotlin" title="Fullscreen Camera Implementation">
```xml
<!-- fragment_camera.xml -->
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@android:color/black">

    <androidx.camera.view.PreviewView
        android:id="@+id/preview_view"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent"
        app:scaleType="fillStart" />

    <ai.setkpsw.objectdetection.ui.views.DetectionOverlay
        android:id="@+id/detection_overlay"
        android:layout_width="0dp"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="@id/preview_view"
        app:layout_constraintEnd_toEndOf="@id/preview_view"
        app:layout_constraintStart_toStartOf="@id/preview_view"
        app:layout_constraintTop_toTopOf="@id/preview_view" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

Update the CameraFragment to handle fullscreen:

```kotlin
@AndroidEntryPoint
class CameraFragment : Fragment() {
    // ... other properties

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // Make the fragment fullscreen
        requireActivity().window.apply {
            addFlags(WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS)
            statusBarColor = Color.TRANSPARENT
            decorView.systemUiVisibility = (View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                    or View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                    or View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION)
        }
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupCamera()
    }

    private fun setupCamera() {
        binding.previewView.apply {
            implementationMode = PreviewView.ImplementationMode.PERFORMANCE
            scaleType = PreviewView.ScaleType.FILL_START
        }

        lifecycleScope.launch {
            try {
                detector.initialize()
                withContext(Dispatchers.Main) {
                    if (allPermissionsGranted()) {
                        startCamera()
                    } else {
                        requestCameraPermission()
                    }
                }
            } catch (e: Exception) {
                Log.e(TAG, "Failed to initialize detector", e)
                withContext(Dispatchers.Main) {
                    Toast.makeText(
                        context,
                        "Failed to initialize detector: ${e.message}",
                        Toast.LENGTH_SHORT
                    ).show()
                    activity?.onBackPressed()
                }
            }
        }
    }

    private fun bindCameraUseCases(cameraProvider: ProcessCameraProvider) {
        // Get screen metrics
        val metrics = DisplayMetrics().also { 
            binding.previewView.display.getRealMetrics(it) 
        }
        val screenAspectRatio = aspectRatio(metrics.widthPixels, metrics.heightPixels)
        val rotation = binding.previewView.display.rotation

        // Preview
        val preview = Preview.Builder()
            .setTargetAspectRatio(screenAspectRatio)
            .setTargetRotation(rotation)
            .build()
            .also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

        // ImageAnalysis
        imageAnalyzer = ImageAnalysis.Builder()
            .setTargetAspectRatio(screenAspectRatio)
            .setTargetRotation(rotation)
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
            cameraProvider.bindToLifecycle(
                viewLifecycleOwner,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageAnalyzer
            )
        } catch (e: Exception) {
            Log.e(TAG, "Use case binding failed", e)
        }
    }

    private fun aspectRatio(width: Int, height: Int): Int {
        val previewRatio = max(width, height).toDouble() / min(width, height)
        if (abs(previewRatio - RATIO_4_3_VALUE) <= abs(previewRatio - RATIO_16_9_VALUE)) {
            return AspectRatio.RATIO_4_3
        }
        return AspectRatio.RATIO_16_9
    }

    override fun onDestroyView() {
        super.onDestroyView()
        // Restore system UI
        requireActivity().window.apply {
            clearFlags(WindowManager.LayoutParams.FLAG_LAYOUT_NO_LIMITS)
            decorView.systemUiVisibility = View.SYSTEM_UI_FLAG_VISIBLE
        }
        _binding = null
    }

    companion object {
        private const val RATIO_4_3_VALUE = 4.0 / 3.0
        private const val RATIO_16_9_VALUE = 16.0 / 9.0
        // ... other companion object properties
    }

    // ... rest of the implementation
}
```

And update the DetectionOverlay view to match the camera preview:

```kotlin
class DetectionOverlay @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {
    private val paint = Paint().apply {
        style = Paint.Style.STROKE
        strokeWidth = 4f
        color = Color.RED
    }

    private val textPaint = Paint().apply {
        color = Color.WHITE
        textSize = 48f
        style = Paint.Style.FILL
        setShadowLayer(4f, 0f, 0f, Color.BLACK)
    }

    private var detectionResults: DetectionResult? = null
    private var scaleFactor: Float = 1f
    private var previewWidth: Int = 0
    private var previewHeight: Int = 0

    fun updateResults(results: DetectionResult) {
        detectionResults = results
        invalidate()
    }

    fun setPreviewSize(width: Int, height: Int) {
        previewWidth = width
        previewHeight = height
        calculateScaleFactor()
    }

    private fun calculateScaleFactor() {
        if (previewWidth == 0 || previewHeight == 0) return
        
        val viewRatio = width.toFloat() / height
        val previewRatio = previewWidth.toFloat() / previewHeight
        
        scaleFactor = if (viewRatio > previewRatio) {
            height.toFloat() / previewHeight
        } else {
            width.toFloat() / previewWidth
        }
    }

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        detectionResults?.let { result ->
            // Scale and draw detection boxes
            result.boundingBoxes.forEachIndexed { index, box ->
                val scaledBox = RectF(
                    box.left * scaleFactor,
                    box.top * scaleFactor,
                    box.right * scaleFactor,
                    box.bottom * scaleFactor
                )
                
                canvas.drawRect(scaledBox, paint)
                
                // Draw label
                val label = "${result.labels[index]} ${(result.confidences[index] * 100).toInt()}%"
                canvas.drawText(label, scaledBox.left, scaledBox.top - 10f, textPaint)
            }
        }
    }
}
```

Key features of this implementation:

1. Fullscreen Camera Preview:
- Uses ConstraintLayout for proper scaling
- Handles different aspect ratios
- Maintains preview quality

2. Properly Scaled Overlay:
- Matches camera preview dimensions
- Scales detection results correctly
- Handles different screen sizes

3. System UI Integration:
- Transparent status bar
- Edge-to-edge display
- Proper cleanup

To make this work, add these to your styles:

```xml
<!-- styles.xml -->
<style name="FullscreenTheme" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="android:windowTranslucentStatus">true</item>
    <item name="android:windowTranslucentNavigation">true</item>
    <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
</style>
```

Would you like me to explain any specific part in more detail or help with additional customization?</antArtifact>