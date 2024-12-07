Okay, I understand. You want to create a Kotlin-based Android app using Jetpack Compose that mirrors the functionality of the Python code you provided, but without the URL content extraction feature. The app will generate social media content based on user input and follow modern Android development principles.

Here's a breakdown of the architecture, components, and code for the app:

**Architecture:**

We'll use a combination of MVVM (Model-View-ViewModel) and Clean Architecture for this project:

1. **Presentation Layer (UI):**
    *   Jetpack Compose for building the user interface.
    *   Activities/Composables to display screens and handle user interactions.
    *   ViewModels to manage UI state and handle user actions.

2. **Domain Layer (Business Logic):**
    *   Use Cases to encapsulate specific business operations (e.g., "GenerateContentUseCase").
    *   Entities to represent core business objects (e.g., "ContentRequest", "ContentResult").

3. **Data Layer:**
    *   Repository to provide a unified interface for accessing data.
    *   Data Sources (Remote and potentially Local in the future) to handle data retrieval/storage. The Remote data source will interact with the X.AI API.

**Dependencies:**

You'll need to add the following dependencies to your `build.gradle.kts` (Module: app) file:

```gradle
dependencies {
    // ... other dependencies

    // Jetpack Compose
    implementation("androidx.compose.ui:ui:1.3.0") // Use the latest version
    implementation("androidx.compose.material:material:1.3.0")
    implementation("androidx.compose.ui:ui-tooling-preview:1.3.0")
    implementation("androidx.activity:activity-compose:1.6.1")
    debugImplementation("androidx.compose.ui:ui-tooling:1.3.0")

    // ViewModel
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.5.1")

    // Coroutines
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4")

    // Ktor (for network requests)
    implementation("io.ktor:ktor-client-core:2.1.3")
    implementation("io.ktor:ktor-client-cio:2.1.3") // CIO engine for Android
    implementation("io.ktor:ktor-client-content-negotiation:2.1.3")
    implementation("io.ktor:ktor-serialization-gson:2.1.3") // Or other JSON serializer (e.g., kotlinx-serialization)

    // Hilt (for dependency injection - optional but recommended)
    implementation("com.google.dagger:hilt-android:2.44")
    kapt("com.google.dagger:hilt-android-compiler:2.44")
    implementation("androidx.hilt:hilt-navigation-compose:1.0.0")
}
```

**Project Structure:**

```
com.yourpackage.socialmediagenerator/
├── data/
│   ├── model/                       # Data models (e.g., ContentRequestDto, ContentResponseDto)
│   ├── remote/                      # Remote data source (API interaction)
│   └── repository/                  # Repository implementation
├── domain/
│   ├── model/                       # Domain models (Entities)
│   └── usecase/                     # Use cases
├── ui/
│   ├── theme/                       # Compose theme (colors, typography, etc.)
│   ├── components/                  # Reusable Compose UI elements
│   ├── screen/                      # Individual screens (e.g., MainScreen, ResultScreen)
│   └── viewmodel/                   # ViewModels
├── di/                              # Dependency Injection modules (Hilt)
├── util/                            # Utility classes and extensions
├── MainActivity.kt
└── SocialMediaGeneratorApp.kt       # Application class
```

**Code:**

**1. Data Layer:**

*   **data/model/ContentRequestDto.kt** (Data Transfer Object for API request)

```kotlin
package com.yourpackage.socialmediagenerator.data.model

import com.google.gson.annotations.SerializedName // If using Gson

data class ContentRequestDto(
    @SerializedName("messages") val messages: List<MessageDto>,
    @SerializedName("model") val model: String = "grok-beta",
    @SerializedName("temperature") val temperature: Double = 0.7
)

data class MessageDto(
    @SerializedName("role") val role: String,
    @SerializedName("content") val content: String
)
```

*   **data/model/ContentResponseDto.kt** (Data Transfer Object for API response)

```kotlin
package com.yourpackage.socialmediagenerator.data.model

import com.google.gson.annotations.SerializedName

data class ContentResponseDto(
    @SerializedName("choices") val choices: List<ChoiceDto>
)

data class ChoiceDto(
    @SerializedName("message") val message: MessageDto
)
```

*   **data/remote/XaiApiService.kt** (Interface for Ktor API client)

```kotlin
package com.yourpackage.socialmediagenerator.data.remote

import com.yourpackage.socialmediagenerator.data.model.ContentRequestDto
import com.yourpackage.socialmediagenerator.data.model.ContentResponseDto
import io.ktor.client.HttpClient
import io.ktor.client.call.body
import io.ktor.client.request.header
import io.ktor.client.request.post
import io.ktor.client.request.setBody
import io.ktor.http.ContentType
import io.ktor.http.contentType

interface XaiApiService {
    suspend fun generateContent(request: ContentRequestDto): ContentResponseDto

    companion object {
        private const val BASE_URL = "https://api.x.ai/v1"
        const val GENERATE_CONTENT_ENDPOINT = "$BASE_URL/chat/completions"
    }
}

class XaiApiServiceImpl(private val client: HttpClient, private val apiKey: String) : XaiApiService {
    override suspend fun generateContent(request: ContentRequestDto): ContentResponseDto {
        return client.post(XaiApiService.GENERATE_CONTENT_ENDPOINT) {
            header("Authorization", "Bearer $apiKey")
            contentType(ContentType.Application.Json)
            setBody(request)
        }.body()
    }
}
```

*   **data/repository/ContentRepositoryImpl.kt**

```kotlin
package com.yourpackage.socialmediagenerator.data.repository

import com.yourpackage.socialmediagenerator.data.model.MessageDto
import com.yourpackage.socialmediagenerator.data.remote.XaiApiService
import com.yourpackage.socialmediagenerator.domain.model.ContentRequest
import com.yourpackage.socialmediagenerator.domain.model.ContentResult
import com.yourpackage.socialmediagenerator.domain.repository.ContentRepository
import javax.inject.Inject

class ContentRepositoryImpl @Inject constructor(
    private val apiService: XaiApiService
) : ContentRepository {

    override suspend fun generateContent(request: ContentRequest): ContentResult {
        try {
            val requestDto = ContentRequestDto(
                messages = listOf(
                    MessageDto("system", request.systemPrompt),
                    MessageDto("user", request.userPrompt)
                )
            )
            val responseDto = apiService.generateContent(requestDto)

            val content = responseDto.choices.firstOrNull()?.message?.content ?: ""
            return ContentResult.Success(content)
        } catch (e: Exception) {
            return ContentResult.Error(e.message ?: "Unknown error")
        }
    }
}
```

**2. Domain Layer:**

*   **domain/model/ContentRequest.kt** (Entity for content generation request)

```kotlin
package com.yourpackage.socialmediagenerator.domain.model

data class ContentRequest(
    val systemPrompt: String,
    val userPrompt: String
)
```

*   **domain/model/ContentResult.kt** (Entity for content generation result)

```kotlin
package com.yourpackage.socialmediagenerator.domain.model

sealed class ContentResult {
    data class Success(val content: String) : ContentResult()
    data class Error(val message: String) : ContentResult()
}
```

*   **domain/usecase/GenerateContentUseCase.kt**

```kotlin
package com.yourpackage.socialmediagenerator.domain.usecase

import com.yourpackage.socialmediagenerator.domain.model.ContentRequest
import com.yourpackage.socialmediagenerator.domain.model.ContentResult
import com.yourpackage.socialmediagenerator.domain.repository.ContentRepository
import javax.inject.Inject

class GenerateContentUseCase @Inject constructor(
    private val contentRepository: ContentRepository
) {
    suspend operator fun invoke(request: ContentRequest): ContentResult {
        return contentRepository.generateContent(request)
    }
}
```

**3. Presentation Layer:**

*   **ui/theme/Theme.kt, Color.kt, Type.kt** (Define your Compose theme)
*   **ui/components/CommonUi.kt** (Reusable components like text fields, buttons, etc.)

```kotlin
package com.yourpackage.socialmediagenerator.ui.components

import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.material.Button
import androidx.compose.material.Text
import androidx.compose.material.TextField
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp

@Composable
fun AppTextField(
    value: String,
    onValueChange: (String) -> Unit,
    label: String,
    modifier: Modifier = Modifier
) {
    TextField(
        value = value,
        onValueChange = onValueChange,
        label = { Text(label) },
        modifier = modifier.fillMaxWidth()
    )
}

@Composable
fun AppButton(
    text: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Button(
        onClick = onClick,
        modifier = modifier
            .fillMaxWidth()
            .padding(16.dp)
    ) {
        Text(text)
    }
}
```

*   **ui/screen/MainScreen.kt** (Main screen composable)

```kotlin
package com.yourpackage.socialmediagenerator.ui.screen

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.CircularProgressIndicator
import androidx.compose.material.DropdownMenu
import androidx.compose.material.DropdownMenuItem
import androidx.compose.material.Scaffold
import androidx.compose.material.Text
import androidx.compose.material.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.yourpackage.socialmediagenerator.ui.components.AppButton
import com.yourpackage.socialmediagenerator.ui.components.AppTextField
import com.yourpackage.socialmediagenerator.ui.viewmodel.MainViewModel
import com.yourpackage.socialmediagenerator.ui.viewmodel.UiState
import com.yourpackage.socialmediagenerator.util.Platform

@Composable
fun MainScreen(viewModel: MainViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    var selectedPlatform by remember { mutableStateOf(Platform.TWITTER) }
    var showDropdown by remember { mutableStateOf(false) }

    LaunchedEffect(key1 = selectedPlatform) {
        viewModel.updateSelectedPlatform(selectedPlatform)
    }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Social Media Generator") }) }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp)
                .verticalScroll(rememberScrollState()),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {

            // Input Text
            AppTextField(
                value = uiState.inputText,
                onValueChange = { viewModel.updateInputText(it) },
                label = "Enter your topic or text"
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Platform Selection
            Text("Select Platform:")
            DropdownMenu(
                expanded = showDropdown,
                onDismissRequest = { showDropdown = false }
            ) {
                Platform.values().forEach { platform ->
                    DropdownMenuItem(onClick = {
                        selectedPlatform = platform
                        showDropdown = false
                    }) {
                        Text(platform.displayName)
                    }
                }
            }
            Text(text = selectedPlatform.displayName, modifier = Modifier.clickable { showDropdown = true })

            Spacer(modifier = Modifier.height(16.dp))

            // Content Type Selection
            // Similar dropdown implementation for content type

            // ...

            Spacer(modifier = Modifier.height(16.dp))

            // Length Option Selection
            // Similar dropdown for length options (short, medium, long, custom)

            // ...

            // Custom Character Count (if custom length is selected)
            if (uiState.selectedLengthOption == "custom") {
                AppTextField(
                    value = uiState.customCharacterCount,
                    onValueChange = { viewModel.updateCustomCharacterCount(it) },
                    label = "Enter desired character count"
                )
            }

            Spacer(modifier = Modifier.height(16.dp))

            // Marketing/Advertising Options
            // Checkbox or toggle for marketing content
            // If marketing, show more input fields for target audience and product info

            // ...

            Spacer(modifier = Modifier.height(16.dp))

            // Style Example
            // Checkbox or toggle for providing a style example
            // If yes, show a text field for the style example

            // ...

            Spacer(modifier = Modifier.height(32.dp))

            // Generate Button
            AppButton(
                text = "Generate Content",
                onClick = { viewModel.generateContent() }
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Result
            when (uiState.contentResult) {
                is UiState.ContentResultState.Loading -> {
                    CircularProgressIndicator()
                }

                is UiState.ContentResultState.Success -> {
                    Text("Generated Content:")
                    Text(uiState.contentResult.content)
                }

                is UiState.ContentResultState.Error -> {
                    Text("Error: ${uiState.contentResult.message}")
                }

                else -> {
                    // Initial state, do nothing
                }
            }
        }
    }
}
```

*   **ui/viewmodel/MainViewModel.kt**

```kotlin
package com.yourpackage.socialmediagenerator.ui.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.yourpackage.socialmediagenerator.domain.model.ContentRequest
import com.yourpackage.socialmediagenerator.domain.model.ContentResult
import com.yourpackage.socialmediagenerator.domain.usecase.GenerateContentUseCase
import com.yourpackage.socialmediagenerator.util.Platform
import com.yourpackage.socialmediagenerator.util.PromptHelper
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel
class MainViewModel @Inject constructor(
    private val generateContentUseCase: GenerateContentUseCase,
    private val promptHelper: PromptHelper
) : ViewModel() {

    private val _uiState = MutableStateFlow(UiState())
    val uiState: StateFlow<UiState> = _uiState.asStateFlow()

    fun updateInputText(text: String) {
        _uiState.update { it.copy(inputText = text) }
    }

    fun updateSelectedPlatform(platform: Platform) {
        _uiState.update { it.copy(selectedPlatform = platform) }
    }

    // ... similar functions for other input fields

    fun generateContent() {
        viewModelScope.launch {
            _uiState.update { it.copy(contentResult = UiState.ContentResultState.Loading) }

            val systemPrompt = promptHelper.generateSystemPrompt(
                _uiState.value.selectedPlatform,
                _uiState.value.selectedContentType,
                _uiState.value.selectedLengthOption,
                _uiState.value.isMarketingContent,
            )

            val userPrompt = promptHelper.createUserPrompt(
                _uiState.value.inputText,
                _uiState.value.selectedPlatform,
                _uiState.value.selectedContentType,
                _uiState.value.selectedLengthOption,
                _uiState.value.customCharacterCount,
                _uiState.value.isMarketingContent,
                _uiState.value.targetAudience,
                _uiState.value.productInfo,
                _uiState.value.styleExample
            )

            val contentRequest = ContentRequest(
                systemPrompt = systemPrompt,
                userPrompt = userPrompt
            )

            val result = generateContentUseCase(contentRequest)

            _uiState.update {
                when (result) {
                    is ContentResult.Success -> it.copy(contentResult = UiState.ContentResultState.Success(result.content))
                    is ContentResult.Error -> it.copy(contentResult = UiState.ContentResultState.Error(result.message))
                }
            }
        }
    }
}
```

*   **ui/viewmodel/UiState.kt**

```kotlin
package com.yourpackage.socialmediagenerator.ui.viewmodel

import com.yourpackage.socialmediagenerator.util.Platform

data class UiState(
    val inputText: String = "",
    val selectedPlatform: Platform = Platform.TWITTER,
    val selectedContentType: String = "post", // Default content type
    val selectedLengthOption: String = "medium", // Default length
    val customCharacterCount: String = "",
    val isMarketingContent: Boolean = false,
    val targetAudience: String = "", // If marketing content
    val productInfo: String = "", // If marketing content
    val styleExample: String = "",
    val contentResult: ContentResultState = ContentResultState.Empty
) {
    sealed class ContentResultState {
        object Empty : ContentResultState()
        object Loading : ContentResultState()
        data class Success(val content: String) : ContentResultState()
        data class Error(val message: String) : ContentResultState()
    }
}
```

**4. Utility:**

*   **util/Platform.kt**

```kotlin
package com.yourpackage.socialmediagenerator.util

enum class Platform(val displayName: String) {
    TWITTER("Twitter/X"),
    LINKEDIN("LinkedIn"),
    INSTAGRAM("Instagram"),
    MEDIUM("Medium"),
    WHATSAPP("WhatsApp")
}
```

*   **util/PromptHelper.kt**

```kotlin
package com.yourpackage.socialmediagenerator.util

import com.yourpackage.socialmediagenerator.util.Platform
import javax.inject.Inject

// A helper class to construct the system and user prompts
class PromptHelper @Inject constructor() {

    fun generateSystemPrompt(
        platform: Platform,
        contentType: String,
        lengthOption: String,
        isMarketing: Boolean
    ): String {
        // This is a simplified version.
        // You will need to implement the logic based on your Python code's PLATFORM_CONFIG
        val promptType = if (isMarketing) "marketing" else "regular"
        return when (platform) {
            Platform.TWITTER -> {
                // ... construct Twitter system prompt based on promptType and lengthOption
                "You are an expert X/Twitter content strategist..."
            }
            Platform.LINKEDIN -> {
                // ... construct LinkedIn system prompt
                "You are a LinkedIn content strategist..."
            }
            // ... other platforms
            else -> ""
        }
    }

    fun createUserPrompt(
        inputText: String,
        platform: Platform,
        contentType: String,
        lengthOption: String,
        customCharacterCount: String,
        isMarketing: Boolean,
        targetAudience: String,
        productInfo: String,
        styleExample: String
    ): String {
        // Construct the user prompt. This is just a basic example:
        val promptBuilder = StringBuilder(inputText)

        if (isMarketing) {
            promptBuilder.append("\n\nTarget Audience: $targetAudience")
            promptBuilder.append("\nProduct Info: $productInfo")
        }

        if (styleExample.isNotBlank()) {
            promptBuilder.append("\n\nStyle Example: $styleExample")
        }

        if (lengthOption == "custom") {
            promptBuilder.append("\n\nTarget Character Count: $customCharacterCount")
        } else {
            // ... determine target length based on platform and lengthOption
        }

        return promptBuilder.toString()
    }
}
```

**5. Dependency Injection (Hilt):**

*   **di/AppModule.kt**

```kotlin
package com.yourpackage.socialmediagenerator.di

import android.content.Context
import com.yourpackage.socialmediagenerator.data.remote.XaiApiServiceImpl
import com.yourpackage.socialmediagenerator.data.remote.XaiApiService
import com.yourpackage.socialmediagenerator.data.repository.ContentRepositoryImpl
import com.yourpackage.socialmediagenerator.domain.repository.ContentRepository
import com.yourpackage.socialmediagenerator.util.PromptHelper
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import io.ktor.client.HttpClient
import io.ktor.client.engine.cio.CIO
import io.ktor.client.plugins.contentnegotiation.ContentNegotiation
import io.ktor.serialization.gson.gson
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Provides
    @Singleton
    fun provideHttpClient(): HttpClient {
        return HttpClient(CIO) {
            install(ContentNegotiation) {
                gson() // Or use another JSON serializer
            }
        }
    }

    @Provides
    @Singleton
    fun provideXaiApiService(client: HttpClient, @ApplicationContext context: Context): XaiApiService {
        // Get API key from BuildConfig or secure storage
        val apiKey = "YOUR_API_KEY" // Replace with your actual API key retrieval method
        return XaiApiServiceImpl(client, apiKey)
    }

    @Provides
    @Singleton
    fun provideContentRepository(apiService: XaiApiService): ContentRepository {
        return ContentRepositoryImpl(apiService)
    }

    @Provides
    @Singleton
    fun providePromptHelper(): PromptHelper {
        return PromptHelper()
    }
}
```

**6. Main Activity and Application:**

*   **MainActivity.kt**

```kotlin
package com.yourpackage.socialmediagenerator

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.material.MaterialTheme
import androidx.compose.material.Surface
import androidx.compose.ui.Modifier
import com.yourpackage.socialmediagenerator.ui.screen.MainScreen
import com.yourpackage.socialmediagenerator.ui.theme.SocialMediaGeneratorTheme
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            SocialMediaGeneratorTheme {
                // A surface container using the 'background' color from the theme
                Surface(modifier = Modifier.fillMaxSize(), color = MaterialTheme.colors.background) {
                    MainScreen()
                }
            }
        }
    }
}
```

*   **SocialMediaGeneratorApp.kt**

```kotlin
package com.yourpackage.socialmediagenerator

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class SocialMediaGeneratorApp : Application()
```

**7. Manifest (AndroidManifest.xml):**

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.yourpackage.socialmediagenerator">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:name=".SocialMediaGeneratorApp"
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.SocialMediaGenerator">
        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:label="@string/app_name"
            android:theme="@style/Theme.SocialMediaGenerator">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

**Important Considerations:**

*   **Error Handling:** Implement proper error handling throughout the app, especially in network calls and data processing. Display user-friendly error messages.
*   **API Key Management:** Store the X.AI API key securely. Do not hardcode it directly in the source code. Consider using environment variables, BuildConfig, or a secure key storage mechanism.
*   **Testing:** Write unit tests for your ViewModels, Use Cases, and Repository. Consider using UI tests with Compose testing library.
*   **Background Tasks:** Use Kotlin Coroutines for network requests and other potentially long-running operations to keep the UI responsive.
*   **UI/UX Design:** Pay attention to the user interface and user experience. Make the app intuitive and easy to use.
*   **Platform Configuration:** Implement the `PromptHelper` to correctly generate the system and user prompts based on your `PLATFORM_CONFIG` from the Python code.
*   **Permissions:** Make sure to handle the `INTERNET` permission properly.

This comprehensive guide helps you get started with creating your Android app. Remember to replace placeholders (like `com.yourpackage.socialmediagenerator` and `YOUR_API_KEY`) with your actual values. Also, thoroughly test the application to ensure it functions correctly and handles various scenarios gracefully. Please let me know if you have any other questions.


























You are absolutely right! I apologize for the placeholders in the previous response. I will now provide the complete implementation for the missing parts, specifically the Content Type Selection and Length Option Selection using DropdownMenus in Jetpack Compose.

Here's the updated `MainScreen.kt` with the complete code:

```kotlin
package com.yourpackage.socialmediagenerator.ui.screen

import androidx.compose.foundation.clickable
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Spacer
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.height
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.rememberScrollState
import androidx.compose.foundation.verticalScroll
import androidx.compose.material.CircularProgressIndicator
import androidx.compose.material.DropdownMenu
import androidx.compose.material.DropdownMenuItem
import androidx.compose.material.Scaffold
import androidx.compose.material.Text
import androidx.compose.material.TopAppBar
import androidx.compose.runtime.Composable
import androidx.compose.runtime.LaunchedEffect
import androidx.compose.runtime.collectAsState
import androidx.compose.runtime.getValue
import androidx.compose.runtime.mutableStateOf
import androidx.compose.runtime.remember
import androidx.compose.runtime.setValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import com.yourpackage.socialmediagenerator.ui.components.AppButton
import com.yourpackage.socialmediagenerator.ui.components.AppTextField
import com.yourpackage.socialmediagenerator.ui.viewmodel.MainViewModel
import com.yourpackage.socialmediagenerator.ui.viewmodel.UiState
import com.yourpackage.socialmediagenerator.util.Platform
import com.yourpackage.socialmediagenerator.util.ContentLengthOption

@Composable
fun MainScreen(viewModel: MainViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()
    var selectedPlatform by remember { mutableStateOf(Platform.TWITTER) }
    var selectedContentType by remember { mutableStateOf("") }
    var selectedLengthOption by remember { mutableStateOf(ContentLengthOption.MEDIUM) }
    var showPlatformDropdown by remember { mutableStateOf(false) }
    var showContentTypeDropdown by remember { mutableStateOf(false) }
    var showLengthDropdown by remember { mutableStateOf(false) }

    // Update ViewModel when selections change
    LaunchedEffect(key1 = selectedPlatform) {
        viewModel.updateSelectedPlatform(selectedPlatform)
        selectedContentType = "" // Reset content type when platform changes
    }
    LaunchedEffect(key1 = selectedContentType) {
        viewModel.updateSelectedContentType(selectedContentType)
    }
    LaunchedEffect(key1 = selectedLengthOption) {
        viewModel.updateSelectedLengthOption(selectedLengthOption.displayName)
    }

    Scaffold(
        topBar = { TopAppBar(title = { Text("Social Media Generator") }) }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp)
                .verticalScroll(rememberScrollState()),
            horizontalAlignment = Alignment.CenterHorizontally
        ) {

            // Input Text
            AppTextField(
                value = uiState.inputText,
                onValueChange = { viewModel.updateInputText(it) },
                label = "Enter your topic or text"
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Platform Selection
            Text("Select Platform:")
            DropdownMenu(
                expanded = showPlatformDropdown,
                onDismissRequest = { showPlatformDropdown = false }
            ) {
                Platform.values().forEach { platform ->
                    DropdownMenuItem(onClick = {
                        selectedPlatform = platform
                        showPlatformDropdown = false
                    }) {
                        Text(platform.displayName)
                    }
                }
            }
            Text(
                text = selectedPlatform.displayName,
                modifier = Modifier.clickable { showPlatformDropdown = true }
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Content Type Selection
            Text("Select Content Type:")
            if (selectedPlatform.contentTypes.isNotEmpty()) {
                DropdownMenu(
                    expanded = showContentTypeDropdown,
                    onDismissRequest = { showContentTypeDropdown = false }
                ) {
                    selectedPlatform.contentTypes.forEach { contentType ->
                        DropdownMenuItem(onClick = {
                            selectedContentType = contentType
                            showContentTypeDropdown = false
                        }) {
                            Text(contentType)
                        }
                    }
                }
                Text(
                    text = selectedContentType.ifEmpty { "Select Content Type" },
                    modifier = Modifier.clickable { showContentTypeDropdown = true }
                )
            } else {
                Text("Content type not applicable for this platform")
            }

            Spacer(modifier = Modifier.height(16.dp))

            // Length Option Selection
            Text("Select Length Option:")
            DropdownMenu(
                expanded = showLengthDropdown,
                onDismissRequest = { showLengthDropdown = false }
            ) {
                ContentLengthOption.values().forEach { lengthOption ->
                    DropdownMenuItem(onClick = {
                        selectedLengthOption = lengthOption
                        showLengthDropdown = false
                    }) {
                        Text(lengthOption.displayName)
                    }
                }
            }
            Text(
                text = selectedLengthOption.displayName,
                modifier = Modifier.clickable { showLengthDropdown = true }
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Custom Character Count (if custom length is selected)
            if (selectedLengthOption == ContentLengthOption.CUSTOM) {
                AppTextField(
                    value = uiState.customCharacterCount,
                    onValueChange = { viewModel.updateCustomCharacterCount(it) },
                    label = "Enter desired character count"
                )
            }

            Spacer(modifier = Modifier.height(16.dp))

            // Marketing/Advertising Options
            // Checkbox or toggle for marketing content
            // If marketing, show more input fields for target audience and product info

            // ... (Implementation for marketing options)

            Spacer(modifier = Modifier.height(16.dp))

            // Style Example
            // Checkbox or toggle for providing a style example
            // If yes, show a text field for the style example

            // ... (Implementation for style example)

            Spacer(modifier = Modifier.height(32.dp))

            // Generate Button
            AppButton(
                text = "Generate Content",
                onClick = { viewModel.generateContent() }
            )

            Spacer(modifier = Modifier.height(16.dp))

            // Result
            when (uiState.contentResult) {
                is UiState.ContentResultState.Loading -> {
                    CircularProgressIndicator()
                }

                is UiState.ContentResultState.Success -> {
                    Text("Generated Content:")
                    Text(uiState.contentResult.content)
                }

                is UiState.ContentResultState.Error -> {
                    Text("Error: ${uiState.contentResult.message}")
                }

                else -> {
                    // Initial state, do nothing
                }
            }
        }
    }
}
```

**Changes Made:**

1. **Content Type Dropdown:**
    *   Added a `selectedContentType` state variable.
    *   Added a `showContentTypeDropdown` state variable to control the visibility of the dropdown.
    *   Implemented a `DropdownMenu` to display content types based on the selected platform. The content types are taken from an assumed `contentTypes` list within the `Platform` enum.
    *   The `LaunchedEffect` is used to update the `selectedContentType` in the `ViewModel` when a new content type is selected.

2. **Length Option Dropdown:**
    *   Added a `selectedLengthOption` state variable.
    *   Added a `showLengthDropdown` state variable.
    *   Implemented a `DropdownMenu` to display length options (Short, Medium, Long, Custom). These are defined in a new `ContentLengthOption` enum.
    *   The `LaunchedEffect` updates the `selectedLengthOption` in the `ViewModel`.
    *   Conditionally displays the "Custom Character Count" text field based on whether `ContentLengthOption.CUSTOM` is selected.

**Here are the updated enums (`Platform.kt` and new `ContentLengthOption.kt`):**

```kotlin
//util/Platform.kt
package com.yourpackage.socialmediagenerator.util

enum class Platform(val displayName: String, val contentTypes: List<String>) {
    TWITTER("Twitter/X", listOf("Thread", "Single Post")),
    LINKEDIN("LinkedIn", listOf("Article", "Post")),
    INSTAGRAM("Instagram", listOf("Caption", "Carousel", "Story")),
    MEDIUM("Medium", listOf("Article")),
    WHATSAPP("WhatsApp", listOf("Broadcast", "Status"))
}

//util/ContentLengthOption.kt
package com.yourpackage.socialmediagenerator.util

enum class ContentLengthOption(val displayName: String) {
    SHORT("Short"),
    MEDIUM("Medium"),
    LONG("Long"),
    CUSTOM("Custom")
}
```

**Updated `MainViewModel.kt`:**

```kotlin
// ... other imports
import com.yourpackage.socialmediagenerator.util.ContentLengthOption

// ...

class MainViewModel @Inject constructor(
    // ... other dependencies
) : ViewModel() {

    // ... other functions

    fun updateSelectedContentType(contentType: String) {
        _uiState.update { it.copy(selectedContentType = contentType) }
    }

    fun updateSelectedLengthOption(lengthOption: String) {
        _uiState.update { it.copy(selectedLengthOption = lengthOption) }
    }

    // ...
}
```

Now, your `MainScreen` should have fully functional dropdown menus for both Content Type and Length Option selection, dynamically updating based on the selected platform and correctly passing the selected values to the `MainViewModel`.
