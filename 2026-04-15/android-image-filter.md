## 💻 四、每日知识（重点）

### Android 相册图像编辑开发：滤镜系统从入门到精通

> 本章节深入讲解 Android 平台上图像编辑应用的核心——滤镜系统。从基础的颜色矩阵变换，到 GPU 加速的高级滤镜，再到商业级图像编辑 SDK 的实现。内容涵盖原理讲解 + 完整代码示例，阅读时长约 15-20 分钟。

---

#### 4.1 滤镜系统概述

图像滤镜本质上是对图片每个像素的颜色值进行数学运算，产生特定的视觉效果。Android 中实现滤镜的主要方式：

| 实现方式 | 优点 | 缺点 | 适用场景 |
|----------|------|------|----------|
| **ColorMatrix** | 原生 API，无需第三方库 | 只能做线性变换 | 基础滤镜（亮度、对比度、饱和度） |
| **Paint + ColorFilter** | 简单易用 | 性能一般 | 简单图片处理 |
| **RenderScript** | GPU 加速 | 已废弃 | 复杂图像运算 |
| **GPUImage/OES** | 高性能，支持复杂效果 | 需要 OpenGL ES | 实时预览滤镜 |
| **Native (NDK/C++)** | 极致性能 | 开发成本高 | 专业图像处理 |

---

#### 4.2 ColorMatrix 颜色矩阵基础

ColorMatrix 是 Android 中最基础的图像处理工具，它使用一个 4×5 的矩阵对 RGBA 颜色进行变换。

**颜色矩阵原理：**

```
| R' |   | m00 m01 m02 m03 m04 |   | R |
| G' | = | m10 m11 m12 m13 m14 | × | G |
| B' |   | m20 m21 m22 m23 m24 |   | B |
| A' |   | m30 m31 m32 m33 m34 |   | A |
```

其中：
- m00-m22：控制 RGB 三通道的缩放（乘法运算）
- m03：控制 Alpha 通道
- m04, m14, m24：控制 RGB 通道的平移（加法运算）

**实战代码：灰度滤镜**

```kotlin
/**
 * 灰度滤镜实现
 * 原理：将 RGB 通道设置为相同值，利用人眼对绿色最敏感的原理
 * Gray = R × 0.299 + G × 0.587 + B × 0.114
 */
fun createGrayscaleMatrix(): ColorMatrix {
    val grayscaleMatrix = ColorMatrix()
    
    // 设置灰度矩阵
    grayscaleMatrix.setSaturation(0f)
    
    // 或者手动设置权重
    // val colorArray = floatArrayOf(
    //     0.299f, 0.587f, 0.114f, 0f, 0f,    // R
    //     0.299f, 0.587f, 0.114f, 0f, 0f,    // G  
    //     0.299f, 0.587f, 0.114f, 0f, 0f,    // B
    //     0f,     0f,     0f,     1f, 0f     // A
    // )
    // grayscaleMatrix.set(colorArray)
    
    return grayscaleMatrix
}

// 应用灰度滤镜
fun applyGrayscaleFilter(source: Bitmap): Bitmap {
    val result = Bitmap.createBitmap(
        source.width, 
        source.height, 
        Bitmap.Config.ARGB_8888
    )
    
    val canvas = Canvas(result)
    val paint = Paint().apply {
        colorFilter = ColorMatrixColorFilter(createGrayscaleMatrix())
    }
    
    canvas.drawBitmap(source, 0f, 0f, paint)
    return result
}
```

**实战代码：亮度调节**

```kotlin
/**
 * 亮度调节滤镜
 * 原理：通过矩阵的平移值（m04, m14, m24）增加/减少颜色值
 * @param brightness 亮度值，范围 -255 到 255
 */
fun createBrightnessMatrix(brightness: Float): ColorMatrix {
    val brightnessMatrix = ColorMatrix()
    
    // 平移矩阵：在对角线上为1，其他为0，实现亮度调节
    // [1 0 0 0 brightness]
    // [0 1 0 0 brightness]
    // [0 0 1 0 brightness]
    // [0 0 0 1 0      ]
    brightnessMatrix.set(floatArrayOf(
        1f, 0f, 0f, 0f, brightness,
        0f, 1f, 0f, 0f, brightness,
        0f, 0f, 1f, 0f, brightness,
        0f, 0f, 0f, 1f, 0f
    ))
    
    return brightnessMatrix
}

// 使用 Paint 应用
fun applyBrightness(source: Bitmap, brightness: Float): Bitmap {
    val result = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(result)
    val paint = Paint().apply {
        colorFilter = ColorMatrixColorFilter(createBrightnessMatrix(brightness))
    }
    canvas.drawBitmap(source, 0f, 0f, paint)
    return result
}
```

**实战代码：对比度调节**

```kotlin
/**
 * 对比度调节滤镜
 * 原理：缩放 RGB 值，使偏离中值的颜色变化更剧烈
 * @param contrast 对比度值，范围 -127 到 127
 */
fun createContrastMatrix(contrast: Float): ColorMatrix {
    val scale = (contrast + 127f) / 127f  // 归一化到 0-2 范围
    val translate = (-.5f * scale + .5f) * 255f
    
    return ColorMatrix(floatArrayOf(
        scale, 0f, 0f, 0f, translate,
        0f, scale, 0f, 0f, translate,
        0f, 0f, scale, 0f, translate,
        0f, 0f, 0f, 1f, 0f
    ))
}
```

---

#### 4.3 高级滤镜：饱和度与色彩旋转

**饱和度调节：**

```kotlin
/**
 * 饱和度调节
 * setSaturation(float sat) 内部实现：
 * [sr  sg  sb  0  sr*255]
 * [sr  sg  sb  0  sg*255]
 * [sr  sg  sb  0  sb*255]
 * [0   0   0   1  0     ]
 * 其中 sr = 1 - 0.213 (红色的权重)
 *      sg = 1 - 0.715 (绿色的权重)  
 *      sb = 1 - 0.072 (蓝色的权重)
 */
fun createSaturationMatrix(saturation: Float): ColorMatrix {
    return ColorMatrix().apply {
        setSaturation(saturation)  // 0 = 灰度, 1 = 原始, 2 = 高度饱和
    }
}
```

**色彩旋转（色相调节）：**

```kotlin
/**
 * 色相旋转
 * 围绕 RGB 轴旋转颜色空间
 * @param axis 0=红色轴, 1=绿色轴, 2=蓝色轴
 * @param degrees 旋转角度
 */
fun createHueRotationMatrix(axis: Int, degrees: Float): ColorMatrix {
    val matrix = ColorMatrix()
    matrix.setRotate(axis, degrees)
    return matrix
}

// 综合色相调节
fun createHueMatrix(degrees: Float): ColorMatrix {
    val mat = ColorMatrix()
    
    val cosVal = Math.cos(Math.toRadians(degrees.toDouble())).toFloat()
    val sinVal = Math.sin(Math.toRadians(degrees.toDouble())).toFloat()
    
    val lumR = 0.213f
    val lumG = 0.715f
    val lumB = 0.072f
    
    val matArray = floatArrayOf(
        lumR + cosVal * (1 - lumR) + sinVal * (-lumR),
        lumG + cosVal * (-lumG) + sinVal * (-lumG),
        lumB + cosVal * (-lumB) + sinVal * (1 - lumB),
        0f, 0f,
        
        lumR + cosVal * (-lumR) + sinVal * (0.143f),
        lumG + cosVal * (1 - lumG) + sinVal * (0.140f),
        lumB + cosVal * (-lumB) + sinVal * (-0.283f),
        0f, 0f,
        
        lumR + cosVal * (-lumR) + sinVal * (-(1 - lumR)),
        lumG + cosVal * (-lumG) + sinVal * (lumG),
        lumB + cosVal * (1 - lumB) + sinVal * (lumB),
        0f, 0f,
        
        0f, 0f, 0f, 1f, 0f
    )
    
    mat.set(matArray)
    return mat
}
```

---

#### 4.4 滤镜叠加与组合

实际应用中，单一滤镜往往不够，需要组合多个滤镜效果：

```kotlin
/**
 * 滤镜组合器
 * 使用 ColorMatrix 的 postConcat 方法组合多个滤镜
 */
class FilterComposer {
    private val colorMatrix = ColorMatrix()
    
    // 添加灰度效果
    fun grayscale(): FilterComposer {
        val grayMatrix = ColorMatrix()
        grayMatrix.setSaturation(0f)
        colorMatrix.postConcat(grayMatrix)
        return this
    }
    
    // 添加亮度
    fun brightness(value: Float): FilterComposer {
        val brightMatrix = ColorMatrix(floatArrayOf(
            1f, 0f, 0f, 0f, value,
            0f, 1f, 0f, 0f, value,
            0f, 0f, 1f, 0f, value,
            0f, 0f, 0f, 1f, 0f
        ))
        colorMatrix.postConcat(brightMatrix)
        return this
    }
    
    // 添加对比度
    fun contrast(value: Float): FilterComposer {
        val scale = (value + 127f) / 127f
        val translate = (-.5f * scale + .5f) * 255f
        val contrastMatrix = ColorMatrix(floatArrayOf(
            scale, 0f, 0f, 0f, translate,
            0f, scale, 0f, 0f, translate,
            0f, 0f, scale, 0f, translate,
            0f, 0f, 0f, 1f, 0f
        ))
        colorMatrix.postConcat(contrastMatrix)
        return this
    }
    
    // 添加饱和度
    fun saturation(value: Float): FilterComposer {
        val satMatrix = ColorMatrix()
        satMatrix.setSaturation(value)
        colorMatrix.postConcat(satMatrix)
        return this
    }
    
    fun build(): ColorMatrix = ColorMatrix(colorMatrix.array)
    
    private fun ColorMatrix.array(): FloatArray {
        // 返回内部矩阵数组
        val field = ColorMatrix::class.java.getDeclaredField("mArray")
        field.isAccessible = true
        return field.get(this) as FloatArray
    }
}

// 使用示例
fun applyVintageFilter(source: Bitmap): Bitmap {
    val result = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
    val canvas = Canvas(result)
    val paint = Paint().apply {
        colorFilter = ColorMatrixColorFilter(
            FilterComposer()
                .saturation(0.7f)     // 降低饱和度
                .contrast(30f)          // 增加对比度
                .brightness(10f)        // 轻微提亮
                .grayscale()           // 添加灰度
                .build()
        )
    }
    canvas.drawBitmap(source, 0f, 0f, paint)
    return result
}
```

---

#### 4.5 GPU 加速滤镜：GPUImage 实战

对于需要实时预览的图像编辑应用，GPU 加速是必须的。使用 GPUImage 库可以轻松实现高性能滤镜：

**添加依赖：**

```groovy
// build.gradle
dependencies {
    implementation 'jp.co.cyberagent.android:gpuimage:2.1.0'
}
```

**GPUImage 滤镜使用：**

```kotlin
class GPUImageFilterManager(private val context: Context) {
    
    private lateinit var gpuImage: GPUImage
    
    // 预设滤镜集合
    enum class FilterType {
        BRIGHTNESS,
        CONTRAST,
        SATURATION,
        GRAYSCALE,
        SEPIA,
        SHARPEN,
        SOBEL_EDGE,
        VIGNETTE,
        FILTER_NONE
    }
    
    /**
     * 获取滤镜实例
     */
    fun getFilter(type: FilterType, intensity: Float = 0.5f): GPUImageFilter {
        return when (type) {
            FilterType.BRIGHTNESS -> GPUImageBrightnessFilter(intensity)
            FilterType.CONTRAST -> GPUImageContrastFilter(intensity * 4 + 1)
            FilterType.SATURATION -> GPUImageSaturationFilter(intensity * 2)
            FilterType.GRAYSCALE -> GPUImageGrayscaleFilter()
            FilterType.SEPIA -> GPUImageSepiaToneFilter(intensity)
            FilterType.SHARPEN -> GPUImageSharpenFilter(intensity * 2)
            FilterType.SOBEL_EDGE -> GPUImageSobelEdgeDetectionFilter()
            FilterType.VIGNETTE -> GPUImageVignetteFilter().apply {
                setVignetteStart(0.3f)
                setVignetteEnd(0.75f)
            }
            FilterType.FILTER_NONE -> GPUImageFilter()
        }
    }
    
    /**
     * 应用滤镜并返回结果 Bitmap
     */
    fun applyFilter(sourceBitmap: Bitmap, type: FilterType, intensity: Float = 0.5f): Bitmap {
        gpuImage = GPUImage(context)
        gpuImage.setImage(sourceBitmap)
        gpuImage.setFilter(getFilter(type, intensity))
        return gpuImage.bitmapWithFilterApplied
    }
    
    /**
     * 组合多个滤镜
     */
    fun createFilterGroup(filters: List<Pair<FilterType, Float>>): GPUImageFilterGroup {
        val filterList = filters.map { (type, intensity) ->
            getFilter(type, intensity)
        }
        return GPUImageFilterGroup(filterList)
    }
}
```

**高级自定义滤镜（饱和度 + 亮度组合滑块）：**

```kotlin
/**
 * 自定义组合滤镜
 * 实现类似美图秀秀的饱和度和亮度调节
 */
class SaturationBrightnessFilter(
    private var saturation: Float = 1f,
    private var brightness: Float = 0f
) : GPUImageFilterGroup() {
    
    private val saturationFilter = GPUImageSaturationFilter()
    private val brightnessFilter = GPUImageBrightnessFilter()
    
    init {
        addFilter(saturationFilter)
        addFilter(brightnessFilter)
    }
    
    override fun onInitialized() {
        super.onInitialized()
        updateFilters()
    }
    
    fun setSaturation(value: Float) {
        saturation = value.coerceIn(0f, 2f)
        updateFilters()
    }
    
    fun setBrightness(value: Float) {
        brightness = value.coerceIn(-1f, 1f)
        updateFilters()
    }
    
    private fun updateFilters() {
        saturationFilter.setSaturation(saturation)
        brightnessFilter.setBrightness(brightness)
    }
}
```

---

#### 4.6 商业级滤镜实现：ColorMatrix 高级用法

以下是一些经典滤镜的完整实现方案：

**复古胶片效果（Sepia + Vignette）：**

```kotlin
/**
 * 复古胶片滤镜
 * 特点：暖色调 + 轻微模糊边缘 + 降低饱和度
 */
fun createVintageMatrix(): ColorMatrix {
    // Sepia 矩阵：将图像转换为棕褐色
    val sepiaMatrix = ColorMatrix(floatArrayOf(
        0.393f, 0.769f, 0.189f, 0f, 0f,
        0.349f, 0.686f, 0.168f, 0f, 0f,
        0.272f, 0.534f, 0.131f, 0f, 0f,
        0f,     0f,     0f,     1f, 0f
    ))
    
    // 降低饱和度
    val saturationMatrix = ColorMatrix()
    saturationMatrix.setSaturation(0.8f)
    
    // 组合矩阵
    sepiaMatrix.postConcat(saturationMatrix)
    return sepiaMatrix
}
```

**负片/反相效果：**

```kotlin
/**
 * 负片效果
 * 原理：用 255 减去每个通道的值
 */
fun createNegativeMatrix(): ColorMatrix {
    return ColorMatrix(floatArrayOf(
        -1f, 0f, 0f, 0f, 255f,
        0f, -1f, 0f, 0f, 255f,
        0f, 0f, -1f, 0f, 255f,
        0f, 0f, 0f, 1f, 0f
    ))
}
```

**卡通/素描效果（结合 GPUImage）：**

```kotlin
// 使用 GPUImageToonFilter 实现卡通效果
val toonFilter = GPUImageToonFilter().apply {
    setThreshold(0.5f)      // 阈值
    setQuantizationLevels(10f)  // 色彩量化级别
}

// 或者使用素描效果
val sketchFilter = GPUImageSketchFilter()
```

---

#### 4.7 滤镜系统架构设计

一个完整的滤镜系统应该具备以下架构：

```
┌─────────────────────────────────────────────────────────┐
│                    FilterManager                         │
│  ┌─────────────────────────────────────────────────┐    │
│  │              FilterChain (滤镜链)                │    │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐          │    │
│  │  │ Bright  │→ │Contrast │→ │Saturation│→ ...    │    │
│  │  │ Filter  │  │ Filter  │  │ Filter  │          │    │
│  │  └─────────┘  └─────────┘  └─────────┘          │    │
│  └─────────────────────────────────────────────────┘    │
│                          ↓                               │
│  ┌─────────────────────────────────────────────────┐    │
│  │           RenderEngine (渲染引擎)                │    │
│  │   ColorMatrix | GPUImage | OpenGL ES | NDK     │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

**核心接口设计：**

```kotlin
/**
 * 滤镜接口
 */
interface ImageFilter {
    val name: String
    fun apply(source: Bitmap): Bitmap
    fun applyWithProgress(source: Bitmap, onProgress: (Float) -> Unit): Bitmap
}

/**
 * 可配置参数的滤镜
 */
interface ConfigurableFilter : ImageFilter {
    fun setParameter(key: String, value: Any)
    fun getParameter(key: String): Any
}

/**
 * 滤镜链
 */
class FilterChain(private val filters: List<ImageFilter>) : ImageFilter {
    override val name: String = filters.joinToString(" + ") { it.name }
    
    override fun apply(source: Bitmap): Bitmap {
        var current = source
        filters.forEach { filter ->
            current = filter.apply(current)
        }
        return current
    }
    
    override fun applyWithProgress(source: Bitmap, onProgress: (Float) -> Unit): Bitmap {
        var current = source
        filters.forEachIndexed { index, filter ->
            current = filter.applyWithProgress(current) { progress ->
                onProgress((index + progress) / filters.size)
            }
        }
        return current
    }
}
```

---

#### 4.8 实战：完整的图像编辑Activity

```kotlin
class PhotoEditorActivity : AppCompatActivity() {
    
    private lateinit var gpuImage: GPUImage
    private lateinit var filterManager: GPUImageFilterManager
    
    private var currentFilterType = GPUImageFilterManager.FilterType.FILTER_NONE
    private var originalBitmap: Bitmap? = null
    
    private val filterSeekBar: SeekBar by bind(R.id.seekBarFilter)
    private val imagePreview: ImageView by bind(R.id.imagePreview)
    private val filterList: RecyclerView by bind(R.id.filterList)
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_photo_editor)
        
        // 初始化 GPUImage
        gpuImage = GPUImage(this)
        filterManager = GPUImageFilterManager(this)
        
        // 加载图片
        loadImage()
        
        // 设置滤镜列表
        setupFilterList()
        
        // 设置调节滑块
        setupSeekBar()
        
        // 保存按钮
        findViewById<Button>(R.id.btnSave).setOnClickListener {
            saveImage()
        }
    }
    
    private fun loadImage() {
        // 从相册或相机获取图片
        val imageUri = intent.data ?: return
        
        contentResolver.openInputStream(imageUri)?.use { inputStream ->
            originalBitmap = BitmapFactory.decodeStream(inputStream)
            originalBitmap?.let { bitmap ->
                // 缩放图片以避免 OOM
                val scaledBitmap = scaleBitmap(bitmap, 1080)
                gpuImage.setImage(scaledBitmap)
                imagePreview.setImageBitmap(scaledBitmap)
            }
        }
    }
    
    private fun scaleBitmap(bitmap: Bitmap, maxSize: Int): Bitmap {
        val ratio = minOf(
            maxSize.toFloat() / bitmap.width,
            maxSize.toFloat() / bitmap.height
        )
        if (ratio >= 1f) return bitmap
        
        val newWidth = (bitmap.width * ratio).toInt()
        val newHeight = (bitmap.height * ratio).toInt()
        return Bitmap.createScaledBitmap(bitmap, newWidth, newHeight, true)
    }
    
    private fun setupFilterList() {
        val filters = GPUImageFilterManager.FilterType.values()
        val adapter = FilterListAdapter(filters) { filterType ->
            applyFilter(filterType)
        }
        filterList.adapter = adapter
    }
    
    private fun applyFilter(filterType: GPUImageFilterManager.FilterType) {
        currentFilterType = filterType
        
        // 获取滤镜
        val filter = filterManager.getFilter(filterType, 0.5f)
        gpuImage.setFilter(filter)
        
        // 实时预览
        val filteredBitmap = gpuImage.bitmapWithFilterApplied
        imagePreview.setImageBitmap(filteredBitmap)
        
        // 根据滤镜类型启用/禁用滑块
        filterSeekBar.isEnabled = filterType != GPUImageFilterManager.FilterType.FILTER_NONE
    }
    
    private fun setupSeekBar() {
        filterSeekBar.setOnSeekBarChangeListener(object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
                if (fromUser) {
                    // 0-100 映射到 0-1
                    val intensity = progress / 100f
                    applyFilterIntensity(currentFilterType, intensity)
                }
            }
            
            override fun onStartTrackingTouch(seekBar: SeekBar?) {}
            override fun onStopTrackingTouch(seekBar: SeekBar?) {}
        })
    }
    
    private fun applyFilterIntensity(filterType: GPUImageFilterManager.FilterType, intensity: Float) {
        val filter = filterManager.getFilter(filterType, intensity)
        gpuImage.setFilter(filter)
        imagePreview.setImageBitmap(gpuImage.bitmapWithFilterApplied)
    }
    
    private fun saveImage() {
        // 显示加载进度
        val progressDialog = ProgressDialog.show(this, "", "保存中...", true)
        
        AsyncTask.execute {
            // 获取最终图片
            val finalBitmap = gpuImage.bitmapWithFilterApplied
            
            // 保存到相册
            saveToGallery(finalBitmap)
            
            runOnUiThread {
                progressDialog.dismiss()
                Toast.makeText(this, "保存成功", Toast.LENGTH_SHORT).show()
                finish()
            }
        }
    }
    
    private fun saveToGallery(bitmap: Bitmap) {
        val savedUri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            ContentValues().apply {
                put(MediaStore.Images.Media.DISPLAY_NAME, "Edited_${System.currentTimeMillis()}.jpg")
                put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
            }
        )
        
        savedUri?.let { uri ->
            contentResolver.openOutputStream(uri)?.use { outputStream ->
                bitmap.compress(Bitmap.CompressFormat.JPEG, 95, outputStream)
            }
        }
    }
}
```

---

#### 4.9 性能优化建议

**避免 OOM：**

```kotlin
// 1. 在加载大图前先获取尺寸
fun getImageSize(uri: Uri): Pair<Int, Int>? {
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true  // 只获取尺寸，不加载到内存
    }
    contentResolver.openInputStream(uri)?.use { inputStream ->
        BitmapFactory.decodeStream(inputStream, null, options)
    }
    return if (options.outWidth > 0 && options.outHeight > 0) {
        Pair(options.outWidth, options.outHeight)
    } else null
}

// 2. 计算采样率
fun calculateInSampleSize(options: BitmapFactory.Options, reqWidth: Int, reqHeight: Int): Int {
    val (height, width) = options.outHeight to options.outWidth
    var inSampleSize = 1
    
    if (height > reqHeight || width > reqWidth) {
        val halfHeight = height / 2
        val halfWidth = width / 2
        while (halfHeight / inSampleSize >= reqHeight && halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}

// 3. 高效加载
fun loadOptimizedBitmap(uri: Uri, maxSize: Int): Bitmap? {
    return getImageSize(uri)?.let { (width, height) ->
        val options = BitmapFactory.Options().apply {
            inSampleSize = calculateInSampleSize(
                BitmapFactory.Options().apply {
                    inJustDecodeBounds = true
                    // 先获取尺寸...
                }.also { opts ->
                    contentResolver.openInputStream(uri)?.use { 
                        BitmapFactory.decodeStream(it, null, opts) 
                    }
                },
                maxSize, maxSize
            )
            inPreferredConfig = Bitmap.Config.ARGB_8888
        }
        contentResolver.openInputStream(uri)?.use { 
            BitmapFactory.decodeStream(it, null, options) 
        }
    }
}
```

---

#### 4.10 滤镜系统实战总结

| 滤镜类型 | 实现方式 | 性能 | 复杂度 |
|----------|----------|------|--------|
| 亮度/对比度/饱和度 | ColorMatrix | 中 | ⭐ |
| 灰度/反相/棕褐 | ColorMatrix | 中 | ⭐ |
| 色相旋转 | ColorMatrix | 中 | ⭐⭐ |
| 实时预览滤镜 | GPUImage | 高 | ⭐⭐ |
| 复杂艺术效果 | 自定义 Shader | 最高 | ⭐⭐⭐⭐ |
| 专业图像处理 | NDK/C++ | 最高 | ⭐⭐⭐⭐⭐ |

**推荐技术栈：**
- **入门级**（独立开发者）：ColorMatrix + 自定义滤镜
- **进阶级**（图片编辑 App）：GPUImage
- **商业级**（专业 App）：自研渲染引擎 / GPUImage + 自定义 Shader
- **专业级**：NDK + OpenGL ES / Vulkan

---

## 📚 五、软考备考
