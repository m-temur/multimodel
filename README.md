Ah, I understand now - you want to split the screen into sections, one for each model's detection results. Let me help you modify the code to create this grid-style display.

// Custom ViewGroup to manage multiple detection views
class DetectionGridLayout @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {
    
    private val overlayViews = mutableListOf<DetectionOverlay>()
    private val modelLabels = mutableListOf<TextView>()
    private var columns = 2 // Default to 2 columns
    
    fun setModels(modelCount: Int) {
        // Remove existing views
        removeAllViews()
        overlayViews.clear()
        modelLabels.clear()
        
        // Calculate grid dimensions
        columns = when {
            modelCount <= 2 -> 1
            modelCount <= 4 -> 2
            else -> 3
        }
        
        // Create views for each model
        repeat(modelCount) { index ->
            // Create container for each model
            val container = LinearLayout(context).apply {
                orientation = LinearLayout.VERTICAL
                layoutParams = LayoutParams(
                    LayoutParams.MATCH_PARENT,
                    LayoutParams.MATCH_PARENT
                )
            }
            
            // Add model label
            val label = TextView(context).apply {
                textSize = 16f
                setTextColor(Color.WHITE)
                gravity = Gravity.CENTER
                setPadding(8, 8, 8, 8)
                setBackgroundColor(Color.argb(150, 0, 0, 0))
            }
            modelLabels.add(label)
            container.addView(label)
            
            // Add detection overlay
            val overlay = DetectionOverlay(context).apply {
                layoutParams = LinearLayout.LayoutParams(
                    LayoutParams.MATCH_PARENT,
                    0,
                    1f
                )
            }
            overlayViews.add(overlay)
            container.addView(overlay)
            
            addView(container)
        }
    }
    
    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val width = MeasureSpec.getSize(widthMeasureSpec)
        val height = MeasureSpec.getSize(heightMeasureSpec)
        
        val childWidth = width / columns
        val rows = ceil(childCount.toFloat() / columns).toInt()
        val childHeight = height / rows
        
        // Measure children
        children.forEach { child ->
            val childWidthSpec = MeasureSpec.makeMeasureSpec(childWidth, MeasureSpec.EXACTLY)
            val childHeightSpec = MeasureSpec.makeMeasureSpec(childHeight, MeasureSpec.EXACTLY)
            child.measure(childWidthSpec, childHeightSpec)
        }
        
        setMeasuredDimension(width, height)
    }
    
    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        val width = r - l
        val childWidth = width / columns
        val rows = ceil(childCount.toFloat() / columns).toInt()
        val childHeight = (b - t) / rows
        
        children.forEachIndexed { index, child ->
            val row = index / columns
            val col = index % columns
            
            val childLeft = col * childWidth
            val childTop = row * childHeight
            val childRight = childLeft + childWidth
            val childBottom = childTop + childHeight
            
            child.layout(childLeft, childTop, childRight, childBottom)
        }
    }
    
    fun updateModelResult(index: Int, modelName: String, bitmap: Bitmap, result: DetectionResult) {
        if (index < overlayViews.size) {
            overlayViews[index].updateResults(bitmap, result)
            modelLabels[index].text = modelName
        }
    }
}

// Modified Fragment code to use the grid layout
class RealtimeDetectionFragment : Fragment() {
    private lateinit var binding: FragmentRealtimeDetectionBinding
    private val detectors = mutableListOf<Detector>()
    
    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        binding = FragmentRealtimeDetectionBinding.inflate(inflater, container, false)
        
        // Initialize detectors and setup grid
        initializeDetectors()
        binding.detectionGrid.setModels(detectors.size)
        
        return binding.root
    }
    
    private fun processImage(bitmap: Bitmap?) {
        if (bitmap == null) {
            Toast.makeText(context, "Failed to capture image", Toast.LENGTH_SHORT).show()
            return
        }

        lifecycleScope.launch(Dispatchers.Default) {
            try {
                detectors.forEachIndexed { index, detector ->
                    val startTime = System.currentTimeMillis()
                    val result = detector.detect(bitmap)
                    val endTime = System.currentTimeMillis()
                    
                    withContext(Dispatchers.Main) {
                        binding.detectionGrid.updateModelResult(
                            index,
                            "${detector::class.simpleName} (${endTime - startTime}ms)",
                            bitmap,
                            result
                        )
                    }
                }
            } catch (e: Exception) {
                withContext(Dispatchers.Main) {
                    showError("Detection failed: ${e.message}")
                }
            }
        }
    }
}

// Layout XML
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.your.package.DetectionGridLayout
        android:id="@+id/detectionGrid"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>

Key changes in this implementation:

1. Created a custom `DetectionGridLayout` that:
   - Automatically arranges detection views in a grid
   - Adjusts grid layout based on number of models
   - Includes labels for each model with inference time
   - Handles proper scaling and positioning of each view

2. Each model gets its own:
   - DetectionOverlay view
   - Label showing model name and inference time
   - Dedicated section of the screen

3. The grid layout automatically:
   - Adjusts columns based on model count (1 column for ≤2 models, 2 columns for 3-4 models, 3 columns for more)
   - Maintains aspect ratio for each section
   - Handles proper scaling of bounding boxes and overlays

To use this:

1. Replace your existing layout with the new grid layout XML
2. Initialize your detectors as before
3. The grid will automatically create and manage views for each detector

Would you like me to explain any part in more detail or help with specific customization of the grid layout?
