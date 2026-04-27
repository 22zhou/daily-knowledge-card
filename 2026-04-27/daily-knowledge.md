## 四、每日知识（重点板块）

# Android相册开发：图像滤镜与特效实现详解

> 本节深入讲解Android平台上图像滤镜和特效的核心技术，包括色彩矩阵、卷积核、OpenGL ES着色器等，带你从原理到实战掌握图像编辑能力。

### 4.1 色彩矩阵（ColorMatrix）基础

色彩矩阵是Android图像处理中最基础也是最强大的工具。通过4x5矩阵可以精确控制图像的亮度、对比度、饱和度、色相等属性。

**色彩矩阵原理：**

```
| R' |   | a b c d e |   | R |
| G' | = | f g h i j | × | G |
| B' |   | k l m n o |   | B |
| A' |   | p q r s t |   | A |
```

其中：
- a,b,c,d 控制红色通道
- e 是红色偏移量
- 其他同理

**亮度矩阵：**
```java
/**
 * 创建亮度调整矩阵
 * @param brightness 亮度值，范围 -255 到 255
 * @return 亮度调整色彩矩阵
 */
public static ColorMatrix createBrightnessMatrix(float brightness) {
    ColorMatrix matrix = new ColorMatrix();
    // 设置缩放因子为1（保持原色）
    matrix.set(new float[] {
        1, 0, 0, 0, brightness,  // R: 1*R + brightness
        0, 1, 0, 0, brightness,  // G: 1*G + brightness
        0, 0, 1, 0, brightness,  // B: 1*B + brightness
        0, 0, 0, 1, 0           // A: 保持不变
    });
    return matrix;
}
```

**对比度矩阵：**
```java
/**
 * 创建对比度调整矩阵
 * @param contrast 对比度值，范围 0 到 2（1为原图）
 * @return 对比度调整色彩矩阵
 */
public static ColorMatrix createContrastMatrix(float contrast) {
    float t = (1f - contrast) / 2f * 255f;
    
    ColorMatrix matrix = new ColorMatrix();
    matrix.set(new float[] {
        contrast, 0, 0, 0, t,
        0, contrast, 0, 0, t,
        0, 0, contrast, 0, t,
        0, 0, 0, 1, 0
    });
    return matrix;
}
```

**饱和度矩阵：**
```java
/**
 * 创建饱和度调整矩阵
 * @param saturation 饱和度值，范围 0 到 2（1为原图，0为灰度）
 * @return 饱和度调整色彩矩阵
 */
public static ColorMatrix createSaturationMatrix(float saturation) {
    ColorMatrix matrix = new ColorMatrix();
    matrix.setSaturation(saturation);
    return matrix;
}
```

### 4.2 图像混合模式（Blend Mode）

Android的`PorterDuff`混合模式可以实现各种图层叠加效果。

**常用混合模式：**
```java
/**
 * 图像混合效果枚举
 */
public enum BlendMode {
    // 正常混合
    NORMAL,
    // 变暗：取两图像中较暗的像素
    DARKEN,
    // 变亮：取两图像中较亮的像素
    LIGHTEN,
    // 正片叠底：类似于重叠两张透明胶片
    MULTIPLY,
    // 滤色：与正片叠底相反
    SCREEN,
    // 叠加：结合正片叠底和滤色效果
    OVERLAY,
    // 柔光：产生柔光效果
    SOFT_LIGHT,
    // 强光：产生强光效果
    HARD_LIGHT
}

/**
 * 应用混合模式到图像
 * @param baseBitmap 基础图像（底层）
 * @param overlayBitmap 叠加图像（顶层）
 * @param mode 混合模式
 * @return 混合后的图像
 */
public Bitmap applyBlendMode(Bitmap baseBitmap, Bitmap overlayBitmap, BlendMode mode) {
    Bitmap result = Bitmap.createBitmap(
        baseBitmap.getWidth(),
        baseBitmap.getHeight(),
        Bitmap.Config.ARGB_8888
    );
    
    Canvas canvas = new Canvas(result);
    canvas.drawBitmap(baseBitmap, 0, 0, null);
    
    // 创建混合模式Paint
    Paint paint = new Paint();
    paint.setXfermode(getPorterDuffXfermode(mode));
    
    canvas.drawBitmap(overlayBitmap, 0, 0, paint);
    
    return result;
}

/**
 * 将BlendMode转换为PorterDuffXfermode
 */
private PorterDuffXfermode getPorterDuffXfermode(BlendMode mode) {
    switch (mode) {
        case MULTIPLY:
            return new PorterDuffXfermode(PorterDuff.Mode.MULTIPLY);
        case SCREEN:
            return new PorterDuffXfermode(PorterDuff.Mode.SCREEN);
        case OVERLAY:
            return new PorterDuffXfermode(PorterDuff.Mode.OVERLAY);
        // ... 其他模式
        default:
            return new PorterDuffXfermode(PorterDuff.Mode.SRC_OVER);
    }
}
```

### 4.3 卷积核滤镜（Convolution Kernel）

卷积核是实现模糊、锐化、边缘检测等效果的核心技术。

**卷积原理：**

对于图像中的每个像素，用其周围像素的加权平均来替代原像素值：

```
R'(x,y) = Σ W(i,j) × R(x+i, y+j)
其中 W 是卷积核，i,j 是核内偏移
```

**常用卷积核：**
```java
/**
 * 卷积核类型枚举
 */
public enum KernelType {
    // 3×3 锐化核 - 增强边缘细节
    SHARPEN(new float[] {
        0, -1, 0,
        -1, 5, -1,
        0, -1, 0
    }),
    
    // 3×3 模糊核（Box Blur）- 平均模糊
    BLUR(new float[] {
        1/9f, 1/9f, 1/9f,
        1/9f, 1/9f, 1/9f,
        1/9f, 1/9f, 1/9f
    }),
    
    // 高斯模糊核 - 自然模糊效果
    GAUSSIAN_BLUR(new float[] {
        1/16f, 2/16f, 1/16f,
        2/16f, 4/16f, 2/16f,
        1/16f, 2/16f, 1/16f
    }),
    
    // 边缘检测核（Sobel X）- 检测垂直边缘
    EDGE_DETECT_X(new float[] {
        -1, 0, 1,
        -2, 0, 2,
        -1, 0, 1
    }),
    
    // 浮雕效果
    EMBOSS(new float[] {
        -2, -1, 0,
        -1, 1, 1,
        0, 1, 2
    })
}

/**
 * 应用卷积核滤镜
 */
public class ConvolutionFilter {
    
    /**
     * 对图像应用卷积核
     * @param source 源图像
     * @param kernelType 卷积核类型
     * @param edgeTreatment 边缘处理方式
     * @return 处理后的图像
     */
    public static Bitmap apply(Bitmap source, KernelType kernelType, EdgeTreatment edgeTreatment) {
        int width = source.getWidth();
        int height = source.getHeight();
        float[] kernel = kernelType.kernel;
        
        Bitmap result = Bitmap.createBitmap(width, height, Bitmap.Config.ARGB_8888);
        
        int[] pixels = new int[width * height];
        int[] resultPixels = new int[width * height];
        source.getPixels(pixels, 0, width, 0, 0, width, height);
        
        int halfKernel = kernel.length / 2;
        
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                float a = 0, r = 0, g = 0, b = 0;
                
                for (int ky = 0; ky < kernel.length; ky++) {
                    for (int kx = 0; kx < kernel.length; kx++) {
                        // 计算采样位置
                        int sx = x + kx - halfKernel;
                        int sy = y + ky - halfKernel;
                        
                        // 边缘处理
                        sx = edgeTreatment.clamp(sx, 0, width - 1);
                        sy = edgeTreatment.clamp(sy, 0, height - 1);
                        
                        int pixel = pixels[sy * width + sx];
                        float weight = kernel[ky * kernel.length + kx];
                        
                        a += ((pixel >> 24) & 0xFF) * weight;
                        r += ((pixel >> 16) & 0xFF) * weight;
                        g += ((pixel >> 8) & 0xFF) * weight;
                        b += (pixel & 0xFF) * weight;
                    }
                }
                
                resultPixels[y * width + x] = 
                    ((int) Math.max(0, Math.min(255, a)) << 24) |
                    ((int) Math.max(0, Math.min(255, r)) << 16) |
                    ((int) Math.max(0, Math.min(255, g)) << 8) |
                    (int) Math.max(0, Math.min(255, b));
            }
        }
        
        result.setPixels(resultPixels, 0, width, 0, 0, width, height);
        return result;
    }
    
    /**
     * 边缘处理策略接口
     */
    public interface EdgeTreatment {
        int clamp(int value, int min, int max);
    }
}
```

### 4.4 GPU加速滤镜（RenderScript & OpenGL ES）

对于复杂的滤镜效果，推荐使用GPU进行加速处理。

**RenderScript实现高斯模糊：**
```java
/**
 * RenderScript高斯模糊
 * 优点：自动利用GPU/NPU加速
 * 缺点：需要NDK支持
 */
public class RenderScriptBlur {
    private final RenderScript rs;
    private final ScriptIntrinsicBlur script;
    
    public RenderScriptBlur(Context context) {
        rs = RenderScript.create(context);
        // 创建高斯模糊内置脚本
        script = ScriptIntrinsicBlur.create(rs, Element.U8_4(rs));
    }
    
    /**
     * 执行高斯模糊
     * @param source 源图像
     * @param radius 模糊半径，范围 1-25
     * @return 模糊后的图像
     */
    public Bitmap blur(Bitmap source, float radius) {
        // 创建输入输出Allocation
        Allocation input = Allocation.createFromBitmap(rs, source);
        Allocation output = Allocation.createTyped(rs, input.getType());
        
        // 设置模糊半径（最大25）
        script.setRadius(Math.min(25f, radius));
        
        // 执行模糊
        script.setInput(input);
        script.forEach(output);
        
        // 复制结果
        Bitmap result = Bitmap.createBitmap(source.getWidth(), source.getHeight(), 
            Bitmap.Config.ARGB_8888);
        output.copyTo(result);
        
        // 释放资源
        input.destroy();
        output.destroy();
        
        return result;
    }
    
    public void destroy() {
        script.destroy();
        rs.destroy();
    }
}
```

**OpenGL ES着色器实现（推荐用于实时预览）：**

```java
/**
 * OpenGL ES图像处理着色器
 * 顶点着色器：simple_vertex.glsl
 */
public class ImageFilterRenderer {
    
    // 顶点着色器代码
    private static final String VERTEX_SHADER = 
        "attribute vec4 aPosition;\n" +
        "attribute vec2 aTextureCoord;\n" +
        "varying vec2 vTextureCoord;\n" +
        "void main() {\n" +
        "    gl_Position = aPosition;\n" +
        "    vTextureCoord = aTextureCoord;\n" +
        "}\n";
    
    // 片段着色器 - 饱和度调整
    private static final String FRAGMENT_SHADER_SATURATION = 
        "precision mediump float;\n" +
        "varying vec2 vTextureCoord;\n" +
        "uniform sampler2D sTexture;\n" +
        "uniform float saturation;\n" +
        "void main() {\n" +
        "    vec4 color = texture2D(sTexture, vTextureCoord);\n" +
        "    float gray = dot(color.rgb, vec3(0.299, 0.587, 0.114));\n" +
        "    vec3 grayColor = vec3(gray, gray, gray);\n" +
        "    vec3 satColor = mix(grayColor, color.rgb, saturation);\n" +
        "    gl_FragColor = vec4(satColor, color.a);\n" +
        "}\n";
    
    // 片段着色器 - 暖色调
    private static final String FRAGMENT_SHADER_WARM = 
        "precision mediump float;\n" +
        "varying vec2 vTextureCoord;\n" +
        "uniform sampler2D sTexture;\n" +
        "uniform float intensity; // 0.0 ~ 1.0\n" +
        "void main() {\n" +
        "    vec4 color = texture2D(sTexture, vTextureCoord);\n" +
        "    color.r = min(1.0, color.r + intensity * 0.1);\n" +
        "    color.g = color.g;\n" +
        "    color.b = max(0.0, color.b - intensity * 0.1);\n" +
        "    gl_FragColor = color;\n" +
        "}\n";
    
    // 片段着色器 - 晕影（Vignette）
    private static final String FRAGMENT_SHADER_VIGNETTE = 
        "precision mediump float;\n" +
        "varying vec2 vTextureCoord;\n" +
        "uniform sampler2D sTexture;\n" +
        "uniform float strength;\n" +
        "uniform float radius;\n" +
        "void main() {\n" +
        "    vec4 color = texture2D(sTexture, vTextureCoord);\n" +
        "    vec2 center = vec2(0.5, 0.5);\n" +
        "    float dist = distance(vTextureCoord, center);\n" +
        "    float factor = smoothstep(radius, radius - 0.2, dist);\n" +
        "    color.rgb = mix(color.rgb * (1.0 - strength), color.rgb, factor);\n" +
        "    gl_FragColor = color;\n" +
        "}\n";
}
```

### 4.5 滤镜链与实时预览

将多个滤镜组合成链，实现复杂的图像处理效果：

```java
/**
 * 滤镜链 - 组合多个滤镜效果
 */
public class FilterChain {
    private final List<Filter> filters = new ArrayList<>();
    
    /**
     * 添加滤镜到链中
     */
    public FilterChain add(Filter filter) {
        filters.add(filter);
        return this;
    }
    
    /**
     * 应用滤镜链到图像
     */
    public Bitmap apply(Bitmap source) {
        Bitmap current = source;
        for (Filter filter : filters) {
            current = filter.apply(current);
        }
        return current;
    }
    
    /**
     * 获取链中滤镜数量
     */
    public int size() {
        return filters.size();
    }
    
    /**
     * 滤镜接口
     */
    public interface Filter {
        Bitmap apply(Bitmap source);
    }
}

/**
 * 滤镜链使用示例
 */
public class PhotoEditorExample {
    
    public void applyPresetEffect(Bitmap source) {
        FilterChain chain = new FilterChain()
            // 1. 调整亮度
            .add(source1 -> {
                ColorMatrix matrix = ImageColorFilter.createBrightnessMatrix(20);
                return ImageColorFilter.applyColorMatrix(source1, matrix);
            })
            // 2. 提高对比度
            .add(source2 -> {
                ColorMatrix matrix = ImageColorFilter.createContrastMatrix(1.2f);
                return ImageColorFilter.applyColorMatrix(source2, matrix);
            })
            // 3. 增加饱和度
            .add(source3 -> {
                ColorMatrix matrix = ImageColorFilter.createSaturationMatrix(1.3f);
                return ImageColorFilter.applyColorMatrix(source3, matrix);
            })
            // 4. 添加暖色调
            .add(source4 -> {
                RenderScript rs = RenderScript.create(context);
                // 使用自定义Script执行
                return source4; // 实际返回处理后的结果
            });
        
        Bitmap result = chain.apply(source);
        // 显示或保存result
    }
}
```

### 4.6 性能优化建议

1. **使用BitmapFactory.Options**：
```java
// 缩放加载大图
BitmapFactory.Options options = new BitmapFactory.Options();
options.inSampleSize = 2; // 缩小为1/2
options.inPreferredConfig = Bitmap.Config.RGB_565; // 减少内存占用
Bitmap thumbnail = BitmapFactory.decodeFile(path, options);
```

2. **避免在主线程处理**：使用`AsyncTask`、`HandlerThread`或`Kotlin Coroutines`

3. **缓存中间结果**：重复使用的滤镜结果应该缓存

4. **使用GPU加速**：复杂滤镜优先使用RenderScript或OpenGL ES

---

