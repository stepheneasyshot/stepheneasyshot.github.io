---
layout: post
description: > 
  本文介绍了Android开发领域各种不同的架构演进历史
image: 
  path: /assets/img/blog/blogs_android_common_cover.png
  srcset: 
    1920w: /assets/img/blog/blogs_android_common_cover.png
    960w:  /assets/img/blog/blogs_android_common_cover.png
    480w:  /assets/img/blog/blogs_android_common_cover.png
accent_image: /assets/img/blog/blogs_android_common_cover.png
excerpt_separator: <!--more-->
sitemap: false
---
# 【Android进阶】Android项目架构演进
对于安卓开发者来说，理解并熟练运用架构模式是提升代码质量、可维护性和可测试性的关键。

我们总是在追求编写更清晰、更健壮、更易于维护的代码。而选择一个合适的应用架构，正是实现这一目标的基石。从经典的 MVC，到后来的 MVP、MVVM，再到如今函数式思想影响下的 MVI，安卓的架构模式一直在演进。

为了方便对比，我们设定一个极其简单的业务场景：

> 一个登录界面，包含输入用户名、密码的输入框和一个登录按钮。点击按钮后，模拟一个网络请求，根据结果（成功/失败）更新 UI。

## MVC (Model-View-Controller)
MVC 是一个非常古老的 UI 架构模式，在安卓早期开发中，它是一种“天然”的结构。

* **Model (模型):** 负责处理数据和业务逻辑。例如，网络请求、数据库操作、数据bean类。
* **View (视图):** 负责展示 UI 界面，并将用户的操作（点击、输入）传递出去。在安卓中，这通常由 XML 布局文件和 Activity/Fragment 扮演。
* **Controller (控制器):** 接收来自 View 的用户操作，调用 Model 处理业务逻辑，然后更新 View 的显示。Activity/Fragment 通常也承担了 Controller 的角色。

### MVC的问题
在安卓中，Activity/Fragment 的职责过重，它既是 View 的一部分，又是 Controller。这 **导致 View 和 Controller 紧密耦合** ，业务 **逻辑和 UI 代码混杂** 在一起，使得代码难以测试和维护。这就是我们常说的超大型Activity和Fragment。
### 简单代码实现

**`UserModel.kt` (Model)**

```kotlin
// M - Model: 负责业务逻辑和数据
data class User(val name: String)

object UserModel {
    // 模拟登录网络请求
    fun login(username: String, callback: (Result<User>) -> Unit) {
        // 模拟延时
        Thread.sleep(1000)
        if (username == "admin") {
            callback(Result.success(User("Administrator")))
        } else {
            callback(Result.failure(Exception("用户名或密码错误")))
        }
    }
}
```

**`activity_login.xml` (View)**

```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <EditText
        android:id="@+id/et_username"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Username"/>

    <Button
        android:id="@+id/btn_login"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Login"/>

    <ProgressBar
        android:id="@+id/progress_bar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:visibility="gone"/>
</LinearLayout>
```

**`LoginActivity.kt` (View & Controller)**

```kotlin
// V & C: Activity 同时扮演视图和控制器的角色
class LoginActivity : AppCompatActivity() {
    
    private lateinit var usernameEditText: EditText
    private lateinit var loginButton: Button
    private lateinit var progressBar: ProgressBar

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        usernameEditText = findViewById(R.id.et_username)
        loginButton = findViewById(R.id.btn_login)
        progressBar = findViewById(R.id.progress_bar)

        loginButton.setOnClickListener {
            handleLogin()
        }
    }

    private fun handleLogin() {
        showLoading()
        val username = usernameEditText.text.toString()

        // 控制器直接调用模型
        // 注意：这里为了简化在主线程调用，实际开发应使用协程或线程池
        Thread {
            UserModel.login(username) { result ->
                runOnUiThread {
                    hideLoading()
                    result.onSuccess { user ->
                        showSuccess("欢迎, ${user.name}!")
                    }.onFailure { error ->
                        showError(error.message ?: "登录失败")
                    }
                }
            }
        }.start()
    }

    private fun showLoading() {
        progressBar.visibility = View.VISIBLE
    }

    private fun hideLoading() {
        progressBar.visibility = View.GONE
    }

    private fun showSuccess(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    private fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
}
```

## MVP (Model-View-Presenter)
为了解决 MVC 中 Controller 和 View 的过度耦合问题，MVP 诞生了。它引入了一个新的角色：Presenter。

* **Model:** 职责不变，处理数据和业务逻辑。
* **View:** 职责更纯粹，只负责 UI 的渲染和事件的传递。Activity/Fragment 属于 View 层。它通常会实现一个接口，供 Presenter 调用。
* **Presenter:** 作为 View 和 Model 之间的桥梁。它从 Model 获取数据，然后调用 View 接口的方法来更新 UI。**Presenter 不持有任何 Android 框架的引用**，这使得它很容易进行单元测试。

解决了一些MVC的痛点，View 和 Presenter 通过接口进行通信，实现了彼此的解耦。Presenter 持有 View 的接口引用，但不关心 View 的具体实现。
### MVP的问题
主要有两点：
* 接口爆炸： 为了清晰地定义View和Presenter之间的契约，你需要为每一个界面（或功能模块）定义至少两个接口： `IView` 和 `IPresenter` 。随着项目模块的增加，接口数量会急剧膨胀，导致代码文件数量非常多。
* 繁琐的绑定与解绑： `Presenter` 需要持有 `View` 的引用，为了避免内存泄漏（尤其是在异步任务回调时），你必须在View（如Activity/Fragment）的生命周期方法（onCreate, onDestroy）中手动进行 `attachView()` 和 `detachView()` 操作。这个过程是重复且容易出错的。

### 代码实现

**`LoginContract.kt` (契约接口)**

```kotlin
// 定义 View 和 Presenter 之间的契约
interface LoginContract {
    // View 必须实现的接口
    interface View {
        fun showLoading()
        fun hideLoading()
        fun showLoginSuccess(message: String)
        fun showLoginError(message: String)
    }

    // Presenter 必须实现的接口
    interface Presenter {
        fun login(username: String)
        fun onDestroy()
    }
}
```

**`LoginPresenter.kt` (Presenter)**

```kotlin
// P - Presenter: 不含任何 Android SDK 代码，纯 Kotlin/Java
class LoginPresenter(private var view: LoginContract.View?) : LoginContract.Presenter {

    // Presenter 持有 Model 的引用
    private val model = UserModel

    override fun login(username: String) {
        view?.showLoading()
        // 同样，这里简化处理，实际应在子线程
        Thread {
            model.login(username) { result ->
                // 回到主线程更新 UI
                (view as? AppCompatActivity)?.runOnUiThread {
                    view?.hideLoading()
                    result.onSuccess { user ->
                        view?.showLoginSuccess("欢迎, ${user.name}!")
                    }.onFailure { error ->
                        view?.showLoginError(error.message ?: "登录失败")
                    }
                }
            }
        }.start()
    }
    
    // 防止内存泄漏
    override fun onDestroy() {
        view = null
    }
}
```

**`LoginActivity.kt` (View)**

```kotlin
// V - View: 只负责 UI 展示和用户事件传递
class LoginActivity : AppCompatActivity(), LoginContract.View {

    private lateinit var presenter: LoginContract.Presenter
    // ... UI 控件声明 ...

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
        
        presenter = LoginPresenter(this)
        // ... findViewById ...

        loginButton.setOnClickListener {
            presenter.login(usernameEditText.text.toString())
        }
    }

    override fun showLoading() {
        progressBar.visibility = View.VISIBLE
    }

    override fun hideLoading() {
        progressBar.visibility = View.GONE
    }

    override fun showLoginSuccess(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }

    override fun showLoginError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_SHORT).show()
    }
    
    override fun onDestroy() {
        presenter.onDestroy()
        super.onDestroy()
    }
}
```

## MVVM (Model-View-ViewModel)
MVVM 是 Google 官方推荐的架构模式，也是 Jetpack 组件（如 `ViewModel`, `LiveData`, `DataBinding`）的核心思想。

* **Model:** 职责不变。
* **View:** 职责依然是 UI 展示。但它不再被动地等待 Presenter 调用，而是主动**观察 (Observe)** ViewModel 中的数据变化来更新自己。
* **ViewModel:** 类似于 Presenter，负责处理业务逻辑并持有数据。但它不直接引用 View。它通过暴露可观察的数据（如 `LiveData`或 `StateFlow`）来通知 View 更新。ViewModel 的生命周期与 UI 控制器（Activity/Fragment）的配置更改无关，因此在屏幕旋转时数据不会丢失。

解决前面两代主流架构的痛点：
**数据绑定 (Data Binding)** 和 **生命周期感知 (Lifecycle-Aware)**。View 和 ViewModel 通过可观察的数据流进行单向或双向绑定，实现了比 MVP 更彻底的解耦。

### MVVM的问题
状态管理的复杂性（Lack of Unidirectional Data Flow）：
* 虽然 MVVM 实现了视图和数据之间的双向绑定，但在复杂的场景下，这可能导致数据流变得难以追踪。当一个数据模型在多个视图或组件之间共享时，任何一方的修改都可能影响到其他部分，使得状态的来源和变化路径变得模糊不清。

不一致的状态（Inconsistent State）：
* 在某些情况下，视图可能会从多个地方接收数据更新（例如，网络请求、本地数据库更新、用户输入）。当这些更新并非同步发生时，视图可能会进入一种不一致的状态，导致用户界面出现意料之外的行为或显示错误。

### 代码实现

**`LoginViewModel.kt` (ViewModel)**

```kotlin
// VM - ViewModel: 持有数据和业务逻辑，通过 LiveData 通知 View
class LoginViewModel : ViewModel() {

    private val model = UserModel
    
    // UI 状态的 LiveData
    private val _loginState = MutableLiveData<LoginUiState>()
    val loginState: LiveData<LoginUiState> = _loginState

    fun login(username: String) {
        _loginState.value = LoginUiState.Loading

        // 使用 ViewModelScope 协程来处理异步操作
        viewModelScope.launch(Dispatchers.IO) {
            model.login(username) { result ->
                val newState = result.fold(
                    onSuccess = { user -> LoginUiState.Success("欢迎, ${user.name}!") },
                    onFailure = { error -> LoginUiState.Error(error.message ?: "登录失败") }
                )
                // 切换回主线程更新 LiveData
                withContext(Dispatchers.Main) {
                    _loginState.value = newState
                }
            }
        }
    }
}

// 定义 UI 状态的密封类
sealed class LoginUiState {
    object Loading : LoginUiState()
    data class Success(val message: String) : LoginUiState()
    data class Error(val message: String) : LoginUiState()
}
```

**`LoginActivity.kt` (View)**

```kotlin
// V - View: 观察 ViewModel 中的数据变化来更新 UI
class LoginActivity : AppCompatActivity() {

    // 通过 ktx 库轻松获取 ViewModel
    private val viewModel: LoginViewModel by viewModels()
    // ... UI 控件声明 ...

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)
        // ... findViewById ...

        loginButton.setOnClickListener {
            viewModel.login(usernameEditText.text.toString())
        }
        
        // 观察 LiveData 的变化
        viewModel.loginState.observe(this) { state ->
            when (state) {
                is LoginUiState.Loading -> showLoading()
                is LoginUiState.Success -> {
                    hideLoading()
                    showSuccess(state.message)
                }
                is LoginUiState.Error -> {
                    hideLoading()
                    showError(state.message)
                }
            }
        }
    }
    
    // ... UI 更新方法 ...
}
```

> 别忘了在 `build.gradle` 文件中添加 ViewModel 和 LiveData 的依赖。

## MVI (Model-View-Intent)

MVI 是一种更现代的架构模式，深受函数式编程思想的影响，它强调**单向数据流**和**状态的唯一可信源**。

  * **Model:** 在 MVI 中，Model 通常指**UI 状态 (State)**。它是一个不可变的数据结构，代表了 UI 在某一时刻的所有状态。
  * **View:** 负责渲染 UI 状态，并**捕获用户意图 (Intent)**，将其发送出去。
  * **Intent:** 不要和安卓的 `Intent` 组件混淆。这里的 Intent 指的是用户的操作意图，例如 `LoginClickedIntent`、`UsernameChangedIntent` 等。

1.  **环形单向数据流:**
      * View 发送 **Intent** (用户意图)。
      * ViewModel (或类似角色) 接收 Intent，处理业务逻辑。
      * ViewModel 生成一个新的 **State** (UI 状态)。
      * View 订阅 State 的变化，并用新状态渲染自己。
2.  **唯一数据源:** UI 的所有状态都由一个 State 对象管理，任何对 UI 的更新都必须通过生成一个新的 State 来实现。这使得状态变化变得可预测和易于调试。这一定程度上解决了 MVVM 架构的多数据来源导致UI变化难以追踪的问题。

### 代码实现

**`LoginContract.kt` (State, Intent, Effect)**

```kotlin
// M - State: UI 的状态，必须是不可变的
data class LoginViewState(
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

// I - Intent: 用户的意图
sealed class LoginIntent {
    data class LoginClicked(val username: String) : LoginIntent()
}

// Side Effect: 一次性事件，如 Toast 或导航
sealed class LoginEffect {
    data class ShowSuccessToast(val message: String) : LoginEffect()
}
```

**`LoginViewModel.kt` (处理 Intent，生成 State)**

```kotlin
// ViewModel: 处理 Intent，更新 State，发送 Effect
class LoginViewModel : ViewModel() {

    private val model = UserModel
    
    private val _state = MutableStateFlow(LoginViewState())
    val state: StateFlow<LoginViewState> = _state.asStateFlow()

    private val _effect = MutableSharedFlow<LoginEffect>()
    val effect: SharedFlow<LoginEffect> = _effect.asSharedFlow()

    // 统一处理所有 Intent
    fun processIntent(intent: LoginIntent) {
        when (intent) {
            is LoginIntent.LoginClicked -> login(intent.username)
        }
    }

    private fun login(username: String) {
        viewModelScope.launch {
            _state.value = _state.value.copy(isLoading = true, errorMessage = null)
            
            // 使用 Coroutine + Flow
            kotlin.runCatching {
                // 模拟异步调用
                withContext(Dispatchers.IO) { model.performLogin(username) }
            }.onSuccess { user ->
                _state.value = _state.value.copy(isLoading = false)
                _effect.emit(LoginEffect.ShowSuccessToast("欢迎, ${user.name}!"))
            }.onFailure { error ->
                _state.value = _state.value.copy(isLoading = false, errorMessage = error.message)
            }
        }
    }
}
// 可以在 Model 中提供一个挂起函数
suspend fun UserModel.performLogin(username: String): User {
    delay(1000)
    if (username == "admin") return User("Administrator")
    else throw Exception("用户名或密码错误")
}
```

**`LoginActivity.kt` (View)**

```kotlin
// V - View: 发送 Intent，订阅 State 和 Effect
class LoginActivity : AppCompatActivity() {
    
    private val viewModel: LoginViewModel by viewModels()
    // ... UI 控件 ...

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // ... setContentView, findViewById ...

        loginButton.setOnClickListener {
            viewModel.processIntent(LoginIntent.LoginClicked(usernameEditText.text.toString()))
        }
        
        // 订阅 State 变化来更新持久性 UI
        lifecycleScope.launch {
            viewModel.state.collect { state ->
                progressBar.visibility = if (state.isLoading) View.VISIBLE else View.GONE
                state.errorMessage?.let { Toast.makeText(this@LoginActivity, it, Toast.LENGTH_SHORT).show() }
            }
        }

        // 订阅 Effect 来处理一次性事件
        lifecycleScope.launch {
            viewModel.effect.collect { effect ->
                when (effect) {
                    is LoginEffect.ShowSuccessToast -> {
                        Toast.makeText(this@LoginActivity, effect.message, Toast.LENGTH_SHORT).show()
                    }
                }
            }
        }
    }
}
```

从 MVC 到 MVI，我们可以清晰地看到一个趋势：**职责分离越来越明确，耦合度越来越低，代码的可测试性和可维护性越来越强**。特别是从 MVVM 到 MVI，我们开始更多地借鉴**函数式编程**的思想，通过管理不可变的状态和单向数据流来构建更加稳健和可预测的应用。
## Clean Architecture
相比于 MVC、MVP、MVVM 这些主要关注于 **UI层** 的架构模式外，Clean Architecture（整洁架构）是一个更高层次、更宏观的架构思想。

简单来说，如果把 MVP/MVVM 看作是如何组织一个具体页面的代码（View、Presenter/ViewModel、Model），那么 **Clean Architecture 就是如何组织整个 App 所有模块代码的宏伟蓝图**。

Clean Architecture 由 Robert C. Martin (人称 "Uncle Bob") 提出，其核心目标是 **“分离关注点” (Separation of Concerns)**，通过将软件系统划分成不同的层次，来创建一个易于维护、独立于框架、可测试性极强的系统。
### 核心思想：依赖关系原则 (The Dependency Rule)
这是 Clean Architecture 最最核心的一条规则：

> **源代码的依赖关系，只能从外层指向内层。**

想象一个洋葱或一组同心圆，内层代码对任何外层代码都一无所知。

这些层次通常代表什么呢？从内到外：

#### Entities (实体层)
这是最核心的内层。 定义了整个应用的核心业务对象和规则。在安卓中，这通常是你的数据模型类（例如 `User`, `Product`, `Order` 等），它们是纯粹的 Kotlin/Java 对象 (POJO/POKO)，不应该包含任何与安卓框架、数据库或网络相关的代码。极度稳定，变动最少。它不知道任何其他层的存在。
#### Use Cases / Interactors (用例层 / 交互器)
这是应用的业务逻辑层。封装并实现了应用的所有业务用例。例如 `LoginUseCase` (处理登录逻辑)、`GetUserProfileUseCase` (获取用户信息)、`PlaceOrderUseCase` (下单逻辑) 等。它们会协调 Entities 来完成具体的业务操作。这一层同样是纯 Kotlin/Java 代码，不依赖任何外层。它知道 Entities，但不知道谁会来调用它，也不知道数据从哪里来（是从网络还是数据库）。
#### Interface Adapters (接口适配器层)
这是数据转换层。负责将 Use Cases 和 Entities 层的数据，转换成适合外层（如UI、数据库）使用的格式，反之亦然。**你熟悉的 MVP 中的 Presenter 和 MVVM 中的 ViewModel 就生活在这一层**。此外，还包括数据库的数据映射器 (Mappers)、网络请求返回的数据模型 (DTOs) 等。

它的作用就像一个双向翻译官。例如，ViewModel 调用 Use Case，获取到纯业务数据 (Entity)，然后 ViewModel 将这个 Entity 转换成 UI 可以直接显示的格式 (UI Model)。
#### Frameworks & Drivers (框架与驱动层)
这是最外层，也是最不稳定的一层。包含所有具体的实现细节。例如：
* **UI:** Activities, Fragments, Jetpack Compose UI
* **数据库:** Room, Realm 的具体实现
* **网络:** Retrofit, Volley 的具体实现
* **安卓框架:** 各种 Android SDK 的调用

这一层是所有东西粘合在一起的地方，依赖关系都指向内部。比如，Activity 持有 ViewModel 的引用，但 ViewModel 对 Activity 一无所知。
### Clean Architecture 与 MVP/MVVM 的关系
很多人会误解 Clean Architecture 是 MVP/MVVM 的替代品，其实不是。它们是互补关系：

* **Clean Architecture 是一个宏观的、全局的架构设计。** 它定义了整个应用的模块划分和依赖方向，比如把业务逻辑 (`domain` 模块) 和数据获取 (`data` 模块) 以及界面展示 (`presentation` 模块) 分开。
* **MVP/MVVM 是一个微观的、专注于 UI 层的设计模式。** 它通常被应用在 Clean Architecture 的最外两层（Interface Adapters 和 Frameworks & Drivers）来组织 UI 代码。

可以这样理解一个典型的请求流程：
1.  **View (Activity/Fragment)** 在最外层，它接收到用户操作。
2.  View 通知 **ViewModel** (位于接口适配器层)。
3.  ViewModel 调用相应的 **Use Case** (位于用例层) 来执行业务逻辑。
4.  Use Case 可能会通过一个 **Repository 接口** (接口定义在 Use Case 层) 来请求数据。
5.  这个接口的具体实现 **RepositoryImpl** (位于框架驱动层或接口适配器层) 会决定是从网络 (Retrofit) 还是从本地数据库 (Room) 获取数据。
6.  数据从内层一步步返回，经过适配器层的转换，最终由 ViewModel 提供给 View 进行展示。

在这个流程中，`Use Case` 层根本不关心数据是来自网络还是数据库，也不关心数据最终是显示在 Activity 上还是一个 Compose 屏幕上。这就实现了彻底的解耦。
### 优点总结
1.  **独立于框架 (Framework Independent):** 核心业务逻辑不依赖于安卓 SDK，可以轻松迁移到其他平台（如桌面应用）。
2.  **可测试性强 (Testable):** 内层的业务逻辑 (Use Cases, Entities) 是纯粹的 Kotlin/Java 代码，可以进行非常快速的单元测试，无需启动模拟器。
3.  **独立于 UI (UI Independent):** 你可以随意更换你的 UI 实现（比如从 XML 布局换成 Jetpack Compose），而无需改动任何业务逻辑。
4.  **独立于数据库 (Database Independent):** 你可以从 Room 切换到其他数据库，只需要改动最外层的具体实现，核心逻辑不受影响。
5.  **代码结构清晰 (Maintainable):** 尤其在大型复杂项目中，严格的分层让代码职责分明，新人更容易上手，代码也更容易维护。