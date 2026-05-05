# 每日知识

# 四、每日知识

> **主题：Android 图像编辑开发 — Bitmap 高效处理与滤镜实现**

本章节将深入讲解 Android 平台上图像编辑应用开发的核心技术，包括：

- Bitmap 内存优化策略
- Room 数据库存储图像元数据
- 滤镜算法的实现原理
- Canvas 绑图编辑核心代码

---

## 4.1 Bitmap 内存优化：BitmapFactory.Options 详解

### 问题背景
一张 4000×3000 像素的照片，采用 ARGB_8888 格式（每像素 4 字节），直接加载需要：
```
4000 × 3000 × 4 = 48 MB
```
如果同时加载多张高清图片，极易触发 OOM。

### 解决方案：BitmapFactory.Options 参数配置

```kotlin
/**
 * 图像采样压缩加载
 * 目标：在保证显示效果的前提下，大幅降低内存占用
 */
fun decodeSampledBitmapFromResource(
    context: Context,
    resId: Int,
    reqWidth: Int,    // 目标显示宽度（像素）
    reqHeight: Int    // 目标显示高度（像素）
): Bitmap {
    // 步骤1：只读取图片尺寸，不加载到内存
    val options = BitmapFactory.Options().apply {
        inJustDecodeBounds = true
    }
    BitmapFactory.decodeResource(context.resources, resId, options)

    // 步骤2：计算采样率
    options.inSampleSize = calculateInSampleSize(options, reqWidth, reqHeight)

    // 步骤3：使用采样率重新解码
    options.inJustDecodeBounds = false
    
    // 可选：使用 inBitmap 实现 Bitmap 复用（避免频繁分配内存）
    options.inBitmap = getReusableBitmap(options)
    
    return BitmapFactory.decodeResource(context.resources, resId, options)
}

/**
 * 计算采样率
 * inSampleSize = 2 表示宽高各缩放为原来的 1/2，图片变为原来的 1/4
 */
private fun calculateInSampleSize(
    options: BitmapFactory.Options,
    reqWidth: Int,
    reqHeight: Int
): Int {
    val (height: Int, width: Int) = options.outHeight to options.outWidth
    var inSampleSize = 1

    if (height > reqHeight || width > reqWidth) {
        val halfHeight: Int = height / 2
        val halfWidth: Int = width / 2

        // 计算最大的 inSampleSize 值，使宽高都 >= 目标尺寸
        while (halfHeight / inSampleSize >= reqHeight &&
               halfWidth / inSampleSize >= reqWidth) {
            inSampleSize *= 2
        }
    }
    return inSampleSize
}
```

### Bitmap 复用机制（Android 4.4+）

```kotlin
/**
 * Bitmap 复用池
 * 避免频繁创建/销毁 Bitmap，减少 GC 压力
 */
class BitmapPool(private val maxSize: Int) {
    private val reusableBitmaps = LinkedHashMap<String, Bitmap>()

    fun put(bitmap: Bitmap) {
        if (isReusable(bitmap)) {
            synchronized(reusableBitmaps) {
                reusableBitmaps[getKey(bitmap)] = bitmap
            }
            trimToSize(maxSize)
        }
    }

    fun get(width: Int, height: Int, config: Bitmap.Config?): Bitmap? {
        synchronized(reusableBitmaps) {
            val key = "${width}_${height}_${config}"
            return reusableBitmaps.remove(key)
        }
    }

    private fun isReusable(bitmap: Bitmap): Boolean {
        return bitmap.isMutable || 
               bitmap.config == Bitmap.Config.RGB_565 ||
               bitmap.hasAlpha() && (bitmap.config == Bitmap.Config.ARGB_4444)
    }
    
    private fun getKey(bitmap: Bitmap): String = 
        "${bitmap.width}_${bitmap.height}_${bitmap.config}"
}
```

### 内存计算参考表

| 格式 | 每像素字节 | 4000×3000 图片内存 |
|------|-----------|-------------------|
| ARGB_8888 | 4 字节 | 48 MB |
| RGB_565 | 2 字节 | 24 MB |
| ALPHA_8 | 1 字节 | 12 MB |

---

## 4.2 色彩调整算法：亮度、对比度、饱和度

### 核心原理
色彩调整的本质是对每个像素的 RGB 值进行数学变换：

```
R' = α × R + β        // 亮度调整
R' = (R - 128) × α + 128  // 对比度调整
```

### 完整实现代码

```kotlin
/**
 * 图像色彩调整器
 * 支持亮度、对比度、饱和度的实时调整
 */
class ColorMatrixHelper {
    
    /**
     * 调整亮度
     * @param brightness 值范围：-255 ~ 255
     * 正值变亮，负值变暗，0 为原始
     */
    fun adjustBrightness(brightness: Float): ColorMatrix {
        val matrix = ColorMatrix()
        matrix.set(
            floatArrayOf(
                1f, 0f, 0f, 0f, brightness,
                0f, 1f, 0f, 0f, brightness,
                0f, 0f, 1f, 0f, brightness,
                0f, 0f, 0f, 1f, 0f
            )
        )
        return matrix
    }

    /**
     * 调整对比度
     * @param contrast 值范围：0 ~ 2（1 为原始）
     * > 1 增强对比，< 1 减弱对比
     */
    fun adjustContrast(contrast: Float): ColorMatrix {
        val scale = contrast
        val translate = (-.5f * scale + .5f) * 255f
        
        return ColorMatrix(floatArrayOf(
            scale, 0f, 0f, 0f, translate,
            0f, scale, 0f, 0f, translate,
            0f, 0f, scale, 0f, translate,
            0f, 0f, 0f, 1f, 0f
        ))
    }

    /**
     * 调整饱和度
     * @param saturation 值范围：0 ~ 2（1 为原始）
     * 0 为灰度，1 为原始，> 1 增强色彩
     */
    fun adjustSaturation(saturation: Float): ColorMatrix {
        val matrix = ColorMatrix()
        matrix.setSaturation(saturation)
        return matrix
    }

    /**
     * 组合多个效果
     */
    fun combineEffects(
        brightness: Float = 0f,
        contrast: Float = 1f,
        saturation: Float = 1f
    ): ColorMatrix {
        val brightnessMatrix = adjustBrightness(brightness)
        val contrastMatrix = adjustContrast(contrast)
        val saturationMatrix = adjustSaturation(saturation)
        
        // 按顺序叠加矩阵
        val combined = ColorMatrix()
        combined.postConcat(brightnessMatrix)
        combined.postConcat(contrastMatrix)
        combined.postConcat(saturationMatrix)
        
        return combined
    }
}
```

### 应用到 ImageView

```kotlin
class ImageFilterView(context: Context) : AppCompatImageView(context) {
    
    private val colorMatrixHelper = ColorMatrixHelper()
    private val paint = Paint()
    private var sourceBitmap: Bitmap? = null
    
    // 当前色彩矩阵
    private var currentMatrix = ColorMatrix()
    
    /**
     * 应用色彩调整效果
     */
    fun applyColorAdjust(
        brightness: Float = 0f,
        contrast: Float = 1f,
        saturation: Float = 1f
    ) {
        currentMatrix = colorMatrixHelper.combineEffects(
            brightness, contrast, saturation
        )
        invalidate()
    }

    override fun onDraw(canvas: Canvas) {
        sourceBitmap?.let { bitmap ->
            val imageMatrix = ColorMatrixColorFilter(currentMatrix)
            paint.colorFilter = imageMatrix
            canvas.drawBitmap(bitmap, 0f, 0f, paint)
        } ?: super.onDraw(canvas)
    }

    /**
     * 设置源图片（带内存优化）
     */
    fun setSourceBitmap(bitmap: Bitmap, maxWidth: Int = 1080) {
        // 限制最大尺寸，防止内存溢出
        val scale = minOf(1f, maxWidth.toFloat() / bitmap.width)
        if (scale < 1f) {
            sourceBitmap = Bitmap.createScaledBitmap(
                bitmap,
                (bitmap.width * scale).toInt(),
                (bitmap.height * scale).toInt(),
                true
            )
        } else {
            sourceBitmap = bitmap
        }
        invalidate()
    }
}
```

---

## 4.3 Room 数据库存储图像元数据

### 为什么需要元数据库？
图像编辑应用需要存储：
- 编辑历史（滤镜参数、裁剪数据）
- 用户收藏和标签
- 最近编辑列表

### 完整实现

```kotlin
// ==================== Entity 定义 ====================

@Entity(tableName = "edited_images")
data class EditedImageEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Long = 0,
    
    @ColumnInfo(name = "original_uri")
    val originalUri: String,  // 原始图片 URI
    
    @ColumnInfo(name = "edited_path")
    val editedPath: String,   // 编辑后保存路径
    
    @ColumnInfo(name = "thumbnail_path")
    val thumbnailPath: String?, // 缩略图路径（节省内存）
    
    @ColumnInfo(name = "filter_type")
    val filterType: String,  // 使用的滤镜类型
    
    // 编辑参数 JSON（亮度、对比度、饱和度等）
    @ColumnInfo(name = "edit_params")
    val editParams: String,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),
    
    @ColumnInfo(name = "updated_at")
    val updatedAt: Long = System.currentTimeMillis()
)

// ==================== DAO 定义 ====================

@Dao
interface EditedImageDao {
    
    @Query("SELECT * FROM edited_images ORDER BY updated_at DESC")
    fun getAllImages(): Flow<List<EditedImageEntity>>
    
    @Query("SELECT * FROM edited_images WHERE id = :id")
    suspend fun getImageById(id: Long): EditedImageEntity?
    
    @Query("SELECT * FROM edited_images ORDER BY updated_at DESC LIMIT :limit")
    fun getRecentImages(limit: Int): Flow<List<EditedImageEntity>>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertImage(image: EditedImageEntity): Long
    
    @Update
    suspend fun updateImage(image: EditedImageEntity)
    
    @Delete
    suspend fun deleteImage(image: EditedImageEntity)
    
    @Query("DELETE FROM edited_images WHERE created_at < :timestamp")
    suspend fun deleteOldImages(timestamp: Long)
    
    @Query("SELECT * FROM edited_images WHERE filter_type = :filterType")
    fun getImagesByFilter(filterType: String): Flow<List<EditedImageEntity>>
}

// ==================== Database 定义 ====================

@Database(
    entities = [EditedImageEntity::class],
    version = 1,
    exportSchema = false
)
abstract class ImageEditDatabase : RoomDatabase() {
    abstract fun editedImageDao(): EditedImageDao
    
    companion object {
        @Volatile
        private var INSTANCE: ImageEditDatabase? = null
        
        fun getInstance(context: Context): ImageEditDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    ImageEditDatabase::class.java,
                    "image_edit_database"
                )
                    .fallbackToDestructiveMigration() // 版本升级时删除旧表
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// ==================== Repository ====================

class ImageEditRepository(context: Context) {
    
    private val database = ImageEditDatabase.getInstance(context)
    private val dao = database.editedImageDao()
    
    // Flow 实现响应式数据流，配合 Compose 或 LiveData 使用
    val allImages: Flow<List<EditedImageEntity>> = dao.getAllImages()
    
    fun getRecentImages(limit: Int = 20): Flow<List<EditedImageEntity>> {
        return dao.getRecentImages(limit)
    }
    
    suspend fun saveEditedImage(
        originalUri: String,
        editedPath: String,
        filterType: String,
        brightness: Float,
        contrast: Float,
        saturation: Float
    ): Long {
        val editParams = EditParams(brightness, contrast, saturation)
        val entity = EditedImageEntity(
            originalUri = originalUri,
            editedPath = editedPath,
            thumbnailPath = null,
            filterType = filterType,
            editParams = Json.encodeToString(editParams.serializer(), editParams)
        )
        return dao.insertImage(entity)
    }
    
    // 清理 30 天前的编辑记录
    suspend fun cleanupOldRecords() {
        val thirtyDaysAgo = System.currentTimeMillis() - 30L * 24 * 60 * 60 * 1000
        dao.deleteOldImages(thirtyDaysAgo)
    }
}

// ==================== 数据类 ====================

@Serializable
data class EditParams(
    val brightness: Float = 0f,
    val contrast: Float = 1f,
    val saturation: Float = 1f,
    val hue: Float = 0f,
    val warmth: Float = 0f
)
```

### ViewModel 集成

```kotlin
class ImageEditViewModel(private val repository: ImageEditRepository) : ViewModel() {
    
    val recentImages: StateFlow<List<EditedImageEntity>> = 
        repository.getRecentImages(20)
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5000),
                initialValue = emptyList()
            )
    
    fun saveEdit(
        originalUri: String,
        editedPath: String,
        params: EditParams
    ) = viewModelScope.launch {
        repository.saveEditedImage(
            originalUri = originalUri,
            editedPath = editedPath,
            filterType = "custom",
            brightness = params.brightness,
            contrast = params.contrast,
            saturation = params.saturation
        )
    }
}
```

---

## 4.4 常见滤镜实现（对比度增强、怀旧、黑白）

```kotlin
/**
 * 预设滤镜集合
 */
object PresetFilters {
    
    /**
     * 原始（无滤镜）
     */
    fun none(): ColorMatrix = ColorMatrix()
    
    /**
     * 黑白滤镜
     */
    fun blackAndWhite(): ColorMatrix {
        val matrix = ColorMatrix()
        matrix.setSaturation(0f)
        return matrix
    }
    
    /**
     * 怀旧/复古滤镜
     * 降低饱和度，增加暖色调
     */
    fun nostalgic(): ColorMatrix {
        // 先去饱和度
        val saturation = ColorMatrix()
        saturation.setSaturation(0.6f)
        
        // 再叠加棕褐色调
        val sepia = ColorMatrix(floatArrayOf(
            1f, 0f, 0f, 0f, 0f,
            0f, 1f, 0f, 0f, 0f,
            0f, 0f, 1f, 0f, 0f,
            0f, 0f, 0f, 1f, 0f
        ))
        // 叠加效果
        saturation.postConcat(sepia)
        
        // 整体偏暖
        val warmMatrix = ColorMatrix(floatArrayOf(
            1.1f, 0f, 0f, 0f, 10f,
            0f, 1f, 0f, 0f, 5f,
            0f, 0f, 0.9f, 0f, 0f,
            0f, 0f, 0f, 1f, 0f
        ))
        saturation.postConcat(warmMatrix)
        
        return saturation
    }
    
    /**
     * 鲜亮滤镜（增强饱和度和对比度）
     */
    fun vivid(): ColorMatrix {
        val saturation = ColorMatrix()
        saturation.setSaturation(1.3f) // 饱和度 +30%
        
        val contrast = ColorMatrix(floatArrayOf(
            1.2f, 0f, 0f, 0f, -25f,
            0f, 1.2f, 0f, 0f, -25f,
            0f, 0f, 1.2f, 0f, -25f,
            0f, 0f, 0f, 1f, 0f
        ))
        
        saturation.postConcat(contrast)
        return saturation
    }
    
    /**
     * 冷色调滤镜
     */
    fun cool(): ColorMatrix {
        return ColorMatrix(floatArrayOf(
            1f, 0f, 0f, 0f, 0f,
            0f, 1f, 0f, 0f, 0f,
            0f, 0f, 1.2f, 0f, 20f,  // B 通道增强
            0f, 0f, 0f, 1f, 0f
        ))
    }
    
    /**
     * 暖色调滤镜
     */
    fun warm(): ColorMatrix {
        return ColorMatrix(floatArrayOf(
            1.2f, 0f, 0f, 0f, 20f,  // R 通道增强
            0f, 1.1f, 0f, 0f, 10f,
            0f, 0f, 1f, 0f, 0f,
            0f, 0f, 0f, 1f, 0f
        ))
    }
}
```

---

## 4.5 实战：完整的滤镜编辑 Activity

```kotlin
class PhotoEditActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityPhotoEditBinding
    private lateinit var viewModel: PhotoEditViewModel
    
    // 当前滤镜类型
    private var currentFilter: FilterType = FilterType.NONE
    
    // 滤镜列表
    private val filters = listOf(
        FilterItem("原图", FilterType.NONE, PresetFilters::none),
        FilterItem("黑白", FilterType.BLACK_WHITE, PresetFilters::blackAndWhite),
        FilterItem("怀旧", FilterType.NOSTALGIC, PresetFilters::nostalgic),
        FilterItem("鲜亮", FilterType.VIVID, PresetFilters::vivid),
        FilterItem("冷色", FilterType.COOL, PresetFilters::cool),
        FilterItem("暖色", FilterType.WARM, PresetFilters::warm)
    )
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityPhotoEditBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        loadImage()
        setupFilterList()
        setupSliders()
        setupButtons()
    }
    
    private fun loadImage() {
        val uri = intent.getStringExtra(EXTRA_IMAGE_URI)?.toUri()
        uri?.let {
            // 从 URI 加载图片（带采样优化）
            val bitmap = contentResolver.openInputStream(it)?.use { stream ->
                decodeSampledBitmapFromStream(stream, 1080, 1920)
            }
            bitmap?.let { bmp ->
                binding.imageView.setSourceBitmap(bmp)
            }
        }
    }
    
    private fun setupFilterList() {
        binding.filterRecyclerView.apply {
            layoutManager = LinearLayoutManager(
                this@PhotoEditActivity,
                LinearLayoutManager.HORIZONTAL,
                false
            )
            adapter = FilterAdapter(filters) { filterItem ->
                applyFilter(filterItem)
            }
        }
    }
    
    private fun setupSliders() {
        binding.sliderBrightness.addOnChangeListener { _, value, _ ->
            applyColorAdjust(
                brightness = value,
                contrast = binding.sliderContrast.value,
                saturation = binding.sliderSaturation.value
            )
        }
        
        binding.sliderContrast.addOnChangeListener { _, value, _ ->
            applyColorAdjust(
                brightness = binding.sliderBrightness.value,
                contrast = value,
                saturation = binding.sliderSaturation.value
            )
        }
        
        binding.sliderSaturation.addOnChangeListener { _, value, _ ->
            applyColorAdjust(
                brightness = binding.sliderBrightness.value,
                contrast = binding.sliderContrast.value,
                saturation = value
            )
        }
    }
    
    private fun applyFilter(filterItem: FilterItem) {
        currentFilter = filterItem.type
        val matrix = filterItem.filter()
        binding.imageView.setColorFilter(ColorMatrixColorFilter(matrix))
    }
    
    private fun applyColorAdjust(brightness: Float, contrast: Float, saturation: Float) {
        if (currentFilter != FilterType.NONE) return // 滤镜优先
        
        binding.imageView.applyColorAdjust(brightness, contrast, saturation)
    }
    
    private fun setupButtons() {
        binding.btnReset.setOnClickListener {
            binding.sliderBrightness.value = 0f
            binding.sliderContrast.value = 1f
            binding.sliderSaturation.value = 1f
            currentFilter = FilterType.NONE
            binding.imageView.applyColorAdjust(0f, 1f, 1f)
        }
        
        binding.btnSave.setOnClickListener {
            saveEditedImage()
        }
    }
    
    private fun saveEditedImage() {
        lifecycleScope.launch {
            val bitmap = binding.imageView.getCurrentBitmap()
            val savedPath = saveBitmapToGallery(bitmap)
            
            if (savedPath != null) {
                Toast.makeText(this@PhotoEditActivity, "保存成功", Toast.LENGTH_SHORT).show()
            } else {
                Toast.makeText(this@PhotoEditActivity, "保存失败", Toast.LENGTH_SHORT).show()
            }
        }
    }
    
    private fun saveBitmapToGallery(bitmap: Bitmap): String? {
        val filename = "IMG_${System.currentTimeMillis()}.jpg"
        return try {
            val savedUri = contentResolver.insert(
                MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
                ContentValues().apply {
                    put(MediaStore.Images.Media.DISPLAY_NAME, filename)
                    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
                    put(MediaStore.Images.Media.RELATIVE_PATH, "Pictures/PhotoEdit")
                }
            )
            
            savedUri?.let { uri ->
                contentResolver.openOutputStream(uri)?.use { out ->
                    bitmap.compress(Bitmap.CompressFormat.JPEG, 95, out)
                }
                uri.toString()
            }
        } catch (e: Exception) {
            e.printStackTrace()
            null
        }
    }
}

enum class FilterType { NONE, BLACK_WHITE, NOSTALGIC, VIVID, COOL, WARM }

data class FilterItem(
    val name: String,
    val type: FilterType,
    val filter: () -> ColorMatrix
)

class FilterAdapter(
    private val filters: List<FilterItem>,
    private val onFilterSelected: (FilterItem) -> Unit
) : RecyclerView.Adapter<FilterAdapter.ViewHolder>() {
    
    // ... RecyclerView 实现省略
}
```

---

## 4.6 性能优化小结

| 优化点 | 具体做法 |
|--------|---------|
| **内存优化** | 使用 `inSampleSize` 采样压缩；启用 `Bitmap` 复用池 |
| **线程安全** | 图像处理在 `Dispatchers.Default` 或自定义线程池执行 |
| **预览优化** | 预览时使用小图，导出时再处理原图 |
| **GC 控制** | 及时 `recycle()` 不需要的 Bitmap；避免在 `onDraw` 中创建对象 |
| **数据库** | 使用 Flow 响应式查询；定期清理过期记录 |

---

