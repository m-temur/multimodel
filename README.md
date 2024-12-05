 I'll provide the complete implementation of CameraFragment:

<antArtifact identifier="complete-camera-fragment" type="application/vnd.ant.code" language="kotlin" title="Complete Camera Fragment Implementation">
```kotlin
package ai.setkpsw.objectdetection.ui.fragments

import android.Manifest
import android.content.pm.PackageManager
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.ImageFormat
import android.graphics.Rect
import android.os.Bundle
import android.util.Log
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.core.content.ContextCompat
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import com.google.android.material.dialog.MaterialAlertDialogBuilder
import ai.setkpsw.objectdetection.R
import ai.setkpsw.objectdetection.databinding.FragmentCameraBinding
import ai.setkpsw.objectdetection.detection.base.BaseDetector
import ai.setkpsw.objectdetection.detection.base.DetectionModel
import ai.setkpsw.objectdetection.detection.factory.DetectionFactory
import ai.setkpsw.objectdetection.detection.factory.DetectionModelFactory
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import java.io.ByteArrayOutputStream
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import javax.inject.Inject
import android.media.Image

@AndroidEntryPoint
class CameraFragment : Fragment() {
    private var _binding: FragmentCameraBinding? = null
    private val binding get() = _binding!!

    private lateinit var cameraExecutor: ExecutorService
    private lateinit var detectionModel: DetectionModel
    private lateinit var detector: BaseDetector
    
    @Inject
    lateinit var detectionFactory: DetectionFactory

    private var imageAnalyzer: ImageAnalysis? = null
    private var isProcessing = false

    companion object {
        private const val TAG = "CameraFragment"
        private const val ARG_MODEL_NAME = "modelName"
        private val REQUIRED_PERMISSIONS = arrayOf(Manifest.permission.CAMERA)

        fun newInstance(modelName: String) = CameraFragment().apply {
            arguments = Bundle().apply {
                putString(ARG_MODEL_NAME, modelName)
            }
        }
    }

    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            startCamera()
        } else {
            Toast.makeText(context, "Camera permission is required", Toast.LENGTH_SHORT).show()
            activity?.onBackPressed()
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentCameraBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val modelName = arguments?.getString(ARG_MODEL_NAME) 
            ?: throw IllegalStateException("Model name not found!")
            
        detectionModel = DetectionModelFactory.createModel(modelName)
            ?: throw IllegalArgumentException("Invalid model name: $modelName")
            
        detector = detectionFactory.createDetector(detectionModel)
        cameraExecutor = Executors.newSingleThreadExecutor()

        // Initialize detector in coroutine
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

    private fun requestCameraPermission() {
        when {
            ContextCompat.checkSelfPermission(
                requireContext(),
                Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                startCamera()
            }
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                showPermissionExplanationDialog()
            }
            else -> {
                requestPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun showPermissionExplanationDialog() {
        MaterialAlertDialogBuilder(requireContext())
            .setTitle("Camera Permission Required")
            .setMessage("This app needs camera permission to perform detection.")
            .setPositiveButton("Grant") { _, _ ->
                requestPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
            .setNegativeButton("Cancel") { _, _ ->
                Toast.makeText(
                    context,
                    "Camera permission is required for detection",
                    Toast.LENGTH_SHORT
                ).show()
                activity?.onBackPressed()
            }
            .show()
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(requireContext())

        cameraProviderFuture.addListener({
            try {
                val cameraProvider = cameraProviderFuture.get()
                bindCameraUseCases(cameraProvider)
            } catch (e: Exception) {
                Log.e(TAG, "Failed to start camera", e)
                Toast.makeText(
                    context,
                    "Failed to start camera: ${e.message}",
                    Toast.LENGTH_SHORT
                ).show()
            }
        }, ContextCompat.getMainExecutor(requireContext()))
    }

    private fun bindCameraUseCases(cameraProvider: ProcessCameraProvider) {
        val preview = Preview.Builder()
            .setTargetAspectRatio(AspectRatio.RATIO_16_9)
            .build()
            .also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

        imageAnalyzer = ImageAnalysis.Builder()
            .setTargetAspectRatio(AspectRatio.RATIO_16_9)
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

    private fun processImage(imageProxy: ImageProxy) {
        isProcessing = true
        
        lifecycleScope.launch(Dispatchers.Default) {
            try {
                val bitmap = imageProxy.toBitmap()
                bitmap?.let {
                    val result = detector.detect(it, detectionModel.detectionType)
                    withContext(Dispatchers.Main) {
                        binding.detectionOverlay.updateResults(result)
                    }
                }
            } catch (e: Exception) {
                Log.e(TAG, "Detection failed", e)
            } finally {
                isProcessing = false
                imageProxy.close()
            }
        }
    }

    private fun ImageProxy.toBitmap(): Bitmap? {
        val yBuffer = planes[0].buffer
        val uBuffer = planes[1].buffer
        val vBuffer = planes[2].buffer

        val ySize = yBuffer.remaining()
        val uSize = uBuffer.remaining()
        val vSize = vBuffer.remaining()

        val nv21 = ByteArray(ySize + uSize + vSize)

        yBuffer.get(nv21, 0, ySize)
        vBuffer.get(nv21, ySize, vSize)
        uBuffer.get(nv21, ySize + vSize, uSize)

        val yuvImage = android.graphics.YuvImage(
            nv21, 
            ImageFormat.NV21, 
            width, 
            height, 
            null
        )
        
        val out = ByteArrayOutputStream()
        yuvImage.compressToJpeg(Rect(0, 0, width, height), 100, out)
        val imageBytes = out.toByteArray()
        
        return BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)
    }

    private fun allPermissionsGranted() = REQUIRED_PERMISSIONS.all {
        ContextCompat.checkSelfPermission(
            requireContext(), it
        ) == PackageManager.PERMISSION_GRANTED
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }

    override fun onDestroy() {
        super.onDestroy()
        detector.release()
        cameraExecutor.shutdown()
    }
}
```

And here's the corresponding layout file:

```xml
<!-- fragment_camera.xml -->
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.camera.view.PreviewView
        android:id="@+id/preview_view"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <ai.setkpsw.objectdetection.ui.views.DetectionOverlay
        android:id="@+id/detection_overlay"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>
```

Key features of this implementation:

1. Complete Lifecycle Management:
- Proper initialization
- Resource cleanup
- Memory management

2. Camera Handling:
- CameraX integration
- Efficient frame processing
- Back-pressure handling

3. Permission Management:
- Runtime permissions
- User feedback
- Permission rationale

4. Error Handling:
- Graceful error recovery
- User feedback
- Proper logging

5. Coroutines Integration:
- Async processing
- Thread management
- Cancellation handling

Dependencies needed:
```gradle
dependencies {
    def camerax_version = "1.3.0-alpha04"
    
    implementation "androidx.camera:camera-core:$camerax_version"
    implementation "androidx.camera:camera-camera2:$camerax_version"
    implementation "androidx.camera:camera-lifecycle:$camerax_version"
    implementation "androidx.camera:camera-view:$camerax_version"
    
    implementation 'androidx.core:core-ktx:1.10.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4"
}
```

Would you like me to explain any specific part in more detail?</antArtifact>