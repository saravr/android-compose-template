# Template to kick start a Jetpack Compose-based Android app

Provides all basic nuts and bolts to get going as fast as possible. Includes:
1. Jetpack Compose
2. Dagger/Hilt
3. Kotlin Serialization
4. Retrofit
5. Timber
6. Room DB

# How to start

Note: Sample built using these steps is at https://github.com/saravr/recipe-list-android.

## Setup
1. Follow "Quick Adds" section below and complete the setup, do Gradle sync!
2. Create the following directories under the app source root: api/service, di, model, repository, ui/viewmodel
3. Add Application class with hilt annotation
```
@HiltAndroidApp
class RecipeApplication: Application() {
}
```
4. Update manifest
- Add permissions needed, e.g., `<uses-permission android:name="android.permission.INTERNET" />`
- Specify application class name under <application> tag, e.g., `android:name=".RecipeApplication"` 

## Create model classes
Use the JSON response payloads from the API to create data classes under the model package. Ideally, these data classes should be separate from domain model classes. For simplicity, the model classes are reused between domain and repository (API) layers.

Note: Use the Android Studio `Kotlin data class file from JSON` plugin to automatically generate data classes from the JSON payload. Enable options "nullable with default values", etc.

## Add Network Module
Add the module class to provide network layer dependencies (using Dagger/Hilt).
```

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    private const val BASE_URL = "https://hf-android-app.s3-eu-west-1.amazonaws.com" // TODO: <-- update!
    private val json = Json {
        prettyPrint = true
        ignoreUnknownKeys = true
        encodeDefaults = true
        coerceInputValues = true // safety!
    }

    @Provides
    @Singleton
    fun provideOkHTTPClient(): OkHttpClient {
        val client = OkHttpClient.Builder()
        if (BuildConfig.DEBUG) {
            client.addInterceptor(HttpLoggingInterceptor())
        }
        return client.build()
    }

    @Provides
    @Singleton
    fun providesRetrofit(okHttpClient: OkHttpClient): Retrofit {
        val retrofit = Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
            .client(okHttpClient)
            .build()
        return retrofit
    }

    @Provides
    @Singleton
    fun provideRecipeService(retrofit: Retrofit): RecipeService {
        return retrofit.create(RecipeService::class.java)
    }
}
```

## Add Retrofit interface
Retrofit needs an interface with its annotations to help make the API calls.
```
interface RecipeService {
    @GET("/android-test/recipes.json")
    suspend fun getRecipes(): List<RecipeItem> // suspend is important, else call will fail
}
```

## Add Repository
```
interface RecipeRepository {
    val recipeList: StateFlow<List<RecipeItem>>
    suspend fun getRecipes()
}

@Singleton
class RecipeRepositoryImpl @Inject constructor(
    private val recipeService: RecipeService,
    @ApplicationScope private val scope: CoroutineScope,
): RecipeRepository {
    private val _recipeList = MutableStateFlow<List<RecipeItem>>(emptyList())
    override val recipeList = _recipeList.asStateFlow()

    init {
        refreshRecipes()
    }

    override fun refreshRecipes() {
        scope.launch {
            val recipeList = try {
                recipeService.getRecipes()
            } catch (exception: Exception) {
                Timber.e("Error fetching recipes: ${exception.message}")
                emptyList()
            }
            _recipeList.update {
                recipeList
            }
        }
    }
}
```

Add injection support for the repository.
```
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindRecipeRepository(
        recipeRepositoryImpl: RecipeRepositoryImpl,
    ): RecipeRepository
}
```

Define ApplicationScope
```
@Qualifier
@Retention(AnnotationRetention.RUNTIME)
annotation class ApplicationScope
```
## Add ViewModel
```
@HiltViewModel
class RecipeViewModel @Inject constructor(
    private val recipeRepository: RecipeRepository,
): ViewModel() {
    val recipeList = recipeRepository.recipeList

    init {
        viewModelScope.launch {
            recipeRepository.refreshRecipes()
        }
    }

    // TODO: avoid multiple simultaneous invocations
    fun refreshRecipes() {
        viewModelScope.launch {
            recipeRepository.refreshRecipes()
        }
    }
}
```

## Add Composable screen
```
@Composable
fun RecipeScreen(
    modifier: Modifier = Modifier,
    recipeList: List<RecipeItem>,
    getRecipes: suspend () -> Unit,
) {
    LaunchedEffect(Unit) {
        getRecipes()
    }

    if (recipeList.isEmpty()) {
        Box(
            modifier = modifier.fillMaxSize(),
            contentAlignment = Alignment.Center,
        ) {
            Text("No content!")
        }
    }

    LazyColumn(
        modifier = modifier,
    ) {
        items(items = recipeList, key = { it.id }) {
            Text(
                it.name,
                style = MaterialTheme.typography.bodyLarge,
                modifier = Modifier.padding(20.dp)
            )
            HorizontalDivider(color = MaterialTheme.colorScheme.surfaceVariant)
        }
    }
}
```

## Wire-up MainActivity

```
@AndroidEntryPoint
class MainActivity : ComponentActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContent {
            val recipeViewModel: RecipeViewModel = hiltViewModel()
            val recipeList by recipeViewModel.recipeList.collectAsStateWithLifecycle()

            RecepiesTheme {
                Scaffold(modifier = Modifier.fillMaxSize()) { innerPadding ->
                    RecipeScreen(
                        modifier = Modifier.padding(innerPadding),
                        recipeList = recipeList,
                        getRecipes = {
                            recipeViewModel.getRecipeList()
                        }
                    )
                }
            }
        }
    }
}
```

# Quick Adds

The following can be added *on top of* the template created by Android Studio (for Compose based app).

## libs.versions.toml

```
[versions]
androidUtilities = "0.0.9"
coilComposeVersion = "2.6.0"
constraintLayoutVersion = "1.1.0"
datastoreVersion = "1.1.2"
hiltNavigationCompose = "1.2.0"
materialIconsExtendedVersion = "1.7.8"
kotlinxSerializationJson = "1.7.3"
loggingInterceptor = "4.12.0"
hiltAndroid = "2.51.1"
material3Version = "1.3.1"
mockk = "1.13.13"
retrofit = "2.11.0"
retrofit2KotlinxSerializationConverter = "1.0.0"
splashscreenVersion = "1.0.1"
timber = "5.0.1"

[libraries]
android-utilities = { module = "com.github.saravr:android-utilities", version.ref = "androidUtilities" }
androidx-constraint-layout = { module = "androidx.constraintlayout:constraintlayout-compose", version.ref = "constraintLayoutVersion" }
androidx-datastore-preferences = { module = "androidx.datastore:datastore-preferences", version.ref = "datastoreVersion" }
androidx-hilt-navigation-compose = { module = "androidx.hilt:hilt-navigation-compose", version.ref = "hiltNavigationCompose" }
androidx-material-icons-extended = { module = "androidx.compose.material:material-icons-extended", version.ref = "materialIconsExtendedVersion" }
coil-compose = { module = "io.coil-kt:coil-compose", version.ref = "coilComposeVersion" }
hilt-android = { module = "com.google.dagger:hilt-android", version.ref = "hiltAndroid" }
hilt-android-compiler = { module = "com.google.dagger:hilt-android-compiler", version.ref = "hiltAndroid" }
androidx-compose-material3-windowsizeclass = { module = "androidx.compose.material3:material3-window-size-class", version.ref = "material3Version" }
kotlinx-serialization-json = { module = "org.jetbrains.kotlinx:kotlinx-serialization-json", version.ref = "kotlinxSerializationJson" }
logging-interceptor = { module = "com.squareup.okhttp3:logging-interceptor", version.ref = "loggingInterceptor" }
mockk = { module = "io.mockk:mockk", version.ref = "mockk" }
mockk-android = { module = "io.mockk:mockk-android", version.ref = "mockk" }
retrofit = { module = "com.squareup.retrofit2:retrofit", version.ref = "retrofit" }
retrofit2-kotlinx-serialization-converter = { module = "com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter", version.ref = "retrofit2KotlinxSerializationConverter" }
splashscreen = { module = "androidx.core:core-splashscreen", version.ref = "splashscreenVersion" }
timber = { module = "com.jakewharton.timber:timber", version.ref = "timber" }
```

## (project-level) build.gradle.kts

Note: ensure kotlin versions mentioned in gradle files should match the one listed in libs.versions.toml.
```
plugins {
    id("com.google.devtools.ksp") version "2.0.21-1.0.27" apply false
    id("com.google.dagger.hilt.android") version "2.51.1" apply false
}
```

## app/build.gradle.kts
```
plugins {
    // add ...
    id("com.google.devtools.ksp")
    id("com.google.dagger.hilt.android")
    kotlin("plugin.serialization") version "2.0.21"
}

android {
   defaultConfig {
        minSdk = 25 // helps!!
   }

   buildFeatures {
      buildConfig = true
   }
}

dependencies {
    // add ...
    // additional android/compose libraries
    implementation(libs.androidx.compose.material3.windowsizeclass)
    implementation(libs.androidx.constraint.layout)
    implementation(libs.splashscreen)

    // for now -- import only needed icons to reduce apk size
    implementation(libs.androidx.material.icons.extended)

    // hilt
    implementation(libs.hilt.android)
    ksp(libs.hilt.android.compiler)
    implementation(libs.androidx.hilt.navigation.compose)

    // kotlin serialization
    implementation(libs.kotlinx.serialization.json)
    implementation(libs.retrofit2.kotlinx.serialization.converter)

    // retrofit
    implementation(libs.retrofit)
    implementation(libs.logging.interceptor)

    // coil
    implementation(libs.coil.compose)

    // timber
    implementation(libs.timber)

    // utilities
    implementation(libs.android.utilities)
    implementation(libs.androidx.datastore.preferences)
}

ksp {
    arg("dagger.hilt.internal.useAggregatingRootProcessor", "false")
}
```

## settings.gradle.kts
Needed only if using the utilities library https://github.com/saravr/android-utilities.

```
dependencyResolutionManagement {
    repositories {
        // add ...
        maven { url = uri("https://jitpack.io") }
    }
}
```
