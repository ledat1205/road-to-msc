### Very High Frequency (almost every module has at least some of these)

|Annotation / Decorator|Where used|Common purpose / real-world use case|Example snippet|
|---|---|---|---|
|@Provides|@Module methods|Create instances of classes you don’t own or need custom construction (Retrofit, OkHttp, Room, Gson, etc.)|@Provides @Singleton fun provideRetrofit(): Retrofit { … }|
|@Binds|@Module abstract methods|Bind interface → concrete implementation (most efficient, no extra codegen)|@Binds abstract fun bindRepo(impl: RepoImpl): Repo|
|@Singleton / @ApplicationScope|Class or @Provides/@Binds|Application-wide single instance (most common scope in Android)|@Singleton class AppDatabase @Inject constructor()|
|@InstallIn|@Module|Tells Hilt which component to install the module into (replaced old @Component(modules = …))|@InstallIn(SingletonComponent::class)|
|@Inject|Constructor, field, method|Constructor injection (most common), field injection (ViewModels, Activities, Fragments)|@Inject constructor(private val repo: Repo)|
|@HiltViewModel|ViewModel classes|Required for Hilt to inject into ViewModels (replaces manual ViewModelFactory)|@HiltViewModel class HomeViewModel @Inject constructor(…)|

### High Frequency (appear in most medium/large apps)

|Annotation / Decorator|Where used|Common purpose / real-world use case|Example snippet|
|---|---|---|---|
|@Qualifier + custom annotation|Custom annotations|Distinguish multiple bindings of the same type (e.g. two OkHttpClients: auth vs public)|@Qualifier @Retention(BINARY) annotation class AuthenticatedClient|
|@Named|@Provides / parameters|Quick & dirty qualifier when you don’t want a custom annotation (less type-safe)|@Provides @Named("logging") fun provideLogger(): Logger|
|@ApplicationContext|Parameters in @Provides|Inject Context as application context (safer than @ActivityContext)|@Provides fun provideSharedPrefs(@ApplicationContext ctx: Context): SharedPreferences|
|@IntoSet / @IntoMap|@Provides / @Binds|Multibindings – collect many implementations into a Set or Map (e.g. interceptors, initializers)|@IntoSet @Provides fun provideInterceptor(): Interceptor|
|@ActivityScoped / @FragmentScoped|Classes / bindings|Scope tied to Activity / Fragment lifecycle (used less since Hilt ViewModel prefers @ViewModelScoped)|@ActivityScoped class AnalyticsTracker|
|@ViewModelScoped|Bindings inside ViewModel|Rare but useful for objects scoped to a specific ViewModel instance|(Used with @InstallIn(ViewModelComponent::class))|

### Medium Frequency (common in mature / large codebases)

|Annotation / Decorator|Where used|Common purpose / real-world use case|Example snippet|
|---|---|---|---|
|@Module(includes = …)|@Module|One module explicitly depends on another (e.g. DataModule includes CoreModule)|@Module(includes = [CoreModule::class, NetworkModule::class])|
|@EntryPoint|Interfaces|Access Hilt-provided objects from non-Hilt-aware code (e.g. BroadcastReceiver, WorkManager, ContentProvider)|@EntryPoint @InstallIn(SingletonComponent::class) interface AppEntryPoint { fun tokenManager(): TokenManager }|
|@TestInstallIn|Test modules|Replace production modules in tests (very common in unit/integration tests)|@TestInstallIn(components = [SingletonComponent::class], replaces = [CoreModule::class])|
|@AssistedInject / @Assisted|Assisted injection classes|Inject objects that need runtime parameters (e.g. ViewModel with SavedStateHandle + extra args)|@AssistedInject constructor(@Assisted val userId: String, repo: Repo)|
|@Reusable|Bindings|Optimization: allow Dagger to reuse instances even if not scoped (less memory than @Singleton)|@Reusable @Provides fun provideMapper(): Mapper|
|@ActivityContext|Parameters|Inject Activity context (use sparingly – prefer @ApplicationContext)|@Provides fun provideNavigator(@ActivityContext ctx: Context): Navigator|

### Lower Frequency but Still Seen in Production

|Annotation / Decorator|Typical use case|
|---|---|
|@Subcomponent|Feature-level subgraphs (e.g. LoginComponent, PaymentComponent)|
|@Component.Factory / @Subcomponent.Factory|Manual component creation with parameters (less common since Hilt)|
|@Module(subcomponents = …)|Declare subcomponents inside a parent module (advanced)|
|@Unscoped|Explicitly mark something as unscoped (rare, mostly for documentation)|
|@MapKey (custom)|Advanced multibindings with complex keys (e.g. @ViewModelKey for ViewModel map)|

### Quick "What to Use When" Cheat Sheet

|You want to…|Most common choice(s)|
|---|---|
|Bind interface → impl|@Binds|
|Create Retrofit / OkHttp / Room|@Provides|
|Scope to whole app|@Singleton|
|Scope to one screen / ViewModel|@HiltViewModel + constructor injection|
|Distinguish two same-type bindings|Custom @Qualifier or @Named|
|Collect many interceptors / plugins|@IntoSet or @IntoMap|
|Make module depend on another module|@Module(includes = [OtherModule::class])|
|Access Hilt from non-Hilt class|@EntryPoint|
|Replace module in tests|@TestInstallIn(replaces = …)|

### Interview / Code Review Talking Points

- "We prefer @Binds over @Provides for our own implementations because it generates less code and is more performant."
- "We use custom @Qualifier annotations instead of @Named for type-safety when we have multiple bindings of the same type."
- "Core infrastructure (network, db) goes into a CoreModule with @Provides, while feature-specific repositories use @Binds and include the core module."
- "In tests we always use @TestInstallIn to swap real Retrofit with a mock server."