I'll help you modify the DetectionOverlay to handle dynamic results using GridLayout. Based on your current code that handles single detector results, here's how we can adapt it for multiple detectors:

// DetectionOverlayAll.kt
class DetectionOverlayAll @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : GridLayout(context, attrs, defStyleAttr) {

    private val overlays = mutableMapOf<ModelType, DetectionOverlay>()
    private var gridSize = 1
    
    init {
        clipChildren = true
        clipToPadding = true
    }

    fun setupGrid(detectorCount: Int) {
        // Calculate grid dimensions (e.g., 2x2 for 4 detectors)
        gridSize = ceil(sqrt(detectorCount.toDouble())).toInt()
        
        rowCount = gridSize
        columnCount = gridSize
        
        // Clear existing overlays
        removeAllViews()
        overlays.clear()
        
        // Create overlay for each detector
        ModelType.values().take(detectorCount).forEachIndexed { index, modelType ->
            val overlay = DetectionOverlay(context).apply {
                layoutParams = createGridLayoutParams(index)
            }
            overlays[modelType] = overlay
            addView(overlay)
        }
    }

    private fun createGridLayoutParams(index: Int): GridLayout.LayoutParams {
        return GridLayout.LayoutParams().apply {
            width = 0
            height = 0
            rowSpec = GridLayout.spec(index / gridSize, 1, 1f)
            columnSpec = GridLayout.spec(index % gridSize, 1, 1f)
            setMargins(2, 2, 2, 2)
        }
    }

    fun updateResults(results: Map<ModelType, DetectionResultWithTiming>) {
        results.forEach { (modelType, result) ->
            overlays[modelType]?.let { overlay ->
                when (result.detectionResult) {
                    is DetectionResult.Anomaly -> {
                        overlay.updateAnomalyDetectionResults(
                            result.detectionResult.heatmap,
                            result.detectionResult.label.toString()
                        )
                    }
                    is DetectionResult.ObjectDetection -> {
                        overlay.updateObjectDetectionResults(result.detectionResult.detections)
                    }
                }
            }
        }
        invalidate()
    }
}

// DetectionOverlay.kt (Modified version of your existing class)
class DetectionOverlay @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    private var scaledBitmap: Bitmap? = null
    private var anomalyHeatmap: Bitmap? = null
    private var boundingBoxes = mutableListOf<Pair<RectF, String>>()
    private var anomalyLabel = ""
    private var captureMode = CaptureMode.PHOTO
    
    private val roiPaint = Paint().apply {
        color = Color.RED
        style = Paint.Style.STROKE
        strokeWidth = 2f
    }

    private val boundingBoxPaint = Paint().apply {
        color = Color.argb(150, 0, 255, 0)
        style = Paint.Style.FILL
    }

    private val textPaint = Paint().apply {
        color = Color.RED
        textSize = 48f
        style = Paint.Style.FILL
        isFakeBoldText = true
    }

    private val heatmapPaint = Paint()

    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)

        scaledBitmap?.let { bitmap ->
            // Calculate scaling to fit view
            val (scaledRect, scale, dx, dy) = calculateScaledDimensions(bitmap)
            
            if (captureMode == CaptureMode.PHOTO) {
                canvas.drawBitmap(bitmap, null, scaledRect, null)
            }

            // Draw anomaly heatmap if available
            anomalyHeatmap?.let { heatmap ->
                val roiRect = RectF(
                    scaledRect.left.toFloat(),
                    scaledRect.top.toFloat(),
                    scaledRect.right.toFloat(),
                    scaledRect.bottom.toFloat()
                )
                canvas.drawBitmap(heatmap, null, roiRect, heatmapPaint)
                
                // Draw anomaly label
                if (anomalyLabel.isNotEmpty()) {
                    canvas.drawText(anomalyLabel, 50f, 100f, textPaint)
                }
            }

            // Draw bounding boxes
            boundingBoxes.forEach { (box, label) ->
                val adjustedRect = RectF(
                    box.left * scale + dx,
                    box.top * scale + dy,
                    box.right * scale + dx,
                    box.bottom * scale + dy
                )
                canvas.drawRect(adjustedRect, boundingBoxPaint)
                
                // Draw label above bounding box
                val textX = adjustedRect.left
                val textY = adjustedRect.top - 10f
                canvas.drawText(label, textX, textY, textPaint)
            }
        }
    }

    private fun calculateScaledDimensions(bitmap: Bitmap): ScaledDimensions {
        val viewWidth = width.toFloat()
        val viewHeight = height.toFloat()
        val imageAspectRatio = bitmap.width.toFloat() / bitmap.height.toFloat()
        val viewAspectRatio = viewWidth / viewHeight
        
        val scale: Float
        val dx: Float
        val dy: Float
        
        if (viewAspectRatio > imageAspectRatio) {
            scale = viewHeight / bitmap.height
            dy = 0f
            dx = (viewWidth - (bitmap.width * scale)) / 2f
        } else {
            scale = viewWidth / bitmap.width
            dx = 0f
            dy = (viewHeight - (bitmap.height * scale)) / 2f
        }
        
        val scaledRect = RectF(
            dx,
            dy,
            viewWidth - dx,
            viewHeight - dy
        )
        
        return ScaledDimensions(scaledRect, scale, dx, dy)
    }

    fun updateResults(
        orgBitmap: Bitmap,
        image: Bitmap,
        detectionResult: DetectionResultWithTiming,
        mode: CaptureMode
    ) {
        scaledBitmap = orgBitmap
        captureMode = mode
        
        when (detectionResult.detectionResult) {
            is DetectionResult.Anomaly -> {
                updateAnomalyDetectionResults(
                    detectionResult.detectionResult.heatmap,
                    detectionResult.detectionResult.label.toString()
                )
            }
            is DetectionResult.ObjectDetection -> {
                updateObjectDetectionResults(detectionResult.detectionResult.detections)
            }
        }
        
        invalidate()
    }

    fun updateObjectDetectionResults(detectionResults: List<Pair<RectF, String>>) {
        boundingBoxes.clear()
        boundingBoxes.addAll(detectionResults)
        anomalyHeatmap = null
        anomalyLabel = ""
        invalidate()
    }

    fun updateAnomalyDetectionResults(heatmap: Bitmap, label: String) {
        anomalyHeatmap = heatmap
        anomalyLabel = label
        boundingBoxes.clear()
        invalidate()
    }

    private data class ScaledDimensions(
        val rect: RectF,
        val scale: Float,
        val dx: Float,
        val dy: Float
    )
}

// Usage in your Fragment/Activity
class MultiDetectionFragment : Fragment() {
    private lateinit var overlayAll: DetectionOverlayAll
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        overlayAll = view.findViewById(R.id.overlayAll)
        
        // Setup grid based on active detectors
        val activeDetectors = ModelType.values().size
        overlayAll.setupGrid(activeDetectors)
        
        // Update results when available
        viewModel.detectionResults.observe(viewLifecycleOwner) { results ->
            overlayAll.updateResults(results)
        }
    }
}

// layout/fragment_multi_detection.xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.camera.view.PreviewView
        android:id="@+id/viewFinder"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <com.your.package.DetectionOverlayAll
        android:id="@+id/overlayAll"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

</FrameLayout>

Key changes made:

1. Created `DetectionOverlayAll` class:
   - Uses GridLayout to arrange multiple overlays
   - Dynamically calculates grid size based on detector count
   - Manages individual overlays for each detector

2. Modified your existing `DetectionOverlay`:
   - Made it more reusable for grid layout
   - Added proper scaling calculations
   - Maintained existing drawing functionality

3. Added dynamic grid management:
   - Automatically calculates rows and columns
   - Equal sizing for all cells
   - Proper margins and spacing

4. Enhanced result handling:
   - Supports both object detection and anomaly results
   - Maintains aspect ratio of inputs
   - Clear separation between detectors

To use this:

1. Replace your single overlay with `DetectionOverlayAll` in your layout
2. Initialize the grid with your detector count
3. Update results as they come in from detectors

Would you like me to:
1. Add additional customization options for the grid layout?
2. Implement alternative layout strategies?
3. Add performance optimizations for the drawing?
