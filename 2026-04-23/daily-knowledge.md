## 🔧 每日知识

# Android相册图像编辑开发完全指南

> 本指南深入讲解Android平台上相册图像编辑功能的开发，涵盖权限处理、图片加载、图像编辑核心功能（裁剪、旋转、滤镜、涂鸦等）以及性能优化。适合需要开发图片处理相关功能的Android开发者。

---

## 一、权限处理与系统集成

### 1.1 权限申请

Android 6.0以后需要动态申请权限，图片编辑功能主要涉及以下权限：

```kotlin
/**
 * Android图片编辑应用权限管理器
 * 
 * 核心权限说明：
 * - READ_EXTERNAL_STORAGE: 读取外部存储（已废弃，使用READ_MEDIA_IMAGES）
 * - READ_MEDIA_IMAGES: Android 13+读取图片
 * - WRITE_EXTERNAL_STORAGE: 写入外部存储（Android 10+ scoped storage）
 * 
 * 注意：从Android 10开始，系统引入Scoped Storage，
 * 大多数情况下不需要WRITE_EXTERNAL_STORAGE权限
 */
class PermissionManager(private val activity: Activity) {

    // Android 13 (API 33) 及以上使用的权限
    private val mediaImagePermission = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        Manifest.permission.READ_MEDIA_IMAGES
    } else {
        // Android 13以下使用旧权限
        Manifest.permission.READ_EXTERNAL_STORAGE
    }

    /**
     * 检查权限是否已授予
     * 
     * @return true if permission is granted, false otherwise
     */
    fun checkPermission(): Boolean {
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            // Android 13+只需要READ_MEDIA_IMAGES
            activity.checkSelfPermission(Manifest.permission.READ_MEDIA_IMAGES) == 
                PackageManager.PERMISSION_GRANTED
        } else {
            // Android 12及以下需要READ_EXTERNAL_STORAGE
            activity.checkSelfPermission(Manifest.permission.READ_EXTERNAL_STORAGE) == 
                PackageManager.PERMISSION_GRANTED
        }
    }

    /**
     * 请求读取图片权限
     * 
     * 使用示例：
     * ```kotlin
     * val permissionManager = PermissionManager(this)
     * if (!permissionManager.checkPermission()) {
     *     permissionManager.requestMediaPermission(REQUEST_CODE_MEDIA)
     * }
     * ```
     */
    fun requestMediaPermission(requestCode: Int) {
        ActivityCompat.requestPermissions(
            activity,
            arrayOf(
                // Android 13+只需要媒体权限
                if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                    Manifest.permission.READ_MEDIA_IMAGES
                } else {
                    Manifest.permission.READ_EXTERNAL_STORAGE
                }
            ),
            requestCode
        )
    }

    /**
     * 处理权限请求结果
     * 
     * 应在Activity的onRequestPermissionsResult中调用
     */
    fun handlePermissionResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray,
        onGranted: () -> Unit,
        onDenied: () -> Unit
    ) {
        if (requestCode == REQUEST_CODE_MEDIA && 
            grantResults.isNotEmpty() && 
            grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            onGranted()
        } else {
            onDenied()
        }
    }

    companion object {
        const val REQUEST_CODE_MEDIA = 1001
    }
}
```

### 1.2 从相册选择图片

使用系统PhotoPicker（Android 13+）或ACTION_OPEN_DOCUMENT获取图片：

```kotlin
/**
 * 图片选择器
 * 
 * 使用PhotoPicker（推荐）：
 * - Android 13+原生支持
 * - 统一的图片选择界面
 * - 无需申请存储权限
 * - 支持多选和单选
 */
class ImagePicker(private val activity: Activity) {

    // PhotoPicker需要ActivityResultLauncher
    private var pickImageLauncher: ActivityResultLauncher<PickVisualMediaRequest>? = null
    private var openDocumentLauncher: ActivityResultLauncher<Uri>? = null
    
    // 回调
    private var onImageSelected: ((Uri) -> Unit)? = null

    /**
     * 初始化launcher
     * 
     * 应在Activity/Fragment的onCreate中调用
     */
    fun registerLaunchers() {
        // 注册PhotoPicker（Android 13+）
        pickImageLauncher = activity.registerForActivityResult(
            ActivityResultContracts.PickVisualMedia()
        ) { uri: Uri? ->
            uri?.let { handleImageSelected(it) }
        }

        // 注册Document访问（兼容旧版本）
        openDocumentLauncher = activity.registerForActivityResult(
            ActivityResultContracts.OpenDocument()
        ) { uri: Uri? ->
            uri?.let { 
                // 获取持久化权限（重要！）
                getPersistedPermission(it)
                handleImageSelected(it)
            }
        }
    }

    /**
     * 打开PhotoPicker（Android 13+推荐）
     * 
     * @param onImageSelected 选中图片的回调
     */
    fun pickImageWithPhotoPicker(onImageSelected: (Uri) -> Unit) {
        this.onImageSelected = onImageSelected
        
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            // 使用PhotoPicker
            pickImageLauncher?.launch(
                PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
            )
        } else {
            // 回退到OpenDocument
            openDocumentLauncher?.launch(arrayOf("image/*"))
        }
    }

    /**
     * 获取持久化权限
     * 
     * 重要：对于从相册选择的图片，需要获取持久化读取权限，
     * 否则可能在后续访问时失去权限
     * 
     * @param uri 图片Uri
     */
    private fun getPersistedPermission(uri: Uri) {
        try {
            // FLAG_GRANT_READ_URI_PERMISSION - 授予读取权限
            // FLAG_GRANT_WRITE_URI_PERMISSION - 授予写入权限（通常不需要）
            activity.contentResolver.takePersistableUriPermission(
                uri,
                Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
        } catch (e: SecurityException) {
            Log.e(TAG, "Failed to persist permission", e)
        }
    }

    /**
     * 处理图片选择结果
     */
    private fun handleImageSelected(uri: Uri) {
        Log.d(TAG, "Image selected: $uri")
        
        // 验证是否为图片类型
        val mimeType = activity.contentResolver.getType(uri)
        if (mimeType?.startsWith("image/") == true) {
            onImageSelected?.invoke(uri)
        } else {
            Log.w(TAG, "Selected file is not an image: $mimeType")
        }
    }

    companion object {
        private const val TAG = "ImagePicker"
    }
}
```

---

## 二、图片加载与显示

### 2.1 使用Coil加载图片

Coil是现代Android开发推荐使用的图片加载库，Kotlin优先，协程支持好：

```kotlin
/**
 * Coil图片加载工具类
 * 
 * 为什么要用Coil而不是Glide/Picasso？
 * 1. Kotlin优先，协程支持
 * 2. 轻量级（核心约1500行代码）
 * 3. 自动管理与生命周期绑定
 * 4. 支持ImagePool和MemoryCache
 */
object ImageLoader {

    // 全局ImageLoader实例
    private lateinit var imageLoader: ImageLoader

    /**
     * 初始化ImageLoader
     * 
     * 应在Application的onCreate中调用
     */
    fun init(context: Context) {
        imageLoader = ImageLoader.Builder(context)
            // 设置内存缓存大小为可用内存的25%
            .memoryCache {
                MemoryCache.Builder(context)
                    .maxSizePercent(0.25)
                    .build()
            }
            // 设置磁盘缓存大小为250MB
            .diskCache {
                DiskCache.Builder()
                    .directory(context.cacheDir.resolve("image_cache"))
                    .maxSizeBytes(250L * 1024 * 1024) // 250MB
                    .build()
            }
            // 跨进程共享磁盘缓存
            .diskCachePolicy(CachePolicy.ENABLED)
            // 内存缓存策略
            .memoryCachePolicy(CachePolicy.ENABLED)
            // 允许采样（适合大图）
            .allowRgb565(true)
            // 组件映射（处理特殊格式）
            .components {
                add(SvgDecoder.Factory()) // 支持SVG
                add(GifDecoder.Factory()) // 支持GIF
            }
            // 跨上下文自动取消
            .respectCacheHeaders(false)
            .build()
    }

    /**
     * 加载图片到ImageView
     * 
     * @param imageView 目标ImageView
     * @param uri 图片Uri（可以是网络、本地、ContentProvider）
     * @param placeholderResId 占位图资源ID
     * @param errorResId 错误图资源ID
     * 
     * 使用示例：
     * ```kotlin
     * ImageLoader.loadImage(
     *     imageView = binding.imageView,
     *     uri = imageUri,
     *     placeholderResId = R.drawable.placeholder,
     *     errorResId = R.drawable.error
     * )
     * ```
     */
    fun loadImage(
        imageView: ImageView,
        uri: Uri,
        placeholderResId: Int? = null,
        errorResId: Int? = null
    ) {
        val request = imageView.context.let { ctx ->
            ImageRequest.Builder(ctx)
                .data(uri)
                .target(imageView)
                .apply {
                    // 跨淡入动画
                    transitions {
                        CrossfadeTransition(300)
                    }
                }
                .apply {
                    placeholderResId?.let {
                        placeholder(it)
                    }
                }
                .apply {
                    errorResId?.let {
                        error(it)
                    }
                }
                .build()
        }
        
        imageLoader.enqueue(request)
    }

    /**
     * 异步加载图片并返回Bitmap
     * 
     * 适用于需要获取Bitmap进行编辑的场景
     * 
     * @param context Context
     * @param uri 图片Uri
     * @param size 目标尺寸（用于内存优化）
     * @return 加载后的Bitmap
     */
    suspend fun loadBitmap(
        context: Context,
        uri: Uri,
        size: Size = Size.ORIGINAL
    ): Bitmap? {
        return withContext(Dispatchers.IO) {
            try {
                val request = ImageRequest.Builder(context)
                    .data(uri)
                    .size(size)
                    .allowHardware(false) // 需要像素数据时必须设为false
                    .build()
                
                val result = imageLoader.execute(request)
                (result.drawable as? BitmapDrawable)?.bitmap
            } catch (e: Exception) {
                Log.e(TAG, "Failed to load bitmap", e)
                null
            }
        }
    }

    /**
     * 清理缓存
     */
    fun clearCache() {
        imageLoader.memoryCache?.clear()
    }

    companion object {
        private const val TAG = "ImageLoader"
    }
}
```

### 2.2 BitmapFactory加载选项

使用BitmapFactory加载本地图片并进行基本处理：

```kotlin
/**
 * Bitmap处理工具类
 * 
 * Bitmap是Android中表示位图的核心类，
 * 开发图片编辑功能时需要熟练掌握Bitmap的操作
 */
object BitmapUtils {

    /**
     * 从Uri加载合适尺寸的Bitmap
     * 
     * 为什么要采样压缩？
     * - 避免OOM（Out Of Memory）
     * - 节省内存占用
     * - 加快处理速度
     * 
     * @param context Context
     * @param uri 图片Uri
     * @param maxWidth 最大宽度（像素）
     * @param maxHeight 最大高度（像素）
     * @return Bitmap或null
     */
    fun loadBitmapFromUri(
        context: Context,
        uri: Uri,
        maxWidth: Int = 1920,
        maxHeight: Int = 1080
    ): Bitmap? {
        return try {
            // 第一次解析：获取图片尺寸（不加载像素数据）
            val options = BitmapFactory.Options().apply {
                inJustDecodeBounds = true // 设为true，只解析尺寸不加载像素
            }
            
            context.contentResolver.openInputStream(uri)?.use { inputStream ->
                BitmapFactory.decodeStream(inputStream, null, options)
            }

            // 计算采样率
            options.inSampleSize = calculateInSampleSize(
                options.outWidth, options.outHeight,
                maxWidth, maxHeight
            )
            
            // 第二次解析：加载采样后的Bitmap
            options.inJustDecodeBounds = false
            // 使用RGB_565减少内存占用（每个像素2字节 vs ARGB_8888的4字节）
            options.inPreferredConfig = Bitmap.Config.RGB_565
            
            context.contentResolver.openInputStream(uri)?.use { inputStream ->
                BitmapFactory.decodeStream(inputStream, null, options)
            }
        } catch (e: Exception) {
            Log.e(TAG, "Failed to load bitmap", e)
            null
        }
    }

    /**
     * 计算采样率
     * 
     * 采样率的平方影响最终Bitmap的内存占用：
     * - inSampleSize = 2, 内存占用为原图的1/4
     * - inSampleSize = 4, 内存占用为原图的1/16
     * 
     * @param srcWidth 原图宽度
     * @param srcHeight 原图高度
     * @param reqWidth 目标宽度
     * @param reqHeight 目标高度
     * @return 采样率（2的幂次方）
     */
    fun calculateInSampleSize(
        srcWidth: Int,
        srcHeight: Int,
        reqWidth: Int,
        reqHeight: Int
    ): Int {
        var inSampleSize = 1
        
        if (srcHeight > reqHeight || srcWidth > reqWidth) {
            // 计算宽高的缩放比例
            val halfHeight = srcHeight / 2
            val halfWidth = srcWidth / 2
            
            // 循环计算最大采样率（取2的幂次方）
            // 使得最终尺寸刚好小于或等于目标尺寸
            while ((halfHeight / inSampleSize) >= reqHeight &&
                   (halfWidth / inSampleSize) >= reqWidth) {
                inSampleSize *= 2
            }
        }
        
        return inSampleSize
    }

    /**
     * 创建可编辑的Bitmap副本
     * 
     * 为什么需要副本？
     * - BitmapFactory返回的Bitmap可能被系统复用
     * - 编辑时需要确保Bitmap是可变的
     * - 避免原图被意外修改
     * 
     * @param source 源Bitmap
     * @return 可变的Bitmap副本
     */
    fun createMutableCopy(source: Bitmap): Bitmap {
        // 创建相同尺寸的Mutable Bitmap
        return Bitmap.createBitmap(
            source.width,
            source.height,
            // 使用ARGB_8888保证编辑质量
            // 如果内存紧张可用RGB_565（但会丢失透明度）
            Bitmap.Config.ARGB_8888
        ).also { mutable ->
            // 将原图内容复制到新Bitmap
            Canvas(mutable).drawBitmap(source, 0f, 0f, null)
        }
    }

    /**
     * 旋转Bitmap
     * 
     * @param source 源Bitmap
     * @param degrees 旋转角度（正数顺时针，负数逆时针）
     * @return 旋转后的Bitmap
     */
    fun rotateBitmap(source: Bitmap, degrees: Float): Bitmap {
        if (degrees == 0f) return source
        
        val matrix = Matrix().apply {
            postRotate(degrees)
        }
        
        val rotatedWidth = source.width
        val rotatedHeight = source.height
        
        return Bitmap.createBitmap(
            source,
            0, 0,
            rotatedWidth, rotatedHeight,
            matrix,
            true // 使用过滤（抗锯齿）
        )
    }

    /**
     * 缩放Bitmap到指定尺寸
     * 
     * @param source 源Bitmap
     * @param newWidth 目标宽度
     * @param newHeight 目标高度
     * @return 缩放后的Bitmap
     */
    fun scaleBitmap(source: Bitmap, newWidth: Int, newHeight: Int): Bitmap {
        return Bitmap.createScaledBitmap(source, newWidth, newHeight, true)
    }

    /**
     * 保存Bitmap到文件
     * 
     * @param bitmap 要保存的Bitmap
     * @param outputStream 输出流
     * @param format 图片格式
     * @param quality 质量（0-100）
     * @return 是否保存成功
     */
    fun saveBitmap(
        bitmap: Bitmap,
        outputStream: OutputStream,
        format: Bitmap.CompressFormat = Bitmap.CompressFormat.JPEG,
        quality: Int = 90
    ): Boolean {
        return try {
            bitmap.compress(format, quality, outputStream)
            outputStream.flush()
            true
        } catch (e: Exception) {
            Log.e(TAG, "Failed to save bitmap", e)
            false
        }
    }

    /**
     * 计算Bitmap占用的内存大小
     * 
     * 用于评估内存使用和OOM风险
     * 
     * @param bitmap Bitmap
     * @return 内存大小（字节）
     */
    fun getBitmapSize(bitmap: Bitmap): Long {
        // ARGB_8888: 每个像素4字节
        // RGB_565: 每个像素2字节
        val bytesPerPixel = when (bitmap.config) {
            Bitmap.Config.ARGB_8888 -> 4
            Bitmap.Config.RGB_565 -> 2
            Bitmap.Config.ALPHA_8 -> 1
            else -> 4
        }
        return bitmap.width.toLong() * bitmap.height * bytesPerPixel
    }

    /**
     * 计算内存占用（MB）
     */
    fun getBitmapSizeMB(bitmap: Bitmap): Float {
        return getBitmapSize(bitmap) / (1024f * 1024f)
    }

    companion object {
        private const val TAG = "BitmapUtils"
    }
}
```

---

## 三、Canvas绘制基础

### 3.1 自定义ImageEditorView

核心绘制组件，用于实现各种编辑功能：

```kotlin
/**
 * 图片编辑器自定义View
 * 
 * 职责：
 * 1. 显示原图和编辑层
 * 2. 处理手势（缩放、平移、旋转）
 * 3. 接收编辑操作并实时预览
 * 
 * 架构说明：
 * - 使用双缓冲Canvas避免闪烁
 * - Matrix实现图片变换（缩放、平移、旋转）
 * - 分层绘制：原图层 + 编辑图层
 */
class ImageEditorView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    // ============ 核心属性 ============
    
    // 原图Bitmap
    private var originalBitmap: Bitmap? = null
    
    // 用于绘制的可编辑Bitmap
    private var workingBitmap: Bitmap? = null
    
    // 绘制画布（关联到workingBitmap）
    private var drawingCanvas: Canvas? = null
    
    // 图片变换矩阵（控制缩放、平移、旋转）
    private val imageMatrix = Matrix()
    
    // 图片变换的反向矩阵（用于触摸坐标转换）
    private val inverseMatrix = Matrix()
    
    // 缩放和平移的边界限制
    private val matrixValues = FloatArray(9)
    
    // 变换参数
    private var scaleFactor = 1f      // 缩放比例
    private var translateX = 0f       // X方向平移
    private var translateY = 0f     // Y方向平移
    private var rotation = 0f        // 旋转角度
    
    // 最大/最小缩放比例
    private val minScale = 0.5f
    private val maxScale = 5f
    
    // ============ 手势处理 ============
    
    // 缩放手势
    private var scaleGestureDetector: ScaleGestureDetector? = null
    
    // 移动手势
    private var gestureDetector: GestureDetector? = null
    
    // 触摸状态
    private var lastTouchX = 0f
    private var lastTouchY = 0f
    private var isScaling = false
    
    // ============ 绘制参数 ============
    
    // 绘制画笔
    private val drawPaint = Paint().apply {
        isAntiAlias = true           // 抗锯齿
        isDither = true              // 防抖动
        style = Paint.Style.STROKE   // 只画边框（空心效果）
        strokeJoin = Paint.Join.ROUND // 圆角连接
        strokeCap = Paint.Cap.ROUND   // 圆角线帽
        strokeWidth = 8f             // 默认线宽
        color = Color.RED            // 默认颜色
    }

    // 图片绘制 Paint
    private val imagePaint = Paint().apply {
        isAntiAlias = true
        filterBitmap = true         // 启用双线性过滤（缩放时平滑）
    }

    // ============ 编辑状态 ============
    
    // 当前编辑模式
    enum class EditMode {
        NONE,           // 无编辑
        DRAW,           // 涂鸦
        TEXT,           // 添加文字
        CROP,           // 裁剪
        ROTATE          // 旋转
    }
    
    private var currentMode = EditMode.NONE
    
    // 涂鸦路径（用于撤销功能）
    private val drawPath = Path()
    private val pathStack = mutableListOf<PathData>() // 支持撤销
    
    // 文字列表
    private val textList = mutableListOf<TextItem>()

    /**
     * Path数据类（用于撤销功能）
     */
    data class PathData(
        val path: Path,
        val paint: Paint
    )

    /**
     * 文字项
     */
    data class TextItem(
        val text: String,
        val x: Float,
        val y: Float,
        val paint: Paint
    )

    init {
        // 初始化手势检测器
        scaleGestureDetector = ScaleGestureDetector(context, ScaleListener())
        gestureDetector = GestureDetector(context, GestureListener())
    }

    /**
     * 设置要编辑的图片
     * 
     * @param bitmap 要编辑的Bitmap
     */
    fun setImage(bitmap: Bitmap) {
        // 创建工作副本
        originalBitmap = bitmap
        
        // 创建可编辑的副本
        workingBitmap = BitmapUtils.createMutableCopy(bitmap)
        
        // 创建关联的Canvas
        drawingCanvas = Canvas(workingBitmap!!)
        
        // 重置变换
        resetTransform()
        
        // 重置编辑状态
        resetEditState()
        
        // 请求重绘
        invalidate()
    }

    /**
     * 重置所有变换
     */
    fun resetTransform() {
        imageMatrix.reset()
        scaleFactor = 1f
        translateX = 0f
        translateY = 0f
        rotation = 0f
        constrainMatrix()
        invalidate()
    }

    /**
     * 重置编辑状态（路径、文字等）
     */
    fun resetEditState() {
        drawPath.reset()
        pathStack.clear()
        textList.clear()
        currentMode = EditMode.NONE
    }

    /**
     * 设置编辑模式
     */
    fun setEditMode(mode: EditMode) {
        currentMode = mode
        // 设置默认光标或工具样式
        when (mode) {
            EditMode.DRAW -> {
                drawPaint.color = Color.RED
                drawPaint.strokeWidth = 8f
            }
            else -> {}
        }
    }

    /**
     * 设置画笔颜色
     */
    fun setDrawColor(color: Int) {
        drawPaint.color = color
    }

    /**
     * 设置画笔宽度
     */
    fun setDrawStrokeWidth(width: Float) {
        drawPaint.strokeWidth = width
    }

    /**
     * 开始绘制（触摸开始）
     */
    override fun onTouchEvent(event: MotionEvent): Boolean {
        // 缩放手势优先
        scaleGestureDetector?.onTouchEvent(event)
        
        // 手势检测（点击、双击等）
        gestureDetector?.onTouchEvent(event)
        
        val x = event.x
        val y = event.y
        
        when (currentMode) {
            EditMode.DRAW -> {
                // 将屏幕坐标转换为图片坐标
                updateInverseMatrix()
                val points = floatArrayOf(x, y)
                inverseMatrix.mapPoints(points)
                val imageX = points[0]
                val imageY = points[1]
                
                when (event.action) {
                    MotionEvent.ACTION_DOWN -> {
                        drawPath.moveTo(imageX, imageY)
                    }
                    MotionEvent.ACTION_MOVE -> {
                        drawPath.lineTo(imageX, imageY)
                        // 实时绘制到工作Bitmap
                        drawingCanvas?.drawPath(drawPath, drawPaint)
                        invalidate()
                    }
                    MotionEvent.ACTION_UP -> {
                        // 保存路径用于撤销
                        val pathCopy = Path(drawPath)
                        val paintCopy = Paint(drawPaint)
                        pathStack.add(PathData(pathCopy, paintCopy))
                        drawPath.reset()
                    }
                }
            }
            EditMode.NONE -> {
                // 仅缩放和平移
                when (event.action) {
                    MotionEvent.ACTION_DOWN -> {
                        lastTouchX = x
                        lastTouchY = y
                    }
                    MotionEvent.ACTION_MOVE -> {
                        if (!isScaling) {
                            translateX += x - lastTouchX
                            translateY += y - lastTouchY
                            constrainMatrix()
                            invalidate()
                        }
                        lastTouchX = x
                        lastTouchY = y
                    }
                }
            }
            else -> {}
        }
        
        return true
    }

    /**
     * 绘制内容
     * 
     * 绘制顺序：
     * 1. 背景色
     * 2. 工作Bitmap（包含编辑内容）
     * 3. 实时绘制路径（未保存的部分）
     */
    override fun onDraw(canvas: Canvas) {
        super.onDraw(canvas)
        
        // 填充背景
        canvas.drawColor(Color.DKGRAY)
        
        workingBitmap?.let { bitmap ->
            // 保存画布状态
            canvas.save()
            
            // 应用变换
            canvas.concat(imageMatrix)
            
            // 绘制图片
            canvas.drawBitmap(bitmap, 0f, 0f, imagePaint)
            
            // 恢复画布状态
            canvas.restore()
        }
    }

    /**
     * 更新反向矩阵（用于触摸坐标转换）
     */
    private fun updateInverseMatrix() {
        imageMatrix.invert(inverseMatrix)
    }

    /**
     * 限制变换在合理范围内
     */
    private fun constrainMatrix() {
        // 获取当前缩放和平移值
        imageMatrix.getValues(matrixValues)
        val currentScale = matrixValues[Matrix.MSCALE_X]
        
        // 限制缩放范围
        if (currentScale < minScale) {
            val scale = minScale / currentScale
            imageMatrix.postScale(scale, scale, width / 2f, height / 2f)
        } else if (currentScale > maxScale) {
            val scale = maxScale / currentScale
            imageMatrix.postScale(scale, scale, width / 2f, height / 2f)
        }
        
        // 限制平移范围（确保图片在视口内）
        workingBitmap?.let { bitmap ->
            updateInverseMatrix()
            
            // 计算图片边界（在应用矩阵变换后）
            val corners = floatArrayOf(
                0f, 0f,
                bitmap.width.toFloat(), 0f,
                bitmap.width.toFloat(), bitmap.height.toFloat(),
                0f, bitmap.height.toFloat()
            )
            inverseMatrix.mapPoints(corners)
            
            // 检查是否完全在视口外，若是则限制平移
            val minX = corners.filterIndexed { i, _ -> i % 2 == 0 }.minOrNull() ?: 0f
            val maxX = corners.filterIndexed { i, _ -> i % 2 == 0 }.maxOrNull() ?: 0f
            val minY = corners.filterIndexed { i, _ -> i % 2 == 1 }.minOrNull() ?: 0f
            val maxY = corners.filterIndexed { i, _ -> i % 2 == 1 }.maxOrNull() ?: 0f
            
            var dx = 0f
            var dy = 0f
            
            if (maxX < width) dx = width - maxX
            if (minX > 0) dx = -minX
            if (maxY < height) dy = height - maxY
            if (minY > 0) dy = -minY
            
            if (dx != 0f || dy != 0f) {
                imageMatrix.postTranslate(dx, dy)
            }
        }
    }

    /**
     * 获取编辑后的Bitmap
     * 
     * @return 包含所有编辑内容的Bitmap
     */
    fun getEditedBitmap(): Bitmap? {
        return workingBitmap?.copy(Bitmap.Config.ARGB_8888, false)
    }

    /**
     * 撤销上一步操作
     */
    fun undo(): Boolean {
        return if (pathStack.isNotEmpty()) {
            // 重新从原图开始绘制
            workingBitmap?.let { bitmap ->
                // 清除并重绘
                drawingCanvas?.drawColor(Color.TRANSPARENT, PorterDuff.Mode.CLEAR)
                drawingCanvas?.drawBitmap(originalBitmap!!, 0f, 0f, imagePaint)
                
                // 重新绘制除最后一个外的所有路径
                pathStack.removeAt(pathStack.size - 1)
                pathStack.forEach { pathData ->
                    drawingCanvas?.drawPath(pathData.path, pathData.paint)
                }
            }
            invalidate()
            true
        } else {
            false
        }
    }

    /**
     * 释放资源
     */
    fun release() {
        originalBitmap?.recycle()
        workingBitmap?.recycle()
        originalBitmap = null
        workingBitmap = null
        drawingCanvas = null
    }

    // ============ 手势监听器 ============

    private inner class ScaleListener : ScaleGestureDetector.SimpleOnScaleGestureListener() {
        override fun onScaleBegin(detector: ScaleGestureDetector): Boolean {
            isScaling = true
            return true
        }

        override fun onScale(detector: ScaleGestureDetector): Boolean {
            val scale = detector.scaleFactor
            imageMatrix.postScale(scale, scale, detector.focusX, detector.focusY)
            constrainMatrix()
            invalidate()
            return true
        }

        override fun onScaleEnd(detector: ScaleGestureDetector) {
            isScaling = false
        }
    }

    private inner class GestureListener : GestureDetector.SimpleOnGestureListener() {
        override fun onDoubleTap(e: MotionEvent): Boolean {
            // 双击重置缩放
            if (scaleFactor > 1f) {
                resetTransform()
            } else {
                // 放大到2倍
                imageMatrix.postScale(2f, 2f, e.x, e.y)
                constrainMatrix()
            }
            invalidate()
            return true
        }
    }
}
```

---

## 四、滤镜处理

### 4.1 常用滤镜实现

```kotlin
/**
 * 图片滤镜处理器
 * 
 * 滤镜原理：
 * 遍历Bitmap的每个像素，根据滤镜算法计算新的颜色值
 * 
 * 性能优化：
 * - 使用Bitmap.getPixels批量获取像素
 * - 使用IntArray存储处理结果
 * - 使用AndroidBitmap.lockPixels进行原地处理
 */
object ImageFilters {

    /**
     * 应用滤镜到Bitmap
     * 
     * @param source 源Bitmap
     * @param filter 滤镜类型
     * @return 应用滤镜后的新Bitmap
     */
    fun applyFilter(source: Bitmap, filter: FilterType): Bitmap {
        // 创建输出Bitmap
        val output = Bitmap.createBitmap(
            source.width,
            source.height,
            Bitmap.Config.ARGB_8888
        )
        
        // 获取像素数据
        val width = source.width
        val height = source.height
        val pixels = IntArray(width * height)
        source.getPixels(pixels, 0, width, 0, 0, width, height)
        
        // 应用滤镜
        val filteredPixels = when (filter) {
            FilterType.GRAYSCALE -> grayscale(pixels)
            FilterType.SEPIA -> sepia(pixels)
            FilterType.WARM -> warm(pixels)
            FilterType.COOL -> cool(pixels)
            FilterType.VINTAGE -> vintage(pixels)
            FilterType.CONTRAST -> contrast(pixels, 1.5f)
            FilterType.BRIGHTNESS -> brightness(pixels, 30)
            FilterType.SATURATION -> saturation(pixels, 1.5f)
            FilterType.NONE -> pixels
        }
        
        // 设置处理后的像素
        output.setPixels(filteredPixels, 0, width, 0, 0, width, height)
        
        return output
    }

    /**
     * 灰度滤镜
     * 
     * 算法：取RGB的平均值，或使用加权平均
     * 人眼对绿色最敏感，所以常用加权公式：
     * Gray = 0.299*R + 0.587*G + 0.114*B
     */
    private fun grayscale(pixels: IntArray): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            // 加权灰度转换
            val gray = (0.299 * r + 0.587 * g + 0.114 * b).toInt()
            
            Color.argb(Color.alpha(pixel), gray, gray, gray)
        }.toIntArray()
    }

    /**
     * 复古/棕褐色调滤镜
     * 
     * 算法：
     * R' = R * 0.393 + G * 0.769 + B * 0.189
     * G' = R * 0.349 + G * 0.686 + B * 0.168
     * B' = R * 0.272 + G * 0.534 + B * 0.131
     */
    private fun sepia(pixels: IntArray): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            val newR = minOf(255, (0.393 * r + 0.769 * g + 0.189 * b).toInt())
            val newG = minOf(255, (0.349 * r + 0.686 * g + 0.168 * b).toInt())
            val newB = minOf(255, (0.272 * r + 0.534 * g + 0.131 * b).toInt())
            
            Color.argb(Color.alpha(pixel), newR, newG, newB)
        }.toIntArray()
    }

    /**
     * 暖色调滤镜
     * 
     * 算法：增加红色和黄色，减少蓝色
     */
    private fun warm(pixels: IntArray): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            // 增加R和G，减少B
            val newR = minOf(255, (r * 1.2).toInt())
            val newG = minOf(255, (g * 1.1).toInt())
            val newB = maxOf(0, (b * 0.9).toInt())
            
            Color.argb(Color.alpha(pixel), newR, newG, newB)
        }.toIntArray()
    }

    /**
     * 冷色调滤镜
     * 
     * 算法：增加蓝色，减少红色和黄色
     */
    private fun cool(pixels: IntArray): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            // 减少R和G，增加B
            val newR = maxOf(0, (r * 0.9).toInt())
            val newG = maxOf(0, (g * 0.95).toInt())
            val newB = minOf(255, (b * 1.2).toInt())
            
            Color.argb(Color.alpha(pixel), newR, newG, newB)
        }.toIntArray()
    }

    /**
     * 复古滤镜
     * 
     * 组合效果：降低对比度 + 棕褐色调 + 噪点纹理
     */
    private fun vintage(pixels: IntArray): IntArray {
        return pixels.map { pixel ->
            var r = Color.red(pixel)
            var g = Color.green(pixel)
            var b = Color.blue(pixel)
            
            // 降低对比度
            val factor = 0.8f
            r = ((r - 128) * factor + 128).toInt()
            g = ((g - 128) * factor + 128).toInt()
            b = ((b - 128) * factor + 128).toInt()
            
            // 添加棕褐色调
            r = (r * 1.1).toInt()
            b = (b * 0.9).toInt()
            
            // 限制范围
            r = r.coerceIn(0, 255)
            g = g.coerceIn(0, 255)
            b = b.coerceIn(0, 255)
            
            Color.argb(Color.alpha(pixel), r, g, b)
        }.toIntArray()
    }

    /**
     * 对比度调整
     * 
     * @param factor 对比度因子（1.0不变，>1.0增加对比度，<1.0降低）
     */
    private fun contrast(pixels: IntArray, factor: Float): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            // 对比度公式：newValue = ((value - 128) * factor) + 128
            val newR = ((r - 128) * factor + 128).toInt().coerceIn(0, 255)
            val newG = ((g - 128) * factor + 128).toInt().coerceIn(0, 255)
            val newB = ((b - 128) * factor + 128).toInt().coerceIn(0, 255)
            
            Color.argb(Color.alpha(pixel), newR, newG, newB)
        }.toIntArray()
    }

    /**
     * 亮度调整
     * 
     * @param value 亮度调整值（正数变亮，负数变暗）
     */
    private fun brightness(pixels: IntArray, value: Int): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            val newR = (r + value).coerceIn(0, 255)
            val newG = (g + value).coerceIn(0, 255)
            val newB = (b + value).coerceIn(0, 255)
            
            Color.argb(Color.alpha(pixel), newR, newG, newB)
        }.toIntArray()
    }

    /**
     * 饱和度调整
     * 
     * @param factor 饱和度因子（1.0不变，0.0灰度，>1.0高饱和）
     */
    private fun saturation(pixels: IntArray, factor: Float): IntArray {
        return pixels.map { pixel ->
            val r = Color.red(pixel)
            val g = Color.green(pixel)
            val b = Color.blue(pixel)
            
            // 转HSV调整饱和度后转回RGB
            val hsv = FloatArray(3)
            Color.RGBToHSV(r, g, b, hsv)
            hsv[1] = (hsv[1] * factor).coerceIn(0f, 1f)
            val newColor = Color.HSVToColor(Color.alpha(pixel), hsv)
            
            newColor
        }.toIntArray()
    }

    /**
     * 滤镜类型枚举
     */
    enum class FilterType {
        NONE,
        GRAYSCALE,
        SEPIA,
        WARM,
        COOL,
        VINTAGE,
        CONTRAST,
        BRIGHTNESS,
        SATURATION
    }
}
```

### 4.2 滤镜预览（使用RenderScript加速）

```kotlin
/**
 * 高性能滤镜处理器（使用RenderScript）
 * 
 * RenderScript优势：
 * - GPU/专用DSP加速
 * - 自动并行化
 * - 比Java代码快数倍
 * 
 * 注意：从Android 12开始，RenderScript被废弃，
 * 建议使用RenderScriptIntrinsic或迁移到GPU
 */
@RequiresApi(Build.VERSION_CODES.R)
object RenderScriptFilters {

    private var renderScript: RenderScript? = null

    /**
     * 初始化RenderScript
     */
    fun init(context: Context) {
        renderScript = RenderScript.create(context)
    }

    /**
     * 应用灰度滤镜（高性能版本）
     */
    fun grayscale(source: Bitmap): Bitmap {
        val rs = renderScript ?: return source
        
        // 创建输入Allocation
        val input = Allocation.createFromBitmap(
            rs,
            source,
            Allocation.MipmapControl.MIPMAP_NONE,
            Allocation.USAGE_SCRIPT or Allocation.USAGE_SHARED
        )
        
        // 创建输出Bitmap
        val output = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
        val outAllocation = Allocation.createFromBitmap(rs, output)
        
        // 创建Script
        val script = ScriptC_grayscale(rs)
        
        // 执行
        script.forEach_root(input, outAllocation)
        outAllocation.copyTo(output)
        
        // 销毁资源
        input.destroy()
        outAllocation.destroy()
        script.destroy()
        
        return output
    }

    /**
     * 释放资源
     */
    fun release() {
        renderScript?.destroy()
        renderScript = null
    }

    /**
     * 释放时调用
     */
    fun onDestroy() {
        release()
    }
}
```

---

## 五、图片保存与分享

### 5.1 保存到相册

```kotlin
/**
 * 图片保存管理器
 * 
 * 保存流程：
 * 1. 将Bitmap保存到应用私有目录或MediaStore
 * 2. 通知系统相册更新
 * 3. 返回保存结果
 */
class ImageSaveManager(private val context: Context) {

    /**
     * 保存Bitmap到相册
     * 
     * 使用MediaStore API（Android 10+推荐）
     * 
     * @param bitmap 要保存的Bitmap
     * @param displayName 显示名称（不含扩展名）
     * @param quality JPEG质量（0-100）
     * @param onComplete 保存完成的回调
     */
    fun saveToGallery(
        bitmap: Bitmap,
        displayName: String = "IMG_${System.currentTimeMillis()}",
        quality: Int = 95,
        onComplete: (Result<Uri>) -> Unit
    ) {
        // 使用协程在后台执行
        CoroutineScope(Dispatchers.IO).launch {
            try {
                val uri = saveImageInternal(bitmap, displayName, quality)
                withContext(Dispatchers.Main) {
                    onComplete(Result.success(uri))
                }
            } catch (e: Exception) {
                Log.e(TAG, "Failed to save image", e)
                withContext(Dispatchers.Main) {
                    onComplete(Result.failure(e))
                }
            }
        }
    }

    /**
     * 内部保存方法
     */
    private fun saveImageInternal(
        bitmap: Bitmap,
        displayName: String,
        quality: Int
    ): Uri {
        val filename = "$displayName.jpg"
        val mimeType = "image/jpeg"
        
        return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            // Android 10+ 使用MediaStore
            saveWithMediaStore(bitmap, filename, mimeType, quality)
        } else {
            // Android 9及以下直接保存到相册目录
            saveToExternalStorage(bitmap, filename, quality)
        }
    }

    /**
     * 使用MediaStore保存（Android 10+）
     */
    @RequiresApi(Build.VERSION_CODES.Q)
    private fun saveWithMediaStore(
        bitmap: Bitmap,
        filename: String,
        mimeType: String,
        quality: Int
    ): Uri {
        val contentValues = ContentValues().apply {
            put(MediaStore.Images.Media.DISPLAY_NAME, filename)
            put(MediaStore.Images.Media.MIME_TYPE, mimeType)
            // 标记为用户可见
            put(MediaStore.Images.Media.IS_PENDING, 1)
        }

        val uri = context.contentResolver.insert(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            contentValues
        ) ?: throw IllegalStateException("Failed to create MediaStore entry")

        context.contentResolver.openOutputStream(uri)?.use { outputStream ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)
            outputStream.flush()
        }

        // 标记完成
        contentValues.clear()
        contentValues.put(MediaStore.Images.Media.IS_PENDING, 0)
        context.contentResolver.update(uri, contentValues, null, null)

        Log.d(TAG, "Image saved to: $uri")
        return uri
    }

    /**
     * 保存到外部存储（Android 9及以下）
     */
    @Suppress("DEPRECATION")
    private fun saveToExternalStorage(
        bitmap: Bitmap,
        filename: String,
        quality: Int
    ): Uri {
        // 获取相册目录
        val picturesDir = Environment.getExternalStoragePublicDirectory(
            Environment.DIRECTORY_PICTURES
        )
        
        // 创建应用专属子目录
        val appDir = File(picturesDir, context.packageName)
        if (!appDir.exists()) {
            appDir.mkdirs()
        }
        
        // 创建文件
        val imageFile = File(appDir, filename)
        
        FileOutputStream(imageFile).use { outputStream ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)
            outputStream.flush()
        }
        
        // 通知相册扫描新文件
        MediaScannerConnection.scanFile(
            context,
            arrayOf(imageFile.absolutePath),
            arrayOf("image/jpeg"),
            null
        )
        
        return Uri.fromFile(imageFile)
    }

    /**
     * 保存到应用私有目录
     * 
     * 优点：不需要存储权限
     * 缺点：其他应用无法访问
     */
    fun saveToAppDirectory(
        bitmap: Bitmap,
        filename: String = "edited_${System.currentTimeMillis()}.jpg",
        quality: Int = 95
    ): Uri? {
        return try {
            val file = File(context.filesDir, "images")
            if (!file.exists()) file.mkdirs()
            
            val imageFile = File(file, filename)
            
            FileOutputStream(imageFile).use { outputStream ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, quality, outputStream)
                outputStream.flush()
            }
            
            Uri.fromFile(imageFile)
        } catch (e: Exception) {
            Log.e(TAG, "Failed to save to app directory", e)
            null
        }
    }

    companion object {
        private const val TAG = "ImageSaveManager"
    }
}
```

### 5.2 分享图片

```kotlin
/**
 * 图片分享管理器
 */
class ShareManager(private val activity: Activity) {

    /**
     * 分享图片
     * 
     * @param bitmap 要分享的Bitmap
     * @param text 分享文字（可选）
     */
    fun shareImage(bitmap: Bitmap, text: String? = null) {
        // 保存到缓存目录（便于分享）
        val cachePath = File(activity.cacheDir, "images")
        cachePath.mkdirs()
        
        val imageFile = File(cachePath, "share_image.jpg")
        
        FileOutputStream(imageFile).use { outputStream ->
            bitmap.compress(Bitmap.CompressFormat.JPEG, 95, outputStream)
        }
        
        // 创建分享Intent
        val uri = FileProvider.getUriForFile(
            activity,
            "${activity.packageName}.fileprovider",
            imageFile
        )
        
        val shareIntent = Intent(Intent.ACTION_SEND).apply {
            type = "image/jpeg"
            putExtra(Intent.EXTRA_STREAM, uri)
            text?.let { putExtra(Intent.EXTRA_TEXT, it) }
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        
        activity.startActivity(
            Intent.createChooser(shareIntent, "分享图片")
        )
    }
}
```

---

## 六、性能优化

### 6.1 内存管理

```kotlin
/**
 * Bitmap内存管理工具
 * 
 * 关键原则：
 * 1. 及时回收不再使用的Bitmap
 * 2. 处理大图时先采样
 * 3. 使用合适颜色的Bitmap配置
 * 4. 注意Context生命周期
 */
object BitmapMemoryManager {

    /**
     * 计算Bitmap的内存占用
     */
    fun getBitmapBytes(bitmap: Bitmap): Long {
        return bitmap.allocationByteCount.toLong()
    }

    /**
     * 获取当前应用的内存使用情况
     */
    fun getMemoryInfo(context: Context): MemoryInfo {
        val runtime = Runtime.getRuntime()
        val memoryInfo = ActivityManager.MemoryInfo()
        
        @Suppress("DEPRECATION")
        val activityManager = context.getSystemService(Context.ACTIVITY_SERVICE) as ActivityManager
        activityManager.getMemoryInfo(memoryInfo)
        
        return MemoryInfo(
            availableMemory = memoryInfo.availMem,
            totalMemory = memoryInfo.totalMem,
            isLowMemory = memoryInfo.lowMemory,
            runtimeMaxMemory = runtime.maxMemory(),
            runtimeFreeMemory = runtime.freeMemory(),
            runtimeTotalMemory = runtime.totalMemory()
        )
    }

    /**
     * 内存信息数据类
     */
    data class MemoryInfo(
        val availableMemory: Long,
        val totalMemory: Long,
        val isLowMemory: Boolean,
        val runtimeMaxMemory: Long,
        val runtimeFreeMemory: Long,
        val runtimeTotalMemory: Long
    ) {
        // 计算已使用内存
        val usedMemory: Long get() = runtimeTotalMemory - runtimeFreeMemory
        
        // 计算可用内存比例
        val availableRatio: Float get() = 
            availableMemory.toFloat() / runtimeMaxMemory
        
        // 是否处于高内存压力状态
        val isHighMemoryPressure: Boolean get() = availableRatio < 0.2f
    }

    /**
     * 安全回收Bitmap
     */
    fun recycleBitmap(bitmap: Bitmap?) {
        if (bitmap != null && !bitmap.isRecycled) {
            bitmap.recycle()
        }
    }

    /**
     * 批量回收Bitmap列表
     */
    fun recycleBitmaps(bitmaps: List<Bitmap?>) {
        bitmaps.forEach { recycleBitmap(it) }
    }

    /**
     * 检查是否可以安全创建指定尺寸的Bitmap
     * 
     * @param width 宽度
     * @param height 高度
     * @param config Bitmap配置
     * @return 是否可以安全创建
     */
    fun canAllocateBitmap(
        context: Context,
        width: Int,
        height: Int,
        config: Bitmap.Config = Bitmap.Config.ARGB_8888
    ): Boolean {
        val bytesPerPixel = when (config) {
            Bitmap.Config.ARGB_8888 -> 4
            Bitmap.Config.RGB_565 -> 2
            Bitmap.Config.ALPHA_8 -> 1
            else -> 4
        }
        
        val requiredMemory = width.toLong() * height * bytesPerPixel
        val memoryInfo = getMemoryInfo(context)
        
        // 检查是否有足够的可用内存（预留50%安全边际）
        val safeLimit = memoryInfo.runtimeMaxMemory / 2
        
        return requiredMemory <= safeLimit
    }
}
```

### 6.2 后台处理

```kotlin
/**
 * 图片处理Worker
 * 
 * 使用Kotlin协程和WorkManager处理耗时的图片操作
 */
object ImageProcessingWorker {

    /**
     * 在后台线程处理图片
     * 
     * @param bitmap 输入Bitmap
     * @param operation 处理操作
     * @param onProgress 进度回调
     * @param onComplete 完成回调
     */
    fun processBitmap(
        bitmap: Bitmap,
        operation: suspend (Bitmap) -> Bitmap,
        onComplete: (Bitmap) -> Unit
    ) {
        CoroutineScope(Dispatchers.Default).launch {
            try {
                val result = operation(bitmap)
                withContext(Dispatchers.Main) {
                    onComplete(result)
                }
            } catch (e: CancellationException) {
                Log.d(TAG, "Processing cancelled")
            } catch (e: Exception) {
                Log.e(TAG, "Processing failed", e)
            }
        }
    }

    /**
     * 应用滤镜（带进度）
     */
    fun applyFilterWithProgress(
        bitmap: Bitmap,
        filter: ImageFilters.FilterType,
        onProgress: (Int) -> Unit,
        onComplete: (Bitmap) -> Unit
    ) {
        CoroutineScope(Dispatchers.Default).launch {
            try {
                val result = withContext(Dispatchers.Default) {
                    // 分块处理，每10%报告一次进度
                    val totalPixels = bitmap.width * bitmap.height
                    val progressStep = totalPixels / 10
                    
                    var processed = 0
                    val pixels = IntArray(totalPixels)
                    bitmap.getPixels(pixels, 0, bitmap.width, 0, 0, bitmap.width, bitmap.height)
                    
                    // 应用滤镜
                    val filteredPixels = when (filter) {
                        ImageFilters.FilterType.GRAYSCALE -> ImageFilters.grayscale(pixels)
                        ImageFilters.FilterType.SEPIA -> ImageFilters.sepia(pixels)
                        else -> pixels
                    }
                    
                    // 创建输出Bitmap
                    val output = Bitmap.createBitmap(
                        bitmap.width,
                        bitmap.height,
                        Bitmap.Config.ARGB_8888
                    )
                    output.setPixels(filteredPixels, 0, bitmap.width, 0, 0, bitmap.width, bitmap.height)
                    
                    // 报告进度
                    withContext(Dispatchers.Main) {
                        onProgress(100)
                    }
                    
                    output
                }
                
                withContext(Dispatchers.Main) {
                    onComplete(result)
                }
            } catch (e: CancellationException) {
                // 任务被取消
            } catch (e: Exception) {
                Log.e(TAG, "Filter application failed", e)
            }
        }
    }
}
```

---

## 七、完整示例：图片编辑器Activity

```kotlin
/**
 * 图片编辑器Activity完整示例
 * 
 * 功能：
 * - 从相册选择图片
 * - 裁剪、旋转、缩放
 * - 涂鸦、文字添加
 * - 滤镜应用
 * - 保存和分享
 */
class ImageEditorActivity : AppCompatActivity() {

    // ViewBinding
    private lateinit var binding: ActivityImageEditorBinding
    
    // 图片选择器
    private lateinit var imagePicker: ImagePicker
    
    // 权限管理器
    private lateinit var permissionManager: PermissionManager
    
    // 保存管理器
    private lateinit var saveManager: ImageSaveManager
    
    // 分享管理器
    private lateinit var shareManager: ShareManager
    
    // 当前编辑的Bitmap
    private var currentBitmap: Bitmap? = null
    
    // 可用的滤镜列表
    private val filters = listOf(
        "原图" to ImageFilters.FilterType.NONE,
        "灰度" to ImageFilters.FilterType.GRAYSCALE,
        "复古" to ImageFilters.FilterType.SEPIA,
        "暖色" to ImageFilters.FilterType.WARM,
        "冷色" to ImageFilters.FilterType.COOL
    )
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityImageEditorBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        // 初始化管理器
        imagePicker = ImagePicker(this)
        permissionManager = PermissionManager(this)
        saveManager = ImageSaveManager(this)
        shareManager = ShareManager(this)
        
        // 注册PhotoPicker
        imagePicker.registerLaunchers()
        
        // 初始化View
        setupViews()
        
        // 自动打开图片选择器
        openImagePicker()
    }

    private fun setupViews() {
        // 选择图片按钮
        binding.btnSelectImage.setOnClickListener {
            openImagePicker()
        }
        
        // 涂鸦模式
        binding.btnDraw.setOnClickListener {
            binding.editorView.setEditMode(ImageEditorView.EditMode.DRAW)
        }
        
        // 撤销按钮
        binding.btnUndo.setOnClickListener {
            binding.editorView.undo()
        }
        
        // 保存按钮
        binding.btnSave.setOnClickListener {
            saveImage()
        }
        
        // 分享按钮
        binding.btnShare.setOnClickListener {
            shareImage()
        }
        
        // 设置画笔颜色
        binding.colorRed.setOnClickListener {
            binding.editorView.setDrawColor(Color.RED)
        }
        binding.colorBlue.setOnClickListener {
            binding.editorView.setDrawColor(Color.BLUE)
        }
        binding.colorGreen.setOnClickListener {
            binding.editorView.setDrawColor(Color.GREEN)
        }
        binding.colorYellow.setOnClickListener {
            binding.editorView.setDrawColor(Color.YELLOW)
        }
        
        // 设置画笔粗细
        binding.sliderStrokeWidth.addOnChangeListener { _, value, _ ->
            binding.editorView.setDrawStrokeWidth(value)
        }
    }

    private fun openImagePicker() {
        imagePicker.pickImageWithPhotoPicker { uri ->
            loadImage(uri)
        }
    }

    private fun loadImage(uri: Uri) {
        // 显示加载状态
        binding.progressBar.visibility = View.VISIBLE
        
        CoroutineScope(Dispatchers.Main).launch {
            val bitmap = ImageLoader.loadBitmap(this@ImageEditorActivity, uri)
            
            bitmap?.let {
                currentBitmap = it
                binding.editorView.setImage(it)
                binding.progressBar.visibility = View.GONE
                
                // 显示滤镜选项
                setupFilterChips()
            } ?: run {
                binding.progressBar.visibility = View.GONE
                Toast.makeText(this@ImageEditorActivity, "加载图片失败", Toast.LENGTH_SHORT).show()
            }
        }
    }

    private fun setupFilterChips() {
        binding.filterChipGroup.removeAllViews()
        
        filters.forEach { (name, filterType) ->
            val chip = Chip(this).apply {
                text = name
                isCheckable = true
                setOnClickListener {
                    applyFilter(filterType)
                }
            }
            binding.filterChipGroup.addView(chip)
        }
    }

    private fun applyFilter(filterType: ImageFilters.FilterType) {
        currentBitmap?.let { original ->
            binding.progressBar.visibility = View.VISIBLE
            
            CoroutineScope(Dispatchers.Default).launch {
                val filtered = ImageFilters.applyFilter(original, filterType)
                
                withContext(Dispatchers.Main) {
                    // 更新编辑器
                    binding.editorView.setImage(filtered)
                    currentBitmap = filtered
                    binding.progressBar.visibility = View.GONE
                }
            }
        }
    }

    private fun saveImage() {
        binding.editorView.getEditedBitmap()?.let { bitmap ->
            binding.progressBar.visibility = View.VISIBLE
            
            saveManager.saveToGallery(bitmap) { result ->
                binding.progressBar.visibility = View.GONE
                
                result.onSuccess { uri ->
                    Toast.makeText(this, "图片已保存到相册", Toast.LENGTH_SHORT).show()
                }.onFailure {
                    Toast.makeText(this, "保存失败: ${it.message}", Toast.LENGTH_SHORT).show()
                }
            }
        }
    }

    private fun shareImage() {
        binding.editorView.getEditedBitmap()?.let { bitmap ->
            shareManager.shareImage(bitmap, "来自图片编辑器")
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        binding.editorView.release()
        BitmapMemoryManager.recycleBitmap(currentBitmap)
    }

    // 权限处理
    @Deprecated("Deprecated in Java")
    override fun onRequestPermissionsResult(
        requestCode: Int,
        permissions: Array<out String>,
        grantResults: IntArray
    ) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults)
        
        permissionManager.handlePermissionResult(
            requestCode,
            permissions,
            grantResults,
            onGranted = { openImagePicker() },
            onDenied = {
                Toast.makeText(this, "需要相册权限才能选择图片", Toast.LENGTH_LONG).show()
            }
        )
    }
}
```

---

## 总结

本文详细介绍了Android相册图像编辑开发的核心知识：

| 模块 | 关键技术 |
|------|----------|
| 权限处理 | PhotoPicker、Scoped Storage、持久化URI权限 |
| 图片加载 | Coil、BitmapFactory采样、内存管理 |
| Canvas绘制 | 自定义View、Matrix变换、手势处理 |
| 滤镜处理 | 像素级操作、RenderScript加速 |
| 保存分享 | MediaStore API、FileProvider |
| 性能优化 | 内存管理、协程后台处理 |

**下一步学习建议**：
1. 学习更高级的滤镜算法（如卷积核滤镜）
2. 了解OpenGL ES实现GPU加速滤镜
3. 研究Material Design 3的图片编辑组件

---

