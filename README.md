# 2do Parcial - Análisis de Errores: SimpsonsApp

## Repositorio original
https://github.com/ExBattou/SimpsonsApp

---

## Errores encontrados

### Error 1 — Bloque `init` fuera del cuerpo de la clase

**Archivo:** `app/src/main/java/com/example/simpsonsapp/domain/model/Episode.kt`
**Líneas:** 13–15

**Problema:**
El bloque `init { return Episode; }` se encuentra fuera del cuerpo de `data class Episode`. En Kotlin, los bloques `init` solo pueden existir **dentro** de una clase. Además, los bloques `init` no permiten la instrucción `return` ya que no retornan ningún valor. Esto es un **error de compilación** que impide que la app compile.

**Concepto de la materia:** Clases en Kotlin, ciclo de vida de inicialización de objetos.

**Solución:** Eliminar el bloque `init` por completo (líneas 13 a 15). Si se requiere validación, debe ubicarse dentro del cuerpo de la clase:

```kotlin
data class Episode(
    val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
// Eliminar las líneas del init {} que están debajo de la clase
```

---

### Error 2 — Nombre de método en snake_case e incompatibilidad interfaz/implementación

**Archivos:**
- `app/src/main/java/com/example/simpsonsapp/domain/repository/EpisodeRepository.kt` — Línea 8
- `app/src/main/java/com/example/simpsonsapp/data/repository/EpisodeRepositoryImpl.kt` — Línea 20
- `app/src/main/java/com/example/simpsonsapp/domain/usecase/GetEpisodesUseCase.kt` — Línea 14

**Problema:**
La interfaz `EpisodeRepository` declara `fun get_episodes()` usando snake_case, pero la implementación `EpisodeRepositoryImpl` define `override fun getEpisodes()` con camelCase. En Kotlin, son dos métodos distintos. Por lo tanto, `EpisodeRepositoryImpl` **no implementa** el método requerido por la interfaz, lo que produce un **error de compilación**. Además, el uso de snake_case viola la convención de nombrado de Kotlin.

**Concepto de la materia:** Convenciones de Kotlin, patrón Repository, contrato interfaz/implementación.

**Solución:** Renombrar `get_episodes()` a `getEpisodes()` en la interfaz y actualizar la llamada en el UseCase:

```kotlin
// EpisodeRepository.kt — línea 8
fun getEpisodes(): Flow<PagingData<Episode>>

// GetEpisodesUseCase.kt — línea 14
return repository.getEpisodes()
```

---

### Error 3 — Falta `baseUrl()` en el builder de Retrofit

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt` — Líneas 31–36

**Problema:**
El builder de Retrofit no incluye la llamada `.baseUrl()`. Retrofit exige obligatoriamente una URL base; sin ella, lanza `IllegalStateException: Base URL required` en tiempo de ejecución y la app **crashea al iniciar**. Agravando el problema, la URL completa fue colocada en la anotación `@GET` de `SimpsonsApi` en lugar de separarse correctamente en base URL y ruta relativa.

**Concepto de la materia:** Retrofit, configuración correcta del cliente HTTP (Clase 7 — Retrofit).

**Solución:**

```kotlin
// DataModule.kt — agregar baseUrl() en el builder
return Retrofit.Builder()
    .baseUrl("https://thesimpsonsapi.com/")
    .client(client)
    .addConverterFactory(GsonConverterFactory.create())
    .build()
    .create(SimpsonsApi::class.java)

// EpisodeRemoteMediator.kt — usar solo la ruta relativa en @GET
@GET("api/episodes")
suspend fun getEpisodes(@Query("page") page: Int): EpisodesResponse
```

---

### Error 4 — `MainScreenViewModel` sin anotaciones de Hilt

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreenViewModel.kt` — Línea 10

**Problema:**
`MainScreenViewModel` no tiene la anotación `@HiltViewModel` ni usa `@Inject constructor`. Sin estas anotaciones, Hilt no puede proveer este ViewModel. Si se intentara obtenerlo con `hiltViewModel()` en un Composable, la app **crashearía en runtime**.

**Concepto de la materia:** Hilt, inyección de dependencias, `@HiltViewModel` (Clase 9 — Hilt).

**Solución:** Agregar las anotaciones de Hilt. O bien eliminar esta clase ya que es un remanente del template inicial que no conecta con la arquitectura real:

```kotlin
@HiltViewModel
class MainScreenViewModel @Inject constructor(
    dataRepository: DataRepository
) : ViewModel() { ... }
```

---

### Error 5 — `DataRepository` con datos hardcodeados (código muerto)

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/DataRepository.kt` — Líneas 8–14

**Problema:**
`DefaultDataRepository` emite datos hardcodeados que no representan ningún dato real de la app. No está registrada en `DataModule.kt` como proveedor de Hilt, no se usa en ninguna pantalla real y su tipo `Flow<List<String>>` no tiene relación con el modelo de dominio `Episode`. Es código muerto que quedó del template inicial de Android Studio.

**Concepto de la materia:** Patrón Repository, Arquitectura MVVM, separación de capas (Clase 5 — Arquitectura).

**Solución:** Eliminar `DataRepository.kt` y `MainScreenViewModel.kt` ya que son artefactos del template inicial. La arquitectura real ya tiene `EpisodeRepository` con su implementación correcta registrada en `DataModule`.

---

### Error 6 — Side effect ejecutado directamente en el cuerpo de un Composable

**Archivo:** `app/src/main/java/com/example/simpsonsapp/main/MainScreen.kt` — Líneas 47–49

**Problema:**

```kotlin
if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
    viewModel.refreshSeasons()
}
```

Esta llamada a `viewModel.refreshSeasons()` se ejecuta directamente en el cuerpo del Composable durante la composición. En Jetpack Compose, los Composables deben ser **funciones puras** sin efectos secundarios directos: pueden recomponerse múltiples veces por frame y esto causaría llamadas repetidas e incontroladas. La forma correcta es encapsular el efecto en un `LaunchedEffect`.

**Concepto de la materia:** Jetpack Compose, efectos secundarios, `LaunchedEffect` (Clase 3 y 4 — Compose).

**Solución:**

```kotlin
LaunchedEffect(episodes.loadState.refresh) {
    if (episodes.loadState.refresh is LoadState.NotLoading && seasons.isEmpty()) {
        viewModel.refreshSeasons()
    }
}
```

---

### Error 7 — Uso del operador `!!` (non-null assertion)

**Archivo:** `app/src/main/java/com/example/simpsonsapp/detail/DetailScreen.kt` — Línea 55

**Problema:**

```kotlin
if (episode != null) {
    EpisodeCard(
        episode = episode!!,  // operador !! innecesario y mala práctica
        ...
    )
}
```

El operador `!!` se usa dentro de un bloque `if (episode != null)`. El compilador no puede hacer smart cast automático porque `episode` es una propiedad delegada por `collectAsState()`. El uso de `!!` es explícitamente desaconsejado en el material de la materia ya que puede lanzar `NullPointerException` y es señal de un manejo incorrecto de nulabilidad.

**Concepto de la materia:** Nulabilidad en Kotlin, operadores seguros `?.` y `?:` (Clase 1 — Kotlin).

**Solución:** Usar una variable local o `let` para acceder de forma segura:

```kotlin
episode?.let { ep ->
    EpisodeCard(
        episode = ep,
        onClick = { },
        isDetailMode = true
    )
}
```

---

### Error 8 — Wildcard imports en `AppNavigation.kt`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/AppNavigation.kt` — Líneas 9–10

**Problema:**

```kotlin
import androidx.navigation.*
import androidx.compose.*
```

Los imports comodín (`*`) importan todos los símbolos de un paquete. Esto es mala práctica porque genera ambigüedad, dificulta la lectura del código, puede introducir conflictos de nombres no intencionados y hace más lento el análisis del IDE. Todos los símbolos necesarios ya están importados explícitamente en las líneas anteriores.

**Concepto de la materia:** Buenas prácticas de Kotlin (Clase 1 — Introducción a Kotlin).

**Solución:** Eliminar las líneas 9 y 10 (`import androidx.navigation.*` e `import androidx.compose.*`), ya que los imports específicos necesarios están declarados en las líneas 3 a 8 del mismo archivo.

---

### Error 9 — `EpisodeEntity` definida como `class` en lugar de `data class`

**Archivo:** `app/src/main/java/com/example/simpsonsapp/data/local/entity/EpisodeEntity.kt` — Línea 7

**Problema:**

```kotlin
@Entity(tableName = "episodes")
class EpisodeEntity(   // debería ser "data class"
    @PrimaryKey val id: Int, ...
)
```

Las entidades de Room deben definirse como `data class` y no como `class` regular. Una `data class` genera automáticamente `equals()`, `hashCode()`, `toString()` y `copy()`. Sin estos métodos, las comparaciones de objetos y la detección de cambios en las listas de Paging3 no funcionan correctamente.

**Concepto de la materia:** Room Database, entidades, `data class` en Kotlin (Clase 8 — Room).

**Solución:** Cambiar `class` por `data class` en la línea 7:

```kotlin
@Entity(tableName = "episodes")
data class EpisodeEntity(
    @PrimaryKey val id: Int,
    val airdate: String,
    val episodeNumber: Int,
    val imagePath: String,
    val name: String,
    val season: Int,
    val synopsis: String
)
```

---

### Error 10 — `HttpLoggingInterceptor` habilitado en todos los builds (incluyendo release)

**Archivo:** `app/src/main/java/com/example/simpsonsapp/di/DataModule.kt` — Líneas 24–29

**Problema:**

```kotlin
val logging = HttpLoggingInterceptor().apply {
    level = HttpLoggingInterceptor.Level.BODY
}
val client = OkHttpClient.Builder()
    .addInterceptor(logging)
    .build()
```

El interceptor de logging está configurado con nivel `BODY` (que registra headers y body completo de todas las requests y responses HTTP) y se añade **de forma incondicional**, incluyendo los builds de release. Esto expone en los logs del dispositivo información sensible de las llamadas a la API. El logging HTTP solo debe activarse en builds de debug. Además, en `build.gradle.kts` la opción `buildConfig = false` impide usar `BuildConfig.DEBUG`, lo que agrava el problema.

**Concepto de la materia:** Retrofit, buenas prácticas de seguridad, configuración por build type (Clase 7 — Retrofit).

**Solución:** Activar `buildConfig = true` en `build.gradle.kts` y condicionar el interceptor:

```kotlin
// build.gradle.kts — dentro de buildFeatures
buildConfig = true

// DataModule.kt
val client = OkHttpClient.Builder().apply {
    if (BuildConfig.DEBUG) {
        val logging = HttpLoggingInterceptor().apply {
            level = HttpLoggingInterceptor.Level.BODY
        }
        addInterceptor(logging)
    }
}.build()
```
