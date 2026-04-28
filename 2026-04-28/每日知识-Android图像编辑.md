# Android 相册开发：图像编辑核心实现指南

*每日知识卡片 - 2026年4月28日*

# 📚 每日知识（重点）

## Android 相册开发：图像编辑核心实现指南

### 一、图像编辑模块概述

Android 相册应用中的图像编辑功能通常包括：
- **基础调整**：亮度、对比度、饱和度
- **滤镜效果**：复古、黑白、暖色调等
- **裁剪旋转**：自由裁剪、固定比例、90度旋转
- **涂鸦标注**：画笔、马赛克、文字添加

今天我们重点讲解 **基础图像调整和滤镜实现**。

### 二、技术选型

```gradle
// build.gradle.kts (Module: app)
dependencies {
    // RenderScript 用于高性能图像处理
    implementation("androidx.renderscript:renderscript:1.0.0")
    
    // Android GPU Image 库（可选）
    implementation("jp.co.cyberagent.android:gpuimage:2.1.0")
    
    // 协程支持
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
}
```

### 三、核心类设计

#### 1. 图像调整处理器

```kotlin
package com.example.photoeditor

import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.ColorMatrix
import android.graphics.ColorMatrixColorFilter
import android.graphics.Paint
import android.renderscript.Allocation
import android.renderscript.Element
import android.renderscript.RenderScript
import android.renderscript.ScriptIntrinsicColorMatrix
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

/**
 * 图像调整器
 * 支持亮度、对比度、饱和度的实时调整
 * 使用 RenderScript 进行 GPU 加速处理
 */
class ImageAdjustProcessor(private val renderScript: RenderScript) {
    
    // 颜色矩阵用于图像调整
    private val colorMatrix = ColorMatrix()
    private val paint = Paint().apply {
        isAntiAlias = true
    }
    
    /**
     * 调整图像参数
     * @param source 源Bitmap
     * @param brightness 亮度值范围 -100 到 100
     * @param contrast 对比度值范围 -100 到 100
     * @param saturation 饱和度值范围 -100 到 100
     * @return 调整后的Bitmap
     */
    suspend fun adjustImage(
        source: Bitmap,
        brightness: Float = 0f,
        contrast: Float = 0f,
        saturation: Float = 0f
    ): Bitmap = withContext(Dispatchers.Default) {
        // 创建输出Bitmap
        val output = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(output)
        
        // 构建颜色矩阵
        buildColorMatrix(brightness, contrast, saturation)
        
        // 应用颜色矩阵
        paint.colorFilter = ColorMatrixColorFilter(colorMatrix)
        canvas.drawBitmap(source, 0f, 0f, paint)
        
        output
    }
    
    /**
     * 构建颜色矩阵
     * 颜色矩阵的原理：将颜色表示为 [R, G, B, A] 四维向量
     * 通过 4x4 矩阵进行线性变换实现各种效果
     */
    private fun buildColorMatrix(brightness: Float, contrast: Float, saturation: Float) {
        colorMatrix.reset()
        
        // 1. 亮度调整：通过对 R/G/B 分别加上偏移量实现
        // 亮度范围 -100 到 100，转换为 -255 到 255
        val brightnessValue = brightness * 2.55f
        val brightnessMatrix = ColorMatrix(floatArrayOf(
            1f, 0f, 0f, 0f, brightnessValue,
            0f, 1f, 0f, 0f, brightnessValue,
            0f, 0f, 1f, 0f, brightnessValue,
            0f, 0f, 0f, 1f, 0f
        ))
        colorMatrix.postConcat(brightnessMatrix)
        
        // 2. 对比度调整：围绕中点(128)进行缩放
        // 对比度值转换为 scale 和 offset
        val contrastScale = (contrast + 100f) / 100f
        val translate = (-.5f * contrastScale + .5f) * 255f
        val contrastMatrix = ColorMatrix(floatArrayOf(
            contrastScale, 0f, 0f, 0f, translate,
            0f, contrastScale, 0f, 0f, translate,
            0f, 0f, contrastScale, 0f, translate,
            0f, 0f, 0f, 1f, 0f
        ))
        colorMatrix.postConcat(contrastMatrix)
        
        // 3. 饱和度调整：使用专用方法
        // saturation = 0 时为灰度图，100 时为原图
        val satMatrix = ColorMatrix()
        satMatrix.setSaturation((saturation + 100f) / 100f)
        colorMatrix.postConcat(satMatrix)
    }
}
```

#### 2. 滤镜处理器

```kotlin
package com.example.photoeditor

import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.ColorMatrix
import android.graphics.ColorMatrixColorFilter
import android.graphics.Paint

/**
 * 滤镜处理器
 * 实现多种预设滤镜效果
 */
class FilterProcessor {
    
    // 滤镜类型枚举
    enum class FilterType {
        NONE,
        VINTAGE,      // 复古
        BLACK_WHITE,  // 黑白
        WARM,         // 暖色调
        COOL,         // 冷色调
        SEPIA,        // 怀旧（褐色）
        VIVID,        // 鲜艳
        PASTORAL      // 清新
    }
    
    private val paint = Paint().apply {
        isAntiAlias = true
    }
    
    /**
     * 应用滤镜
     * @param source 源Bitmap
     * @param filterType 滤镜类型
     * @return 应用滤镜后的Bitmap
     */
    fun applyFilter(source: Bitmap, filterType: FilterType): Bitmap {
        if (filterType == FilterType.NONE) return source
        
        val output = Bitmap.createBitmap(source.width, source.height, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(output)
        
        val matrix = getFilterColorMatrix(filterType)
        paint.colorFilter = ColorMatrixColorFilter(matrix)
        canvas.drawBitmap(source, 0f, 0f, paint)
        
        return output
    }
    
    /**
     * 获取滤镜颜色矩阵
     * 每种滤镜本质上是一个特定的颜色变换矩阵
     */
    private fun getFilterColorMatrix(filterType: FilterType): ColorMatrix {
        return when (filterType) {
            FilterType.VINTAGE -> {
                // 复古风格：降低饱和度，稍微偏暖
                ColorMatrix().apply {
                    setSaturation(0.7f)  // 降低饱和度
                    val warmMatrix = ColorMatrix(floatArrayOf(
                        1.1f, 0f, 0f, 0f, 10f,
                        0f, 1.0f, 0f, 0f, 5f,
                        0f, 0f, 0.9f, 0f, -10f,
                        0f, 0f, 0f, 1f, 0f
                    ))
                    postConcat(warmMatrix)
                }
            }
            
            FilterType.BLACK_WHITE -> {
                // 黑白滤镜
                ColorMatrix().apply {
                    setSaturation(0f)
                }
            }
            
            FilterType.WARM -> {
                // 暖色调：增强红色和黄色
                ColorMatrix(floatArrayOf(
                    1.2f, 0f, 0f, 0f, 20f,
                    0f, 1.1f, 0f, 0f, 10f,
                    0f, 0f, 0.9f, 0f, -10f,
                    0f, 0f, 0f, 1f, 0f
                ))
            }
            
            FilterType.COOL -> {
                // 冷色调：增强蓝色
                ColorMatrix(floatArrayOf(
                    0.9f, 0f, 0f, 0f, -10f,
                    0f, 1.0f, 0f, 0f, 0f,
                    0f, 0f, 1.2f, 0f, 20f,
                    0f, 0f, 0f, 1f, 0f
                ))
            }
            
            FilterType.SEPIA -> {
                // 怀旧褐色调
                ColorMatrix(floatArrayOf(
                    0.393f, 0.769f, 0.189f, 0f, 0f,
                    0.349f, 0.686f, 0.168f, 0f, 0f,
                    0.272f, 0.534f, 0.131f, 0f, 0f,
                    0f, 0f, 0f, 1f, 0f
                ))
            }
            
            FilterType.VIVID -> {
                // 鲜艳模式
                ColorMatrix().apply {
                    setSaturation(1.5f)  // 超饱和
                    val contrastMatrix = ColorMatrix(floatArrayOf(
                        1.2f, 0f, 0f, 0f, -20f,
                        0f, 1.2f, 0f, 0f, -20f,
                        0f, 0f, 1.2f, 0f, -20f,
                        0f, 0f, 0f, 1f, 0f
                    ))
                    postConcat(contrastMatrix)
                }
            }
            
            FilterType.PASTORAL -> {
                // 清新风格：略微过曝，低饱和
                ColorMatrix().apply {
                    setSaturation(0.6f)
                    val lightMatrix = ColorMatrix(floatArrayOf(
                        1.1f, 0f, 0f, 0f, 15f,
                        0f, 1.1f, 0f, 0f, 15f,
                        0f, 0f, 1.1f, 0f, 15f,
                        0f, 0f, 0f, 1f, 0f
                    ))
                    postConcat(lightMatrix)
                }
            }
            
            else -> ColorMatrix()  // 无效果
        }
    }
    
    /**
     * 获取滤镜名称
     */
    fun getFilterName(filterType: FilterType): String {
        return when (filterType) {
            FilterType.NONE -> "原图"
            FilterType.VINTAGE -> "复古"
            FilterType.BLACK_WHITE -> "黑白"
            FilterType.WARM -> "暖阳"
            FilterType.COOL -> "清凉"
            FilterType.SEPIA -> "怀旧"
            FilterType.VIVID -> "鲜艳"
            FilterType.PASTORAL -> "清新"
        }
    }
}
```

#### 3. 图像变换处理器

```kotlin
package com.example.photoeditor

import android.graphics.Bitmap
import android.graphics.Canvas
import android.graphics.Matrix
import android.graphics.Paint
import kotlin.math.roundToInt

/**
 * 图像变换处理器
 * 支持旋转、翻转、缩放等操作
 */
class TransformProcessor {
    
    private val paint = Paint().apply {
        isAntiAlias = true
        isFilterBitmap = true  // 启用双线性过滤，缩放时更平滑
    }
    
    /**
     * 旋转图像
     * @param source 源Bitmap
     * @param degrees 旋转角度（正值顺时针）
     * @return 旋转后的Bitmap
     */
    fun rotate(source: Bitmap, degrees: Float): Bitmap {
        val matrix = Matrix().apply {
            postRotate(degrees)
        }
        
        // 计算旋转后的尺寸
        val width = source.width
        val height = source.height
        val cos = kotlin.math.cos(Math.toRadians(degrees.toDouble())).toFloat().absoluteValue
        val sin = kotlin.math.sin(Math.toRadians(degrees.toDouble())).toFloat().absoluteValue
        
        val newWidth = (width * cos + height * sin).roundToInt()
        val newHeight = (width * sin + height * cos).roundToInt()
        
        matrix.postTranslate(newWidth / 2f - width / 2f, newHeight / 2f - height / 2f)
        
        val output = Bitmap.createBitmap(newWidth, newHeight, Bitmap.Config.ARGB_8888)
        val canvas = Canvas(output)
        canvas.drawBitmap(source, matrix, paint)
        
        return output
    }
    
    /**
     * 水平翻转
     */
    fun flipHorizontal(source: Bitmap): Bitmap {
        val matrix = Matrix().apply {
            preScale(-1f, 1f)
        }
        return Bitmap.createBitmap(source, 0, 0, source.width, source.height, matrix, true)
    }
    
    /**
     * 垂直翻转
     */
    fun flipVertical(source: Bitmap): Bitmap {
        val matrix = Matrix().apply {
            preScale(1f, -1f)
        }
        return Bitmap.createBitmap(source, 0, 0, source.width, source.height, matrix, true)
    }
    
    /**
     * 缩放图像
     * @param source 源Bitmap
     * @param scaleX 水平缩放比例
     * @param scaleY 垂直缩放比例
     */
    fun scale(source: Bitmap, scaleX: Float, scaleY: Float): Bitmap {
        val newWidth = (source.width * scaleX).roundToInt()
        val newHeight = (source.height * scaleY).roundToInt()
        return Bitmap.createScaledBitmap(source, newWidth, newHeight, true)
    }
    
    /**
     * 裁剪图像
     * @param source 源Bitmap
     * @param x 裁剪起点X
     * @param y 裁剪起点Y
     * @param width 裁剪宽度
     * @param height 裁剪高度
     */
    fun crop(source: Bitmap, x: Int, y: Int, width: Int, height: Int): Bitmap {
        // 边界检查
        val cropX = x.coerceAtLeast(0)
        val cropY = y.coerceAtLeast(0)
        val cropWidth = width.coerceAtMost(source.width - cropX)
        val cropHeight = height.coerceAtMost(source.height - cropY)
        
        return Bitmap.createBitmap(source, cropX, cropY, cropWidth, cropHeight)
    }
}
```

### 四、使用示例

```kotlin
package com.example.photoeditor

import android.os.Bundle
import android.widget.SeekBar
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

class PhotoEditorActivity : AppCompatActivity() {
    
    private lateinit var imageView: android.widget.ImageView
    private lateinit var brightnessSeekBar: SeekBar
    private lateinit var contrastSeekBar: SeekBar
    private lateinit var saturationSeekBar: SeekBar
    
    // 图像处理器
    private val adjustProcessor by lazy { 
        ImageAdjustProcessor(android.renderscript.RenderScript.create(this))
    }
    private val filterProcessor = FilterProcessor()
    private val transformProcessor = TransformProcessor()
    
    // 当前编辑的图像
    private var originalBitmap: Bitmap? = null
    private var currentBitmap: Bitmap? = null
    
    // 当前调整参数
    private var currentBrightness = 0f
    private var currentContrast = 0f
    private var currentSaturation = 0f
    private var currentFilter = FilterProcessor.FilterType.NONE
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_photo_editor)
        
        // 加载图片
        loadImage()
        
        // 初始化视图
        initViews()
        
        // 设置调整监听
        setupAdjustmentListeners()
    }
    
    private fun loadImage() {
        // 从 Intent 或其他方式获取图片路径
        val imagePath = intent.getStringExtra("image_path") ?: return
        originalBitmap = android.graphics.BitmapFactory.decodeFile(imagePath)
        currentBitmap = originalBitmap
        imageView.setImageBitmap(currentBitmap)
    }
    
    private fun initViews() {
        imageView = findViewById(R.id.imageView)
        brightnessSeekBar = findViewById(R.id.brightnessSeekBar)
        contrastSeekBar = findViewById(R.id.contrastSeekBar)
        saturationSeekBar = findViewById(R.id.saturationSeekBar)
        
        // 设置 SeekBar 范围 -100 到 100
        listOf(brightnessSeekBar, contrastSeekBar, saturationSeekBar).forEach { seekBar ->
            seekBar.max = 200
            seekBar.progress = 100  // 中心位置
        }
    }
    
    private fun setupAdjustmentListeners() {
        val seekBarChangeListener = object : SeekBar.OnSeekBarChangeListener {
            override fun onProgressChanged(seekBar: SeekBar?, progress: Int, fromUser: Boolean) {
                if (fromUser) {
                    // 将 progress (0-200) 转换为实际值 (-100 到 100)
                    val value = progress - 100
                    when (seekBar) {
                        brightnessSeekBar -> currentBrightness = value.toFloat()
                        contrastSeekBar -> currentContrast = value.toFloat()
                        saturationSeekBar -> currentSaturation = value.toFloat()
                    }
                    applyAdjustments()
                }
            }
            
            override fun onStartTrackingTouch(seekBar: SeekBar?) {}
            override fun onStopTrackingTouch(seekBar: SeekBar?) {}
        }
        
        brightnessSeekBar.setOnSeekBarChangeListener(seekBarChangeListener)
        contrastSeekBar.setOnSeekBarChangeListener(seekBarChangeListener)
        saturationSeekBar.setOnSeekBarChangeListener(seekBarChangeListener)
    }
    
    /**
     * 应用当前调整参数
     * 使用协程在后台线程处理，避免阻塞UI
     */
    private fun applyAdjustments() {
        originalBitmap ?: return
        
        lifecycleScope.launch {
            // 先应用滤镜
            var tempBitmap = filterProcessor.applyFilter(originalBitmap!!, currentFilter)
            
            // 再应用调整
            tempBitmap = adjustProcessor.adjustImage(
                source = tempBitmap,
                brightness = currentBrightness,
                contrast = currentContrast,
                saturation = currentSaturation
            )
            
            currentBitmap = tempBitmap
            imageView.setImageBitmap(currentBitmap)
        }
    }
    
    /**
     * 应用滤镜
     */
    fun applyFilter(filterType: FilterProcessor.FilterType) {
        currentFilter = filterType
        applyAdjustments()
    }
    
    /**
     * 重置所有调整
     */
    fun resetAll() {
        currentBrightness = 0f
        currentContrast = 0f
        currentSaturation = 0f
        currentFilter = FilterProcessor.FilterType.NONE
        
        brightnessSeekBar.progress = 100
        contrastSeekBar.progress = 100
        saturationSeekBar.progress = 100
        
        currentBitmap = originalBitmap
        imageView.setImageBitmap(currentBitmap)
    }
    
    /**
     * 保存编辑结果
     */
    fun saveImage() {
        currentBitmap ?: return
        
        lifecycleScope.launch {
            // 在后台线程保存
            withContext(Dispatchers.IO) {
                val outputPath = getExternalFilesDir(null)?.absolutePath + "/edited_${System.currentTimeMillis()}.jpg"
                currentBitmap?.compress(Bitmap.CompressFormat.JPEG, 95, java.io.FileOutputStream(outputPath))
            }
            
            // 显示保存成功
            android.widget.Toast.makeText(this@PhotoEditorActivity, "图片已保存", android.widget.Toast.LENGTH_SHORT).show()
        }
    }
}
```

### 五、性能优化建议

```kotlin
/**
 * 图像处理性能优化工具类
 */
object ImageProcessorOptimize {
    
    /**
     * 创建缩略图用于预览
     * 避免在编辑过程中处理原图大小的Bitmap
     */
    fun createThumbnail(source: Bitmap, maxSize: Int = 1024): Bitmap {
        val ratio = minOf(
            maxSize.toFloat() / source.width,
            maxSize.toFloat() / source.height
        )
        
        if (ratio >= 1f) return source
        
        val newWidth = (source.width * ratio).toInt()
        val newHeight = (source.height * ratio).toInt()
        
        return Bitmap.createScaledBitmap(source, newWidth, newHeight, true)
    }
    
    /**
     * 回收Bitmap内存
     */
    fun recycleBitmap(bitmap: Bitmap?) {
        if (bitmap != null && !bitmap.isRecycled) {
            bitmap.recycle()
        }
    }
    
    /**
     * 计算内存使用量
     */
    fun getBitmapMemorySize(bitmap: Bitmap): Int {
        // ARGB_8888 格式每个像素4字节
        return bitmap.width * bitmap.height * 4
    }
}
```

### 六、技术总结

| 功能 | 实现方式 | 关键类 |
|------|----------|--------|
| 亮度调整 | ColorMatrix 平移 | `ColorMatrix.postConcat()` |
| 对比度调整 | ColorMatrix 缩放 | 围绕中点(128)缩放 |
| 饱和度调整 | `setSaturation()` | ColorMatrix |
| 滤镜 | 预设 ColorMatrix | 固定参数矩阵 |
| 旋转/翻转 | Matrix 变换 | `postRotate()`/`preScale()` |
| 裁剪 | `createBitmap()` | 指定起点和尺寸 |

**性能优化要点**：
1. 使用协程在后台线程处理图像
2. 编辑预览时使用缩略图
3. 及时回收不再使用的Bitmap
4. 复杂处理考虑 RenderScript GPU 加速

---

