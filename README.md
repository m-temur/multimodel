 I'll provide the UI code that perfectly aligns with your existing flow, maintaining your current architecture.

<antArtifact identifier="ui-implementation" type="application/vnd.ant.code" language="kotlin" title="UI Implementation Following Existing Flow">
// DetectionActivity.kt
@AndroidEntryPoint
class DetectionActivity : AppCompatActivity() {
    private lateinit var binding: ActivityDetectionBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityDetectionBinding.inflate(layoutInflater)
        setContentView(binding.root)

        setupViews()
    }

    private fun setupViews() {
        binding.apply {
            realtimeDetectionButton.setOnClickListener {
                navigateToModelSelection("REALTIME")
            }
            photoDetectionButton.setOnClickListener {
                navigateToModelSelection("PHOTO")
            }
        }
    }

    private fun navigateToModelSelection(detectionMode: String) {
        val fragment = ModelSelectionFragment.newInstance(detectionMode)
        supportFragmentManager.beginTransaction()
            .replace(R.id.fragment_container, fragment)
            .addToBackStack(null)
            .commit()
    }
}

// activity_detection.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <FrameLayout
        android:id="@+id/fragment_container"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1" />

    <LinearLayout
        android:id="@+id/button_container"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical">

        <Button
            android:id="@+id/realtime_detection_button"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:text="Real-time Detection" />

        <Button
            android:id="@+id/photo_detection_button"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="8dp"
            android:text="Photo Detection" />
    </LinearLayout>
</LinearLayout>

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

        getAvailableModels().forEach { modelName ->
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

    private fun getAvailableModels(): List<String> {
        return when (detectionMode) {
            "REALTIME" -> listOf("YOLO", "MobileNet")
            "PHOTO" -> listOf("PADIM", "PatchCore")
            else -> emptyList()
        }
    }

    private fun navigateToDetection(modelName: String) {
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

// fragment_model_selection.xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:id="@+id/selection_title"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"
        android:textAlignment="center"
        android:textAppearance="?attr/textAppearanceHeadline6" />

    <ScrollView
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <LinearLayout
            android:id="@+id/model_list_container"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:orientation="vertical" />
    </ScrollView>
</LinearLayout>

// PhotoDetectionFragment.kt
@AndroidEntryPoint
class PhotoDetectionFragment : Fragment() {
    private lateinit var binding: FragmentPhotoDetectionBinding
    private var selectedImageBitmap: Bitmap? = null
    
    @Inject lateinit var detector: AnomalyDetector

    companion object {
        private const val ARG_MODEL_NAME = "modelName"
        private const val REQUEST_IMAGE_PICK = 1

        fun newInstance(modelName: String) = PhotoDetectionFragment().apply {
            arguments = Bundle().apply {
                putString(ARG_MODEL_NAME, modelName)
            }
        }
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
        }
    }

    private fun selectImage() {
        val intent = Intent(Intent.ACTION_PICK).apply {
            type = "image/*"
        }
        startActivityForResult(intent, REQUEST_IMAGE_PICK)
    }

    private fun runDetection() {
        selectedImageBitmap?.let { bitmap ->
            lifecycleScope.launch {
                try {
                    val result = detector.detect(bitmap, DetectionType.ANOMALY)
                    binding.detectionOverlay.updateResults(result)
                } catch (e: Exception) {
                    Toast.makeText(context, "Detection failed", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)
        if (requestCode == REQUEST_IMAGE_PICK && resultCode == Activity.RESULT_OK) {
            data?.data?.let { uri ->
                loadImage(uri)
            }
        }
    }

    private fun loadImage(uri: Uri) {
        lifecycleScope.launch {
            try {
                val bitmap = withContext(Dispatchers.IO) {
                    MediaStore.Images.Media.getBitmap(
                        requireContext().contentResolver,
                        uri
                    )
                }
                selectedImageBitmap = bitmap
                binding.imageView.setImageBitmap(bitmap)
                binding.detectButton.isEnabled = true
            } catch (e: Exception) {
                Toast.makeText(context, "Failed to load image", Toast.LENGTH_SHORT).show()
            }
        }
    }
}
</antArtifact>

Key aspects of this implementation:

1. Simple Activity/Fragment Navigation:
- DetectionActivity -> ModelSelectionFragment -> Camera/PhotoFragment

2. Clean UI Structure:
- Activity hosts main detection mode buttons
- ModelSelectionFragment displays available models
- PhotoDetectionFragment handles image selection and detection

3. Maintains Existing Architecture:
- No changes to your detection flow
- Uses existing detector implementations
- Follows established pattern

4. Simple ViewBinding:
- Safer view access
- Better compile-time safety

To use this, add these layout dependencies:

```gradle
dependencies {
    implementation 'com.google.android.material:material:1.11.0'
    implementation 'androidx.fragment:fragment-ktx:1.6.2'
}
```

Would you like me to explain any specific part in more detail?