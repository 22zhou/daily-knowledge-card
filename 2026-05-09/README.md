# Android相册图像编辑开发

## 💻 每日知识：Android相册图像编辑开发（重点板块）

### 一、核心技术架构：MVVM + Clean Architecture

#### 1.1 架构分层设计
```kotlin
// Presentation Layer - 展示层
class PhotoEditViewModel(
    private val editImageUseCase: EditImageUseCase,
    private val saveImageUseCase: SaveImageUseCase
) : ViewModel() {
    
    private val _editState = MutableStateFlow<EditState>(EditState.Idle)
    val editState: StateFlow<EditState> = _editState.asStateFlow()
    
    fun applyFilter(filterType: FilterType) {
        viewModelScope.launch {
            _editState.value = EditState.Loading
            val result = editImageUseCase(currentBitmap, filterType)
            _editState.value = EditState.Success(result)
        }
    }
}

// Domain Layer - 领域层
interface EditImageUseCase {
    suspend operator fun invoke(bitmap: Bitmap, filterType: FilterType): Bitmap
}

class EditImageUseCaseImpl(
    private val imageProcessor: ImageProcessor
) : EditImageUseCase {
    override suspend fun invoke(bitmap: Bitmap, filterType: FilterType): Bitmap {
        return withContext(Dispatchers.Default) {
            imageProcessor.applyFilter(bitmap, filterType)
        }
    }
}

// Data Layer - 数据层
class ImageProcessorImpl : ImageProcessor {
    override fun applyFilter(bitmap: Bitmap, filterType: FilterType): Bitmap {
        return when (filterType) {
            FilterType.GRAYSCALE -> applyGrayscaleFilter(bitmap)
            FilterType.BRIGHTNESS -> adjustBrightness(bitmap, 50f)
            FilterType.CONTRAST -> adjustContrast(bitmap, 1.2f)
        }
    }
}
```

#### 1.2 依赖注入配置（Hilt）
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object ImageEditModule {
    
    @Provides
    fun provideImageProcessor(): ImageProcessor {
        return ImageProcessorImpl()
    }
    
    @Provides
    fun provideEditImageUseCase(
        imageProcessor: ImageProcessor
    ): EditImageUseCase {
        return EditImageUseCaseImpl(imageProcessor)
    }
}
```

---

### 二、基础图像编辑实现

#### 2.1 图片裁剪
```kotlin
/**
 * 裁剪Bitmap指定区域
 * @param srcBitmap 原图
 * @param x 裁剪起始X坐标
 * @param y 裁剪起始Y坐标
 * @param width 裁剪宽度
 * @param height 裁剪高度
 * @return 裁剪后的Bitmap
 */
fun cropBitmap(
    srcBitmap: Bitmap,
    x: Int,
    y: Int,
    width: Int,
    height: Int
): Bitmap? {
    if (srcBitmap.isRecycled || width <= 0 || height <= 0) {
        return null
    }
    
    // 边界检查
    val safeX = x.coerceIn(0, srcBitmap.width - 1)
    val safeY = y.coerceIn(0, srcBitmap.height - 1)
    val safeWidth = width.coerceAtMost(srcBitmap.width - safeX)
    val safeHeight = height.coerceAtMost(srcBitmap.height - safeY)
    
    return try {
        Bitmap.createBitmap(srcBitmap, safeX, safeY, safeWidth, safeHeight)
    } catch (e: Exception) {
        e.printStackTrace()
        null
    }
}

// 使用示例
fun demoCrop() {
    val original = getOriginalBitmap()
    val cropped = cropBitmap(original, 100, 100, 500, 500)
    imageView.setImageBitmap(cropped)
}
```

#### 2.2 图片缩放
```kotlin
/**
 * 按比例缩放Bitmap
 * @param srcBitmap 原图
 * @param ratio 缩放比例（1.0为原大小，>1放大，<1缩小）
 * @param filter 是否使用滤波（true质量更好但稍慢）
 * @return 缩放后的Bitmap
 */
fun scaleBitmapByRatio(
    srcBitmap: Bitmap,
    ratio: Float,
    filter: Boolean = true
): Bitmap? {
    if (srcBitmap.isRecycled || ratio <= 0) {
        return null
    }
    
    val newWidth = (srcBitmap.width * ratio).toInt()
    val newHeight = (srcBitmap.height * ratio).toInt()
    
    return Bitmap.createScaledBitmap(srcBitmap, newWidth, newHeight, filter)
}

/**
 * 缩放到指定尺寸（保持宽高比）
 */
fun scaleBitmapToSize(
    srcBitmap: Bitmap,
    targetWidth: Int,
    targetHeight: Int
): Bitmap {
    val widthRatio = targetWidth.toFloat() / srcBitmap.width
    val heightRatio = targetHeight.toFloat() / srcBitmap.height
    val ratio = min(widthRatio, heightRatio)
    
    return scaleBitmapByRatio(srcBitmap, ratio) ?: srcBitmap
}
```

#### 2.3 图片旋转
```kotlin
/**
 * 旋转Bitmap指定角度
 * @param srcBitmap 原图
 * @param degrees 旋转角度（正为顺时针，负为逆时针）
 * @return 旋转后的Bitmap
 */
fun rotateBitmap(srcBitmap: Bitmap, degrees: Float): Bitmap {
    if (srcBitmap.isRecycled || degrees % 360 == 0f) {
        return srcBitmap
    }
    
    val matrix = Matrix()
    matrix.postRotate(degrees)
    
    return Bitmap.createBitmap(
        srcBitmap,
        0, 0,
        srcBitmap.width,
        srcBitmap.height,
        matrix,
        true // 抗锯齿
    )
}

/**
 * 读取EXIF信息并修正图片旋转方向
 * 这是相册开发中非常重要的功能！
 */
suspend fun fixBitmapRotation(imagePath: String): Bitmap = withContext(Dispatchers.IO) {
    val exif = ExifInterface(imagePath)
    val orientation = exif.getAttributeInt(
        ExifInterface.TAG_ORIENTATION,
        ExifInterface.ORIENTATION_NORMAL
    )
    
    val rotationDegrees = when (orientation) {
        ExifInterface.ORIENTATION_ROTATE_90 -> 90
        ExifInterface.ORIENTATION_ROTATE_180 -> 180
        ExifInterface.ORIENTATION_ROTATE_270 -> 270
        else -> 0
    }
    
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = false
        inSampleSize = calculateInSampleSize(this, 1080, 1920)
    }
    
    val bitmap = BitmapFactory.decodeFile(imagePath, options)
    
    if (rotationDegrees != 0) {
        rotateBitmap(bitmap, rotationDegrees.toFloat())
    } else {
        bitmap
    }
}

// 计算采样率（重要的内存优化手段）
fun calculateInSampleSize(
    options: BitmapFactory.Options,
    reqWidth: Int,
    reqHeight: Int
): Int {
    val height = options.outHeight
    val width = options.outWidth
    var inSampleSize = 1
    
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        
        while (halfHeight / inSampleSize >= reqHeight 
               && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    
    return inSampleSize
}
```

---

### 三、进阶：滤镜效果实现

#### 3.1 黑白滤镜（像素级处理）
```kotlin
/**
 * 黑白滤镜（灰度化）- 优化版：批量处理像素
 * 依据ITU-R BT.601标准：Y = 0.299R + 0.587G + 0.114B
 */
fun applyGrayscaleFilter(srcBitmap: Bitmap): Bitmap {
    val width = srcBitmap.width
    val height = srcBitmap.height
    val totalPixels = width * height
    
    // 批量获取像素数组（比逐个getPixel快10倍以上）
    val pixels = IntArray(totalPixels)
    srcBitmap.getPixels(pixels, 0, width, 0, 0, width, height)
    
    // 遍历处理每个像素
    for (i in pixels.indices) {
        val pixel = pixels[i]
        val r = Color.red(pixel)
        val g = Color.green(pixel)
        val b = Color.blue(pixel)
        val a = Color.alpha(pixel)
        
        // 加权平均计算灰度值
        val gray = (0.299 * r + 0.587 * g + 0.114 * b).toInt()
        
        // 设置新像素
        pixels[i] = Color.argb(a, gray, gray, gray)
    }
    
    // 创建结果Bitmap
    val result = Bitmap.createBitmap(width, height, srcBitmap.config)
    result.setPixels(pixels, 0, width, 0, 0, width, height)
    
    return result
}
```

#### 3.2 使用ColorMatrix实现高性能滤镜
```kotlin
/**
 * 使用ColorMatrix实现亮度调整
 * 性能最优，底层由Skia引擎执行
 */
fun adjustBrightnessWithMatrix(
    srcBitmap: Bitmap,
    brightness: Float // -255到255，0为不变
): Bitmap {
    val result = Bitmap.createBitmap(
        srcBitmap.width,
        srcBitmap.height,
        Bitmap.Config.ARGB_8888
    )
    
    val canvas = Canvas(result)
    val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    // 亮度调整矩阵
    val brightnessMatrix = floatArrayOf(
        1f, 0f, 0f, 0f, brightness,  // R
        0f, 1f, 0f, 0f, brightness,  // G
        0f, 0f, 1f, 0f, brightness,  // B
        0f, 0f, 0f, 1f, 0f           // Alpha
    )
    
    val colorMatrix = ColorMatrix(brightnessMatrix)
    paint.colorFilter = ColorMatrixColorFilter(colorMatrix)
    canvas.drawBitmap(srcBitmap, 0f, 0f, paint)
    
    return result
}

/**
 * 对比度调整
 * @param contrast 对比度值（1.0为不变，>1增强，<1减弱）
 */
fun adjustContrastWithMatrix(
    srcBitmap: Bitmap,
    contrast: Float
): Bitmap {
    val result = Bitmap.createBitmap(
        srcBitmap.width,
        srcBitmap.height,
        Bitmap.Config.ARGB_8888
    )
    
    val canvas = Canvas(result)
    val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    // 对比度计算公式：newValue = (value - 128) * contrast + 128
    val translate = ((1 - contrast) * 128).toFloat()
    
    val contrastMatrix = floatArrayOf(
        contrast, 0f, 0f, 0f, translate,
        0f, contrast, 0f, 0f, translate,
        0f, 0f, contrast, 0f, translate,
        0f, 0f, 0f, 1f, 0f
    )
    
    val colorMatrix = ColorMatrix(contrastMatrix)
    paint.colorFilter = ColorMatrixColorFilter(colorMatrix)
    canvas.drawBitmap(srcBitmap, 0f, 0f, paint)
    
    return result
}

/**
 * 复合滤镜：先调整饱和度，再调整对比度，最后调整亮度
 * 注意：矩阵乘法顺序很重要！
 */
fun applyCombinedFilter(
    srcBitmap: Bitmap,
    saturation: Float,
    contrast: Float,
    brightness: Float
): Bitmap {
    val result = Bitmap.createBitmap(
        srcBitmap.width,
        srcBitmap.height,
        Bitmap.Config.ARGB_8888
    )
    
    val canvas = Canvas(result)
    val paint = Paint(Paint.ANTI_ALIAS_FLAG)
    
    // 创建复合矩阵
    val combinedMatrix = ColorMatrix()
    
    // 1. 先调整饱和度
    combinedMatrix.setSaturation(saturation)
    
    // 2. 再调整对比度
    val contrastMatrix = ColorMatrix()
    val translate = ((1 - contrast) * 128).toFloat()
    contrastMatrix.setScale(contrast, contrast, contrast, 1f)
    contrastMatrix.postTranslate(translate, translate, translate, 0f)
    combinedMatrix.postConcat(contrastMatrix)
    
    // 3. 最后调整亮度
    val brightnessMatrix = ColorMatrix()
    brightnessMatrix.set(floatArrayOf(
        1f, 0f, 0f, 0f, brightness,
        0f, 1f, 0f, 0f, brightness,
        0f, 0f, 1f, 0f, brightness,
        0f, 0f, 0f, 1f, 0f
    ))
    combinedMatrix.postConcat(brightnessMatrix)
    
    paint.colorFilter = ColorMatrixColorFilter(combinedMatrix)
    canvas.drawBitmap(srcBitmap, 0f, 0f, paint)
    
    return result
}
```

---

### 四、性能优化最佳实践

#### 4.1 内存管理
```kotlin
/**
 * 安全的Bitmap操作包装类
 * 使用Kotlin扩展函数确保资源正确释放
 */
inline fun <T> Bitmap.use(block: (Bitmap) -> T): T {
    try {
        return block(this)
    } finally {
        if (!isRecycled) {
            recycle()
        }
    }
}

// 使用示例
fun processImageSafety() {
    loadOriginalBitmap().use { original ->
        val cropped = cropBitmap(original, 100, 100, 500, 500)
        cropped?.use { croppedBitmap ->
            val filtered = applyGrayscaleFilter(croppedBitmap)
            filtered.use { result ->
                saveToFile(result)
            }
        }
    }
}

/**
 * 使用BitmapPool复用Bitmap（Glide内置）
 */
class BitmapPoolManager(private val bitmapPool: BitmapPool) {
    
    fun getReusableBitmap(width: Int, height: Int, config: Bitmap.Config): Bitmap {
        return bitmapPool.get(width, height, config)
    }
    
    fun putBitmap(bitmap: Bitmap) {
        bitmapPool.put(bitmap)
    }
}
```

#### 4.2 后台线程处理
```kotlin
/**
 * 使用协程在后台线程执行图像处理
 */
class ImageEditWorker(
    private val coroutineScope: CoroutineScope
) {
    sealed class Result {
        data class Success(val bitmap: Bitmap) : Result()
        data class Progress(val percent: Int) : Result()
        data class Error(val message: String) : Result()
    }
    
    fun processImageAsync(
        original: Bitmap,
        operations: List<ImageOperation>
    ): Flow<Result> = flow {
        var currentBitmap = original
        
        operations.forEachIndexed { index, operation ->
            emit(Result.Progress((index * 100) / operations.size))
            
            currentBitmap = withContext(Dispatchers.Default) {
                when (operation) {
                    is ImageOperation.Crop -> cropBitmap(currentBitmap, operation.rect)
                    is ImageOperation.Rotate -> rotateBitmap(currentBitmap, operation.degrees)
                    is ImageOperation.Filter -> applyFilter(currentBitmap, operation.filterType)
                } ?: currentBitmap
            }
        }
        
        emit(Result.Progress(100))
        emit(Result.Success(currentBitmap))
    }.flowOn(Dispatchers.Default)
}

// 在ViewModel中使用
fun startImageProcessing() {
    viewModelScope.launch {
        imageEditWorker.processImageAsync(originalBitmap, operations)
            .collect { result ->
                when (result) {
                    is ImageEditWorker.Result.Progress -> {
                        updateProgress(result.percent)
                    }
                    is ImageEditWorker.Result.Success -> {
                        showResult(result.bitmap)
                    }
                    is ImageEditWorker.Result.Error -> {
                        showError(result.message)
                    }
                }
            }
    }
}
```

#### 4.3 大图分段处理
```kotlin
/**
 * 超大图片分段处理，避免一次性加载导致OOM
 */
suspend fun processLargeImageInChunks(
    imagePath: String,
    chunkSize: Int = 256
): Bitmap = withContext(Dispatchers.Default) {
    val options = BitmapFactory.Options()
    options.inJustDecodeBounds = true
    BitmapFactory.decodeFile(imagePath, options)
    
    val fullWidth = options.outWidth
    val fullHeight = options.outHeight
    
    // 创建结果Bitmap
    val result = Bitmap.createBitmap(fullWidth, fullHeight, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(result)
    
    // 分段加载和处理
    var y = 0
    while (y < fullHeight) {
        val height = min(chunkSize, fullHeight - y)
        
        val regionOptions = BitmapFactory.Options().apply {
            inJustDecodeBounds = false
            inSampleSize = 1
        }
        
        // 加载区域（需要Android 10+支持）
        val region = Rect(0, y, fullWidth, y + height)
        regionOptions.inPreferredConfig = Bitmap.Config.ARGB_8888
        
        val regionDecoder = ImageDecoder.createSource(File(imagePath))
        val chunk = ImageDecoder.decodeBitmap(regionDecoder) { decoder, info, source ->
            decoder.setCrop(region)
        }
        
        // 处理当前块
        val processedChunk = applyGrayscaleFilter(chunk)
        
        // 绘制到结果
        canvas.drawBitmap(processedChunk, 0f, y.toFloat(), null)
        
        processedChunk.recycle()
        chunk.recycle()
        
        y += chunkSize
    }
    
    result
}
```

---

### 五、完整编辑功能实现

#### 5.1 编辑操作历史（撤销/重做）
```kotlin
/**
 * 编辑历史管理器，支持撤销和重做
 */
class EditHistoryManager(private val maxHistorySize: Int = 20) {
    private val historyStack = mutableListOf<Bitmap>()
    private var currentIndex = -1
    
    /**
     * 保存当前状态到历史
     */
    fun saveState(bitmap: Bitmap) {
        // 清除当前位置之后的历史（重做栈）
        if (currentIndex < historyStack.lastIndex) {
            while (historyStack.size > currentIndex + 1) {
                historyStack.removeLast().recycle()
            }
        }
        
        // 添加新状态
        historyStack.add(bitmap.copy(bitmap.config, true))
        currentIndex++
        
        // 限制历史大小
        if (historyStack.size > maxHistorySize) {
            historyStack.removeFirst().recycle()
            currentIndex--
        }
    }
    
    /**
     * 撤销
     */
    fun undo(): Bitmap? {
        if (currentIndex > 0) {
            currentIndex--
            return historyStack[currentIndex]
        }
        return null
    }
    
    /**
     * 重做
     */
    fun redo(): Bitmap? {
        if (currentIndex < historyStack.lastIndex) {
            currentIndex++
            return historyStack[currentIndex]
        }
        return null
    }
    
    /**
     * 是否可以撤销
     */
    fun canUndo() = currentIndex > 0
    
    /**
     * 是否可以重做
     */
    fun canRedo() = currentIndex < historyStack.lastIndex
    
    /**
     * 清理所有历史
     */
    fun clear() {
        historyStack.forEach { it.recycle() }
        historyStack.clear()
        currentIndex = -1
    }
}
```

#### 5.2 图片保存功能
```kotlin
/**
 * 保存编辑后的图片到相册
 */
suspend fun saveEditedImageToGallery(
    context: Context,
    bitmap: Bitmap,
    displayName: String,
    quality: Int = 90
): Uri? = withContext(Dispatchers.IO) {
    val contentValues = ContentValues().apply {
        put(MediaStore.Images.Media.DISPLAY_NAME, displayName)
        put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
        put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
        put(MediaStore.Images.Media.IS_PENDING, 1)
    }
    
    val resolver = context.contentResolver
    val uri = resolver.insert(
        MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
        contentValues
    ) ?: return@withContext null
    
    try {
        resolver.openOutputStream(uri).use { outputStream ->
            if (!bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)) {
                throw IOException("Failed to compress bitmap")
            }
        }
        
        // 更新状态为已完成
        contentValues.clear()
        contentValues.put(MediaStore.Images.Media.IS_PENDING, 0)
        resolver.update(uri, contentValues, null, null)
        
        uri
    } catch (e: Exception) {
        resolver.delete(uri, null, null)
        null
    }
}
```

---
