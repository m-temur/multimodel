Let me help clarify. There are two approaches we can take:

1. Using existing DetectionOverlay:
- Create multiple DetectionOverlay views, one for each model
- Arrange them in a grid layout
- Each will handle its own drawing

2. Creating a new combined overlay:
- Single view that handles all model results
- Divides its canvas into sections
- Draws all results itself

Given that you already have a working DetectionOverlay class, I'd recommend approach #1. It will be simpler and reuse your existing code. Let me show you a cleaner version:

class DetectionOverlayAll @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : ViewGroup(context, attrs, defStyleAttr) {
    
    private val overlayViews = mutableListOf<DetectionOverlay>()
    private val modelLabels = mutableListOf<TextView>()
    private var columns = 2

    fun setModels(modelCount: Int) {
        removeAllViews()
        overlayViews.clear()
        modelLabels.clear()
        
        // Calculate grid size
        columns = when {
            modelCount <= 2 -> 1
            modelCount <= 4 -> 2
            else -> 3
        }
        
        // Create views for each model
        repeat(modelCount) { index ->
            // Create label
            val label = TextView(context).apply {
                textSize = 16f
                setTextColor(Color.WHITE)
                gravity = Gravity.CENTER
                setPadding(8, 8, 8, 8)
                setBackgroundColor(Color.argb(150, 0, 0, 0))
            }
            modelLabels.add(label)
            addView(label)
            
            // Create overlay
            val overlay = DetectionOverlay(context)
            overlayViews.add(overlay)
            addView(overlay)
        }
    }

    override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
        val width = MeasureSpec.getSize(widthMeasureSpec)
        val height = MeasureSpec.getSize(heightMeasureSpec)
        
        val childWidth = width / columns
        val rows = ceil(overlayViews.size.toFloat() / columns).toInt()
        val childHeight = height / rows
        
        // Measure all children
        for (i in 0 until childCount step 2) {
            // Measure label
            val label = getChildAt(i) as TextView
            label.measure(
                MeasureSpec.makeMeasureSpec(childWidth, MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(height/10, MeasureSpec.AT_MOST)
            )
            
            // Measure overlay
            val overlay = getChildAt(i + 1) as DetectionOverlay
            overlay.measure(
                MeasureSpec.makeMeasureSpec(childWidth, MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(childHeight - label.measuredHeight, MeasureSpec.EXACTLY)
            )
        }
        
        setMeasuredDimension(width, height)
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        val width = r - l
        val childWidth = width / columns
        val rows = ceil((childCount/2).toFloat() / columns).toInt()
        val totalHeight = b - t
        val cellHeight = totalHeight / rows
        
        for (i in 0 until childCount step 2) {
            val index = i / 2
            val row = index / columns
            val col = index % columns
            
            val cellLeft = col * childWidth
            val cellTop = row * cellHeight
            
            // Layout label
            val label = getChildAt(i) as TextView
            label.layout(
                cellLeft,
                cellTop,
                cellLeft + childWidth,
                cellTop + label.measuredHeight
            )
            
            // Layout overlay
            val overlay = getChildAt(i + 1) as DetectionOverlay
            overlay.layout(
                cellLeft,
                cellTop + label.measuredHeight,
                cellLeft + childWidth,
                cellTop + cellHeight
            )
        }
    }

    fun updateModelResult(index: Int, bitmap: Bitmap, result: DetectionResult, modelName: String, inferenceTime: Long) {
        if (index < overlayViews.size) {
            overlayViews[index].updateResults(bitmap, result)
            modelLabels[index].text = "$modelName (${inferenceTime}ms)"
        }
    }
}

This implementation:
1. Extends ViewGroup to manage multiple DetectionOverlay views
2. Creates a grid of DetectionOverlay instances
3. Adds labels above each overlay
4. Reuses your existing DetectionOverlay class

Usage in fragment:
```kotlin
class RealtimeDetectionFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        binding.detectionOverlayAll.setModels(detectors.size)
    }
    
    private fun processImage(bitmap: Bitmap?) {
        if (bitmap == null) return
        
        lifecycleScope.launch(Dispatchers.Default) {
            detectors.forEachIndexed { index, detector ->
                val startTime = System.currentTimeMillis()
                val result = detector.detect(bitmap)
                val endTime = System.currentTimeMillis()
                
                withContext(Dispatchers.Main) {
                    binding.detectionOverlayAll.updateModelResult(
                        index,
                        bitmap,
                        result,
                        detector::class.simpleName ?: "Unknown",
                        endTime - startTime
                    )
                }
            }
        }
    }
}
```

XML layout:
```xml
<com.your.package.DetectionOverlayAll
    android:id="@+id/detectionOverlayAll"
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

Would you like me to explain any part in more detail?
