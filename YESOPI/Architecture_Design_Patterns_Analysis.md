# Análisis de Arquitectura y Patrones de Diseño — Habitica Android

## Información General

| Campo | Valor |
|-------|-------|
| **Proyecto** | Habitica Android |
| **Repositorio** | `Yesopi/habitica-android` |
| **Lenguaje principal** | Kotlin |
| **Arquitectura general** | MVVM + Clean Architecture con Repository Pattern |
| **Framework de DI** | Hilt (Dagger) |
| **Base de datos local** | Realm |
| **Programación reactiva** | Kotlin Coroutines + Flow + LiveData |

---

## Estructura del Proyecto

```
habitica-android/
├── Habitica/                          # Módulo principal de la app Android
│   └── src/main/java/com/habitrpg/android/habitica/
│       ├── api/                       # Definiciones de servicios API
│       ├── data/                      # Interfaces de repositorios
│       │   ├── implementation/        # Implementaciones de repositorios
│       │   └── local/                 # Fuentes de datos locales (Realm)
│       ├── modules/                   # Módulos de inyección de dependencias (Hilt)
│       ├── ui/                        # Capa de UI
│       │   ├── viewmodels/            # ViewModels (MVVM)
│       │   ├── adapter/               # Adaptadores de RecyclerView
│       │   ├── activities/            # Activities
│       │   ├── fragments/             # Fragments
│       │   └── views/                 # Vistas personalizadas
│       ├── interactors/               # Casos de uso / Interactors
│       ├── helpers/                   # Utilidades y managers
│       ├── models/                    # Modelos de datos
│       └── extensions/                # Funciones de extensión de Kotlin
├── common/                            # Componentes compartidos
├── shared/                            # Código compartido entre módulos
├── wearos/                            # Módulo Wear OS
└── build-logic/                       # Lógica de configuración de build
```

---

## Patrón 1: Singleton (Creacional)

### Descripción

El **patrón Singleton** garantiza que una clase tenga una única instancia y proporciona un punto de acceso global a ella. En Habitica Android, este patrón se implementa mediante la anotación `@Singleton` de Hilt/Dagger, que asegura que el contenedor de inyección de dependencias cree una sola instancia del componente durante todo el ciclo de vida de la aplicación.

### Ejemplo: `SoundManager`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/helpers/SoundManager.kt`

```kotlin
package com.habitrpg.android.habitica.helpers

import com.habitrpg.common.habitica.helpers.launchCatching
import kotlinx.coroutines.MainScope
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class SoundManager
@Inject
constructor(var soundFileLoader: SoundFileLoader) {
    var soundTheme: String = SOUND_THEME_OFF

    private val loadedSoundFiles: MutableMap<String, SoundFile> = HashMap()

    fun preloadAllFiles() {
        loadedSoundFiles.clear()
        if (soundTheme == SOUND_THEME_OFF) {
            return
        }

        val soundFiles = ArrayList<SoundFile>()
        soundFiles.add(SoundFile(soundTheme, SOUND_ACHIEVEMENT_UNLOCKED))
        soundFiles.add(SoundFile(soundTheme, SOUND_CHAT))
        soundFiles.add(SoundFile(soundTheme, SOUND_DAILY))
        soundFiles.add(SoundFile(soundTheme, SOUND_DEATH))
        soundFiles.add(SoundFile(soundTheme, SOUND_ITEM_DROP))
        soundFiles.add(SoundFile(soundTheme, SOUND_LEVEL_UP))
        soundFiles.add(SoundFile(soundTheme, SOUND_MINUS_HABIT))
        soundFiles.add(SoundFile(soundTheme, SOUND_PLUS_HABIT))
        soundFiles.add(SoundFile(soundTheme, SOUND_REWARD))
        soundFiles.add(SoundFile(soundTheme, SOUND_TODO))
        MainScope().launchCatching {
            soundFileLoader.download(soundFiles)
        }
    }

    fun loadAndPlayAudio(type: String) {
        if (soundTheme == SOUND_THEME_OFF) {
            return
        }

        if (loadedSoundFiles.containsKey(type)) {
            loadedSoundFiles[type]?.play()
        } else {
            val soundFiles = ArrayList<SoundFile>()
            soundFiles.add(SoundFile(soundTheme, type))
            MainScope().launchCatching {
                val newFiles = soundFileLoader.download(soundFiles)
                // ...
            }
        }
    }

    companion object {
        const val SOUND_ACHIEVEMENT_UNLOCKED = "Achievement_Unlocked"
        const val SOUND_CHAT = "Chat"
        const val SOUND_DAILY = "Daily"
        const val SOUND_DEATH = "Death"
        const val SOUND_ITEM_DROP = "Item_Drop"
        const val SOUND_LEVEL_UP = "Level_Up"
        const val SOUND_MINUS_HABIT = "Minus_Habit"
        const val SOUND_PLUS_HABIT = "Plus_Habit"
        const val SOUND_REWARD = "Reward"
        const val SOUND_TODO = "Todo"
        const val SOUND_THEME_OFF = "off"
    }
}
```

### Ejemplo adicional: `AppModule` proporcionando Singletons

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/modules/AppModule.kt`

```kotlin
@InstallIn(SingletonComponent::class)
@Module
class AppModule {
    @Provides
    @Singleton
    fun provideSharedPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return PreferenceManager.getDefaultSharedPreferences(context)
    }

    @Provides
    fun provideKeyStore(): KeyStore? {
        try {
            val keyStore = KeyStore.getInstance("AndroidKeyStore")
            keyStore.load(null)
            return keyStore
        } catch (e: KeyStoreException) {
            HLogger.logException("KeyHelper", "Error initializing", e)
        } catch (e: CertificateException) {
            HLogger.logException("KeyHelper", "Error initializing", e)
        } catch (e: NoSuchAlgorithmException) {
            HLogger.logException("KeyHelper", "Error initializing", e)
        } catch (e: IOException) {
            HLogger.logException("KeyHelper", "Error initializing", e)
        }
        return null
    }
}
```

### ¿Por qué es Singleton?

- La anotación `@Singleton` garantiza una **única instancia** de `SoundManager` en toda la app.
- La inyección con `@Inject constructor` permite que Hilt gestione su ciclo de vida.
- `SharedPreferences` también se provee como `@Singleton` para garantizar coherencia global.

---

## Patrón 2: Repository (Estructural / Arquitectónico)

### Descripción

El **patrón Repository** actúa como una capa de abstracción entre la capa de datos (API remota, base de datos local) y la lógica de negocio. Proporciona una interfaz limpia para acceder a los datos, ocultando los detalles de implementación de las fuentes de datos.

En Habitica Android, cada dominio tiene su propia interfaz de repositorio y su implementación que coordina datos locales (Realm) y remotos (API REST).

### Interfaz base: `BaseRepository`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/BaseRepository.kt`

```kotlin
package com.habitrpg.android.habitica.data

import com.habitrpg.android.habitica.models.BaseObject

interface BaseRepository {
    val isClosed: Boolean

    fun close()

    fun <T : BaseObject> getUnmanagedCopy(obj: T): T

    fun <T : BaseObject> getUnmanagedCopy(list: List<T>): List<T>
}
```

### Implementación base: `BaseRepositoryImpl`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/BaseRepositoryImpl.kt`

```kotlin
package com.habitrpg.android.habitica.data.implementation

import com.habitrpg.android.habitica.data.ApiClient
import com.habitrpg.android.habitica.data.BaseRepository
import com.habitrpg.android.habitica.data.local.BaseLocalRepository
import com.habitrpg.android.habitica.models.BaseObject
import com.habitrpg.android.habitica.modules.AuthenticationHandler

abstract class BaseRepositoryImpl<T : BaseLocalRepository>(
    protected val localRepository: T,
    protected val apiClient: ApiClient,
    protected val authenticationHandler: AuthenticationHandler
) : BaseRepository {
    val currentUserID: String
        get() = authenticationHandler.currentUserID ?: ""

    override fun close() {
        this.localRepository.close()
    }

    override fun <T : BaseObject> getUnmanagedCopy(list: List<T>): List<T> {
        return localRepository.getUnmanagedCopy(list)
    }

    override val isClosed: Boolean
        get() = localRepository.isClosed

    override fun <T : BaseObject> getUnmanagedCopy(obj: T): T {
        return localRepository.getUnmanagedCopy(obj)
    }
}
```

### Ejemplo concreto: `UserRepository`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/UserRepository.kt`

```kotlin
package com.habitrpg.android.habitica.data

import com.habitrpg.android.habitica.models.user.User
import kotlinx.coroutines.flow.Flow

interface UserRepository : BaseRepository {
    fun getUser(): Flow<User?>

    fun getUser(userID: String): Flow<User?>

    suspend fun updateUser(updateData: Map<String, Any?>): User?

    suspend fun updateUser(key: String, value: Any?): User?

    suspend fun retrieveUser(
        withTasks: Boolean = false,
        forced: Boolean = false,
        overrideExisting: Boolean = false
    ): User?

    suspend fun revive(): Equipment?

    suspend fun resetTutorial(): User?

    suspend fun sleep(user: User): User?

    fun getSkills(user: User): Flow<List<Skill>>

    suspend fun useSkill(key: String, target: String?, taskId: String): SkillResponse?

    suspend fun disableClasses(): User?

    suspend fun changeClass(selectedClass: String? = null): User?

    suspend fun runCron(tasks: MutableList<Task>)

    suspend fun runCron()

    // ... más métodos
}
```

### Implementación: `UserRepositoryImpl`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/UserRepositoryImpl.kt`

```kotlin
class UserRepositoryImpl(
    localRepository: UserLocalRepository,
    apiClient: ApiClient,
    authenticationHandler: AuthenticationHandler,
    private val taskRepository: TaskRepository,
    private val appConfigManager: AppConfigManager
) : BaseRepositoryImpl<UserLocalRepository>(
    localRepository, apiClient, authenticationHandler
), UserRepository {

    override fun getUser(): Flow<User?> =
        authenticationHandler.userIDFlow.flatMapLatest { getUser(it) }

    override fun getUser(userID: String): Flow<User?> =
        localRepository.getUser(userID)

    override suspend fun syncUserStats(): User? {
        val user = apiClient.syncUserStats()
        if (user != null && (user.stats?.toNextLevel ?: 0) > 1 &&
            (user.stats?.maxMP ?: 0) > 1
        ) {
            localRepository.saveUser(user)
        } else {
            retrieveUser(false, true)
        }
        return user
    }

    private suspend fun updateUser(
        userID: String,
        updateData: Map<String, Any?>
    ): User? {
        val networkUser = apiClient.updateUser(updateData) ?: return null
        val oldUser = localRepository.getUser(userID).firstOrNull()
        return mergeUser(oldUser, networkUser)
    }
}
```

### Provisión en el módulo de DI: `RepositoryModule`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/modules/RepositoryModule.kt`

```kotlin
@InstallIn(SingletonComponent::class)
@Module
open class RepositoryModule {
    @Provides
    open fun providesRealm(): Realm {
        return Realm.getDefaultInstance()
    }

    @Provides
    fun providesContentLocalRepository(realm: Realm): ContentLocalRepository {
        return RealmContentLocalRepository(realm)
    }

    @Provides
    fun providesContentRepository(
        contentLocalRepository: ContentLocalRepository,
        apiClient: ApiClient,
        @ApplicationContext context: Context,
        authenticationHandler: AuthenticationHandler
    ): ContentRepository {
        return ContentRepositoryImpl(
            contentLocalRepository,
            apiClient,
            context,
            authenticationHandler
        )
    }
}
```

### Lista completa de repositorios en la app

| Interfaz | Implementación |
|----------|---------------|
| `UserRepository` | `UserRepositoryImpl` |
| `TaskRepository` | `TaskRepositoryImpl` |
| `ContentRepository` | `ContentRepositoryImpl` |
| `SocialRepository` | `SocialRepositoryImpl` |
| `ChallengeRepository` | `ChallengeRepositoryImpl` |
| `CustomizationRepository` | `CustomizationRepositoryImpl` |
| `InventoryRepository` | `InventoryRepositoryImpl` |
| `TagRepository` | `TagRepositoryImpl` |
| `FAQRepository` | `FAQRepositoryImpl` |
| `TutorialRepository` | `TutorialRepositoryImpl` |

### ¿Por qué es Repository?

- Separa la lógica de acceso a datos de la lógica de negocio.
- Cada repositorio tiene una **interfaz** y una **implementación** que coordina datos locales (Realm) y remotos (API).
- `BaseRepositoryImpl` contiene lógica común (acceso al `apiClient`, `localRepository`, `authenticationHandler`).

---

## Patrón 3: MVVM — Model-View-ViewModel (Arquitectónico)

### Descripción

El patrón **MVVM (Model-View-ViewModel)** separa la interfaz de usuario (View) de la lógica de presentación (ViewModel) y los datos (Model). El ViewModel expone datos reactivos que la View observa, eliminando la dependencia directa entre ambos.

En Habitica Android, los `ViewModel` extienden de `BaseViewModel` y usan `LiveData` y `Flow` para comunicarse reactivamente con la UI (Activities/Fragments).

### Base ViewModel

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/ui/viewmodels/BaseViewModel.kt`

```kotlin
package com.habitrpg.android.habitica.ui.viewmodels

import androidx.lifecycle.LiveData
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.habitrpg.android.habitica.data.UserRepository
import com.habitrpg.android.habitica.models.user.User
import com.habitrpg.common.habitica.helpers.ExceptionHandler
import kotlinx.coroutines.launch

abstract class BaseViewModel(
    val userRepository: UserRepository,
    val userViewModel: MainUserViewModel
) : ViewModel() {
    val user: LiveData<User?> by lazy {
        userViewModel.user
    }

    override fun onCleared() {
        userRepository.close()
        super.onCleared()
    }

    fun updateUser(
        path: String,
        value: Any
    ) {
        viewModelScope.launch(ExceptionHandler.coroutine()) {
            userRepository.updateUser(path, value)
        }
    }
}
```

### Flujo del patrón MVVM en la app

```
┌─────────────┐       observa        ┌──────────────┐      solicita datos    ┌──────────────┐
│    View      │ ◄──────────────────  │  ViewModel   │  ──────────────────►  │  Repository  │
│ (Fragment/   │    LiveData / Flow   │ (BaseViewModel│                      │ (UserRepo,   │
│  Activity)   │                      │  TasksVM...)  │  ◄──────────────────  │  TaskRepo...)│
└─────────────┘                       └──────────────┘      Flow<Data>       └──────────────┘
                                                                                    │
                                                                            ┌───────┴───────┐
                                                                            │               │
                                                                      ┌─────┴─────┐   ┌────┴────┐
                                                                      │ Local DB  │   │ API     │
                                                                      │ (Realm)   │   │ (REST)  │
                                                                      └───────────┘   └─────────┘
```

### Lista de ViewModels en la app

| ViewModel | Responsabilidad |
|-----------|----------------|
| `BaseViewModel` | Clase base abstracta con acceso al usuario |
| `MainUserViewModel` | Datos principales del usuario |
| `TasksViewModel` | Gestión de tareas (habits, dailies, todos) |
| `MainActivityViewModel` | Estado de la actividad principal |
| `AuthenticationViewModel` | Flujo de autenticación |
| `PartyViewModel` | Datos del grupo/party |
| `GroupViewModel` | Gestión de grupos |
| `InboxViewModel` | Bandeja de mensajes |
| `StableViewModel` | Mascotas y monturas |
| `SetupViewModel` | Flujo de configuración inicial |
| `NotificationsViewModel` | Notificaciones |
| `EquipmentOverviewViewModel` | Gestión de equipamiento |

### ¿Por qué es MVVM?

- Los `ViewModel` exponen `LiveData<User?>` que las Views observan reactivamente.
- `viewModelScope.launch` ejecuta operaciones asíncronas sin bloquear la UI.
- Los ViewModels **no** tienen referencias directas a Views — la comunicación es mediante observación de datos.

---

## Patrón 4: Inyección de Dependencias — Dependency Injection (Estructural)

### Descripción

La **Inyección de Dependencias (DI)** es un patrón donde los objetos reciben sus dependencias en lugar de crearlas internamente. Habitica Android usa **Hilt** (basado en Dagger) para gestionar toda la inyección de dependencias, lo que promueve el desacoplamiento y la testabilidad.

### Módulo de DI: `AppModule`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/modules/AppModule.kt`

```kotlin
@InstallIn(SingletonComponent::class)
@Module
class AppModule {
    @Provides
    @Singleton
    fun provideSharedPreferences(
        @ApplicationContext context: Context
    ): SharedPreferences {
        return PreferenceManager.getDefaultSharedPreferences(context)
    }

    @Provides
    fun provideKeyStore(): KeyStore? {
        try {
            val keyStore = KeyStore.getInstance("AndroidKeyStore")
            keyStore.load(null)
            return keyStore
        } catch (e: KeyStoreException) {
            HLogger.logException("KeyHelper", "Error initializing", e)
        }
        return null
    }

    @Provides
    @Singleton
    fun provideKeyHelper(
        @ApplicationContext context: Context,
        keyStore: KeyStore?
    ): KeyHelper {
        return getInstance(context, keyStore)
    }
}
```

### Inyección en constructores

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/interactors/BuyRewardUseCase.kt`

```kotlin
class BuyRewardUseCase
@Inject
constructor(
    private val taskRepository: TaskRepository,
    private val soundManager: SoundManager
) : UseCase<BuyRewardUseCase.RequestValues, TaskScoringResult?>() {
    override suspend fun run(requestValues: RequestValues): TaskScoringResult? {
        val response =
            taskRepository.taskChecked(
                requestValues.user,
                requestValues.task,
                false,
                false,
                requestValues.notifyFunc
            )
        soundManager.loadAndPlayAudio(SoundManager.SOUND_REWARD)
        return response
    }

    class RequestValues(
        internal val user: User?,
        val task: Task,
        val notifyFunc: (TaskScoringResult) -> Unit
    ) : UseCase.RequestValues
}
```

### Módulo de Repositorios: `RepositoryModule`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/modules/RepositoryModule.kt`

```kotlin
@InstallIn(SingletonComponent::class)
@Module
open class RepositoryModule {
    @Provides
    open fun providesRealm(): Realm {
        return Realm.getDefaultInstance()
    }

    @Provides
    fun providesContentLocalRepository(realm: Realm): ContentLocalRepository {
        return RealmContentLocalRepository(realm)
    }

    @Provides
    fun providesContentRepository(
        contentLocalRepository: ContentLocalRepository,
        apiClient: ApiClient,
        @ApplicationContext context: Context,
        authenticationHandler: AuthenticationHandler
    ): ContentRepository {
        return ContentRepositoryImpl(
            contentLocalRepository,
            apiClient,
            context,
            authenticationHandler
        )
    }
}
```

### Módulos de DI en la app

| Módulo | Responsabilidad |
|--------|----------------|
| `AppModule` | Dependencias a nivel de aplicación (SharedPreferences, KeyStore) |
| `ApiModule` | Dependencias de red (ApiClient, HostConfig) |
| `RepositoryModule` | Bindings de repositorios (interfaces → implementaciones) |
| `UserRepositoryModule` | Repositorios específicos del usuario |
| `DeveloperModule` | Dependencias de desarrollo/debug |
| `UserModule` | Servicios relacionados con el usuario |

### Anotaciones clave usadas

| Anotación | Propósito |
|-----------|-----------|
| `@Inject` | Marca constructores o campos para inyección |
| `@Module` | Define una clase que provee dependencias |
| `@Provides` | Marca métodos que crean instancias de dependencias |
| `@InstallIn` | Especifica el componente de Hilt donde se instala el módulo |
| `@Singleton` | Garantiza una única instancia en el scope del componente |
| `@ApplicationContext` | Inyecta el contexto de la aplicación |

### ¿Por qué es Dependency Injection?

- Las clases **no crean** sus dependencias directamente — las reciben a través del constructor.
- Los `@Module` definen **cómo** se crean las dependencias.
- Hilt gestiona automáticamente el **ciclo de vida** de las instancias.
- Permite **sustituir** implementaciones fácilmente (por ejemplo, para testing).

---

## Patrón 5: Observer / Patrón Observador (Comportamiento)

### Descripción

El **patrón Observer** define una dependencia de uno a muchos entre objetos, de modo que cuando un objeto cambia de estado, todos sus dependientes son notificados automáticamente. En Habitica Android se implementa mediante:

- **`LiveData`**: Observable lifecycle-aware para datos en la UI
- **`Flow`**: Streams reactivos de Kotlin Coroutines para datos en repositorios

### Ejemplo con LiveData en BaseViewModel

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/ui/viewmodels/BaseViewModel.kt`

```kotlin
abstract class BaseViewModel(
    val userRepository: UserRepository,
    val userViewModel: MainUserViewModel
) : ViewModel() {
    // LiveData que las Views observan reactivamente
    val user: LiveData<User?> by lazy {
        userViewModel.user
    }
}
```

### Ejemplo con Flow en UserRepositoryImpl

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/UserRepositoryImpl.kt`

```kotlin
class UserRepositoryImpl(
    localRepository: UserLocalRepository,
    apiClient: ApiClient,
    authenticationHandler: AuthenticationHandler,
    private val taskRepository: TaskRepository,
    private val appConfigManager: AppConfigManager
) : BaseRepositoryImpl<UserLocalRepository>(
    localRepository, apiClient, authenticationHandler
), UserRepository {

    // Flow reactivo que emite automáticamente cuando el usuario cambia
    override fun getUser(): Flow<User?> =
        authenticationHandler.userIDFlow.flatMapLatest { getUser(it) }

    override fun getUser(userID: String): Flow<User?> =
        localRepository.getUser(userID)
}
```

### Ejemplo de interfaz UserRepository con Flow

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/data/UserRepository.kt`

```kotlin
interface UserRepository : BaseRepository {
    fun getUser(): Flow<User?>
    fun getUser(userID: String): Flow<User?>
    fun getSkills(user: User): Flow<List<Skill>>
    fun getAchievements(): Flow<List<Achievement>>
    fun getQuestAchievements(): Flow<List<QuestAchievement>>
    fun getUserQuestStatus(): Flow<UserQuestStatus>
    fun getTeamPlans(): Flow<List<TeamPlan>>
    fun getTeamPlan(teamID: String): Flow<Group?>
    // ... métodos suspend para operaciones únicas
}
```

### ¿Por qué es Observer?

- `LiveData` notifica automáticamente a los observadores (Views) cuando los datos cambian.
- `Flow` emite datos reactivamente — cuando la base de datos local cambia, los observadores reciben la actualización.
- `flatMapLatest` permite cambiar de stream cuando el userID cambia, manteniendo la reactividad.

---

## Patrón 6: Use Case / Interactor (Clean Architecture)

### Descripción

El patrón **Use Case** (también llamado **Interactor**) encapsula una operación de negocio específica en una clase independiente. Cada caso de uso tiene una única responsabilidad y recibe sus datos a través de un objeto `RequestValues`.

### Clase base abstracta: `UseCase`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/interactors/UseCase.kt`

```kotlin
package com.habitrpg.android.habitica.interactors

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

abstract class UseCase<Q : UseCase.RequestValues?, T> {
    protected abstract suspend fun run(requestValues: Q): T

    suspend fun callInteractor(requestValues: Q): T {
        return withContext(Dispatchers.Main) {
            run(requestValues)
        }
    }

    interface RequestValues
}
```

### Implementación concreta: `BuyRewardUseCase`

**Archivo:** `Habitica/src/main/java/com/habitrpg/android/habitica/interactors/BuyRewardUseCase.kt`

```kotlin
package com.habitrpg.android.habitica.interactors

import com.habitrpg.android.habitica.data.TaskRepository
import com.habitrpg.android.habitica.helpers.SoundManager
import com.habitrpg.android.habitica.models.tasks.Task
import com.habitrpg.android.habitica.models.user.User
import com.habitrpg.shared.habitica.models.responses.TaskScoringResult
import javax.inject.Inject

class BuyRewardUseCase
@Inject
constructor(
    private val taskRepository: TaskRepository,
    private val soundManager: SoundManager
) : UseCase<BuyRewardUseCase.RequestValues, TaskScoringResult?>() {
    override suspend fun run(requestValues: RequestValues): TaskScoringResult? {
        val response =
            taskRepository.taskChecked(
                requestValues.user,
                requestValues.task,
                false,
                false,
                requestValues.notifyFunc
            )
        soundManager.loadAndPlayAudio(SoundManager.SOUND_REWARD)
        return response
    }

    class RequestValues(
        internal val user: User?,
        val task: Task,
        val notifyFunc: (TaskScoringResult) -> Unit
    ) : UseCase.RequestValues
}
```

### Lista de Use Cases / Interactors en la app

| Use Case | Responsabilidad |
|----------|----------------|
| `BuyRewardUseCase` | Compra de recompensas |
| `CheckClassSelectionUseCase` | Selección de clase del personaje |
| `DisplayItemDropUseCase` | Mostrar notificación de item obtenido |
| `FeedPetUseCase` | Alimentar mascota |
| `HatchPetUseCase` | Eclosión de mascota |
| `LevelUpUseCase` | Lógica de subir de nivel |
| `NotifyUserUseCase` | Notificaciones al usuario |
| `ShareAvatarUseCase` | Compartir avatar |
| `ShareMountUseCase` | Compartir montura |
| `SharePetUseCase` | Compartir mascota |
| `ScoreTaskLocallyInteractor` | Puntuación local de tareas |
| `ShowNotificationInteractor` | Mostrar notificaciones en pantalla |

### ¿Por qué es Use Case / Interactor?

- Cada clase encapsula **una sola operación** de negocio (Principio de Responsabilidad Única).
- Usa `RequestValues` como **objeto de entrada** estandarizado.
- `callInteractor()` ejecuta la lógica en el hilo correcto usando coroutines.
- Se inyectan dependencias mediante `@Inject constructor` para desacopling y testabilidad.

---

## Diagrama de Arquitectura General

```
┌─────────────────────────────────────────────────────────────┐
│                         UI LAYER                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Activities │  │Fragments │  │ Adapters │  │Composables│   │
│  └─────┬────┘  └─────┬────┘  └──────────┘  └──────────┘   │
│        │              │                                      │
│        ▼              ▼                                      │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              ViewModels (LiveData / Flow)             │   │
│  │  BaseViewModel, TasksViewModel, PartyViewModel...    │   │
│  └──────────────────────┬───────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────┘
                          │
┌─────────────────────────┼───────────────────────────────────┐
│                   DOMAIN LAYER                               │
│  ┌──────────────────────┴───────────────────────────────┐   │
│  │         Use Cases / Interactors                       │   │
│  │  BuyRewardUseCase, LevelUpUseCase, FeedPetUseCase... │   │
│  └──────────────────────┬───────────────────────────────┘   │
└─────────────────────────┼───────────────────────────────────┘
                          │
┌─────────────────────────┼───────────────────────────────────┐
│                    DATA LAYER                                │
│  ┌──────────────────────┴───────────────────────────────┐   │
│  │       Repository Interfaces (Contratos)               │   │
│  │  UserRepository, TaskRepository, SocialRepository...  │   │
│  └──────────────────────┬───────────────────────────────┘   │
│                          │                                    │
│         ┌────────────────┼────────────────┐                  │
│         ▼                                 ▼                  │
│  ┌─────────────────┐             ┌─────────────────┐        │
│  │  Local Source    │             │  Remote Source   │        │
│  │  (Realm DB)     │             │  (REST API)      │        │
│  │  RealmUserLocal  │             │  ApiClient       │        │
│  │  Repository      │             │                  │        │
│  └─────────────────┘             └─────────────────┘        │
└─────────────────────────────────────────────────────────────┘
                          │
┌─────────────────────────┼───────────────────────────────────┐
│              DEPENDENCY INJECTION (Hilt)                      │
│  AppModule, ApiModule, RepositoryModule, UserModule...       │
│  Gestiona la creación y el ciclo de vida de todas las        │
│  dependencias mediante @Module, @Provides, @Singleton        │
└─────────────────────────────────────────────────────────────┘
```

---

## Resumen de Patrones Identificados

| # | Patrón | Tipo | Ejemplo Principal | Archivos Clave |
|---|--------|------|-------------------|----------------|
| 1 | **Singleton** | Creacional | `SoundManager`, `SharedPreferences` | `SoundManager.kt`, `AppModule.kt` |
| 2 | **Repository** | Arquitectónico | `UserRepository` → `UserRepositoryImpl` | `BaseRepository.kt`, `UserRepository.kt`, `UserRepositoryImpl.kt` |
| 3 | **MVVM** | Arquitectónico | `BaseViewModel` con `LiveData` | `BaseViewModel.kt`, ViewModels en `/ui/viewmodels/` |
| 4 | **Dependency Injection** | Estructural | Hilt modules con `@Provides` | `AppModule.kt`, `RepositoryModule.kt` |
| 5 | **Observer** | Comportamiento | `LiveData`, `Flow` | `BaseViewModel.kt`, `UserRepositoryImpl.kt` |
| 6 | **Use Case / Interactor** | Clean Architecture | `BuyRewardUseCase` | `UseCase.kt`, `BuyRewardUseCase.kt` |

---

## Conclusiones

La aplicación Habitica Android demuestra una **arquitectura moderna y bien estructurada** que sigue los principios de **Clean Architecture** y **MVVM**:

1. **Separación de responsabilidades**: Cada capa (UI, Domain, Data) tiene responsabilidades claramente definidas.
2. **Desacoplamiento**: Gracias a la inyección de dependencias con Hilt y el uso de interfaces, las implementaciones pueden ser sustituidas fácilmente.
3. **Reactividad**: El uso combinado de `LiveData` y `Flow` permite una comunicación reactiva eficiente entre capas.
4. **Testabilidad**: La inyección de dependencias y las interfaces de repositorio facilitan la creación de mocks para pruebas unitarias.
5. **Escalabilidad**: El patrón Repository con fuentes de datos locales y remotas permite agregar nuevas funcionalidades sin modificar la arquitectura existente.
6. **Principio de Responsabilidad Única**: Los Use Cases encapsulan operaciones de negocio individuales, manteniendo el código limpio y mantenible.
