## 每日知识 💡

> **本期主题：Android相册图像编辑 - 滤镜原理与ColorMatrix实战**
> 
> 结合用户当前Android相册开发项目，深入讲解图像滤镜的核心技术原理与实现

---

### 一、滤镜技术概述

在Android相册应用中，图像编辑功能是核心模块之一。滤镜（Filter）本质上是**对图像像素进行数学变换**，从而改变图像的视觉效果。常见的滤镜效果包括：

- **亮度调整**：改变像素的整体明暗程度
- **对比度调整**：增强或减弱像素之间的差异
- **饱和度调整**：增强或减弱色彩的鲜艳程度
- **色相调整**：改变颜色的基本色调
- **滤镜叠加**：如怀旧、黑白、复古等艺术效果

这些滤镜的底层实现，都离不开 **ColorMatrix（颜色矩阵）**。

---

### 二、ColorMatrix 颜色矩阵原理

#### 2.1 什么是ColorMatrix

`ColorMatrix` 是Android中用于处理图像颜色变换的类，本质上是一个 **4×5 的矩阵**。每个像素的颜色用 RGBA（红、绿、蓝、透明度）四个分量表示，范围都是 0-255 或 0-1。

#### 2.2 数学原理

对于任意像素 `[R, G, B, A]`，颜色矩阵变换的公式为：

```
R' = a*R + b*G + c*B + d*A + e
G' = f*R + g*G + h*B + i*A + j
B' = k*R + l*G + m*B + n*A + o
A' = p*R + q*G + r*B + s*A + t
```

用矩阵表示：

```
| R' |   | a  b  c  d  e |   | R |
| G' | = | f  g  h  i  j | × | G |
| B' |   | k  l  m  n  o |   | B |
| A' |   | p  q  r  s  t |   | A |
                               | 1 |
```

#### 2.3 常用颜色矩阵变换

**（1）亮度调整（Brightness）**

亮度调整的原理是将每个颜色分量加上一个常数。正值为变亮，负值为变暗。

```kotlin
/**
 * 创建亮度调整矩阵
 * @param brightness 亮度值，范围 -255 到 255
 *                  正值变亮，负值变暗
 */
fun createBrightnessMatrix(brightness: Float): ColorMatrix {
    val matrix = ColorMatrix()
    // 设置缩放为1（保持原有比例）
    matrix.setScale(1f, 1f, 1f, 1f)
    
    // 创建亮度调整的临时矩阵
    val brightnessMatrix = ColorMatrix(
        floatArrayOf(
            1f, 0f, 0f, 0f, brightness,   // R: 保持R，加上brightness
            0f, 1f, 0f, 0f, brightness,   // G: 保持G，加上brightness
            0f, 0f, 1f, 0f, brightness,   // B: 保持B，加上brightness
            0f, 0f, 0f, 1f, 0f            // A: 透明度不变
        )
    )
    
    // 将亮度矩阵与缩放矩阵相乘
    matrix.postConcat(brightnessMatrix)
    return matrix
}
```

**（2）对比度调整（Contrast）**

对比度调整是让颜色值远离或靠近中点(128)。

```kotlin
/**
 * 创建对比度调整矩阵
 * @param contrast 对比度值，范围 0 到 2
 *                 1.0 表示原始对比度
 *                 大于1增强对比度，小于1降低对比度
 */
fun createContrastMatrix(contrast: Float): ColorMatrix {
    // 对比度调整公式: output = ((input - 128) * contrast) + 128
    // 拆分为缩放 + 平移
    val scale = contrast
    val translate = (1f - contrast) * 128f
    
    return ColorMatrix(
        floatArrayOf(
            scale, 0f, 0f, 0f, translate,
            0f, scale, 0f, 0f, translate,
            0f, 0f, scale, 0f, translate,
            0f, 0f, 0f, 1f, 0f
        )
    )
}
```

**（3）饱和度调整（Saturation）**

饱和度控制色彩的鲜艳程度。

```kotlin
/**
 * 创建饱和度调整矩阵
 * @param saturation 饱和度值，范围 0 到 2
 *                    0 为灰度图
 *                    1 为原始饱和度
 *                    2 为高度饱和
 */
fun createSaturationMatrix(saturation: Float): ColorMatrix {
    val matrix = ColorMatrix()
    matrix.setSaturation(saturation)
    return matrix
}
```

**（4）色相调整（Hue）**

色相调整是让颜色在色轮上旋转。

```kotlin
/**
 * 创建色相调整矩阵
 * @param degrees 色相旋转角度，范围 0 到 360 度
 */
fun createHueMatrix(degrees: Float): ColorMatrix {
    val matrix = ColorMatrix()
    matrix.setRotate(0, degrees)  // 红色通道旋转
    matrix.setRotate(1, degrees)  // 绿色通道旋转
    matrix.setRotate(2, degrees)  // 蓝色通道旋转
    return matrix
}
```

---

### 三、组合滤镜实战

在实际应用中，通常需要组合多个滤镜效果。下面是一个完整的滤镜工具类：

```kotlin
/**
 * 滤镜效果工具类
 * 提供常用滤镜的创建方法
 */
object FilterUtils {

    /**
     * 调整图像亮度
     * @param brightness 亮度值 -255 到 255
     */
    fun adjustBrightness(brightness: Float): ColorMatrix {
        return ColorMatrix(
            floatArrayOf(
                1f, 0f, 0f, 0f, brightness,
                0f, 1f, 0f, 0f, brightness,
                0f, 0f, 1f, 0f, brightness,
                0f, 0f, 0f, 1f, 0f
            )
        )
    }

    /**
     * 调整图像对比度
     * @param contrast 对比度值 0 到 2
     */
    fun adjustContrast(contrast: Float): ColorMatrix {
        val translate = (1f - contrast) * 128f
        return ColorMatrix(
            floatArrayOf(
                contrast, 0f, 0f, 0f, translate,
                0f, contrast, 0f, 0f, translate,
                0f, 0f, contrast, 0f, translate,
                0f, 0f, 0f, 1f, 0f
            )
        )
    }

    /**
     * 调整图像饱和度
     * @param saturation 饱和度值 0 到 2
     */
    fun adjustSaturation(saturation: Float): ColorMatrix {
        val matrix = ColorMatrix()
        matrix.setSaturation(saturation)
        return matrix
    }

    /**
     * 组合多个滤镜效果
     * @param matrices 要组合的多个颜色矩阵
     */
    fun combineMatrices(vararg matrices: ColorMatrix): ColorMatrix {
        val result = ColorMatrix()
        matrices.forEach { matrix ->
            result.postConcat(matrix)
        }
        return result
    }

    /**
     * 怀旧滤镜（Sepia）
     * 将图像转换为棕褐色调，产生复古效果
     */
    fun sepiaMatrix(): ColorMatrix {
        return ColorMatrix(
            floatArrayOf(
                0.393f, 0.769f, 0.189f, 0f, 0f,
                0.349f, 0.686f, 0.168f, 0f, 0f,
                0.272f, 0.534f, 0.131f, 0f, 0f,
                0f, 0f, 0f, 1f, 0f
            )
        )
    }

    /**
     * 黑白滤镜（Grayscale）
     * 将图像转换为灰度图
     */
    fun grayscaleMatrix(): ColorMatrix {
        val matrix = ColorMatrix()
        matrix.setSaturation(0f)
        return matrix
    }

    /**
     * 负片滤镜（Negative/Invert）
     * 反转图像颜色
     */
    fun invertMatrix(): ColorMatrix {
        return ColorMatrix(
            floatArrayOf(
                -1f, 0f, 0f, 0f, 255f,
                0f, -1f, 0f, 0f, 255f,
                0f, 0f, -1f, 0f, 255f,
                0f, 0f, 0f, 1f, 0f
            )
        )
    }

    /**
     * 锐化滤镜（Sharpen）
     * 增强图像边缘细节
     */
    fun sharpenMatrix(): ColorMatrix {
        // 锐化核
        return ColorMatrix(
            floatArrayOf(
                0f, -1f, 0f, 0f, 0f,
                -1f, 5f, -1f, 0f, 0f,
                0f, -1f, 0f, 0f, 0f,
                0f, 0f, 0f, 1f, 0f
            )
        )
    }

    /**
     * 复古滤镜（Vintage）
     * 组合效果：降低饱和度 + 增加暖色调 + 轻微暗角
     */
    fun vintageMatrix(): ColorMatrix {
        // 降低饱和度
        val saturationMatrix = ColorMatrix().apply { setSaturation(0.3f) }
        
        // 增加暖色调
        val warmMatrix = ColorMatrix(
            floatArrayOf(
                1.2f, 0f, 0f, 0f, 10f,
                0f, 1.1f, 0f, 0f, 5f,
                0f, 0f, 0.9f, 0f, 0f,
                0f, 0f, 0f, 1f, 0f
            )
        )
        
        // 增加对比度
        val contrastMatrix = adjustContrast(1.15f)
        
        return combineMatrices(saturationMatrix, warmMatrix, contrastMatrix)
    }
}
```

---

### 四、在Android中应用滤镜

#### 4.1 使用ColorMatrixColorFilter

```kotlin
/**
 * 在自定义View中应用滤镜
 */
class FilterImageView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : AppCompatImageView(context, attrs, defStyleAttr) {

    private var colorMatrix: ColorMatrix? = null
    private var colorFilter: ColorMatrixColorFilter? = null

    /**
     * 应用滤镜效果
     * @param matrix 颜色矩阵
     */
    fun applyFilter(matrix: ColorMatrix) {
        this.colorMatrix = matrix
        colorFilter = ColorMatrixColorFilter(matrix)
        this.colorFilter = colorFilter
        invalidate() // 重绘视图
    }

    /**
     * 清除滤镜，恢复原图
     */
    fun clearFilter() {
        this.colorMatrix = null
        this.colorFilter = null
        this.colorFilter = null
        invalidate()
    }

    /**
     * 调整亮度
     */
    fun setBrightness(brightness: Float) {
        val matrix = FilterUtils.adjustBrightness(brightness)
        applyFilter(matrix)
    }

    /**
     * 调整对比度
     */
    fun setContrast(contrast: Float) {
        val matrix = FilterUtils.adjustContrast(contrast)
        applyFilter(matrix)
    }

    /**
     * 调整饱和度
     */
    fun setSaturation(saturation: Float) {
        val matrix = FilterUtils.adjustSaturation(saturation)
        applyFilter(matrix)
    }

    /**
     * 应用预设滤镜
     */
    fun applyPresetFilter(filterType: FilterType) {
        val matrix = when (filterType) {
            FilterType.SEPIA -> FilterUtils.sepiaMatrix()
            FilterType.GRAYSCALE -> FilterUtils.grayscaleMatrix()
            FilterType.INVERT -> FilterUtils.invertMatrix()
            FilterType.SHARPEN -> FilterUtils.sharpenMatrix()
            FilterType.VINTAGE -> FilterUtils.vintageMatrix()
        }
        applyFilter(matrix)
    }
}

/**
 * 滤镜类型枚举
 */
enum class FilterType {
    SEPIA,      // 怀旧
    GRAYSCALE,  // 黑白
    INVERT,     // 负片
    SHARPEN,    // 锐化
    VINTAGE     // 复古
}
```

#### 4.2 使用Paint应用滤镜到Bitmap

```kotlin
/**
 * Bitmap滤镜处理工具类
 */
object BitmapFilterProcessor {

    /**
     * 对Bitmap应用颜色矩阵滤镜
     * @param source 原始Bitmap
     * @param colorMatrix 颜色矩阵
     * @return 处理后的新Bitmap
     */
    fun applyColorMatrix(source: Bitmap, colorMatrix: ColorMatrix): Bitmap {
        // 创建输出Bitmap
        val output = Bitmap.createBitmap(
            source.width,
            source.height,
            Bitmap.Config.ARGB_8888
        )
        
        val canvas = Canvas(output)
        val paint = Paint().apply {
            isAntiAlias = true  // 抗锯齿
            colorFilter = ColorMatrixColorFilter(colorMatrix)
        }
        
        // 绘制应用了滤镜的图像
        canvas.drawBitmap(source, 0f, 0f, paint)
        
        return output
    }

    /**
     * 应用亮度调整
     */
    fun adjustBrightness(source: Bitmap, brightness: Float): Bitmap {
        val matrix = FilterUtils.adjustBrightness(brightness)
        return applyColorMatrix(source, matrix)
    }

    /**
     * 应用对比度调整
     */
    fun adjustContrast(source: Bitmap, contrast: Float): Bitmap {
        val matrix = FilterUtils.adjustContrast(contrast)
        return applyColorMatrix(source, matrix)
    }

    /**
     * 应用饱和度调整
     */
    fun adjustSaturation(source: Bitmap, saturation: Float): Bitmap {
        val matrix = FilterUtils.adjustSaturation(saturation)
        return applyColorMatrix(source, matrix)
    }

    /**
     * 应用预设滤镜
     */
    fun applyPreset(source: Bitmap, filterType: FilterType): Bitmap {
        val matrix = when (filterType) {
            FilterType.SEPIA -> FilterUtils.sepiaMatrix()
            FilterType.GRAYSCALE -> FilterUtils.grayscaleMatrix()
            FilterType.INVERT -> FilterUtils.invertMatrix()
            FilterType.SHARPEN -> FilterUtils.sharpenMatrix()
            FilterType.VINTAGE -> FilterUtils.vintageMatrix()
        }
        return applyColorMatrix(source, matrix)
    }

    /**
     * 应用组合滤镜
     */
    fun applyCombinedFilters(
        source: Bitmap,
        brightness: Float = 0f,
        contrast: Float = 1f,
        saturation: Float = 1f,
        presetFilter: FilterType? = null
    ): Bitmap {
        val matrices = mutableListOf<ColorMatrix>()
        
        if (brightness != 0f) {
            matrices.add(FilterUtils.adjustBrightness(brightness))
        }
        if (contrast != 1f) {
            matrices.add(FilterUtils.adjustContrast(contrast))
        }
        if (saturation != 1f) {
            matrices.add(FilterUtils.adjustSaturation(saturation))
        }
        presetFilter?.let {
            matrices.add(when (it) {
                FilterType.SEPIA -> FilterUtils.sepiaMatrix()
                FilterType.GRAYSCALE -> FilterUtils.grayscaleMatrix()
                FilterType.INVERT -> FilterUtils.invertMatrix()
                FilterType.SHARPEN -> FilterUtils.sharpenMatrix()
                FilterType.VINTAGE -> FilterUtils.vintageMatrix()
            })
        }
        
        if (matrices.isEmpty()) return source
        
        val combinedMatrix = FilterUtils.combineMatrices(*matrices.toTypedArray())
        return applyColorMatrix(source, combinedMatrix)
    }
}
```

---

### 五、性能优化建议

#### 5.1 避免频繁创建ColorMatrix

```kotlin
// ❌ 不推荐：每次滑动都创建新矩阵
slider.addOnChangeListener { _, value, _ ->
    imageView.colorFilter = ColorMatrixColorFilter(
        ColorMatrix().apply { setSaturation(value) }
    )
}

// ✅ 推荐：复用ColorMatrix对象
private val saturationMatrix = ColorMatrix()
slider.addOnChangeListener { _, value, _ ->
    saturationMatrix.setSaturation(value)
    imageView.colorFilter = ColorMatrixColorFilter(saturationMatrix)
}
```

#### 5.2 使用ARGB_8888还是RGB_565？

- **ARGB_8888**：每个像素32位，质量高，支持透明度
- **RGB_565**：每个像素16位，内存占用少50%，不支持透明度

对于照片编辑，**推荐使用ARGB_8888**，以保证滤镜效果的准确性。

#### 5.3 使用RenderScript进行高级滤镜

对于更复杂的滤镜（如模糊、锐化），可以使用RenderScript：

```kotlin
// 使用RenderScript进行高斯模糊
fun applyBlur(context: Context, source: Bitmap, radius: Float): Bitmap {
    val input = Allocation.createFromBitmap(source)
    val output = Allocation.createTyped(input.type)
    
    val script = ScriptC_blur(context).apply {
        // 设置模糊半径
        setRadius(radius.coerceIn(0f, 25f))
    }
    
    script.forEach_blur(input, output)
    output.copyTo(source)
    
    return source
}
```

---

### 六、总结

| 滤镜效果 | ColorMatrix实现 | 典型应用场景 |
|---------|---------------|-------------|
| 亮度 | 矩阵加法 | 曝光补偿 |
| 对比度 | 缩放+平移 | 增强视觉冲击力 |
| 饱和度 | setSaturation() | 色彩增强/减弱 |
| 色相 | 通道旋转 | 色调调整 |
| 怀旧 | 固定矩阵 | 复古风格 |
| 锐化 | 卷积核 | 细节增强 |

**核心要点：**
1. ColorMatrix是Android图像处理的基础
2. 通过矩阵运算实现各种颜色变换
3. 组合多个矩阵实现复杂滤镜效果
4. 注意性能优化，避免频繁创建对象

---

