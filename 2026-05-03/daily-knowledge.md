# 知识卡片 - 2026-05-03

---

# 📱 科技热点

## 1. 中国硬核科技突破

**算电协同绿电直供项目投运**
5月2日，中国大唐中卫云基地50万千瓦光伏电站正式投运，这是我国首个大规模"算电协同"绿电直供项目，标志着"东数西算"工程实现从沙漠风光电到数字算力的直连直通。

**抽水蓄能电站刷新纪录**
浙江松阳抽水蓄能电站两条深度超600米的竖井实现全线贯通，深度达637米，刷新我国抽水蓄能领域最深竖井工程纪录。

**克隆牦牛技术突破**
中国科研团队首次实现批量克隆10头牦牛，标志着牦牛"全基因组选择+体细胞克隆复合技术"取得重大成功。

## 2. AI与半导体动态

**DeepSeek-V4发布**
- 首次同时适配华为昇腾和英伟达芯片
- 百万Token超长上下文全系标配
- 代码生成准确率提升30%以上
- API价格仅为GPT-5.5的1/700

**三星HBM4商业化**
三星成为首家商业化出货HBM4的厂商，下半年销售额预计占比超50%。

**存储芯片股暴涨**
闪迪涨超8%、希捷涨近8%、美光涨超4%，费城半导体指数再创历史新高。

## 3. 新能源汽车

- **零跑4月销量**：71387台居首
- **比亚迪4月销售**：321123辆，海外出口134542辆创单月新高
- **小米SU7**：4月交付超3万台，143城495家门店覆盖

## 4. 具身智能

- 银河太空舱机器人落地沈阳，提供24小时自主零售服务
- 小鹏第五代机器人定档2026年推出
- Meta完成对ARI的收购

---

# 💰 商业风向

## 2026年普通人最靠谱的5条副业线

### 1. AI辅助的技能服务型副业 ⭐最推荐

**核心逻辑**：卖结果，不赌平台流量

适合的技能：
- Excel、表格整理、数据分析
- 简单设计、海报、封面
- 短视频剪辑、配字幕
- 基础运营支持、文案撰写

**变现路径**：
```
接单(PPT优化/简历优化) → 沉淀模板/工具包 → 打包报价体系
```

### 2. 垂类内容型副业

**关键转变**：从"做博主"到"做垂类认知"

推荐方向：
- AI工具实测号
- iOS/安卓应用测评号
- 副业避坑号
- 运营干货号
- 简历/求职优化号

**变现组合**：广告分成 + 带货佣金 + 私域引流

### 3. 数字产品型副业

**轻量化产品**：
- 简历模板、PPT模板
- AI提示词包
- 副业选题库
- 面试问答资料包

**定价策略**：
- 低价走量：9.9-49元建立口碑
- 高价精品：199-999元提升利润

### 4. 本地生活代运营

- 团购套餐上架：200-500元/店
- 门店日常引流：包月300-800元
- 抖音/朋友圈内容代发

### 5. 有声内容创作

- 有声书录制（喜马拉雅、番茄小说）
- 播客制作服务
- AI语音训练样本

---

# 📚 程序员英语

## 常用开发词汇

| 英文 | 音标 | 中文 | 例句 |
|------|------|------|------|
| bug | /bʌɡ/ | 程序漏洞 | Fix the bug |
| merge | /ˈmɜːdʒ/ | 合并代码 | Merge pull request |
| deploy | /dɪˈplɔɪ/ | 部署上线 | Deploy to production |
| refactor | /riːˈfæktə/ | 重构 | Refactor the codebase |
| debug | /diːˈbʌɡ/ | 调试排错 | Debug the issue |
| feature | /ˈfiːtʃə/ | 功能模块 | New feature release |
| sync | /sɪŋk/ | 同步 | Let's sync on this |
| deploy | /dɪˈplɔɪ/ | 部署 | CI/CD pipeline |
| build | /bɪld/ | 构建 | Build the project |
| release | /rɪˈliːs/ | 发布 | Release v2.0 |

## 日常口语短句

- **Got it** - 明白了，收到
- **Makes sense** - 有道理，说得通
- **I'll take a look** - 我来看看
- **That's a good point** - 这点说得好
- **Let's align on this** - 我们对齐一下
- **It's a blocker** - 这是阻塞项
- **What's the status?** - 状态如何？
- **Can you reproduce it?** - 你能复现吗？

---

# 📖 每日知识

# Android协程与Flow深入解析

> 本章节详细讲解Kotlin协程与Flow的核心原理、实战应用及最佳实践，帮助你深入理解异步编程的本质。

## 一、协程基础概念

### 1.1 什么是协程？

协程（Coroutine）是一种轻量级的线程，它比传统线程更轻量、更高效。在Android开发中，协程主要用于处理异步任务，避免回调地狱，让异步代码看起来像同步代码。

**与传统线程的区别**：

| 特性 | 线程 | 协程 |
|------|------|------|
| 创建开销 | 较大（需向OS申请） | 极小（用户态） |
| 切换成本 | 高（内核态切换） | 低（用户态协作） |
| 资源占用 | MB级别 | KB级别 |
| 并发数量 | 有限（通常几十个） | 数十万级 |

### 1.2 协程的核心优势

```kotlin
// 传统回调写法 - 回调地狱
fun fetchUserCallback(userId: String) {
    api.getUser(userId) { user ->
        api.getPosts(user.id) { posts ->
            api.getComments(posts.first().id) { comments ->
                // 嵌套越来越深...
            }
        }
    }
}

// 协程写法 - 结构化并发
suspend fun fetchUserData(userId: String): UserData {
    val user = api.getUser(userId)      // 挂起，不阻塞
    val posts = api.getPosts(user.id)   // 顺序执行，逻辑清晰
    val comments = api.getComments(posts.first().id)
    return UserData(user, posts, comments)
}
```

## 二、协程上下文与调度器

### 2.1 CoroutineContext

`CoroutineContext` 是协程的"运行环境"，包含以下关键元素：

```kotlin
// CoroutineContext 的组成
public interface CoroutineContext {
    // 包含多个Element，每个Element也是一个Context
    public operator fun <E : Element> get(key: Key<E>): E?
    public fun <R> fold(initial: R, operation: (R, Element) -> R): R
    public fun minusKey(key: Key<*>): CoroutineContext
    public operator fun plus(context: CoroutineContext): CoroutineContext
}

// 常见的Context Element
public interface CoroutineExceptionHandler : Element
public interface ContinuationInterceptor : Element  // 调度器
public interface CoroutineName : Element           // 协程名称
```

### 2.2 调度器（Dispatcher）

调度器决定协程在哪个线程或线程池执行：

```kotlin
// 内置调度器
val Dispatchers.Main = Android主线程调度器（UI线程）
val Dispatchers.IO = IO密集型任务调度器（适合网络、文件操作）
val Dispatchers.Default = CPU密集型任务调度器（适合计算、排序）
val Dispatchers.Unconfined = 不受限调度器（在调用者线程启动）

// 自定义调度器
val myDispatcher = newFixedThreadPoolContext(4, "MyPool")

// 切换调度器
suspend fun loadData() = withContext(Dispatchers.IO) {
    // IO操作在IO线程池执行
    api.fetchData()
}

// 切换到主线程更新UI
suspend fun updateUI() = withContext(Dispatchers.Main) {
    textView.text = "Loaded"
}
```

### 2.3 完整示例：分页数据加载

```kotlin
class UserRepository(private val api: UserApi) {

    /**
     * 实际项目中常见的用法：
     * 1. IO调度器处理网络请求
     * 2. Main调度器处理UI更新
     * 3. 异常统一处理
     */
    suspend fun loadUsers(page: Int, pageSize: Int): Result<List<User>> {
        return withContext(Dispatchers.IO) {
            try {
                val response = api.getUsers(page, pageSize)
                Result.success(response.data)
            } catch (e: Exception) {
                Result.failure(e)
            }
        }
    }
}

// ViewModel中的使用
class UserViewModel(private val repository: UserRepository) : ViewModel() {
    
    private val _users = MutableLiveData<List<User>>()
    val users: LiveData<List<User>> = _users
    
    fun loadUsers(page: Int) {
        viewModelScope.launch {
            // 加载状态处理
            _isLoading.value = true
            
            // 执行数据加载（自动切换到IO线程）
            repository.loadUsers(page, PAGE_SIZE)
                .onSuccess { userList ->
                    // 切换到Main线程更新UI
                    withContext(Dispatchers.Main) {
                        _users.value = userList
                    }
                }
                .onFailure { error ->
                    withContext(Dispatchers.Main) {
                        _error.value = error.message
                    }
                }
            
            _isLoading.value = false
        }
    }
}
```

## 三、协程启动方式

### 3.1 launch vs async

```kotlin
// launch - 启动一个不需要返回值的协程
// 返回 Job，可用于取消
fun main() = runBlocking {
    val job = launch {
        // 这里不需要返回值
        delay(1000L)
        println("Task completed")
    }
    
    job.join()  // 等待任务完成
}

// async - 启动一个需要返回值的协程
// 返回 Deferred，类似于Future
suspend fun fetchTwoValues(): Pair<Int, Int> = coroutineScope {
    val deferred1 = async { fetchValue1() }
    val deferred2 = async { fetchValue2() }
    
    // 并行执行两个任务，提高效率
    Pair(deferred1.await(), deferred2.await())
}
```

### 3.2 suspendCoroutine 与 Continuation

`suspendCoroutine` 是协程底层的核心API，几乎所有挂起函数都基于它实现：

```kotlin
/**
 * suspendCoroutine 的作用：
 * 1. 暂停当前协程的执行
 * 2. 获取 Continuation 对象用于恢复
 * 3. 将回调风格的代码转换为协程风格
 */

// 示例：将Callback API转为协程
suspend fun fetchUserSuspend(userId: String): User {
    // 关键：暂停协程，获取恢复能力
    return suspendCoroutine { continuation ->
        // 原来的Callback写法
        api.getUser(userId, object : Callback<User> {
            override fun onSuccess(user: User) {
                // 成功时恢复协程，并返回值
                continuation.resume(user)
            }
            
            override fun onError(error: Throwable) {
                // 失败时恢复协程，并抛出异常
                continuation.resumeWithException(error)
            }
        })
    }
}

/**
 * suspendCoroutine 与 suspendCoroutineCancellable 的区别：
 * - suspendCoroutineCancellable 支持可取消的挂起
 * - 在协程取消时，可以主动中断正在进行的操作
 */
suspend fun fetchDataCancellable(): Data = suspendCoroutineCancellable { continuation ->
    val call = api.fetchData { result ->
        continuation.resume(result)
    }
    
    // 协程取消时，中断网络请求
    continuation.invokeOnCancellation {
        call.cancel()
    }
}
```

## 四、Flow响应式编程

### 4.1 Flow vs LiveData vs RxJava

```kotlin
/**
 * Flow 的特点：
 * - Kotlin原生，无需额外依赖
 * - 背压支持好
 * - 与协程完美集成
 * - 冷流（Cold Stream），按需生成数据
 */

// LiveData vs Flow 对比
class ViewModel {
    // LiveData（旧方式）
    private val _usersLiveData = MutableLiveData<List<User>>()
    val usersLiveData: LiveData<List<User>> = _usersLiveData
    
    // Flow（新方式）- 更强大、更灵活
    private val _usersFlow = MutableStateFlow<List<User>>(emptyList())
    val usersFlow: StateFlow<List<User>> = _usersFlow
    
    // SharedFlow - 支持多订阅者，广播场景
    private val _events = MutableSharedFlow<Event>()
    val events: SharedFlow<Event> = _events
}
```

### 4.2 StateFlow 与 SharedFlow

```kotlin
/**
 * StateFlow：
 * - 有状态，最新值始终可用
 * - 初始值必须提供
 * - 适合UI状态管理
 * - 新订阅者立即收到最新值
 */
class CounterViewModel : ViewModel() {
    // 初始值为0，UI订阅后立即收到0
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count
    
    fun increment() {
        _count.value++  // 直接修改值
    }
}

/**
 * SharedFlow：
 * - 无状态，需要手动发射
 * - 支持多订阅者
 * - 可配置缓存、重播策略
 * - 适合事件总线、一性事件
 */
class EventBus {
    private val _events = MutableSharedFlow<Event>(
        replay = 0,      // 不重播历史事件
        extraBufferCapacity = 64  // 缓冲区大小
    )
    val events: SharedFlow<Event> = _events
    
    suspend fun emitEvent(event: Event) {
        _events.emit(event)
    }
    
    // 非 suspend 方式发射
    fun tryEmitEvent(event: Event): Boolean {
        return _events.tryEmit(event)
    }
}
```

### 4.3 Flow 操作符详解

```kotlin
/**
 * 常用Flow操作符分类：
 * 
 * 1. 转换操作符：map, filter, transform, onEach
 * 2. 限制操作符：take, takeWhile, debounce
 * 3. 组合操作符：zip, combine, merge
 * 4. 聚合操作符：reduce, fold, toList
 * 5. 错误处理：catch, retry, retryWhen
 */

// 示例：搜索防抖 + 自动刷新
class SearchViewModel(private val repository: SearchRepository) : ViewModel() {
    
    private val _searchQuery = MutableStateFlow("")
    val searchQuery: StateFlow<String> = _searchQuery
    
    // 自动搜索：防抖500ms + 去重 + 网络请求
    val searchResults: Flow<SearchResult> = _searchQuery
        .debounce(500)           // 等待500ms无输入才执行
        .distinctUntilChanged()  // 过滤重复查询
        .flatMapLatest { query -> // 取消旧请求，启动新请求
            if (query.isBlank()) {
                flowOf(SearchResult.Empty)
            } else {
                repository.search(query)
                    .onStart { emit(SearchResult.Loading) }
                    .catch { emit(SearchResult.Error(it)) }
            }
        }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = SearchResult.Idle
        )
}
```

### 4.4 完整实战：相册图片列表

```kotlin
/**
 * 实战场景：Android相册图片列表
 * 需求：
 * 1. 分页加载本地图片
 * 2. 支持下拉刷新、上拉加载更多
 * 3. 搜索过滤
 * 4. 实时更新（监看文件系统变化）
 */

// Repository层
class PhotoRepository(private val contentResolver: ContentResolver) {
    
    /**
     * Flow版本：支持实时更新
     * 每次数据库变化自动触发UI更新
     */
    fun observePhotos(bucketId: String?): Flow<List<Photo>> = callbackFlow {
        val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
            override fun onChange(selfChange: Boolean) {
                trySend(queryPhotos(bucketId))
            }
        }
        
        contentResolver.registerContentObserver(
            MediaStore.Images.Media.EXTERNAL_CONTENT_URI,
            true,
            observer
        )
        
        // 初始数据
        trySend(queryPhotos(bucketId))
        
        awaitClose {
            contentResolver.unregisterContentObserver(observer)
        }
    }
    
    private fun queryPhotos(bucketId: String?): List<Photo> {
        // 查询逻辑...
        return emptyList()
    }
}

// ViewModel层
@HiltViewModel
class PhotoViewModel @Inject constructor(
    private val repository: PhotoRepository
) : ViewModel() {
    
    // 状态管理
    private val _uiState = MutableStateFlow(PhotoUiState())
    val uiState: StateFlow<PhotoUiState> = _uiState
    
    // 当前页码
    private var currentPage = 0
    private var hasMorePages = true
    
    init {
        // 监听照片变化
        viewModelScope.launch {
            repository.observePhotos(null)
                .collect { photos ->
                    _uiState.update { it.copy(photos = photos) }
                }
        }
    }
    
    // 加载更多
    fun loadMore() {
        if (!hasMorePages) return
        
        viewModelScope.launch {
            _uiState.update { it.copy(isLoadingMore = true) }
            
            repository.getPhotos(page = currentPage + 1)
                .catch { e ->
                    _uiState.update { it.copy(error = e.message) }
                }
                .collect { newPhotos ->
                    currentPage++
                    hasMorePages = newPhotos.size >= PAGE_SIZE
                    _uiState.update { state ->
                        state.copy(
                            photos = state.photos + newPhotos,
                            isLoadingMore = false
                        )
                    }
                }
        }
    }
    
    // 搜索过滤
    fun search(query: String) {
        viewModelScope.launch {
            repository.searchPhotos(query)
                .collect { results ->
                    _uiState.update { it.copy(photos = results) }
                }
        }
    }
}

// UI层（Compose）
@Composable
fun PhotoScreen(viewModel: PhotoViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    
    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        state = listState
    ) {
        items(uiState.photos, key = { it.id }) { photo ->
            PhotoItem(
                photo = photo,
                onClick = { /* 预览 */ }
            )
        }
        
        // 加载更多触发器
        if (uiState.isLoadingMore) {
            item {
                Box(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    contentAlignment = Alignment.Center
                ) {
                    CircularProgressIndicator()
                }
            }
        }
    }
    
    // 无限滚动检测
    LaunchedEffect(listState) {
        snapshotFlow { listState.layoutInfo.visibleItemsInfo.lastOrNull()?.index }
            .filter { it != null && it >= uiState.photos.size - 5 }
            .collect { viewModel.loadMore() }
    }
}
```

## 五、协程与Flow最佳实践

### 5.1 ViewModelScope 与 LifecycleScope

```kotlin
/**
 * 协程作用域选择原则：
 * 
 * ViewModelScope：
 * - ViewModel销毁时自动取消
 * - 用于与UI相关的数据加载
 * 
 * LifecycleScope：
 * - Activity/Fragment销毁时自动取消
 * - 用于短生命周期任务
 * 
 * GlobalScope：
 * - ⚠️ 极度不推荐使用
 * - 除非明确知道任务需要与应用同生命周期
 */

// Hilt + ViewModelScope（推荐）
@HiltViewModel
class MyViewModel @Inject constructor(
    private val repository: MyRepository
) : ViewModel() {
    
    // viewModelScope 自动注入，可直接使用
    fun loadData() {
        viewModelScope.launch {
            // ViewModel销毁时自动取消
            repository.fetchData()
        }
    }
}

// LaunchedEffect（Compose专用）
@Composable
fun MyScreen() {
    var data by remember { mutableStateOf<Data?>(null) }
    
    // 跟着Composable生命周期，自动取消
    LaunchedEffect(key1 = someId) {
        data = repository.fetchData(someId)
    }
}
```

### 5.2 异常处理策略

```kotlin
/**
 * Flow 异常处理：
 * 
 * 1. catch 操作符 - 捕获上游异常
 * 2. retry/retryWhen - 重新执行失败的操作
 * 3. catch + 发射默认值 - 兜底策略
 */

// 策略1：catch + collect
flowOf(1, 2, 3)
    .map { it / 0 }  // 抛出异常
    .catch { e -> 
        emit(-1)  // 捕获后发射默认值
    }
    .collect { println(it) }  // 输出: -1

// 策略2：retry 指数退避
suspend fun fetchWithRetry(url: String): String = flow {
    emit(api.fetch(url))
}.retryWhen { cause, attempt ->
    // 最多重试3次，间隔时间指数增长
    if (attempt < 3 && cause is IOException) {
        delay(1000L * (attempt + 1))
        true
    } else {
        false
    }
}.first()

// 策略3：结构化异常处理
suspend fun safeFetch(): Result<Data> = runCatching {
    repository.fetchData()
}
```

### 5.3 性能优化技巧

```kotlin
/**
 * Flow 性能优化：
 * 
 * 1. 使用 shareIn / stateIn 共享Flow，避免重复订阅
 * 2. 使用 distinctUntilChanged 去重
 * 3. 使用 flowOn 切换执行上下文
 * 4. 合理设置缓冲区大小
 */

// 共享Flow，避免重复执行
val sharedFlow = expensiveFlow()
    .shareIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        replay = 1
    )

// 线程切换，避免主线程阻塞
val resultFlow = flow {
    // 耗时操作
}.flowOn(Dispatchers.IO)  // 这里在IO线程执行
    .catch { /* 错误处理 */ }
    .flowOn(Dispatchers.Main)  // 这里回到主线程（实际在collect处）
```

## 六、面试高频问题

### Q1: 协程为什么是轻量级的？

**答**：协程的轻量级源于以下设计：

1. **用户态实现**：协程切换发生在用户态，不需要内核态切换
2. **栈空间复用**：协程共享同一线程的栈空间，按需增长/收缩
3. **无序创建**：协程在JVM堆上分配，没有固定大小限制

```kotlin
// 实际对比
fun main() {
    // 创建10万个协程，内存消耗约几十MB
    runBlocking {
        repeat(100_000) {
            launch {
                delay(1000L)
                println("Coroutine $it")
            }
        }
    }
    // 如果是10万个线程，内存消耗会超过10GB
}
```

### Q2: 什么情况下会使用 Flow 而不是 suspend？

**答**：Flow适用于以下场景：

| 场景 | suspend | Flow |
|------|---------|------|
| 单次异步请求 | ✅ | ❌ |
| 多个值序列 | ❌ | ✅ |
| 实时数据更新 | ❌ | ✅ |
| UI状态管理 | ❌ | ✅ |
| 事件总线 | ❌ | ✅ |

### Q3: StateFlow 和 LiveData 的区别？

**答**：主要区别：

```kotlin
// LiveData
- 有生命周期感知（自动在onStop后暂停）
- 必须主线程创建和更新（大部分情况）
- 需要用observe()订阅

// StateFlow
- 无生命周期感知（需要手动管理）
- 可在任何线程更新
- Compose原生支持
- 功能更强大（背压、缓冲区等）
```

## 七、总结

**协程核心要点**：
- 协程 = 可暂停的执行单元
- `suspend` 关键字标记可暂停函数
- `CoroutineContext` 管理执行环境
- `Dispatchers` 控制线程选择

**Flow核心要点**：
- Flow = 冷数据流，按需产生值
- `StateFlow` = 有状态的数据流，适合UI状态
- `SharedFlow` = 事件总线，适合一次性事件
- 操作符链式处理数据转换

**最佳实践**：
1. Repository层返回Flow，ViewModel层用StateFlow暴露
2. 使用`viewModelScope`管理协程生命周期
3. 合理使用`flowOn`和`catch`处理线程和异常
4. 善用`shareIn`/`stateIn`共享Flow

---

# 📝 软考备考

## 系统架构设计师 - 软件架构风格

### 核心概念

软件架构风格是描述某一特定应用领域中系统组织的惯用模式。它定义了系统组件的组织方式、通信方式和约束条件。

### 五大经典架构风格

#### 1. 管道-过滤器风格

**特点**：数据流处理，每个处理步骤封装在过滤器中，数据通过管道传递。

```
┌─────────┐    ┌─────────┐    ┌─────────┐
│ 过滤器1 │───▶│ 过滤器2 │───▶│ 过滤器3 │
└─────────┘    └─────────┘    └─────────┘
     ▲              ▲
     │              │
   管道          管道
```

**优点**：良好的封装性、可复用、可并行处理
**缺点**：不适合交互式系统、进程间通信开销大

**应用场景**：编译器、数据处理管道

```kotlin
// 伪代码示例：管道-过滤器
interface Pipe {
    fun write(data: Any)
    fun read(): Any
}

interface Filter {
    fun setInput(pipe: Pipe)
    fun setOutput(pipe: Pipe)
    fun process()
}

// 过滤器实现
class UpperCaseFilter : Filter {
    private var input: Pipe? = null
    private var output: Pipe? = null
    
    override fun setInput(pipe: Pipe) { input = pipe }
    override fun setOutput(pipe: Pipe) { output = pipe }
    
    override fun process() {
        val data = input?.read() as? String ?: return
        output?.write(data.uppercase())
    }
}
```

#### 2. 面向对象风格

**特点**：数据和操作封装在对象中，通过消息传递通信。

**核心原则**：封装、继承、多态

```kotlin
// 面向对象架构示例
abstract class Component {
    abstract fun operation()
}

class ConcreteComponent : Component() {
    override fun operation() {
        println("Concrete operation")
    }
}

// 装饰器模式
abstract class Decorator(protected var component: Component) : Component() {
    override fun operation() {
        component.operation()
    }
}

class LoggingDecorator(component: Component) : Decorator(component) {
    override fun operation() {
        println("Before operation")
        super.operation()
        println("After operation")
    }
}
```

#### 3. 事件驱动风格（隐式调用）

**特点**：组件不直接调用，而是发布/订阅事件。

```
┌─────────┐                    ┌─────────┐
│ 组件A   │───发布事件───▶│ 事件总线 │───通知───▶│ 组件B   │
└─────────┘                    └─────────┘
                                        │
                                        └───▶│ 组件C   │
```

**优点**：松耦合、易扩展
**缺点**：控制复杂、难以预测行为

**应用**：IDE重构功能、自动化构建系统

#### 4. 分层风格

**特点**：系统分为多个层次，每层只依赖下层。

```
┌─────────────────────────────┐
│      表现层 (UI)            │
├─────────────────────────────┤
│      业务逻辑层 (Service)   │
├─────────────────────────────┤
│      数据访问层 (Repository)│
├─────────────────────────────┤
│      基础设施层 (Framework) │
└─────────────────────────────┘
```

**Android MVVM 示例**：

```kotlin
// 分层架构示例
// 表现层
class MainActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    
    override fun onCreate(savedInstanceState: Bundle?) {
        // UI绑定
    }
}

// 业务逻辑层
class UserViewModel(
    private val getUserUseCase: GetUserUseCase
) : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user
    
    fun loadUser(id: String) {
        viewModelScope.launch {
            _user.value = getUserUseCase(id)
        }
    }
}

// 数据访问层
class UserRepository(private val api: UserApi) {
    suspend fun getUser(id: String): User = api.getUser(id)
}
```

#### 5. 客户端-服务器风格

**特点**：请求-响应模式，客户端发起请求，服务器提供服务。

**变体**：
- 二层C/S：客户端 + 数据库服务器
- 三层C/S：表示层 + 功能层 + 数据层
- N层C/S：进一步分解功能层

### 架构质量属性

| 属性 | 说明 | 关注点 |
|------|------|--------|
| 性能 | 响应时间和吞吐量 | 缓存、异步、负载均衡 |
| 可用性 | 系统正常运行时间 | 冗余、故障转移 |
| 安全性 | 数据保护能力 | 加密、认证、授权 |
| 可维护性 | 易于修改的程度 | 模块化、可测试性 |
| 可扩展性 | 处理增长的能力 | 水平扩展、垂直扩展 |

---

# 💼 面试题

## Android高频面试题

### 1. 什么是ANR？如何避免？

**ANR（Application Not Responding）**：应用无响应

**触发条件**：
- Activity输入事件响应超时：5秒
- BroadcastReceiver处理超时：10秒
- Service执行超时：20秒

**常见原因**：
1. 主线程执行耗时操作（网络请求、数据库查询）
2. 主线程被锁阻塞
3. Binder通信超时

**解决方案**：
```kotlin
// ❌ 错误：在主线程执行网络请求
fun fetchData() {
    val response = api.getData()  // 阻塞主线程！
    updateUI(response)
}

// ✅ 正确：使用协程
fun fetchData() {
    viewModelScope.launch {
        val response = repository.getData()  // 自动切换到IO线程
        _uiState.value = response
    }
}

// ✅ 正确：使用WorkManager处理后台任务
WorkManager.getInstance(context)
    .enqueue(OneTimeWorkRequestBuilder<DataSyncWorker>().build())
```

### 2. 什么是OOM？如何优化？

**OOM（Out Of Memory）**：内存溢出

**常见原因**：
1. 大图加载未压缩
2. Bitmap未及时回收
3. 内存泄漏

**优化方案**：
```kotlin
// 图片压缩示例
fun compressImage(bitmap: Bitmap, maxWidth: Int, maxHeight: Int): Bitmap {
    val ratio = minOf(
        maxWidth.toFloat() / bitmap.width,
        maxHeight.toFloat() / bitmap.height
    )
    
    val width = (bitmap.width * ratio).toInt()
    val height = (bitmap.height * ratio).toInt()
    
    return Bitmap.createScaledBitmap(bitmap, width, height, true)
}

// 使用Glide/Coil自动处理
Glide.with(context)
    .load(url)
    .override(1080, 1920)  // 指定尺寸
    .diskCacheStrategy(DiskCacheStrategy.ALL)
    .into(imageView)
```

### 3. Handler机制原理

```kotlin
/**
 * Handler机制核心组件：
 * 
 * Message: 消息对象，包含数据和处理目标
 * MessageQueue: 消息队列，存储待处理消息
 * Looper: 不断从队列中取消息并分发给Handler
 * Handler: 发送和处理消息
 */

// 主线程Looper初始化
class ActivityThread {
    public static void main(String[] args) {
        // 创建主线程Looper
        Looper.prepareMainLooper();
        
        // 创建Handler
        new Handler(Looper.getMainLooper()) {
            @Override
            public void handleMessage(@NonNull Message msg) {
                // 处理消息
            }
        };
        
        // 启动消息循环
        Looper.loop();
    }
}

// 发送消息
val handler = Handler(Looper.getMainLooper())
handler.sendMessage(Message.obtain().apply { what = 1, obj = data })
handler.post { /* 在主线程执行 */ }

// 消息处理
override fun handleMessage(msg: Message) {
    when (msg.what) {
        1 -> updateUI(msg.obj as Data)
    }
}
```

### 4. Service与WorkManager对比

| 特性 | Service | WorkManager |
|------|---------|-------------|
| 执行时机 | 可立即执行 | 最佳时机执行 |
| 后台类型 | 前台/后台 | 后台（可延迟） |
| 生命周期 | 需要手动管理 | 自动管理 |
| 电池优化 | 无 | 有（电池友好） |
| 可靠性 | 可能被杀死 | 有保证 |
| 系统要求 | Android 8.0前需前台服务 | API 14+ |

```kotlin
// WorkManager：推荐用于后台任务
WorkManager.getInstance(context)
    .enqueueUniqueWork(
        "sync_data",
        ExistingWorkPolicy.REPLACE,
        OneTimeWorkRequestBuilder<SyncWorker>()
            .setConstraints(
                Constraints.Builder()
                    .setRequiredNetworkType(NetworkType.CONNECTED)
                    .build()
            )
            .build()
    )

// Service：仅用于需要立即执行的前台任务
class MyForegroundService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        startForeground(NOTIFICATION_ID, createNotification())
        // 执行任务
        return START_STICKY
    }
}
```

### 5. LiveData与StateFlow对比

```kotlin
/**
 * 关键区别：
 * 
 * LiveData：
 * - 生命周期感知，自动在onStop后暂停
 * - 主线程观察者必须主线程更新
 * - Compose需要额外observeAsState()
 * 
 * StateFlow：
 * - 无生命周期感知，需要手动管理
 * - 任何线程更新
 * - Compose原生支持collectAsState()
 */

// LiveData用法
class UserViewModel : ViewModel() {
    private val _user = MutableLiveData<User>()
    val user: LiveData<User> = _user
}

@Composable
fun Screen(viewModel: UserViewModel) {
    val user by viewModel.user.observeAsState()
}

// StateFlow用法
class UserViewModel : ViewModel() {
    private val _user = MutableStateFlow<User?>(null)
    val user: StateFlow<User?> = _user
}

@Composable
fun Screen(viewModel: UserViewModel) {
    val user by viewModel.user.collectAsState()
}
```

---

# ✨ 金句

> **"代码是写给人看的，顺便能在机器上运行。"**
> 
> —— 《计算机程序设计艺术》作者 Donald Knuth
> 
> **"最好的架构是那些可以轻松改变的架构。不要过度设计，但要为扩展留有余地。"**
> 
> **"程序员的成长不在于写了多少代码，而在于解决了多少问题。"**

---

## 📅 今日总结

**日期**：2026年5月3日 星期日

**学习要点**：
1. 协程挂起原理、Context与调度器
2. Flow的StateFlow vs SharedFlow使用场景
3. 软件架构五大经典风格
4. Android性能优化与内存管理

**行动建议**：
- 在项目中尝试用Flow重构现有LiveData代码
- 阅读协程源码，理解suspend实现原理
- 整理架构风格笔记，形成自己的知识体系

---

*本知识卡片由 AI 自动生成*
*关注公众号「方哲的知识库」获取更多内容*
