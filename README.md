 I see you have a layout for real-time detection with PreviewView and DetectionOverlay. Here's the corresponding CameraFragment implementation that matches your layout:

<antArtifact identifier="camera-fragment" type="application/vnd.ant.code" language="kotlin" title="Camera Fragment Implementation">
```kotlin
@AndroidEntryPoint
class CameraFragment : Fragment() {
    private var _binding: FragmentCameraBinding? = null
    private val binding get() = _binding!!

    private lateinit var cameraExecutor: ExecutorService
    private lateinit var detector: BaseDetector
    private lateinit var detectionModel: DetectionModel
    
    @Inject
    lateinit var detectionFactory: DetectionFactory

    companion object {
        private const val TAG = "CameraFragment"
        private const val REQUEST_CODE_PERMISSIONS = 10
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

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        val modelName = arguments?.getString(ARG_MODEL_NAME) 
            ?: throw IllegalStateException("Model name not found!")
            
        detectionModel = DetectionModelFactory.createModel(modelName)
            ?: throw IllegalArgumentException("Invalid model name: $modelName")
            
        detector = detectionFactory.createDetector(detectionModel)
        cameraExecutor = Executors.newSingleThreadExecutor()
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentCameraBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        if (allPermissionsGranted()) {
            startCamera()
        } else {
            requestPermissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(requireContext())

        cameraProviderFuture.addListener({
            try {
                val cameraProvider = cameraProviderFuture.get()
                bindCameraUseCases(cameraProvider)
            } catch (e: Exception) {
                Log.e(TAG, "Use case binding failed", e)
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

        val imageAnalyzer = ImageAnalysis.Builder()
            .setTargetAspectRatio(AspectRatio.RATIO_16_9)
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(cameraExecutor) { imageProxy ->
                    processImage(imageProxy)
                }
            }

        try {
            // Unbind any bound use cases before rebinding
            cameraProvider.unbindAll()

            // Bind use cases to camera
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
        lifecycleScope.launch(Dispatchers.Default) {
            try {
                imageProxy.use { proxy ->
                    // Convert ImageProxy to Bitmap
                    val bitmap = proxy.toBitmap()
                    
                    bitmap?.let {
                        // Run detection
                        val result = detector.detect(it, detectionModel.detectionType)
                        
                        // Update UI on main thread
                        withContext(Dispatchers.Main) {
                            binding.detectionOverlay.updateResults(result)
                        }
                    }
                }
            } catch (e: Exception) {
                Log.e(TAG, "Detection failed", e)
            }
        }
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
        cameraExecutor.shutdown()
    }
}

// FragmentCameraBinding.kt
class FragmentCameraBinding private constructor(
    private val rootView: View,
    val previewView: PreviewView,
    val detectionOverlay: DetectionOverlay
) {
    val root: View = rootView

    companion object {
        fun inflate(
            inflater: LayoutInflater,
            parent: ViewGroup?,
            attachToParent: Boolean
        ): FragmentCameraBinding {
            val root = inflater.inflate(
                R.layout.fragment_camera,
                parent,
                attachToParent
            )

            return FragmentCameraBinding(
                rootView = root,
                previewView = root.findViewById(R.id.preview_view),
                detectionOverlay = root.findViewById(R.id.detection_overlay)
            )
        }
    }
}
```

And your layout file (fragment_camera.xml):
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
</antArtifact>

Key features of this implementation:

1. Proper Camera Setup:
- Uses CameraX for preview and analysis
- Handles permissions correctly
- Efficient image processing pipeline

2. Detection Integration:
- Uses your detector and model architecture
- Processes frames on background thread
- Updates UI on main thread

3. Lifecycle Management:
- Proper binding cleanup
- Executor shutdown
- View binding nulling

4. Error Handling:
- Permission checks
- Camera setup failures
- Processing exceptions

To optimize performance, you might want to add:

1. Frame Processing Control:
```kotlin
private var isProcessing = AtomicBoolean(false)

private fun processImage(imageProxy: ImageProxy) {
    if (isProcessing.get()) {
        imageProxy.close()
        return
    }
    
    isProcessing.set(true)
    // ... your processing code ...
    isProcessing.set(false)
}
```

2. Resolution Control:
```kotlin
private fun getTargetResolution(): Size {
    val screenSize = Point()
    requireActivity().windowManager.defaultDisplay.getSize(screenSize)
    return Size(screenSize.x, screenSize.y)
}
```

3. Camera Selector:
```kotlin
private fun getCameraSelector(): CameraSelector {
    return try {
        if (hasBackCamera()) CameraSelector.DEFAULT_BACK_CAMERA
        else CameraSelector.DEFAULT_FRONT_CAMERA
    } catch (e: Exception) {
        CameraSelector.DEFAULT_BACK_CAMERA
    }
}
```

Would you like me to explain any part in more detail or show how to implement additional features?