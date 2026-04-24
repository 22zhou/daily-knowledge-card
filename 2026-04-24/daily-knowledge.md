## 🔧 每日知识

> 本板块详细讲解 Android 相册图像编辑开发

# Android 图像编辑实战：滤镜实现原理与自定义滤镜开发

作为Android相册开发的核心功能之一，图像滤镜能够为照片添加各种视觉效果。本文将深入讲解滤镜的实现原理，并带你实现一个完整的自定义滤镜系统。

## 一、滤镜的本质：颜色矩阵变换

### 1.1 什么是颜色矩阵

在Android中，每个像素的颜色由ARGB四个分量组成：
- **A (Alpha)**：透明度，0-255
- **R (Red)**：红色，0-255
- **G (Green)**：绿色，0-255
- **B (Blue)**：蓝色，0-255

颜色矩阵是一个4x5的矩阵，用于对颜色进行线性变换：

```
| R' |   | m00 m01 m02 m03 m04 |   | R |
| G' | = | m10 m11 m12 m13 m14 | × | G |
| B' |   | m20 m21 m22 m23 m24 |   | B |
| A' |   | m30 m31 m32 m33 m34 |   | A |
                                   | 1 |
```

变换公式：
```
R' = m00×R + m01×G + m02×B + m03×A + m04
G' = m10×R + m11×G + m12×B + m13×A + m14
B' = m20×R + m21×G + m22×B + m23×A + m24
A' = m30×R + m31×G + m32×B + m33×A + m34
```

### 1.2 单位矩阵（保持原样）

```java
// 单位矩阵 - 不改变任何颜色
public static final float[] IDENTITY_MATRIX = {
    1, 0, 0, 0, 0,  // R
    0, 1, 0, 0, 0,  // G
    0, 0, 1, 0, 0,  // B
    0, 0, 0, 1, 0   // A
};
```

### 1.3 亮度调整矩阵

```java
/**
 * 创建亮度调整矩阵
 * @param brightness 亮度值，范围 -255 到 255
 *                   负数变暗，正数变亮
 * @return 颜色矩阵
 */
public static float[] createBrightnessMatrix(float brightness) {
    // 亮度调整通过在矩阵的平移分量添加偏移实现
    return new float[] {
        1, 0, 0, 0, brightness,
        0, 1, 0, 0, brightness,
        0, 0, 1, 0, brightness,
        0, 0, 0, 1, 0
    };
}
```

### 1.4 对比度调整矩阵

```java
/**
 * 创建对比度调整矩阵
 * @param contrast 对比度值，范围 0 到 2
 *                 1 = 原图，<1 降低对比度，>1 增加对比度
 * @return 颜色矩阵
 */
public static float[] createContrastMatrix(float contrast) {
    // 对比度通过缩放矩阵实现
    float t = (1.0f - contrast) / 2.0f;
    return new float[] {
        contrast, 0, 0, 0, t * 255,
        0, contrast, 0, 0, t * 255,
        0, 0, contrast, 0, t * 255,
        0, 0, 0, 1, 0
    };
}
```

### 1.5 饱和度调整矩阵

```java
/**
 * 创建饱和度调整矩阵
 * @param saturation 饱和度值，范围 0 到 2
 *                   0 = 灰度图，1 = 原图，2 = 高饱和度
 * @return 颜色矩阵
 */
public static float[] createSaturationMatrix(float saturation) {
    // 灰度权重：人眼对绿色最敏感，蓝色最不敏感
    float gray = 0.2125f * (1 - saturation);
    float grayG = 0.7154f * (1 - saturation);
    float grayB = 0.0721f * (1 - saturation);
    
    return new float[] {
        gray + saturation, grayG, grayB, 0, 0,
        gray, grayG + saturation, grayB, 0, 0,
        gray, grayG, grayB + saturation, 0, 0,
        0, 0, 0, 1, 0
    };
}
```

## 二、滤镜的Android实现

### 2.1 滤镜工具类设计

```java
import android.graphics.ColorMatrix;

/**
 * 滤镜工具类
 * 提供各种预设滤镜和自定义滤镜能力
 */
public class FilterUtils {
    
    // ==================== 预设滤镜 ====================
    
    /**
     * 原图 - 不做任何处理
     */
    public static ColorMatrix getOriginal() {
        return new ColorMatrix();
    }
    
    /**
     * 灰度滤镜
     * 将图像转换为黑白照片效果
     */
    public static ColorMatrix getGrayscale() {
        ColorMatrix matrix = new ColorMatrix();
        matrix.setSaturation(0);
        return matrix;
    }
    
    /**
     * 怀旧滤镜
     * 模拟老照片的暖黄色调
     */
    public static ColorMatrix getVintage() {
        // 怀旧效果：降低蓝色，增加红色和绿色
        float[] vintageMatrix = {
            0.393f, 0.769f, 0.189f, 0, 0,
            0.349f, 0.686f, 0.168f, 0, 0,
            0.272f, 0.534f, 0.131f, 0, 0,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(vintageMatrix);
    }
    
    /**
     * 冷色调滤镜
     * 增加蓝色调，营造冷峻氛围
     */
    public static ColorMatrix getCool() {
        float[] coolMatrix = {
            1, 0, 0, 0, 0,
            0, 1, 0, 0, 0,
            0, 0, 1.2f, 0, 20,  // 增加蓝色分量
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(coolMatrix);
    }
    
    /**
     * 暖色调滤镜
     * 增加红黄色调，营造温暖氛围
     */
    public static ColorMatrix getWarm() {
        float[] warmMatrix = {
            1.2f, 0, 0, 0, 20,
            0, 1.1f, 0, 0, 10,
            0, 0, 0.9f, 0, 0,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(warmMatrix);
    }
    
    /**
     * 鲜艳滤镜
     * 增强色彩饱和度
     */
    public static ColorMatrix getVivid() {
        ColorMatrix matrix = new ColorMatrix();
        matrix.setSaturation(1.5f);
        return matrix;
    }
    
    /**
     * 日系小清新滤镜
     * 低对比度、高亮度、略偏蓝绿
     */
    public static ColorMatrix getJapaneseStyle() {
        // 日系风格特点：低饱和、略过曝、偏青
        float[] japaneseMatrix = {
            1.1f, 0.1f, 0.1f, 0, 15,
            0.05f, 1.1f, 0.05f, 0, 10,
            0.05f, 0.1f, 0.95f, 0, 0,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(japaneseMatrix);
    }
}
```

### 2.2 滤镜应用组件

```java
import android.content.Context;
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.ColorMatrix;
import android.graphics.ColorMatrixColorFilter;
import android.graphics.Paint;
import android.graphics.drawable.BitmapDrawable;
import android.util.AttributeSet;
import androidx.appcompat.widget.AppCompatImageView;

/**
 * 支持滤镜的ImageView
 * 用于实时预览滤镜效果
 */
public class FilterImageView extends AppCompatImageView {
    
    private ColorMatrix currentFilter = null;
    private Paint filterPaint = new Paint();
    
    public FilterImageView(Context context) {
        super(context);
        init();
    }
    
    public FilterImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }
    
    private void init() {
        // 开启抗锯齿
        filterPaint.setAntiAlias(true);
    }
    
    /**
     * 应用滤镜
     * @param filter 颜色矩阵
     */
    public void setFilter(ColorMatrix filter) {
        this.currentFilter = filter;
        if (filter != null) {
            filterPaint.setColorFilter(
                new ColorMatrixColorFilter(filter)
            );
        } else {
            filterPaint.setColorFilter(null);
        }
        invalidate();
    }
    
    /**
     * 获取当前滤镜
     */
    public ColorMatrix getCurrentFilter() {
        return currentFilter;
    }
    
    @Override
    protected void onDraw(Canvas canvas) {
        BitmapDrawable bd = (BitmapDrawable) getDrawable();
        if (bd == null || bd.getBitmap() == null) {
            super.onDraw(canvas);
            return;
        }
        
        Bitmap bitmap = bd.getBitmap();
        
        if (currentFilter != null) {
            // 绘制带滤镜的图像
            canvas.save();
            canvas.concat(getImageMatrix());
            canvas.drawBitmap(bitmap, 0, 0, filterPaint);
            canvas.restore();
        } else {
            super.onDraw(canvas);
        }
    }
}
```

### 2.3 滤镜渲染器（完整示例）

```java
import android.graphics.Bitmap;
import android.graphics.Canvas;
import android.graphics.ColorMatrix;
import android.graphics.ColorMatrixColorFilter;
import android.graphics.Paint;
import android.graphics.PixelFormat;
import android.graphics.PorterDuff;
import android.graphics.PorterDuffXfermode;

/**
 * 图像滤镜渲染器
 * 提供完整的滤镜处理流程
 */
public class ImageFilterRenderer {
    
    /**
     * 为Bitmap应用颜色矩阵滤镜
     * @param source 源图像
     * @param colorMatrix 颜色矩阵
     * @return 应用滤镜后的新Bitmap
     */
    public static Bitmap applyColorMatrix(Bitmap source, ColorMatrix colorMatrix) {
        if (source == null) {
            return null;
        }
        
        int width = source.getWidth();
        int height = source.getHeight();
        
        // 创建可变Bitmap用于保存结果
        Bitmap result = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        
        Canvas canvas = new Canvas(result);
        Paint paint = new Paint();
        paint.setAntiAlias(true);
        
        if (colorMatrix != null) {
            // 设置颜色滤镜
            ColorMatrixColorFilter colorFilter = new ColorMatrixColorFilter(colorMatrix);
            paint.setColorFilter(colorFilter);
        }
        
        // 绘制图像
        canvas.drawBitmap(source, 0, 0, paint);
        
        return result;
    }
    
    /**
     * 组合多个滤镜效果
     * @param source 源图像
     * @param matrices 多个颜色矩阵
     * @return 组合滤镜后的Bitmap
     */
    public static Bitmap combineFilters(Bitmap source, ColorMatrix... matrices) {
        if (source == null || matrices == null || matrices.length == 0) {
            return source;
        }
        
        // 将多个矩阵相乘得到组合矩阵
        ColorMatrix combinedMatrix = new ColorMatrix();
        for (ColorMatrix matrix : matrices) {
            combinedMatrix.postConcat(matrix);
        }
        
        return applyColorMatrix(source, combinedMatrix);
    }
    
    /**
     * 锐化滤镜
     * 通过增强像素对比实现锐化效果
     */
    public static ColorMatrix getSharpenMatrix() {
        // 锐化矩阵：中心像素增强，周围像素减弱
        float[] sharpenMatrix = {
            0, -1, 0, 0, 0,
            -1, 5, -1, 0, 0,
            0, -1, 0, 0, 0,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(sharpenMatrix);
    }
    
    /**
     * 边缘检测滤镜
     * 提取图像边缘
     */
    public static ColorMatrix getEdgeDetectionMatrix() {
        float[] edgeMatrix = {
            -1, -1, -1, 0, 255,
            -1, 8, -1, 0, 255,
            -1, -1, -1, 0, 255,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(edgeMatrix);
    }
    
    /**
     * 棕褐色滤镜（复古）
     */
    public static ColorMatrix getSepiaMatrix() {
        float[] sepiaMatrix = {
            0.393f, 0.769f, 0.189f, 0, 0,
            0.349f, 0.686f, 0.168f, 0, 0,
            0.272f, 0.534f, 0.131f, 0, 0,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(sepiaMatrix);
    }
    
    /**
     * 反色滤镜
     * 类似胶片负片效果
     */
    public static ColorMatrix getInvertMatrix() {
        float[] invertMatrix = {
            -1, 0, 0, 0, 255,
            0, -1, 0, 0, 255,
            0, 0, -1, 0, 255,
            0, 0, 0, 1, 0
        };
        return new ColorMatrix(invertMatrix);
    }
}
```

## 三、自定义滤镜开发实战

### 3.1 定义滤镜接口

```java
/**
 * 滤镜接口
 * 定义所有滤镜必须实现的方法
 */
public interface IImageFilter {
    
    /**
     * 获取滤镜名称
     */
    String getName();
    
    /**
     * 获取滤镜预览缩略图
     * @param source 源图像
     * @param width 预览图宽度
     * @param height 预览图高度
     */
    Bitmap getPreview(Bitmap source, int width, int height);
    
    /**
     * 应用滤镜到图像
     * @param source 源图像
     * @return 应用滤镜后的图像
     */
    Bitmap apply(Bitmap source);
    
    /**
     * 设置滤镜强度
     * @param intensity 强度值，范围 0.0 到 1.0
     */
    void setIntensity(float intensity);
    
    /**
     * 获取当前强度
     */
    float getIntensity();
}
```

### 3.2 实现亮度-对比度-饱和度滤镜

```java
/**
 * BCS滤镜：亮度、对比度、饱和度
 */
public class BcsFilter implements IImageFilter {
    
    private float brightness = 0;  // 亮度: -100 ~ 100
    private float contrast = 1;   // 对比度: 0 ~ 2
    private float saturation = 1; // 饱和度: 0 ~ 2
    
    @Override
    public String getName() {
        return "BCS";
    }
    
    @Override
    public Bitmap apply(Bitmap source) {
        // 构建组合矩阵
        ColorMatrix matrix = new ColorMatrix();
        
        // 应用饱和度变换
        if (saturation != 1) {
            ColorMatrix satMatrix = new ColorMatrix();
            satMatrix.setSaturation(saturation);
            matrix.postConcat(satMatrix);
        }
        
        // 应用对比度变换
        if (contrast != 1) {
            float scale = contrast;
            float translate = (1 - scale) * 128f;
            ColorMatrix contrastMatrix = new ColorMatrix(new float[] {
                scale, 0, 0, 0, translate,
                0, scale, 0, 0, translate,
                0, 0, scale, 0, translate,
                0, 0, 0, 1, 0
            });
            matrix.postConcat(contrastMatrix);
        }
        
        // 应用亮度变换
        if (brightness != 0) {
            // 将-100~100映射到-255~255
            float translate = brightness * 2.55f;
            ColorMatrix brightnessMatrix = new ColorMatrix(new float[] {
                1, 0, 0, 0, translate,
                0, 1, 0, 0, translate,
                0, 0, 1, 0, translate,
                0, 0, 0, 1, 0
            });
            matrix.postConcat(brightnessMatrix);
        }
        
        return ImageFilterRenderer.applyColorMatrix(source, matrix);
    }
    
    @Override
    public void setIntensity(float intensity) {
        // intensity 0.0~1.0 映射到对比度和饱和度的中心值
        this.contrast = 0.5f + intensity;
        this.saturation = 0.5f + intensity;
    }
    
    @Override
    public float getIntensity() {
        return (contrast + saturation) / 2f - 0.5f;
    }
    
    // Setter方法
    public void setBrightness(float brightness) {
        this.brightness = Math.max(-100, Math.min(100, brightness));
    }
    
    public void setContrast(float contrast) {
        this.contrast = Math.max(0, Math.min(2, contrast));
    }
    
    public void setSaturation(float saturation) {
        this.saturation = Math.max(0, Math.min(2, saturation));
    }
}
```

## 四、性能优化注意事项

### 4.1 避免频繁创建Bitmap

```java
// ❌ 错误：每次绘制都创建新Bitmap
public void onDraw(Canvas canvas) {
    Bitmap filtered = applyFilter(originalBitmap);
    canvas.drawBitmap(filtered, 0, 0, paint);
    // 忘记回收 filtered!
}

// ✅ 正确：复用Bitmap
private Bitmap filteredBitmap;

public void applyFilterToView() {
    // 复用或创建固定大小的Bitmap
    if (filteredBitmap == null || 
        filteredBitmap.getWidth() != originalBitmap.getWidth()) {
        filteredBitmap = Bitmap.createBitmap(
            originalBitmap.getWidth(),
            originalBitmap.getHeight(),
            Bitmap.Config.ARGB_8888
        );
    }
    applyFilterInPlace(originalBitmap, filteredBitmap);
}
```

### 4.2 使用RGB_565节省内存

```java
// 对于不需要透明通道的滤镜，可以使用RGB_565
Bitmap.Config config = hasAlpha ? 
    Bitmap.Config.ARGB_8888 : 
    Bitmap.Config.RGB_565;

Bitmap result = Bitmap.createBitmap(width, height, config);
```

### 4.3 子线程处理

```java
// 在协程中执行滤镜处理
CoroutineScope(Dispatchers.Default).launch {
    val filtered = withContext(Dispatchers.Default) {
        filter.apply(originalBitmap)
    }
    withContext(Dispatchers.Main) {
        imageView.setImageBitmap(filtered)
    }
}
```

## 五、总结

本文介绍了Android图像滤镜的核心原理和实现方法：

1. **颜色矩阵是基础**：所有滤镜本质上都是对ARGB颜色分量的线性变换
2. **矩阵组合实现复杂效果**：通过`postConcat`组合多个变换
3. **性能优化关键**：复用Bitmap、选择合适的像素格式、子线程处理
4. **接口设计**：定义统一的`IImageFilter`接口便于扩展

在实际项目中，可以继续扩展：
- LUT (Lookup Table) 滤镜实现更复杂效果
- OpenGL ES 硬件加速
- 滤镜链式调用

---

