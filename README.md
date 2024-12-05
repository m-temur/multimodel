 I understand you want to update the `CameraFragment` to handle real-time detection while maintaining your existing project structure. Here's the compatible implementation:

<antArtifact identifier="camera-fragment" type="application/vnd.ant.code" language="kotlin" title="Updated Camera Fragment Implementation">
package ai.setkpsw.objectdetection.ui.fragments

import android.Manifest
import android.content.pm.PackageManager
import android.os.Bundle
import android.util.Log
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Toast
import androidx.activity.result.contract.ActivityResultContracts
import androidx.camera.core.*
import androidx.camera.lifecycle.ProcessCameraProvider
import androidx.camera.view.PreviewView
import androidx.core.content.ContextCompat
import androidx.fragment.app.Fragment
import androidx.lifecycle.lifecycleScope
import ai.setkpsw.objectdetection.R
import ai.setkpsw.objectdetection.detection.factory.DetectionFactory
import ai.setkpsw.objectdetection.detection.base.DetectionModel
import ai.setkpsw.objectdetection.detection.factory.DetectionModelFactory
import dagger.hilt.android.AndroidEntryPoint
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch
import kotlinx.coroutines.withContext
import java.util.concurrent.ExecutorService
import java.util.concurrent.Executors
import javax.inject.Inject

@AndroidEntryPoint
class CameraFragment : Fragment() {
    @Inject
    lateinit var detectionFactory: DetectionFactory

    private lateinit var detectionModel: DetectionModel
    private lateinit var cameraExecutor: ExecutorService
    private lateinit var previewView: PreviewView
    private lateinit var detectionOverlay: DetectionOverlay
    private var imageAnalyzer: ImageAnalysis? = null
    private var camera: Camera? = null
    
    private var processingFrame = false

    companion object {
        private const val TAG = "CameraFragment"
        private const val ARG_MODEL_NAME = "modelName"

        fun newInstance(modelName: String): CameraFragment {
            val args = Bundle()
            args.putString(ARG_MODEL_NAME, modelName)
            val fragment = CameraFragment()
            fragment.arguments = args
            return fragment
        }
    }

    private val requestPermissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            startCamera()
        } else {
            Toast.makeText(context, "Camera permission is required", Toast.LENGTH_SHORT).show()
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        return inflater.inflate(R.layout.fragment_camera, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        previewView = view.findViewById(R.id.preview_view)
        detectionOverlay = view.findViewById(R.id.detection_overlay)
        cameraExecutor = Executors.newSingleThreadExecutor()

        val modelName = arguments?.getString(ARG_MODEL_NAME) ?: return
        detectionModel = DetectionModelFactory.createModel(modelName)
            ?: throw IllegalArgumentException("Invalid model name: $modelName")

        val detector = detectionFactory.createDetector(detectionModel)
        
        checkCameraPermissionAndStart(detector)
    }

    private fun checkCameraPermissionAndStart(detector: BaseDetector) {
        when {
            ContextCompat.checkSelfPermission(
                requireContext(),
                Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> {
                startCamera(detector)
            }
            else -> {
                requestPermissionLauncher.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun startCamera(detector: BaseDetector) {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(requireContext())

        cameraProviderFuture.addListener({
            try {
                val cameraProvider = cameraProviderFuture.get()
                bindCameraPreview(cameraProvider, detector)
            } catch (e: Exception) {
                Log.e(TAG, "Failed to start camera", e)
            }
        }, ContextCompat.getMainExecutor(requireContext()))
    }

    private fun bindCameraPreview(cameraProvider: ProcessCameraProvider, detector: BaseDetector) {
        // Configure Preview
        val preview = Preview.Builder()
            .setTargetResolution(Size(
                detectionModel.modelConfig.inputSize.first,
                detectionModel.modelConfig.inputSize.second
            ))
            .build()
            .also {
                it.setSurfaceProvider(previewView.surfaceProvider)
            }

        // Configure ImageAnalysis
        imageAnalyzer = ImageAnalysis.Builder()
            .setTargetResolution(Size(
                detectionModel.modelConfig.inputSize.first,
                detectionModel.modelConfig.inputSize.second
            ))
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(cameraExecutor) { imageProxy ->
                    if (!processingFrame) {
                        processImage(imageProxy, detector)
                    } else {
                        imageProxy.close()
                    }
                }
            }

        try {
            // Unbind any bound use cases before rebinding
            cameraProvider.unbindAll()

            // Bind use cases to camera
            camera = cameraProvider.bindToLifecycle(
                viewLifecycleOwner,
                CameraSelector.DEFAULT_BACK_CAMERA,
                preview,
                imageAnalyzer
            )
        } catch (e: Exception) {
            Log.e(TAG, "Use case binding failed", e)
        }
    }

    private fun processImage(imageProxy: ImageProxy, detector: BaseDetector) {
        processingFrame = true
        
        lifecycleScope.launch(Dispatchers.Default) {
            try {
                val bitmap = imageProxy.toBitmap()
                
                // Use the detector to process the image
                val detectionResult = detector.detect(bitmap, detectionModel.detectionType)
                
                // Update UI with results
                withContext(Dispatchers.Main) {
                    detectionOverlay.updateResults(detectionResult)
                }
            } catch (e: Exception) {
                Log.e(TAG, "Error processing frame: ${e.message}")
            } finally {
                processingFrame = false
                imageProxy.close()
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        cameraExecutor.shutdown()
    }
}
</antArtifact>

Key points about this implementation:

1. It maintains your existing dependency injection structure using Hilt
2. Uses your existing `DetectionFactory` and `DetectionModel` abstractions
3. Properly handles camera lifecycle within the Fragment
4. Processes frames using your detector implementation
5. Updates the UI using your detection overlay

Make sure you have the following layout file (fragment_camera.xml):

```xml
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

The implementation:
1. Integrates with your existing model selection workflow
2. Handles camera permissions
3. Sets up real-time frame analysis
4. Uses your detection factory and models
5. Updates the UI with detection results

Would you like me to explain any part in more detail or help with additional components?