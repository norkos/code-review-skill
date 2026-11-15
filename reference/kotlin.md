# Kotlin / Android Code Review Guide

> Kotlin/Android code review guide covering core topics such as coroutine scopes and cancellation, Flow pitfalls, Compose recomposition, null safety, memory leaks, architectural layering, and state modeling with sealed classes.

## Table of Contents

- [Coroutines: Scopes and Cancellation](#coroutines-scopes-and-cancellation)
- [Flow Pitfalls](#flow-pitfalls)
- [Jetpack Compose Recomposition](#jetpack-compose-recomposition)
- [Null Safety Patterns](#null-safety-patterns)
- [Memory Leaks](#memory-leaks)
- [Architecture: ViewModel and Repository](#architecture-viewmodel-and-repository)
- [Sealed Classes and State Management](#sealed-classes-and-state-management)
- [Review Checklist](#review-checklist)

---

## Coroutines: Scopes and Cancellation

> 📖 For general concurrency patterns and cross-language examples, see the [Async & Concurrency Cross-Language Guide](cross-cutting/async-concurrency-patterns.md)

### Avoid GlobalScope

```kotlin
// ❌ GlobalScope's lifecycle is uncontrolled; the coroutine keeps running after the Activity/Fragment is destroyed
GlobalScope.launch {
    val data = api.fetchData()
    binding.textView.text = data.title // Crash: view destroyed
}

// ✅ Use viewModelScope; it's cancelled automatically when the ViewModel is cleared
class MyViewModel(private val repo: Repository) : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = repo.fetchData()
            _uiState.value = UiState.Success(data)
        }
    }
}

// ✅ Use lifecycleScope in an Activity/Fragment
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        lifecycleScope.launch {
            val data = repo.fetchData()
            binding.textView.text = data.title
        }
    }
}
```

### Don't Swallow CancellationException

```kotlin
// ❌ Catching all exceptions loses the cancellation signal
viewModelScope.launch {
    try {
        repo.fetchData()
    } catch (e: Exception) {
        // CancellationException is swallowed, so the coroutine can't be cancelled
        showError(e)
    }
}

// ✅ Rethrow CancellationException
viewModelScope.launch {
    try {
        repo.fetchData()
    } catch (e: CancellationException) {
        throw e // Must rethrow
    } catch (e: Exception) {
        showError(e)
    }
}

// ✅ Or use catch together with ensureActive
viewModelScope.launch {
    try {
        repo.fetchData()
    } catch (e: Exception) {
        ensureActive() // Rethrows if cancelled
        showError(e)
    }
}
```

### CPU-Bound Tasks Must Check for Cancellation

```kotlin
// ❌ CPU-intensive computation doesn't respond to cancellation; it runs to completion even after the coroutine is cancelled
viewModelScope.launch(Dispatchers.Default) {
    for (item in largeList) {
        heavyComputation(item)
    }
}

// ✅ Periodically check isActive or call ensureActive
viewModelScope.launch(Dispatchers.Default) {
    for (item in largeList) {
        ensureActive() // Throws CancellationException if cancelled
        heavyComputation(item)
    }
}

// ✅ Or use yield to give up execution
viewModelScope.launch(Dispatchers.Default) {
    for (item in largeList) {
        yield() // Checks cancellation + yields to other coroutines
        heavyComputation(item)
    }
}
```

### Use runInterruptible for Blocking Operations

```kotlin
// ❌ Calling blocking I/O directly in a coroutine blocks a thread-pool thread
viewModelScope.launch(Dispatchers.IO) {
    val result = blockingLibraryCall() // Blocks IO thread
}

// ✅ Wrap blocking calls with runInterruptible to support cancellation via interruption
viewModelScope.launch(Dispatchers.IO) {
    val result = runInterruptible {
        blockingLibraryCall() // Interrupted on cancellation
    }
}
```

### Choose the Right Dispatcher

```kotlin
// ❌ CPU-intensive task on the IO dispatcher (thread pool is too large, wastes resources)
viewModelScope.launch(Dispatchers.IO) {
    val bitmap = decodeImage(byteArray) // CPU-bound on IO pool
}

// ✅ Default for CPU-intensive work, IO for I/O operations
viewModelScope.launch(Dispatchers.Default) {
    val bitmap = decodeImage(byteArray) // CPU-bound on Default pool
}

// ❌ I/O operation on the Default dispatcher (thread pool is too small, prone to starvation)
viewModelScope.launch(Dispatchers.Default) {
    val response = okHttpClient.newCall(request).execute() // I/O on Default pool
}

// ✅ Use the IO dispatcher for I/O operations
viewModelScope.launch(Dispatchers.IO) {
    val response = okHttpClient.newCall(request).execute()
}
```

### launch vs async

```kotlin
// ❌ async used for "fire and forget" with no return value needed
viewModelScope.launch {
    async { analytics.trackEvent("click") } // Overkill
}

// ✅ Use launch when no return value is needed
viewModelScope.launch {
    launch { analytics.trackEvent("click") }
}

// ✅ Use async when a return value is needed and work may run in parallel
viewModelScope.launch {
    val deferredA = async { api.fetchA() }
    val deferredB = async { api.fetchB() }
    val result = combine(deferredA.await(), deferredB.await())
}
```

### Don't Break the Parent-Child Relationship with Job()

```kotlin
// ❌ Job() cuts off cancellation propagation from the parent coroutine
viewModelScope.launch {
    launch(Job()) { // Detached from parent scope!
        importantWork() // Will NOT be cancelled when viewModelScope cancels
    }
}

// ✅ Keep the default parent-child relationship
viewModelScope.launch {
    launch { // Child of viewModelScope
        importantWork() // Cancelled when viewModelScope cancels
    }
}

// ✅ If an independent lifecycle is truly needed, manage it explicitly and document why
class MyManager(private val scope: CoroutineScope) {
    // Independent lifecycle managed by MyManager.shutdown()
    private val managerJob = Job(scope.coroutineContext[Job])
    private val managerScope = scope + managerJob + Dispatchers.IO

    fun shutdown() {
        managerJob.cancel()
    }
}
```

### Using NonCancellable Correctly

```kotlin
// ❌ The whole coroutine is wrapped in withContext(NonCancellable), so it can't be cancelled
viewModelScope.launch {
    withContext(NonCancellable) { // Entire block is uncancellable!
        val data = repo.fetchData() // Cannot be cancelled
        db.saveData(data) // Cannot be cancelled
        analytics.track("saved")
    }
}

// ✅ Use NonCancellable only for cleanup operations
viewModelScope.launch {
    try {
        val data = repo.fetchData()
        db.saveData(data)
    } catch (e: CancellationException) {
        throw e
    } finally {
        withContext(NonCancellable) {
            db.cleanup() // Only cleanup is uncancellable
        }
    }
}
```

---

## Flow Pitfalls

### Confusing Cold and Hot Flows

```kotlin
// ❌ The flow {} block re-executes on every collect (cold-flow behavior misunderstood)
val userFlow = flow {
    emit(api.fetchUser()) // Called once per collector!
}

// Two collectors = two network requests
lifecycleScope.launch { userFlow.collect { } }
lifecycleScope.launch { userFlow.collect { } }

// ✅ Use StateFlow/SharedFlow (hot flows) for shared data
class MyViewModel(private val repo: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    init {
        viewModelScope.launch {
            _uiState.value = UiState.Success(repo.fetchUser())
        }
    }
}
// Multiple collectors share the same StateFlow
```

### Don't Switch Context Inside flow {}

```kotlin
// ❌ Using withContext inside a flow builder violates the context-preservation constraint
val dataFlow = flow {
    withContext(Dispatchers.IO) { // IllegalStateException!
        emit(api.fetchData())
    }
}

// ✅ Use the flowOn operator to switch the upstream context
val dataFlow = flow {
    emit(api.fetchData()) // Runs on IO via flowOn
}.flowOn(Dispatchers.IO)

// ✅ Or use channelFlow / callbackFlow when switching is required
val dataFlow = channelFlow {
    withContext(Dispatchers.IO) {
        send(api.fetchData()) // send() is safe in channelFlow
    }
}
```

### collect Must Be Lifecycle-Aware

```kotlin
// ❌ collect in an Activity/Fragment without lifecycle awareness
lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        binding.textView.text = state.title // Crash if view destroyed
    }
}

// ✅ In a Fragment, use viewLifecycleOwner.lifecycleScope + repeatOnLifecycle
viewLifecycleOwner.lifecycleScope.launch {
    viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            binding.textView.text = state.title
        }
    }
}

// ✅ In Compose, use collectAsStateWithLifecycle
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    // ...
}
```

### Exception Transparency: Use the catch Operator

```kotlin
// ❌ try-catch around collect to handle upstream exceptions
viewModelScope.launch {
    try {
        dataFlow.collect { data ->
            processData(data)
        }
    } catch (e: Exception) {
        // This also catches exceptions from processData, not just upstream
        showError(e)
    }
}

// ✅ Use the catch operator to preserve exception transparency
viewModelScope.launch {
    dataFlow
        .catch { e -> showError(e) } // Only catches upstream exceptions
        .collect { data ->
            processData(data) // Exceptions here propagate normally
        }
}
```

### Choosing Between StateFlow and SharedFlow

```kotlin
// ❌ Emulating StateFlow with SharedFlow loses latest-value semantics
private val _state = MutableSharedFlow<UiState>()
val state: SharedFlow<UiState> = _state

// ✅ Use StateFlow for UI state: always has a value, and new subscribers immediately get the latest value
class MyViewModel : ViewModel() {
    private val _uiState = MutableStateFlow(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

// ✅ Use SharedFlow + replay(0) for events (one-time notifications)
class MyViewModel : ViewModel() {
    private val _navigationEvent = MutableSharedFlow<NavTarget>(extraBufferCapacity = 1)
    val navigationEvent: SharedFlow<NavTarget> = _navigationEvent.asSharedFlow()

    fun navigate(target: NavTarget) {
        _navigationEvent.tryEmit(target)
    }
}

// ✅ Channel for one-time events (alternative)
private val _navigationEvent = Channel<NavTarget>(Channel.BUFFERED)
val navigationEvent = _navigationEvent.receiveAsFlow()
```

---

## Jetpack Compose Recomposition

### Unstable Parameters Cause Extra Recompositions

```kotlin
// ❌ Using an unstable class as a parameter, so Compose can't tell whether it changed
data class UserProfile(
    val name: String,
    val friends: List<String>, // Unstable! List is not @Stable
)

@Composable
fun ProfileCard(profile: UserProfile) { // Recomposes even if profile didn't change
    Text(profile.name)
}

// ✅ Annotate with @Immutable or use stable collection types
@Immutable
data class UserProfile(
    val name: String,
    val friends: ImmutableList<String>, // kotlinx.collections.immutable
)

// ✅ Or extract unstable properties into separate parameters
@Composable
fun ProfileCard(
    name: String, // Stable: String is primitive
    friendCount: Int, // Stable: Int is primitive
) {
    Text(name)
    Text("$friendCount friends")
}
```

### Unstable Lambdas and Memoization

```kotlin
// ❌ A new lambda is created on every recomposition, causing unnecessary child recompositions
@Composable
fun MyScreen(viewModel: MyViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                onClick = { viewModel.handleClick(item.id) } // New lambda each recomposition!
            )
        }
    }
}

// ✅ Wrap the lambda with remember, or have the ViewModel expose stable callbacks
@Composable
fun MyScreen(viewModel: MyViewModel) {
    LazyColumn {
        items(items, key = { it.id }) { item ->
            ItemRow(
                item = item,
                onClick = remember(item.id) { { viewModel.handleClick(item.id) } }
            )
        }
    }
}
```

### Use derivedStateOf to Avoid High-Frequency Recomposition

```kotlin
// ❌ The whole component recomposes on every scroll
@Composable
fun ScrollToTopButton(lazyListState: LazyListState) {
    val showButton = lazyListState.firstVisibleItemIndex > 0 // Recomposes on every scroll
    if (showButton) {
        Button(onClick = { /* scroll to top */ }) {
            Text("Top")
        }
    }
}

// ✅ Use derivedStateOf to trigger recomposition only when the result changes
@Composable
fun ScrollToTopButton(lazyListState: LazyListState) {
    val showButton by remember {
        derivedStateOf { lazyListState.firstVisibleItemIndex > 0 }
    }
    if (showButton) {
        Button(onClick = { /* scroll to top */ }) {
            Text("Top")
        }
    }
}
```

### Don't Run Side Effects in the Composable Function Body

```kotlin
// ❌ Triggering a side effect directly in the Composable body runs it on every recomposition
@Composable
fun MyScreen(userId: String, viewModel: MyViewModel) {
    viewModel.loadUser(userId) // Called on every recomposition!
    val user by viewModel.user.collectAsStateWithLifecycle()
    Text(user?.name ?: "Loading...")
}

// ✅ Use LaunchedEffect to run side effects when the key changes
@Composable
fun MyScreen(userId: String, viewModel: MyViewModel) {
    LaunchedEffect(userId) {
        viewModel.loadUser(userId) // Only when userId changes
    }
    val user by viewModel.user.collectAsStateWithLifecycle()
    Text(user?.name ?: "Loading...")
}

// ✅ Use remember { ... } for one-time initialization
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val initialData = remember { viewModel.getInitialData() }
}
```

### State Hoisting

```kotlin
// ❌ State and logic coupled inside the Composable, so it can't be reused or tested
@Composable
fun ToggleButton() {
    var isChecked by remember { mutableStateOf(false) }
    Switch(
        checked = isChecked,
        onCheckedChange = { isChecked = it }
    )
}

// ✅ State hoisting: the caller controls the state
@Composable
fun ToggleButton(
    isChecked: Boolean,
    onCheckedChange: (Boolean) -> Unit,
    modifier: Modifier = Modifier,
) {
    Switch(
        checked = isChecked,
        onCheckedChange = onCheckedChange,
        modifier = modifier,
    )
}

// ✅ The caller holds the state
@Composable
fun ParentScreen() {
    var enabled by rememberSaveable { mutableStateOf(false) }
    ToggleButton(
        isChecked = enabled,
        onCheckedChange = { enabled = it },
    )
}
```

---

## Null Safety Patterns

### Avoid the Non-Null Assertion !!

```kotlin
// ❌ Non-null assertion: throws an NPE immediately if null
val user = getUser()!!
val name = user.name!!

// ✅ Safe call + elvis operator
val name = getUser()?.name ?: "Unknown"

// ✅ requireNotNull provides a meaningful error message
val user = requireNotNull(getUser()) { "User must not be null at this point" }

// ✅ Early return
fun process(user: User?) {
    val nonNullUser = user ?: return
    nonNullUser.doSomething()
}
```

### lateinit vs nullable vs lazy

```kotlin
// ❌ lateinit used for a value that may be null (wrong semantics)
lateinit var optionalConfig: Config // Might never be set

// ✅ lateinit for values guaranteed to be initialized before use
class MyActivity : AppCompatActivity() {
    lateinit var binding: ActivityMainBinding // Set in onCreate

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
    }
}

// ✅ nullable vs lateinit vs lazy depends on when initialization happens
// lateinit: the lifecycle guarantees initialization before use
// nullable: unclear whether it will be initialized; requires null checks
// lazy: initialized on first access

class MyViewModel(private val repo: Repository) : ViewModel() {
    // lazy: initialized on first access, thread-safe
    val expensiveObject by lazy { ExpensiveObject(repo) }

    // nullable: may never be initialized
    var cachedData: Data? = null
        private set
}
```

### Java Interop: Leaking Platform Types

```kotlin
// ❌ Java returns a platform type (possibly null), which Kotlin treats as non-null
// Java:
// public User getUser() { return null; }
val name: String = javaService.getUser().name // NPE!

// ✅ Receive Java return values with a nullable type
val user: User? = javaService.getUser()
val name = user?.name ?: "Unknown"

// ✅ Wrap the Java API on the Kotlin side to provide safe types
class SafeUserService(private val delegate: JavaUserService) {
    fun getUser(): User? = delegate.getUser() // Explicitly nullable
}
```

---

## Memory Leaks

### Avoid Capturing Context/View in Long-Lived Coroutines

```kotlin
// ❌ The coroutine captures the Activity Context, so the Activity can't be garbage-collected after it's destroyed
class MyActivity : AppCompatActivity() {
    fun loadData() {
        // Leaking Activity via coroutine
        GlobalScope.launch {
            val data = repo.fetchData()
            // 'this' (Activity) is captured
            binding.textView.text = data // Activity leaked!
        }
    }
}

// ✅ Use viewModelScope + lifecycle awareness
class MyViewModel(private val repo: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun loadData() {
        viewModelScope.launch {
            val data = repo.fetchData()
            _uiState.value = UiState.Success(data) // No Activity reference
        }
    }
}
```

### Unregister Listeners

```kotlin
// ❌ Listener registered but never unregistered
class MyFragment : Fragment() {
    private val sensorListener = object : SensorEventListener {
        override fun onSensorChanged(event: SensorEvent) { }
        override fun onAccuracyChanged(sensor: Sensor, accuracy: Int) { }
    }

    override fun onResume() {
        super.onResume()
        sensorManager.registerListener(sensorListener, sensor, SensorManager.SENSOR_DELAY_UI)
        // Never unregistered!
    }
}

// ✅ Unregister in onPause/onDestroyView
override fun onResume() {
    super.onResume()
    sensorManager.registerListener(sensorListener, sensor, SensorManager.SENSOR_DELAY_UI)
}

override fun onPause() {
    super.onPause()
    sensorManager.unregisterListener(sensorListener)
}
```

### Cancel Custom CoroutineScopes

```kotlin
// ❌ CoroutineScope created but never cancelled
class MyManager(private val scope: CoroutineScope) {
    private val job = SupervisorJob()
    private val managerScope = scope + job + Dispatchers.IO

    fun start() {
        managerScope.launch {
            while (isActive) {
                pollServer()
                delay(5000)
            }
        }
    }
    // Never cancelled! job lives forever.
}

// ✅ Provide a shutdown method that cancels the Job
class MyManager(private val scope: CoroutineScope) {
    private val job = SupervisorJob()
    private val managerScope = scope + job + Dispatchers.IO

    fun start() {
        managerScope.launch {
            while (isActive) {
                pollServer()
                delay(5000)
            }
        }
    }

    fun shutdown() {
        job.cancel()
    }
}

// ✅ In a ViewModel, just use the built-in viewModelScope; no need to manage the lifecycle yourself
class MyViewModel : ViewModel() {
    private val scope = viewModelScope + Dispatchers.IO
    // Automatically cancelled when ViewModel is cleared
}
```

---

## Architecture: ViewModel and Repository

### ViewModels Should Not Expose Mutable State

```kotlin
// ❌ Exposing MutableStateFlow directly lets outside code modify it freely
class MyViewModel : ViewModel() {
    val uiState = MutableStateFlow<UiState>(UiState.Loading) // Mutable!

    fun load() {
        viewModelScope.launch {
            uiState.value = UiState.Success(repo.fetchData())
        }
    }
}

// ✅ Expose an immutable interface and keep the mutable version internal
class MyViewModel(private val repo: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun load() {
        viewModelScope.launch {
            _uiState.value = UiState.Success(repo.fetchData())
        }
    }
}
```

### Push Business Logic Down into the Repository

```kotlin
// ❌ ViewModel contains data processing and business rule logic
class UserViewModel(private val api: Api) : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            val raw = api.getUsers()
            val filtered = raw.filter { it.isActive }
            val sorted = filtered.sortedBy { it.name.lowercase() }
            val enriched = sorted.map { user ->
                user.copy(displayName = "${user.firstName} ${user.lastName}")
            }
            _users.value = enriched
        }
    }
}

// ✅ ViewModel only manages state; logic is pushed down into the Repository
class UserRepository(private val api: Api) {
    suspend fun getActiveUsersSorted(): List<User> {
        return api.getUsers()
            .filter { it.isActive }
            .sortedBy { it.name.lowercase() }
            .map { it.copy(displayName = "${it.firstName} ${it.lastName}") }
    }
}

class UserViewModel(private val repo: UserRepository) : ViewModel() {
    private val _users = MutableStateFlow<List<User>>(emptyList())
    val users: StateFlow<List<User>> = _users.asStateFlow()

    fun loadUsers() {
        viewModelScope.launch {
            _users.value = repo.getActiveUsersSorted()
        }
    }
}
```

### Single Source of Truth (Offline-First)

```kotlin
// ❌ ViewModel fetches directly from the network, with no cache and no offline support
class MyViewModel(private val api: Api) : ViewModel() {
    fun load() {
        viewModelScope.launch {
            _uiState.value = UiState.Success(api.fetchData())
        }
    }
}

// ✅ Repository as the single source of truth: show the local cache first, then update with network data
class MyRepository(
    private val api: Api,
    private val dao: DataDao,
) {
    val data: Flow<List<Data>> = dao.getAll()
        .map { entities -> entities.map { it.toDomain() } }

    suspend fun refresh() {
        val remote = api.fetchData()
        dao.replaceAll(remote.map { it.toEntity() })
    }
}

class MyViewModel(private val repo: MyRepository) : ViewModel() {
    val uiState = repo.data.map { UiState.Success(it) }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), UiState.Loading)

    fun refresh() {
        viewModelScope.launch { repo.refresh() }
    }
}
```

### Use Cases for Complex Business Logic

```kotlin
// ❌ Repository method names turn into verb phrases and responsibilities bloat
class OrderRepository {
    suspend fun validateAndSubmitOrder(order: Order) { }
    suspend fun calculateOrderTotalWithDiscounts(order: Order): Money { }
    suspend fun checkInventoryAndReserve(items: List<Item>) { }
}

// ✅ Use Use Cases to encapsulate complex business logic; the Repository only handles data access
class SubmitOrderUseCase(
    private val orderRepo: OrderRepository,
    private val inventoryRepo: InventoryRepository,
    private val paymentRepo: PaymentRepository,
) {
    suspend operator fun invoke(order: Order): Result<OrderConfirmation> {
        val validated = order.validate()
        inventoryRepo.reserve(validated.items)
        val total = CalculateOrderTotalUseCase().invoke(validated)
        return paymentRepo.charge(total).map { confirmation ->
            orderRepo.save(validated.copy(status = OrderStatus.CONFIRMED))
            confirmation
        }
    }
}

class OrderRepository {
    suspend fun save(order: Order) { }
    suspend fun getById(id: String): Order? { }
    fun observeOrders(): Flow<List<Order>> { }
}
```

---

## Sealed Classes and State Management

### Modeling UI State: Make Impossible States Unrepresentable

```kotlin
// ❌ Representing state with a combination of nullables can produce invalid combinations
data class UiState(
    val isLoading: Boolean = false,
    val data: List<Item>? = null,
    val error: String? = null,
)
// Invalid: isLoading=true AND error != null
// Invalid: data != null AND error != null

// ✅ Model with a sealed class so each state is mutually exclusive
sealed interface UiState {
    data object Loading : UiState
    data class Success(val data: List<Item>) : UiState
    data class Error(val message: String, val cause: Throwable? = null) : UiState
}

class MyViewModel(private val repo: Repository) : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()
}

// ✅ Exhaustive when in Compose
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()
    when (state) {
        is UiState.Loading -> CircularProgressIndicator()
        is UiState.Success -> DataList((state as UiState.Success).data)
        is UiState.Error -> ErrorMessage((state as UiState.Error).message)
    }
}
```

### Modeling Navigation Events

```kotlin
// ❌ Representing navigation events with enums or strings can't carry parameters
sealed class NavEvent {
    object ToDetail : NavEvent()
    object ToSettings : NavEvent()
}
// How to pass orderId to ToDetail?

// ✅ Sealed classes carry type-safe parameters
sealed interface NavEvent {
    data class ToDetail(val orderId: String) : NavEvent
    data class ToSettings(val tab: SettingsTab) : NavEvent
    data class ToProfile(val userId: String, val mode: ProfileMode) : NavEvent
}

// ✅ Handling navigation events
navController.handleNavEvent { event ->
    when (event) {
        is NavEvent.ToDetail -> navController.navigate(DetailRoute(event.orderId))
        is NavEvent.ToSettings -> navController.navigate(SettingsRoute(event.tab))
        is NavEvent.ToProfile -> navController.navigate(ProfileRoute(event.userId, event.mode))
    }
}
```

### Wrapping Network Results

```kotlin
// ❌ Representing network results with Result? or nullable loses error information
suspend fun fetchUser(id: String): User? {
    return try {
        api.getUser(id)
    } catch (e: Exception) {
        null // What went wrong?
    }
}

// ✅ Wrap network results in a sealed class
sealed interface NetworkResult<out T> {
    data class Success<T>(val data: T) : NetworkResult<T>
    data class Error(val code: Int, val message: String) : NetworkResult<Nothing>
    data class Exception(val cause: Throwable) : NetworkResult<Nothing>
}

suspend fun fetchUser(id: String): NetworkResult<User> {
    return try {
        val response = api.getUser(id)
        if (response.isSuccessful) {
            NetworkResult.Success(response.body()!!)
        } else {
            NetworkResult.Error(response.code(), response.message())
        }
    } catch (e: Exception) {
        NetworkResult.Exception(e)
    }
}

// ✅ Map to UI state in the ViewModel
fun loadUser(id: String) {
    viewModelScope.launch {
        when (val result = repo.fetchUser(id)) {
            is NetworkResult.Success -> _uiState.value = UiState.Success(result.data)
            is NetworkResult.Error -> _uiState.value = UiState.Error("Server error: ${result.code}")
            is NetworkResult.Exception -> _uiState.value = UiState.Error(result.cause.message ?: "Unknown")
        }
    }
}
```

---

## Review Checklist

### Coroutines

- [ ] No `GlobalScope`; use `viewModelScope` / `lifecycleScope`
- [ ] `CancellationException` is correctly rethrown, not swallowed
- [ ] CPU-intensive tasks use `Dispatchers.Default`; I/O operations use `Dispatchers.IO`
- [ ] Long-running CPU tasks periodically call `ensureActive()` or `yield()`
- [ ] Blocking calls are wrapped with `runInterruptible`
- [ ] `Job()` is not used in a way that breaks the parent-child coroutine relationship
- [ ] `NonCancellable` is used only for cleanup in `finally` blocks
- [ ] `launch` when no return value is needed; `async` when parallel results are needed

### Flow

- [ ] The difference between cold flows (`flow {}`) and hot flows (`StateFlow`/`SharedFlow`) is understood
- [ ] No `withContext` inside a `flow {}` builder; use the `flowOn` operator
- [ ] `collect` is used with `repeatOnLifecycle` or `collectAsStateWithLifecycle`
- [ ] Exceptions are handled with the `.catch` operator rather than `try-catch` around `collect`
- [ ] `StateFlow` for UI state; `SharedFlow` or `Channel` for one-time events

### Compose

- [ ] Composable parameters use stable types to avoid unnecessary recomposition
- [ ] Lambda parameters are wrapped with `remember` to avoid creating new instances on every recomposition
- [ ] Derived state uses `derivedStateOf` to avoid high-frequency recomposition
- [ ] Side effects use `LaunchedEffect` / `SideEffect`, not direct calls in the function body
- [ ] State is hoisted correctly (state hoisting); Composables are stateless and reusable

### Null Safety

- [ ] Non-null assertion `!!` is not abused; use safe call `?.` or elvis `?:`
- [ ] `lateinit` is used only for properties whose initialization is guaranteed by the lifecycle
- [ ] Java interop return values are received with nullable types
- [ ] `lazy` is used for expensive objects initialized on first access

### Memory Leaks

- [ ] Coroutines don't capture short-lived objects such as `Context` / `View`
- [ ] Listeners are properly unregistered in `onPause` / `onDestroyView`
- [ ] Custom `CoroutineScope`s provide a cancellation mechanism
- [ ] Singletons don't hold `Activity` / `Fragment` references

### Architecture

- [ ] ViewModels don't expose `MutableStateFlow` / `MutableLiveData`; use immutable interfaces
- [ ] Business logic is pushed down into Repositories / Use Cases; ViewModels only manage state
- [ ] Offline-first is implemented: the Repository is the single source of truth
- [ ] Complex business logic is encapsulated in dedicated Use Case classes

### Sealed Classes and State

- [ ] UI state is modeled with sealed classes, making impossible states unrepresentable
- [ ] Navigation events use sealed classes carrying type-safe parameters
- [ ] Network request results are wrapped in sealed classes without losing error information
- [ ] `when` expressions cover all branches (exhaustive check)
