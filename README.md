// ModelAdapter.kt
class ModelAdapter(
    private val models: List<ModelItem>,
    private val onModelSelected: (String) -> Unit
) : RecyclerView.Adapter<ModelAdapter.ModelViewHolder>() {

    class ModelViewHolder(private val binding: ItemModelBinding) : 
        RecyclerView.ViewHolder(binding.root) {
        
        fun bind(item: ModelItem, onModelSelected: (String) -> Unit) {
            binding.apply {
                modelTitle.text = item.name
                modelDescription.text = item.description
                modelType.text = item.type
                modelIcon.setImageResource(item.iconRes)
                
                root.setOnClickListener {
                    onModelSelected(item.name)
                }
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ModelViewHolder {
        val binding = ItemModelBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ModelViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ModelViewHolder, position: Int) {
        holder.bind(models[position], onModelSelected)
    }

    override fun getItemCount() = models.size
}

// ModelItem.kt
data class ModelItem(
    val name: String,
    val description: String,
    val type: String,
    @DrawableRes val iconRes: Int
)

// ModelSelectionFragment.kt
@AndroidEntryPoint
class ModelSelectionFragment : Fragment() {
    private var _binding: FragmentModelSelectionBinding? = null
    private val binding get() = _binding!!

    private lateinit var detectionMode: String

    companion object {
        private const val ARG_DETECTION_MODE = "detectionMode"

        fun newInstance(detectionMode: String): ModelSelectionFragment {
            return ModelSelectionFragment().apply {
                arguments = Bundle().apply {
                    putString(ARG_DETECTION_MODE, detectionMode)
                }
            }
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentModelSelectionBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        detectionMode = arguments?.getString(ARG_DETECTION_MODE)
            ?: throw IllegalStateException("Detection mode not found!")

        setupToolbar()
        setupRecyclerView()
    }

    private fun setupToolbar() {
        binding.toolbar.apply {
            title = when (detectionMode) {
                "REALTIME" -> "Select Real-time Detection Model"
                "PHOTO" -> "Select Photo Detection Model"
                else -> "Select Model"
            }
            setNavigationOnClickListener {
                findNavController().navigateUp()
            }
        }
    }

    private fun setupRecyclerView() {
        val models = getAvailableModels()
        
        binding.recyclerView.apply {
            layoutManager = LinearLayoutManager(context)
            addItemDecoration(
                MaterialDividerItemDecoration(
                    context, 
                    LinearLayoutManager.VERTICAL
                ).apply {
                    dividerInsetStart = resources.getDimensionPixelSize(R.dimen.divider_inset)
                    dividerInsetEnd = resources.getDimensionPixelSize(R.dimen.divider_inset)
                }
            )
            adapter = ModelAdapter(models) { modelName ->
                startDetection(modelName)
            }
        }
    }

    private fun getAvailableModels(): List<ModelItem> {
        return when (detectionMode) {
            "REALTIME" -> listOf(
                ModelItem(
                    name = "YOLO",
                    description = "Real-time object detection using YOLO model",
                    type = "Object Detection",
                    iconRes = R.drawable.ic_object_detection
                ),
                // Add other real-time models
            )
            "PHOTO" -> listOf(
                ModelItem(
                    name = "PADIM",
                    description = "Industrial anomaly detection using PADIM",
                    type = "Anomaly Detection",
                    iconRes = R.drawable.ic_anomaly_detection
                ),
                // Add other photo-based models
            )
            else -> emptyList()
        }
    }

    private fun startDetection(modelName: String) {
        val fragment = when (detectionMode) {
            "REALTIME" -> CameraFragment.newInstance(modelName)
            "PHOTO" -> PhotoDetectionFragment.newInstance(modelName)
            else -> throw IllegalArgumentException("Unknown detection mode: $detectionMode")
        }

        parentFragmentManager.beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .addToBackStack(null)
            .commit()
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

Layout files:

```xml
<!-- fragment_model_selection.xml -->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.google.android.material.appbar.MaterialToolbar
        android:id="@+id/toolbar"
        android:layout_width="match_parent"
        android:layout_height="?attr/actionBarSize"
        android:background="?attr/colorPrimary"
        android:elevation="4dp"
        app:navigationIcon="@drawable/ic_arrow_back"
        app:titleTextColor="?attr/colorOnPrimary" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:clipToPadding="false"
        android:padding="8dp" />

</LinearLayout>

<!-- item_model.xml -->
<?xml version="1.0" encoding="utf-8"?>
<com.google.android.material.card.MaterialCardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_marginHorizontal="8dp"
    android:layout_marginVertical="4dp"
    app:cardElevation="2dp">

    <androidx.constraintlayout.widget.ConstraintLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="16dp">

        <ImageView
            android:id="@+id/modelIcon"
            android:layout_width="48dp"
            android:layout_height="48dp"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent"
            tools:src="@drawable/ic_object_detection" />

        <TextView
            android:id="@+id/modelTitle"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginStart="16dp"
            android:textAppearance="?attr/textAppearanceTitleMedium"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toEndOf="@id/modelIcon"
            app:layout_constraintTop_toTopOf="@id/modelIcon"
            tools:text="YOLO Model" />

        <com.google.android.material.chip.Chip
            android:id="@+id/modelType"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="4dp"
            app:layout_constraintStart_toStartOf="@id/modelTitle"
            app:layout_constraintTop_toBottomOf="@id/modelTitle"
            tools:text="Object Detection" />

        <TextView
            android:id="@+id/modelDescription"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_marginTop="8dp"
            android:textAppearance="?attr/textAppearanceBodyMedium"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="@id/modelTitle"
            app:layout_constraintTop_toBottomOf="@id/modelType"
            tools:text="Real-time object detection using YOLO model" />

    </androidx.constraintlayout.widget.ConstraintLayout>

</com.google.android.material.card.MaterialCardView>
```

Add these values to your resources:

```xml
<!-- dimens.xml -->
<resources>
    <dimen name="divider_inset">16dp</dimen>
</resources>
```

This implementation provides:

1. Material Design card-based UI for model selection
2. Clear model information display with icons and descriptions
3. Proper navigation handling
4. Different model lists based on detection mode
5. View Binding for safe view access
6. RecyclerView with dividers and proper spacing

To use this, you'll need these dependencies:

```gradle
dependencies {
    implementation 'androidx.recyclerview:recyclerview:1.3.2'
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
}
```

Would you like me to explain any part in more detail or show how to implement additional features?



 Ah, I apologize for the misunderstanding! You're absolutely right. Each model (whether it's for anomaly or object detection) should work with both real-time and photo detection methods. Let me correct the implementation:

<antArtifact identifier="corrected-ui-implementation" type="application/vnd.ant.code" language="kotlin" title="Corrected UI Implementation">
// ModelSelectionFragment.kt
@AndroidEntryPoint
class ModelSelectionFragment : Fragment() {
    private lateinit var binding: FragmentModelSelectionBinding
    private lateinit var detectionMode: String

    companion object {
        private const val ARG_DETECTION_MODE = "detectionMode"

        fun newInstance(detectionMode: String) = ModelSelectionFragment().apply {
            arguments = Bundle().apply {
                putString(ARG_DETECTION_MODE, detectionMode)
            }
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        binding = FragmentModelSelectionBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        detectionMode = arguments?.getString(ARG_DETECTION_MODE) 
            ?: throw IllegalStateException("Detection mode not found!")

        setupModelList()
    }

    private fun setupModelList() {
        binding.modelListContainer.removeAllViews()

        // All models are available for both detection modes
        DetectionModelFactory.getAvailableModels().forEach { modelName ->
            val modelButton = MaterialButton(requireContext()).apply {
                text = modelName
                layoutParams = LinearLayout.LayoutParams(
                    LinearLayout.LayoutParams.MATCH_PARENT,
                    LinearLayout.LayoutParams.WRAP_CONTENT
                ).apply {
                    setMargins(0, 0, 0, 16)
                }
                setOnClickListener {
                    navigateToDetection(modelName)
                }
            }
            binding.modelListContainer.addView(modelButton)
        }
    }

    private fun navigateToDetection(modelName: String) {
        // Create the appropriate fragment based on detection mode
        val fragment = when (detectionMode) {
            "REALTIME" -> CameraFragment.newInstance(modelName)
            "PHOTO" -> PhotoDetectionFragment.newInstance(modelName)
            else -> throw IllegalArgumentException("Unknown detection mode: $detectionMode")
        }

        parentFragmentManager.beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .addToBackStack(null)
            .commit()
    }
}

// PhotoDetectionFragment.kt
@AndroidEntryPoint
class PhotoDetectionFragment : Fragment() {
    private lateinit var binding: FragmentPhotoDetectionBinding
    private var selectedImageBitmap: Bitmap? = null
    private lateinit var detector: BaseDetector
    private lateinit var detectionModel: DetectionModel

    @Inject
    lateinit var detectionFactory: DetectionFactory

    companion object {
        private const val ARG_MODEL_NAME = "modelName"
        private const val REQUEST_IMAGE_PICK = 1

        fun newInstance(modelName: String) = PhotoDetectionFragment().apply {
            arguments = Bundle().apply {
                putString(ARG_MODEL_NAME, modelName)
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Initialize detector based on model name
        val modelName = arguments?.getString(ARG_MODEL_NAME) 
            ?: throw IllegalStateException("Model name not found!")
            
        detectionModel = DetectionModelFactory.createModel(modelName)
            ?: throw IllegalArgumentException("Invalid model name: $modelName")
            
        detector = detectionFactory.createDetector(detectionModel)
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        binding = FragmentPhotoDetectionBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        setupViews()
    }

    private fun setupViews() {
        binding.apply {
            selectImageButton.setOnClickListener { selectImage() }
            detectButton.setOnClickListener { runDetection() }
            modelNameText.text = detectionModel::class.simpleName
        }
    }

    private fun runDetection() {
        selectedImageBitmap?.let { bitmap ->
            lifecycleScope.launch {
                try {
                    val result = detector.detect(
                        bitmap, 
                        detectionModel.detectionType
                    )
                    binding.detectionOverlay.updateResults(result)
                } catch (e: Exception) {
                    Toast.makeText(
                        context, 
                        "Detection failed: ${e.message}", 
                        Toast.LENGTH_SHORT
                    ).show()
                }
            }
        }
    }

    // ... rest of the PhotoDetectionFragment implementation
}

// CameraFragment.kt
@AndroidEntryPoint
class CameraFragment : Fragment() {
    private lateinit var binding: FragmentCameraBinding
    private lateinit var detector: BaseDetector
    private lateinit var detectionModel: DetectionModel
    private lateinit var cameraExecutor: ExecutorService

    @Inject
    lateinit var detectionFactory: DetectionFactory

    companion object {
        private const val ARG_MODEL_NAME = "modelName"

        fun newInstance(modelName: String) = CameraFragment().apply {
            arguments = Bundle().apply {
                putString(ARG_MODEL_NAME, modelName)
            }
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Initialize detector based on model name
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
        binding = FragmentCameraBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        binding.modelNameText.text = detectionModel::class.simpleName
        
        if (allPermissionsGranted()) {
            startCamera()
        } else {
            requestPermissions()
        }
    }

    private fun startCamera() {
        val cameraProviderFuture = ProcessCameraProvider.getInstance(requireContext())

        cameraProviderFuture.addListener({
            val cameraProvider = cameraProviderFuture.get()
            bindCameraUseCases(cameraProvider)
        }, ContextCompat.getMainExecutor(requireContext()))
    }

    private fun bindCameraUseCases(cameraProvider: ProcessCameraProvider) {
        // Set up camera preview
        val preview = Preview.Builder()
            .build()
            .also {
                it.setSurfaceProvider(binding.previewView.surfaceProvider)
            }

        // Set up image analysis
        val imageAnalyzer = ImageAnalysis.Builder()
            .setBackpressureStrategy(ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST)
            .build()
            .also {
                it.setAnalyzer(cameraExecutor) { imageProxy ->
                    processImage(imageProxy)
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
        // Convert ImageProxy to Bitmap and run detection
        val bitmap = imageProxy.toBitmap()
        lifecycleScope.launch {
            try {
                val result = detector.detect(
                    bitmap, 
                    detectionModel.detectionType
                )
                binding.detectionOverlay.updateResults(result)
            } catch (e: Exception) {
                Log.e(TAG, "Detection failed", e)
            } finally {
                imageProxy.close()
            }
        }
    }

    // ... rest of the CameraFragment implementation
}
</antArtifact>

Key changes in this corrected version:

1. Both PhotoDetectionFragment and CameraFragment:
- Can work with any model type (YOLO, MobileNet, PADIM, PatchCore)
- Use the generic BaseDetector interface
- Initialize the appropriate detector through DetectionFactory

2. ModelSelectionFragment:
- Shows all available models for both detection modes
- No filtering based on model type

3. Proper Model Initialization:
- Models are created through DetectionModelFactory
- Detectors are created through DetectionFactory
- Each fragment can handle any type of detection

This structure allows:
- Any model to be used in both real-time and photo modes
- Proper dependency injection
- Clear separation of concerns
- Type-safe detection

Would you like me to explain any specific part in more detail?


 Yes, we need to implement the toBitmap() extension function for ImageProxy. Here's the implementation:

<antArtifact identifier="image-processing" type="application/vnd.ant.code" language="kotlin" title="Image Processing Extensions">
// ImageExtensions.kt
import android.graphics.Bitmap
import android.graphics.BitmapFactory
import android.graphics.ImageFormat
import android.graphics.Matrix
import android.graphics.Rect
import android.graphics.YuvImage
import android.media.Image
import androidx.camera.core.ImageProxy
import java.io.ByteArrayOutputStream

fun ImageProxy.toBitmap(): Bitmap? {
    val nv21 = yuv420888ToNv21(this)
    val yuvImage = YuvImage(nv21, ImageFormat.NV21, width, height, null)
    return yuvImage.toBitmap()
}

private fun yuv420888ToNv21(image: ImageProxy): ByteArray {
    val pixelCount = image.cropRect.width() * image.cropRect.height()
    val pixelSizeBits = ImageFormat.getBitsPerPixel(ImageFormat.YUV_420_888)
    val outputBuffer = ByteArray(pixelCount * pixelSizeBits / 8)
    
    imageToByteBuffer(image, outputBuffer, pixelCount)
    return outputBuffer
}

private fun imageToByteBuffer(image: ImageProxy, outputBuffer: ByteArray, pixelCount: Int) {
    assert(image.format == ImageFormat.YUV_420_888)
    
    val imageCrop = image.cropRect
    val imagePlanes = image.planes
    
    imagePlanes.forEachIndexed { planeIndex, plane ->
        // How many values are read in input for each output value written
        val outputStride: Int
        
        // The index in the output buffer the next value will be written at
        var outputOffset: Int
        
        when (planeIndex) {
            0 -> {
                outputStride = 1
                outputOffset = 0
            }
            1 -> {
                outputStride = 2
                outputOffset = pixelCount + 1
            }
            2 -> {
                outputStride = 2
                outputOffset = pixelCount
            }
            else -> {
                // Image contains more than 3 planes, something strange is going on
                return@forEachIndexed
            }
        }
        
        val buffer = plane.buffer
        val rowStride = plane.rowStride
        val pixelStride = plane.pixelStride
        
        val planeCrop = if (planeIndex == 0) {
            imageCrop
        } else {
            Rect(
                imageCrop.left / 2,
                imageCrop.top / 2,
                imageCrop.right / 2,
                imageCrop.bottom / 2
            )
        }
        
        val planeWidth = planeCrop.width()
        val planeHeight = planeCrop.height()
        
        buffer.position(
            rowStride * planeCrop.top + pixelStride * planeCrop.left
        )
        
        for (row in 0 until planeHeight) {
            var outputPos = outputOffset
            if (pixelStride == 1 && outputStride == 1) {
                // When there is a single stride value for pixel and output, we can just copy
                // the entire row in a single step
                buffer.get(outputBuffer, outputPos, planeWidth)
                outputPos += planeWidth
            } else {
                // When either pixel or output have a stride > 1 we must copy pixel by pixel
                for (col in 0 until planeWidth) {
                    outputBuffer[outputPos] = buffer.get()
                    outputPos += outputStride
                    if (col < planeWidth - 1) {
                        buffer.position(buffer.position() + pixelStride - 1)
                    }
                }
            }
            
            if (row < planeHeight - 1) {
                buffer.position(
                    buffer.position() + rowStride - (pixelStride * planeWidth)
                )
            }
            
            outputOffset += outputStride * planeWidth
        }
    }
}

fun YuvImage.toBitmap(): Bitmap? {
    val out = ByteArrayOutputStream()
    if (!compressToJpeg(Rect(0, 0, width, height), 100, out)) {
        return null
    }
    val imageBytes: ByteArray = out.toByteArray()
    return BitmapFactory.decodeByteArray(imageBytes, 0, imageBytes.size)
}

// Optional: Add rotation handling if needed
fun Bitmap.rotate(degrees: Float): Bitmap {
    val matrix = Matrix().apply { postRotate(degrees) }
    return Bitmap.createBitmap(this, 0, 0, width, height, matrix, true)
}
</antArtifact>

Now you can update the image processing in CameraFragment:

```kotlin
private fun processImage(imageProxy: ImageProxy) {
    lifecycleScope.launch(Dispatchers.Default) {
        try {
            // Convert image to bitmap
            val bitmap = imageProxy.toBitmap()
            
            bitmap?.let {
                // Run detection
                val result = detector.detect(it, detectionModel.detectionType)
                
                // Update UI on main thread
                withContext(Dispatchers.Main) {
                    binding.detectionOverlay.updateResults(result)
                }
            }
        } catch (e: Exception) {
            Log.e(TAG, "Detection failed", e)
        } finally {
            imageProxy.close()
        }
    }
}
```

Key features of this implementation:

1. Handles YUV_420_888 format from CameraX
2. Efficient byte buffer operations
3. Proper image plane handling
4. Optional rotation support

Additional considerations:

1. Add image rotation if needed:
```kotlin
// Check image rotation
val rotation = imageProxy.imageInfo.rotationDegrees
bitmap = bitmap?.rotate(rotation.toFloat())
```

2. Add image resizing if needed:
```kotlin
fun Bitmap.resize(targetWidth: Int, targetHeight: Int): Bitmap {
    return Bitmap.createScaledBitmap(this, targetWidth, targetHeight, true)
}
```

3. Add memory management:
```kotlin
private fun processImage(imageProxy: ImageProxy) {
    lifecycleScope.launch(Dispatchers.Default) {
        var bitmap: Bitmap? = null
        try {
            bitmap = imageProxy.toBitmap()
            // Process bitmap...
        } finally {
            bitmap?.recycle()
            imageProxy.close()
        }
    }
}
```

Would you like me to explain any part in more detail or show how to handle specific image processing scenarios?