 I see the error is related to uninitialized OnnxExecutor session. Let's fix this initialization issue in the AnomalyDetector and OnnxExecutor classes:

<antArtifact identifier="fixed-executor" type="application/vnd.ant.code" language="kotlin" title="Fixed Executor Implementation">
```kotlin
// OnnxExecutor.kt
@Singleton
class OnnxExecutor @Inject constructor(
    private val context: Context,
    private val ortEnvironment: OrtEnvironment
) : BaseExecutor {
    private var ortSession: OrtSession? = null
    private val initializationLock = Object()

    override suspend fun initialize(modelConfig: ModelConfig) = withContext(Dispatchers.IO) {
        synchronized(initializationLock) {
            if (ortSession == null) {
                try {
                    val modelBytes = context.assets.open(modelConfig.modelPath).readBytes()
                    ortSession = ortEnvironment.createSession(modelBytes)
                } catch (e: Exception) {
                    throw ExecutorInitializationException("Failed to initialize ONNX session", e)
                }
            }
        }
    }

    override suspend fun runInference(preprocessed: Any): Any {
        val session = ortSession ?: throw ExecutorNotInitializedException("ONNX session not initialized")
        
        return withContext(Dispatchers.Default) {
            try {
                val inputTensor = when (preprocessed) {
                    is FloatArray -> OnnxTensor.createTensor(
                        ortEnvironment,
                        FloatBuffer.wrap(preprocessed),
                        longArrayOf(1, 3, 224, 224) // Adjust based on your model
                    )
                    else -> throw IllegalArgumentException("Unsupported input type")
                }

                inputTensor.use { input ->
                    val outputTensor = session.run(mapOf("input" to input))
                    outputTensor[0].value
                }
            } catch (e: Exception) {
                throw ExecutorInferenceException("Failed to run ONNX inference", e)
            }
        }
    }

    override fun close() {
        ortSession?.close()
        ortSession = null
    }
}

// ExecutorInitializationException.kt
class ExecutorInitializationException(message: String, cause: Throwable? = null) : 
    Exception(message, cause)

class ExecutorNotInitializedException(message: String) : Exception(message)

class ExecutorInferenceException(message: String, cause: Throwable? = null) : 
    Exception(message, cause)

// AnomalyDetector.kt
@Singleton
class AnomalyDetector @Inject constructor(
    private val executor: BaseExecutor,
    private val processingStrategy: ProcessingStrategy
) : BaseDetector {

    private var isInitialized = false
    private val initializationLock = Object()

    override suspend fun initialize(model: DetectionModel) {
        synchronized(initializationLock) {
            if (!isInitialized) {
                executor.initialize(model.modelConfig)
                isInitialized = true
            }
        }
    }

    override suspend fun detect(input: Bitmap, detectionType: DetectionType): DetectionResult {
        if (!isInitialized) {
            throw DetectorNotInitializedException("Detector not initialized")
        }

        return withContext(Dispatchers.Default) {
            try {
                // Preprocess
                val preprocessStartTime = System.nanoTime()
                val preprocessed = processingStrategy.preprocess(input)
                val preprocessTime = System.nanoTime() - preprocessStartTime

                // Run inference
                val inferenceStartTime = System.nanoTime()
                val rawOutput = executor.runInference(preprocessed)
                val inferenceTime = System.nanoTime() - inferenceStartTime

                // Postprocess
                val postprocessStartTime = System.nanoTime()
                val result = processingStrategy.postprocess(rawOutput)
                val postprocessTime = System.nanoTime() - postprocessStartTime

                // Return result with timing information
                DetectionResultWithTiming(
                    result = result,
                    preprocessTime = preprocessTime / 1_000_000, // Convert to milliseconds
                    inferenceTime = inferenceTime / 1_000_000,
                    postprocessTime = postprocessTime / 1_000_000
                )
            } catch (e: Exception) {
                throw DetectionException("Detection failed", e)
            }
        }
    }

    override fun close() {
        executor.close()
        isInitialized = false
    }
}

// CameraFragment.kt - Update the initialization
@AndroidEntryPoint
class CameraFragment : Fragment() {
    // ... other properties ...

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        lifecycleScope.launch {
            try {
                val modelName = arguments?.getString(ARG_MODEL_NAME) 
                    ?: throw IllegalStateException("Model name not found!")
                
                detectionModel = DetectionModelFactory.createModel(modelName)
                    ?: throw IllegalArgumentException("Invalid model name: $modelName")
                
                detector = detectionFactory.createDetector(detectionModel)
                // Initialize the detector
                detector.initialize(detectionModel)
                
                // Start camera after initialization
                withContext(Dispatchers.Main) {
                    if (allPermissionsGranted()) {
                        startCamera()
                    } else {
                        requestPermissionLauncher.launch(Manifest.permission.CAMERA)
                    }
                }
            } catch (e: Exception) {
                Log.e(TAG, "Failed to initialize detector", e)
                withContext(Dispatchers.Main) {
                    Toast.makeText(
                        context,
                        "Failed to initialize detector: ${e.message}",
                        Toast.LENGTH_LONG
                    ).show()
                    activity?.onBackPressed()
                }
            }
        }
    }

    // ... rest of the implementation ...
}
```

Key changes made:

1. Added proper initialization in OnnxExecutor:
- Thread-safe session initialization
- Proper error handling
- Session state checking

2. Enhanced AnomalyDetector:
- Thread-safe initialization
- State validation
- Detailed timing information

3. Updated CameraFragment:
- Asynchronous initialization
- Better error handling
- UI feedback

4. Added specific exceptions:
- ExecutorInitializationException
- ExecutorNotInitializedException
- ExecutorInferenceException

The initialization flow is now:
1. CameraFragment creates detector
2. Detector initializes executor
3. Executor creates ONNX session
4. Camera starts only after successful initialization

To use this:

1. Ensure proper dependencies:
```gradle
dependencies {
    implementation "com.microsoft.onnxruntime:onnxruntime-android:1.12.1"
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4"
}
```

2. Add proguard rules if needed:
```proguard
-keep class com.microsoft.onnxruntime.** { *; }
```

Would you like me to explain any part in more detail or add additional functionality?</antArtifact>



 I'll provide the TensorFlow initialization and executor implementation:

<antArtifact identifier="tensorflow-executor" type="application/vnd.ant.code" language="kotlin" title="TensorFlow Executor Implementation">
```kotlin
// TFLiteExecutor.kt
@Singleton
class TFLiteExecutor @Inject constructor(
    private val context: Context
) : BaseExecutor {
    private var interpreter: Interpreter? = null
    private val initializationLock = Object()

    override suspend fun initialize(modelConfig: ModelConfig) = withContext(Dispatchers.IO) {
        synchronized(initializationLock) {
            if (interpreter == null) {
                try {
                    // Create interpreter options
                    val options = Interpreter.Options().apply {
                        setNumThreads(4) // Adjust based on your needs
                        setUseNNAPI(true) // Enable Neural Network API if available
                    }

                    // Load model file
                    val modelFile = FileUtil.loadMappedFile(
                        context,
                        modelConfig.modelPath
                    )

                    // Create interpreter
                    interpreter = Interpreter(modelFile, options)

                } catch (e: Exception) {
                    throw ExecutorInitializationException(
                        "Failed to initialize TFLite interpreter", 
                        e
                    )
                }
            }
        }
    }

    override suspend fun runInference(preprocessed: Any): Any {
        val tflite = interpreter ?: throw ExecutorNotInitializedException(
            "TFLite interpreter not initialized"
        )

        return withContext(Dispatchers.Default) {
            try {
                when (preprocessed) {
                    is FloatArray -> {
                        // Get input and output shapes from the model
                        val inputShape = tflite.getInputTensor(0).shape()
                        val outputShape = tflite.getOutputTensor(0).shape()

                        // Prepare input buffer
                        val inputBuffer = preprocessed.toDirectBuffer()

                        // Prepare output buffer based on output shape
                        val outputBuffer = ByteBuffer.allocateDirect(
                            outputShape.reduce { acc, i -> acc * i } * 4
                        ).apply {
                            order(ByteOrder.nativeOrder())
                        }

                        // Run inference
                        tflite.run(inputBuffer, outputBuffer)

                        // Convert output buffer to float array
                        val outputArray = FloatArray(outputShape.reduce { acc, i -> acc * i })
                        outputBuffer.rewind()
                        outputBuffer.asFloatBuffer().get(outputArray)

                        outputArray
                    }
                    is ByteBuffer -> {
                        val outputBuffer = ByteBuffer.allocateDirect(
                            tflite.getOutputTensor(0).shape().reduce { acc, i -> acc * i } * 4
                        ).apply {
                            order(ByteOrder.nativeOrder())
                        }

                        tflite.run(preprocessed, outputBuffer)
                        outputBuffer
                    }
                    else -> throw IllegalArgumentException("Unsupported input type")
                }
            } catch (e: Exception) {
                throw ExecutorInferenceException("Failed to run TFLite inference", e)
            }
        }
    }

    override fun close() {
        interpreter?.close()
        interpreter = null
    }

    private fun FloatArray.toDirectBuffer(): ByteBuffer {
        return ByteBuffer.allocateDirect(size * 4).apply {
            order(ByteOrder.nativeOrder())
            asFloatBuffer().put(this@toDirectBuffer)
        }
    }
}

// Extension functions for TensorFlow Lite
object TFLiteUtils {
    fun loadModelFile(context: Context, modelPath: String): MappedByteBuffer {
        return context.assets.openFd(modelPath).use { fileDescriptor ->
            FileInputStream(fileDescriptor.fileDescriptor).use { inputStream ->
                val startOffset = fileDescriptor.startOffset
                val declaredLength = fileDescriptor.declaredLength
                val fileChannel = inputStream.channel
                fileChannel.map(
                    FileChannel.MapMode.READ_ONLY,
                    startOffset,
                    declaredLength
                )
            }
        }
    }

    fun createInterpreterOptions(
        numThreads: Int = 4,
        useNNAPI: Boolean = true,
        useGPU: Boolean = false
    ): Interpreter.Options {
        return Interpreter.Options().apply {
            setNumThreads(numThreads)
            if (useNNAPI) {
                addDelegate(NnApiDelegate())
            }
            if (useGPU) {
                val compatList = CompatibilityList()
                if (compatList.isDelegateSupportedOnThisDevice) {
                    addDelegate(GpuDelegate(compatList.bestOptionsForThisDevice))
                }
            }
        }
    }

    fun getInputOutputDetails(interpreter: Interpreter): Pair<Array<Int>, Array<Int>> {
        val inputShape = interpreter.getInputTensor(0).shape()
        val outputShape = interpreter.getOutputTensor(0).shape()
        return Pair(inputShape, outputShape)
    }
}

// Custom Configuration for TFLite
data class TFLiteConfig(
    val numThreads: Int = 4,
    val useNNAPI: Boolean = true,
    val useGPU: Boolean = false,
    val inputShape: Array<Int>,
    val outputShape: Array<Int>
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false

        other as TFLiteConfig

        if (numThreads != other.numThreads) return false
        if (useNNAPI != other.useNNAPI) return false
        if (useGPU != other.useGPU) return false
        if (!inputShape.contentEquals(other.inputShape)) return false
        if (!outputShape.contentEquals(other.outputShape)) return false

        return true
    }

    override fun hashCode(): Int {
        var result = numThreads
        result = 31 * result + useNNAPI.hashCode()
        result = 31 * result + useGPU.hashCode()
        result = 31 * result + inputShape.contentHashCode()
        result = 31 * result + outputShape.contentHashCode()
        return result
    }
}

// Usage in YoloModel or other TFLite-based models
data class YoloModel(
    override val modelConfig: ModelConfig = ModelConfig(
        metaDataPath = "yolo_metadata.json",
        modelNameWithExtension = "yolo.tflite",
        modelPath = "yolo.tflite",
        inputSize = Pair(416, 416),
        threshold = 0.5f
    ),
    override val executorType: ExecutorType = ExecutorType.TFLITE,
    override val detectionType: DetectionType = DetectionType.OBJECT,
    val tfliteConfig: TFLiteConfig = TFLiteConfig(
        inputShape = arrayOf(1, 416, 416, 3),
        outputShape = arrayOf(1, 80, 4) // Adjust based on your YOLO model
    )
) : DetectionModel
```

Add these dependencies to your build.gradle:
```gradle
dependencies {
    // TensorFlow Lite
    implementation 'org.tensorflow:tensorflow-lite:2.9.0'
    implementation 'org.tensorflow:tensorflow-lite-gpu:2.9.0'
    implementation 'org.tensorflow:tensorflow-lite-support:0.4.2'
    
    // Optional: for NNAPI delegate
    implementation 'org.tensorflow:tensorflow-lite-api:2.9.0'
}
```

And these features to your AndroidManifest.xml:
```xml
<manifest>
    <uses-feature android:name="android.hardware.camera" />
    <uses-feature android:name="android.hardware.camera.autofocus" />
    
    <!-- Optional: for GPU acceleration -->
    <uses-feature android:name="android.hardware.opengles.aep" android:required="false" />
</manifest>
```

Key features:

1. Flexible Input/Output Handling:
- Supports both FloatArray and ByteBuffer inputs
- Automatic buffer management
- Shape inference

2. Performance Optimizations:
- NNAPI support
- GPU delegate support
- Multi-threading
- Direct buffer usage

3. Resource Management:
- Proper interpreter lifecycle
- Memory efficient
- Thread-safe initialization

4. Error Handling:
- Specific exceptions
- Detailed error messages
- Graceful fallbacks

Usage example:
```kotlin
// In your detector
val tfliteExecutor = TFLiteExecutor(context)
tfliteExecutor.initialize(modelConfig)

// Run inference
val result = tfliteExecutor.runInference(preprocessedData)

// Clean up
tfliteExecutor.close()
```

Would you like me to explain any specific part in more detail or add additional functionality?</antArtifact>