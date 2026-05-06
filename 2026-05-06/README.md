# 📖 每日知识：Jetpack Compose NestedScrollConnection 实现吸顶效果

**日期**: 2026年5月6日  
**阅读时长**: 约15分钟  
**标签**: Android | Jetpack Compose | UI开发

---

## 一、为什么选择 NestedScrollConnection？

在现代 Android 开发中，吸顶效果（sticky header）是常见的 UI 模式。本节将深入讲解如何利用 Compose 的 NestedScrollConnection 实现高性能的吸顶效果，并与传统的 CoordinatorLayout + AppBarLayout 方案进行对比。

传统的 CoordinatorLayout 方案虽然成熟，但在 Compose 中存在几个问题：
1. 需要维护 XML 和 Compose 混合代码
2. 状态同步复杂
3. 性能开销相对较大

NestedScrollConnection 是 Compose 提供的新 API，优势明显：
- **纯 Kotlin/Compose 实现**，无 XML 依赖
- **状态管理更简单**
- **性能更优**

## 二、核心原理

NestedScrollConnection 是 NestedScrollDispatcher 的核心组件，它允许你"拦截"嵌套滚动事件。理解三个关键概念：

### 滚动消费机制

```
┌─────────────────────────────────────────┐
│           View Hierarchy                │
├─────────────────────────────────────────┤
│  Parent (NestedScrollConnection)        │
│    ├── 接收 pre-scroll 事件（滚动前）    │
│    ├── 消费 dy 的一部分                  │
│    └── 接收 post-scroll 事件（滚动后）   │
│                                         │
│  Child (实际滚动的列表)                  │
│    ├── 滚动前询问父视图是否消费 dy       │
│    ├── 执行自己的滚动                     │
│    └── 滚动后通知父视图                  │
└─────────────────────────────────────────┘
```

### 核心回调方法

| 方法 | 时机 | 用途 |
|------|------|------|
| `onPreScroll` | 滚动发生前 | 拦截并消费滚动量 |
| `onPostScroll` | 滚动发生后 | 处理剩余滚动量 |
| `onPreFling` | 惯性滑动前 | 拦截 fling 事件 |
| `onPostFling` | 惯性滑动后 | 处理剩余 fling |

## 三、完整代码实现

```kotlin
import androidx.compose.animation.AnimatedVisibility
import androidx.compose.animation.slideInVertically
import androidx.compose.animation.slideOutVertically
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Modifier
import androidx.compose.ui.geometry.Offset
import androidx.compose.ui.input.nestedscroll.NestedScrollConnection
import androidx.compose.ui.input.nestedscroll.NestedScrollSource
import androidx.compose.ui.input.nestedscroll.nestedScroll
import androidx.compose.ui.platform.LocalDensity
import androidx.compose.ui.unit.Dp
import androidx.compose.ui.unit.dp

/**
 * 吸顶效果的实现类
 * 
 * 工作原理：
 * 1. 列表向上滚动时，当header即将完全滚出屏幕，触发"吸顶"
 * 2. 吸顶后，header固定在顶部，列表继续滚动
 * 3. 列表向下滚动时，header从顶部滚出，恢复正常布局
 */
class StickyHeaderNestedScrollConnection(
    private val headerHeight: Dp,
    private val onStickyStateChange: (Boolean) -> Unit
) : NestedScrollConnection {
    
    // 记录当前已滚动的偏移量
    private var scrollOffset = 0f
    
    // 标记是否处于吸顶状态
    private var isSticky = false
    
    private val headerHeightPx: Float
        @Composable get() = with(LocalDensity.current) { headerHeight.toPx() }
    
    override fun onPreScroll(
        available: Offset,
        source: NestedScrollSource
    ): Offset {
        // 这里是关键：我们先于列表处理滚动事件
        // available.y < 0 表示向上滚动（列表内容向上移动 = 手指向上滑）
        // available.y > 0 表示向下滚动
        
        val delta = available.y
        
        // 计算新的偏移量
        val newOffset = (scrollOffset + delta).coerceIn(
            minimumValue = 0f,
            maximumValue = headerHeightPx
        )
        
        // 计算我们实际消费的滚动量
        val consumed = newOffset - scrollOffset
        scrollOffset = newOffset
        
        // 更新吸顶状态
        updateStickyState()
        
        // 返回消耗的滚动量，告诉系统我们处理了一部分
        // 这样列表实际滚动的距离会减少
        return Offset(0f, consumed)
    }
    
    override fun onPostScroll(
        consumed: Offset,
        available: Offset,
        source: NestedScrollSource
    ) {
        // 列表滚动后会触发这个回调
        // available 是列表没有消费完的滚动量
        // 但在我们的场景中，onPreScroll 已经消费了大部分，所以这里通常为 0
    }
    
    private fun updateStickyState() {
        val shouldBeSticky = scrollOffset >= headerHeightPx - 1f // 允许1px误差
        if (shouldBeSticky != isSticky) {
            isSticky = shouldBeSticky
            onStickyStateChange(isSticky)
        }
    }
}

/**
 * 使用吸顶效果的 LazyColumn
 * 
 * @param headerHeight header的高度
 * @param headerContent 顶部header的内容
 * @param listContent 列表内容
 */
@Composable
fun StickyHeaderLazyColumn(
    headerHeight: Dp = 56.dp,
    headerContent: @Composable () -> Unit,
    listContent: @Composable () -> Unit
) {
    var isSticky by remember { mutableStateOf(false) }
    
    // 创建 NestedScrollConnection
    val nestedScrollConnection = remember {
        StickyHeaderNestedScrollConnection(
            headerHeight = headerHeight,
            onStickyStateChange = { sticky -> isSticky = sticky }
        )
    }
    
    Box(
        modifier = Modifier.fillMaxSize()
    ) {
        // 正常的列表内容
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .nestedScroll(nestedScrollConnection)
        ) {
            // Header 占位，保持空间
            item {
                Spacer(modifier = Modifier.height(headerHeight))
            }
            
            // 列表内容
            listContent()
        }
        
        // 吸顶的 Header
        AnimatedVisibility(
            visible = isSticky,
            enter = slideInVertically(initialOffsetY = { -it }),
            exit = slideOutVertically(targetOffsetY = { -it })
        ) {
            Surface(
                modifier = Modifier.fillMaxWidth(),
                shadowElevation = 4.dp,
                color = MaterialTheme.colorScheme.surface
            ) {
                headerContent()
            }
        }
        
        // 非吸顶状态的 Header（只在非吸顶时显示）
        if (!isSticky) {
            headerContent()
        }
    }
}
```

## 四、进阶：带滑动隐藏的吸顶效果

有时我们想要实现"向上滚动隐藏header，向下滚动显示header"的效果：

```kotlin
/**
 * 带滑动隐藏功能的 NestedScrollConnection
 * 
 * 应用场景：类似 Google Play 商店的商品详情页
 * - 向上滚动：header 逐渐隐藏
 * - 向下滚动：header 逐渐显示
 */
class ScrollHideNestedScrollConnection(
    private val headerHeight: Dp,
    private val onOffsetChange: (Float) -> Unit
) : NestedScrollConnection {
    
    private var totalScrollOffset = 0f
    
    // 隐藏状态的阈值（完全隐藏需要的偏移量）
    private var hideThreshold = 0f
    
    @Composable
    private fun getHideThreshold(): Float {
        return with(LocalDensity.current) {
            if (hideThreshold == 0f) {
                hideThreshold = headerHeight.toPx()
            }
            hideThreshold
        }
    }
    
    override fun onPreScroll(
        available: Offset,
        source: NestedScrollSource
    ): Offset {
        getHideThreshold()
        
        // 根据滚动方向决定行为
        when {
            // 向上滚动：隐藏header
            available.y < 0 -> {
                val newOffset = (totalScrollOffset + available.y)
                    .coerceIn(-hideThreshold, 0f)
                val consumed = newOffset - totalScrollOffset
                totalScrollOffset = newOffset
                // 返回 -1 到 0 的值，表示隐藏进度
                onOffsetChange(totalScrollOffset / hideThreshold)
                return Offset(0f, consumed)
            }
            
            // 向下滚动：显示header
            available.y > 0 -> {
                // 如果已经隐藏，需要先把header滚出来
                if (totalScrollOffset < 0) {
                    val newOffset = (totalScrollOffset + available.y)
                        .coerceIn(-hideThreshold, 0f)
                    val consumed = newOffset - totalScrollOffset
                    totalScrollOffset = newOffset
                    onOffsetChange(totalScrollOffset / hideThreshold)
                    return Offset(0f, consumed)
                }
            }
        }
        
        return Offset.Zero
    }
}

/**
 * 应用滑动隐藏效果的组件
 */
@Composable
fun CollapsibleHeader(
    headerHeight: Dp = 200.dp,
    headerContent: @Composable (Float) -> Unit, // 传入隐藏进度
    listContent: @Composable () -> Unit
) {
    var collapseProgress by remember { mutableFloatStateOf(0f) }
    
    val nestedScrollConnection = remember {
        ScrollHideNestedScrollConnection(
            headerHeight = headerHeight,
            onOffsetChange = { progress -> collapseProgress = progress }
        )
    }
    
    Box(modifier = Modifier.fillMaxSize()) {
        LazyColumn(
            modifier = Modifier
                .fillMaxSize()
                .nestedScroll(nestedScrollConnection)
        ) {
            // 动态高度的 header 占位
            item {
                val currentHeight = with(LocalDensity.current) {
                    (headerHeight.toPx() * (1f - collapseProgress)).toDp()
                }
                Spacer(modifier = Modifier.height(currentHeight))
            }
            
            listContent()
        }
        
        // 覆盖在顶部的 header
        headerContent(collapseProgress)
    }
}
```

## 五、性能优化建议

### 1. 使用 remember 缓存 Connection 实例

```kotlin
// ❌ 错误：每次重组都会创建新实例
LazyColumn(
    modifier = Modifier.nestedScroll(StickyHeaderConnection())
) { ... }

// ✅ 正确：使用 remember 缓存实例
val nestedScrollConnection = remember {
    StickyHeaderConnection()
}
LazyColumn(
    modifier = Modifier.nestedScroll(nestedScrollConnection)
) { ... }
```

### 2. 避免在 Connection 中创建新对象

每次滚动都会调用 `onPreScroll`，避免在这里创建临时对象：

```kotlin
// ❌ 错误：在滚动回调中创建对象
override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
    val connection = StickyScrollInfo(...) // 每帧都创建！
    return Offset.Zero
}

// ✅ 正确：使用类的成员变量
class MyConnection {
    private var scrollInfo = ScrollInfo()
    
    override fun onPreScroll(available: Offset, source: NestedScrollSource): Offset {
        scrollInfo.update(available) // 复用对象
        return Offset.Zero
    }
}
```

### 3. 使用 SideEffect 处理副作用

```kotlin
@Composable
fun MyScreen() {
    var isSticky by remember { mutableStateOf(false) }
    
    LaunchedEffect(isSticky) {
        // 处理吸顶状态变化
        // 比如更新 Analytics、触发某些动画等
    }
}
```

### 4. 懒加载计算密集型属性

```kotlin
@Composable
private fun rememberHeaderHeightPx(headerHeight: Dp): Float {
    return remember {
        with(LocalDensity.current) { headerHeight.toPx() }
    }
}
```

## 六、与传统方案的对比

| 特性 | CoordinatorLayout | NestedScrollConnection |
|------|-------------------|------------------------|
| 代码形式 | XML + Compose | 纯 Compose |
| 状态同步 | 复杂 | 简单 |
| 性能 | 一般 | 优秀 |
| 灵活性 | 受限 | 高 |
| 学习曲线 | 陡峭 | 平缓 |
| 可测试性 | 需要 Instrumented Test | 可以 Unit Test |

### 迁移指南

如果你正在使用传统的 CoordinatorLayout + AppBarLayout：

```kotlin
// 原有 XML 布局
<androidx.coordinatorlayout.widget.CoordinatorLayout>
    <com.google.android.material.appbar.AppBarLayout>
        <com.google.android.material.appbar.CollapsingToolbarLayout>
            <!-- Header 内容 -->
        </com.google.android.material.appbar.CollapsingToolbarLayout>
    </com.google.android.material.appbar.AppBarLayout>
    
    <androidx.recyclerview.widget.RecyclerView />
</androidx.coordinatorlayout.widget.CoordinatorLayout>

// 使用 NestedScrollConnection 重构
@Composable
fun CollapsibleHeaderScreen() {
    StickyHeaderLazyColumn(
        headerHeight = 200.dp,
        headerContent = { /* 新的 Header 实现 */ }
    ) {
        // 列表内容
    }
}
```

## 七、完整使用示例

```kotlin
@Composable
fun ArticleListScreen() {
    val articles = remember { /* 获取文章列表 */ }
    
    StickyHeaderLazyColumn(
        headerHeight = 56.dp,
        headerContent = {
            // Header 样式
            Surface(
                modifier = Modifier.fillMaxWidth(),
                color = MaterialTheme.colorScheme.primary
            ) {
                Text(
                    text = "文章列表",
                    style = MaterialTheme.typography.titleLarge,
                    modifier = Modifier.padding(16.dp)
                )
            }
        }
    ) {
        items(articles) { article ->
            ArticleItem(article = article)
        }
    }
}

@Composable
private fun ArticleItem(article: Article) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 8.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = article.title,
                style = MaterialTheme.typography.titleMedium
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = article.summary,
                style = MaterialTheme.typography.bodyMedium,
                color = MaterialTheme.colorScheme.onSurfaceVariant
            )
        }
    }
}
```

## 八、总结

NestedScrollConnection 是一种更现代、更灵活的嵌套滚动处理方式，特别适合 Compose 项目。它让你可以精确控制滚动行为，实现各种复杂的吸顶、交错滚动效果。

### 核心要点

1. **`onPreScroll` 是消费滚动量的地方**，返回 Offset 表示你消耗了多少
2. **记住实例**，避免重组时重新创建
3. **配合 AnimatedVisibility** 实现平滑的吸顶动画
4. **考虑性能**，避免在滚动回调中执行耗时操作
5. **复用对象**，不要在回调中创建新的对象实例

### 延伸学习

- [官方文档: NestedScrollConnection](https://developer.android.com/reference/kotlin/androidx/compose/ui/input/nestedscroll/NestedScrollConnection)
- [理解 Compose 中的 NestedScroll](https://developer.android.com/jetpack/compose/gestures#nested-scroll)
- [实现滑动折叠效果](https://developer.android.com/reference/kotlin/androidx/compose/material3/api/dokka.skip)

---

*本文属于「知识卡片」系列，每日更新。*
*如果你觉得有用，欢迎分享给更多需要的朋友！*
