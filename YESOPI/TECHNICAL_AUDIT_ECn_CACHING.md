# Technical Audit — Eventual Connectivity (ECn) y Caching

Repositorio auditado: `/home/runner/work/habitica-android/habitica-android`

Fecha de auditoría: 2026-04-19

---

## A) EVENTUAL CONNECTIVITY (ECn)

## A1. Estrategias ECn buenas encontradas

### 1) **Cache then network** + **Cached on network response** (estado principal de usuario)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/viewmodels/MainUserViewModel.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/viewmodels/MainActivityViewModel.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/UserRepositoryImpl.kt`

**Métodos / clases**
- `MainUserViewModel.user`
- `MainActivityViewModel.retrieveUser(...)`
- `UserRepositoryImpl.retrieveUser(...)`

**Snippet (evidencia)**
- `MainUserViewModel.kt:49`
```kotlin
val user: LiveData<User?> = userRepository.getUser().asLiveData()
```
- `MainActivityViewModel.kt:93-97, 125-127`
```kotlin
fun retrieveUser(forced: Boolean = false) {
    if (hostConfig.hasAuthentication()) {
        viewModelScope.launch(ExceptionHandler.coroutine()) {
            contentRepository.retrieveWorldState()
            userRepository.retrieveUser(true, forced)
            ...
            inventoryRepository.retrieveInAppRewards()
            contentRepository.retrieveContent()
        }
    }
}
```
- `UserRepositoryImpl.kt:100-106`
```kotlin
if (forced || lastSync == null || Date().time - (lastSync?.time ?: 0) > 180000) {
    val user = apiClient.retrieveUser(withTasks) ?: return null
    lastSync = Date()
    withContext(Dispatchers.Main) {
        localRepository.saveUser(user)
    }
```

**Por qué encaja**
- La UI observa datos locales (`getUser().asLiveData()`) y luego se hace refresh remoto, guardando respuesta en local (`saveUser`).

**Feature**
- Pantalla principal / estado del usuario autenticado.

---

### 2) **Cache, falling back to network** (offline-first en Team Plan)

**Archivo**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/UserRepositoryImpl.kt`

**Método**
- `getTeamPlan(teamID: String)`

**Snippet**
- `UserRepositoryImpl.kt:484-489`
```kotlin
override fun getTeamPlan(teamID: String): Flow<Group?> {
    return localRepository.getTeamPlan(teamID)
        .map {
            it ?: retrieveTeamPlan(teamID)
        }
}
```

**Por qué encaja**
- Primero intenta local cache; si no existe (`it == null`), consulta red.

**Feature**
- Detalle de Team Plan.

---

### 3) **Generic fallback** (errores de red informados, sin crash global)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/ApiClientImpl.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/activities/MainActivity.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/res/values/strings.xml`

**Métodos**
- `ApiClientImpl.accept(...)`
- `MainActivity.showConnectionProblem(...)`

**Snippet**
- `ApiClientImpl.kt:337-341`
```kotlin
} else if (throwableClass == SocketTimeoutException::class.java || UnknownHostException::class.java == throwableClass || IOException::class.java == throwableClass) {
    this.showConnectionProblemDialog(
        R.string.network_error_no_network_body,
        isUserInputCall
    )
}
```
- `MainActivity.kt:1029-1035`
```kotlin
binding.content.connectionIssueView.visibility = View.VISIBLE
binding.content.connectionIssueTextview.text = message
errorJob = lifecycleScope.launch(Dispatchers.Main) {
    delay(1.toDuration(DurationUnit.MINUTES))
    binding.content.connectionIssueView.visibility = View.GONE
}
```
- `strings.xml:56-57`
```xml
<string name="network_error_no_network_body">You are not connected to the internet.</string>
<string name="internal_error_api">Server connection lost</string>
```

**Por qué encaja**
- Hay manejo explícito de fallos de conectividad con feedback de UI.

**Feature**
- Manejo global de errores API.

---

### 4) **Always offline** (lectura FAQ desde local)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/FAQRepositoryImpl.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/fragments/support/FAQOverviewFragment.kt`

**Métodos**
- `FAQRepositoryImpl.getArticles()`
- `FAQOverviewFragment.loadArticles()`

**Snippet**
- `FAQRepositoryImpl.kt:20-22`
```kotlin
override fun getArticles(): Flow<List<FAQArticle>> {
    return localRepository.articles
}
```
- `FAQOverviewFragment.kt:199-220`
```kotlin
faqRepository.getArticles().collect {
    ...
    for (article in it) {
        ...
    }
}
```

**Por qué encaja**
- Este flujo consume directo desde almacenamiento local (sin request online en ese path).

**Feature**
- FAQ / soporte.

---

## A2. Estrategias ECn malas / anti-patrones

### Anti-pattern #4 **LOST CONTENT** + #3 **NON-INFORMATIVE MESSAGE** (Party Seeking)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/SocialRepositoryImpl.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/fragments/social/party/PartySeekingFragment.kt`

**Métodos / código**
- `SocialRepositoryImpl.retrievePartySeekingUsers(...)`
- `PartySeekingPagingSource.load(...)`

**Snippet**
- `SocialRepositoryImpl.kt:67-68`
```kotlin
override suspend fun retrievePartySeekingUsers(page: Int): List<Member>? {
    return apiClient.retrievePartySeekingUsers(page)
}
```
- `PartySeekingFragment.kt:395-400`
```kotlin
val response = repository.retrievePartySeekingUsers(page)
LoadResult.Page(
    data = response ?: emptyList(),
    prevKey = if (page == 0) null else page.minus(1),
    nextKey = if ((response?.size ?: 0) < 30) null else page.plus(1)
)
```
- `PartySeekingFragment.kt:338-340, 359-360`
```kotlin
is LoadState.Error -> {
}
```

**Por qué es anti-patrón**
- Cuando falla red/null, el listado puede terminar como “vacío normal” sin mensaje útil.

**Cómo se manifiesta**
- Usuario interpreta que no hay candidatos, cuando realmente hubo fallo de conectividad.

**Pasos de reproducción**
1. Entrar a Party Invite (tab de buscar miembros).
2. Activar modo avión.
3. Pull-to-refresh o abrir de nuevo la pantalla.
4. Ver estado vacío/no informativo.

**Fix propuesto**
- No convertir fallo a `emptyList()` en silencio.
- Forzar `LoadResult.Error` ante falla de conectividad.
- UI explícita en `LoadState.Error` con mensaje/retry.
- Cache local de último resultado para fallback visual.

---

### Anti-pattern #5 **NON-EXISTENT RESULT NOTIFICATION** + #8 **UNCLEAR BEHAVIOR** (enviar chat)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/ApiClientImpl.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/viewmodels/GroupViewModel.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/views/social/ChatBarView.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/fragments/social/ChatFragment.kt`

**Snippet**
- `ApiClientImpl.kt:98-123` (captura excepción y retorna `null`)
```kotlin
private suspend fun <T> process(apiCall: suspend () -> Response<HabitResponse<T>>,...): T? {
    try {
        return processResponse(apiCall())
    } catch (throwable: Throwable) {
        ...
    }
    return null
}
```
- `GroupViewModel.kt:297-307`
```kotlin
val response = socialRepository.postGroupChat(groupID, chatText)
if (response != null) {
    ...
}
onComplete()
```
- `ChatBarView.kt:157-162`
```kotlin
private fun sendButtonPressed() {
    val chatText = message
    if (chatText.isNotEmpty()) {
        binding.chatEditText.text = null
        sendAction?.invoke(chatText)
    }
}
```
- `ChatFragment.kt:314-317`
```kotlin
viewModel.postGroupChat(
    chatText,
    { binding?.recyclerView?.scrollToPosition(0) }
) { binding?.chatBarView?.message = chatText }
```

**Por qué es anti-patrón**
- En caso de error silencioso (`response == null`) se ejecuta `onComplete()` y no se activa una ruta de error explícita para el usuario.

**Cómo se manifiesta**
- Mensaje desaparece del input y puede no enviarse, con feedback ambiguo.

**Pasos de reproducción**
1. Abrir chat de grupo.
2. Modo avión.
3. Escribir y enviar mensaje.
4. Input se limpia; mensaje no aparece/sin confirmación clara.

**Fix propuesto**
- Si `response == null`, llamar `onError()` y no `onComplete()`.
- Introducir estado de mensaje pendiente/error con retry.
- Usar `Result` tipado en repositorio/API en vez de `null` ambiguo.

---

### Anti-pattern #9 **UNEXPECTED RESULT FROM BACKGROUND PROCESS** + #5 (quick reply en notificación)

**Archivo**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/receivers/LocalNotificationActionReceiver.kt`

**Snippet**
- `LocalNotificationActionReceiver.kt:136-141`
```kotlin
socialRepository.postGroupChat(it, message)
context?.let { c ->
    NotificationManagerCompat.from(c).cancel(it.hashCode())
}
```

**Por qué es anti-patrón**
- Se cierra la notificación aunque el envío haya fallado (sin confirmación de resultado).

**Cómo se manifiesta**
- Usuario cree que se envió desde notificación; al abrir chat no está.

**Pasos de reproducción**
1. Recibir notificación de mensaje grupal.
2. Activar modo avión.
3. Responder desde la notificación.
4. Notificación se cierra; mensaje puede no haberse enviado.

**Fix propuesto**
- Cancelar notificación solo cuando `postGroupChat` retorne éxito.
- En error, mantener notificación o mostrar “no enviado / reintentar”.

---

### Anti-pattern #4 **LOST CONTENT** (path de cache de chat no aprovechado de forma efectiva)

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/SocialRepositoryImpl.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/local/SocialLocalRepository.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/fragments/social/ChatFragment.kt`

**Snippet**
- `SocialRepositoryImpl.kt:92-95`
```kotlin
val messages = apiClient.listGroupChat(groupId, limit, before)
messages?.forEach { it.groupId = groupId }
return messages
```
- `SocialLocalRepository.kt:53-56` define `saveChatMessages(...)`
```kotlin
fun saveChatMessages(
    groupId: String?,
    chatMessages: List<ChatMessage>
)
```
- `ChatFragment.kt:116, 134` fuerza refetch remoto
```kotlin
viewModel.retrieveGroupChat { _ -> }
```

**Por qué es anti-patrón**
- Hay infraestructura local para cache de chat, pero el flujo principal de recuperación de chat mostrado depende del fetch remoto y puede degradar offline.

**Cómo se manifiesta**
- En mala conectividad la experiencia de historial de chat puede vaciarse o quedar desactualizada.

**Pasos de reproducción**
1. Abrir chat online y navegar.
2. Cerrar vista, luego modo avión.
3. Reabrir chat y forzar refresh.
4. Historial puede no comportarse como fallback robusto esperado.

**Fix propuesto**
- Guardar recuperación remota en local (`saveChatMessages`) y renderizar local-first + refresh remoto.

---

## A3. Protecciones ECn faltantes

### Faltante 1: chequeo explícito de conectividad antes de flujos críticos (módulo móvil principal)

**Evidencia**
- En `Habitica/src/main/java` no se observa uso de `ConnectivityManager` / `NetworkCapabilities` para pre-check general en navegación de features de red.
- Ejemplo de redirección directa a flujo de red:
  - `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/ui/fragments/social/party/PartyInvitePagerFragment.kt:68-71`

**Anti-pattern probable**
- #6 / #10 Redirection without connectivity check
- #4 Lost content

**Cómo corregir**
- Estado de conectividad centralizado, guard de navegación, fallback local y CTA de reintento.

---

### Faltante 2: cola persistente de acciones offline (outbox)

**Evidencia**
- Cliente API retorna `null` en error (`ApiClientImpl.kt:98-123`) sin mecanismo persistente de reintento diferido para acciones de usuario.

**Anti-pattern probable**
- #5 Non-existent result notification
- #9 Unexpected result from background process

**Cómo corregir**
- Outbox local (acciones pendientes), worker de sincronización y estados de entrega visibles en UI.

---

## A4. Tabla resumen ECn

| Feature | Estrategia ECn | Archivos / snippets | Good o anti-pattern | Reproducción | Fix |
|---|---|---|---|---|---|
| Estado de usuario | Cache then network + cached on network response | `MainUserViewModel.kt:49`, `MainActivityViewModel.kt:93-127`, `UserRepositoryImpl.kt:100-106` | Good | Abrir app con cache local y luego refresh | Mantener patrón, añadir indicador de “datos no actualizados” |
| Team Plan | Cache fallback to network | `UserRepositoryImpl.kt:484-489` | Good | Abrir Team Plan sin red con dato local | Mantener, agregar timestamp/última sync |
| Manejo global errores API | Generic fallback | `ApiClientImpl.kt:337-341`, `MainActivity.kt:1029-1035`, `strings.xml:56-57` | Good | Apagar red y disparar API | Mantener, mensajes por contexto |
| Party Seeking | Network-only con fallback a vacío | `SocialRepositoryImpl.kt:67-68`, `PartySeekingFragment.kt:395-400,338-340` | Anti #4/#3 | Abrir PartySeeking offline | `LoadResult.Error` + UI error/retry + cache local |
| Envío chat | Resultado ambiguo ante error | `ApiClientImpl.kt:98-123`, `GroupViewModel.kt:297-307`, `ChatBarView.kt:157-162` | Anti #5/#8 | Enviar chat offline | Separar success/error, estado pending/failed |
| Quick reply notificación | Proceso en background sin confirmación real | `LocalNotificationActionReceiver.kt:136-141` | Anti #9/#5 | Responder notificación offline | Cancelar notif solo en éxito + feedback error |
| Historial chat | Fallback local incompleto en flujo principal | `SocialRepositoryImpl.kt:92-95`, `SocialLocalRepository.kt:53-56`, `ChatFragment.kt:116,134` | Anti #4 | Reabrir chat con mala red | Persistir chat recuperado y render local-first |

---

## B) CACHING STRATEGIES (>= 3 estructuras)

## B1. Estructura de caché #1 — **HTTP Disk Cache (OkHttp Cache)**

**Archivo**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/implementation/ApiClientImpl.kt`

**Método**
- `buildRetrofit()`

**Snippet**
- `ApiClientImpl.kt:158-165`
```kotlin
val cacheSize: Long = 10 * 1024 * 1024 // 10 MB
val cache = Cache(File(context.cacheDir, "http_cache"), cacheSize)

val client =
    OkHttpClient.Builder()
        .cache(cache)
```

**Tipo**
- Temporal cache (disco, evictable).

**Relevancia**
- Reduce latencia, consumo de red y carga de backend cuando el protocolo HTTP permite cache.

---

## B2. Estructura de caché #2 — **Realm local persistence como cache permanente de dominio**

**Archivos**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/local/implementation/RealmContentLocalRepository.kt`
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/data/local/implementation/RealmUserLocalRepository.kt`

**Métodos**
- `saveContent(...)`, `saveWorldState(...)`, `saveUser(...)`

**Snippet**
- `RealmContentLocalRepository.kt:16-38`
```kotlin
executeTransaction { realm1 ->
    ...
    realm1.insertOrUpdate(contentResult.faq)
    realm1.insertOrUpdate(contentResult.mystery)
}
```
- `RealmUserLocalRepository.kt:85-106`
```kotlin
override fun saveUser(user: User, overrideExisting: Boolean) {
    ...
    executeTransaction { realm1 -> realm1.insertOrUpdate(user) }
}
```

**Tipo**
- Permanente cache (persistente hasta limpieza de datos).

**Relevancia**
- Base para offline/degradado y lectura rápida de entidades principales.

---

## B3. Estructura de caché #3 — **In-memory cache (bounded map) para markdown parseado**

**Archivo**
- `/home/runner/work/habitica-android/habitica-android/common/src/main/java/com/habitrpg/common/habitica/helpers/MarkdownParser.kt`

**Método**
- `parseMarkdown(...)`

**Snippet**
- `MarkdownParser.kt:33, 103-105, 115-117`
```kotlin
private val cache = sortedMapOf<Int, Spanned>()
...
if (cache.containsKey(hashCode)) {
    return cache[hashCode] ?: SpannableString(preProcessedInput)
}
...
cache[hashCode] = result
if (cache.size > 100) {
    cache.remove(cache.firstKey())
}
```

**Tipo**
- Temporal cache en memoria.

**Relevancia**
- Evita reparseo costoso en textos repetidos y mejora fluidez UI.

---

## B4. Estructura de caché #4 — **File cache de audios de notificación**

**Archivo**
- `/home/runner/work/habitica-android/habitica-android/Habitica/src/main/java/com/habitrpg/android/habitica/helpers/SoundFileLoader.kt`

**Método**
- `download(files: List<SoundFile>)`

**Snippet**
- `SoundFileLoader.kt:34-39, 53-55`
```kotlin
if (file.exists() && file.length() > 5000) {
    file.setReadable(true, false)
    audioFile.file = file
    return@withContext audioFile
}
...
val sink = file.sink().buffer()
sink.writeAll(response.body!!.source())
```

**Tipo**
- Temporal cache en sistema de archivos.

**Relevancia**
- Reutiliza contenido descargado y reduce llamadas repetidas a red.

---

## B5. Hallazgo explícito de ausencia: **LruCache**

**Resultado de búsqueda**
- No se encontraron usos de `LruCache` o `android.util.LruCache` en el repositorio.

**Relevancia**
- Para el marco teórico de clase, el proyecto usa otras estrategias (OkHttp cache, Realm, map in-memory propio), pero no `LruCache` de Android.

---

## Conclusión breve

El repo sí tiene base ECn sólida en varios flujos (local-first en partes críticas, persistencia Realm, manejo global de errores de red), pero persisten debilidades en flujos sociales (Party Seeking / envío de chat / quick reply) donde aún hay comportamiento ambiguo o pérdida de contenido bajo fallos de conectividad.
