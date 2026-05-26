# Capítulo 2 — Networking con Retrofit y Coroutines

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Avanzado  
> **Reemplaza:** "Capítulo 2 Avanzados Móviles" (ContentProviders, AsyncTask para red)

---

## Agenda

1. Coroutines: el modelo de concurrencia moderno (reemplaza AsyncTask)
2. Retrofit: cliente HTTP declarativo
3. Serialización con kotlinx.serialization
4. Manejo de errores en red
5. Caché y estrategia offline-first
6. Content Providers modernos: MediaStore y FileProvider
7. Flujo completo: API REST → Room → UI

---

## 1. Coroutines: el reemplazo de AsyncTask

### Por qué murió AsyncTask

`AsyncTask` fue **deprecado en API 30 (Android 11)** por múltiples razones:
- Fugas de memoria si la Activity se destruía antes de terminar
- API confusa con genéricos crípticos (`AsyncTask<Params, Progress, Result>`)
- No manejaba bien el ciclo de vida de la Activity
- No era composable (difícil de encadenar operaciones)

### Comparación directa

```kotlin
// ANTIGUO — AsyncTask (no usar)
class ObtenerUsuariosTask : AsyncTask<Void, Void, List<Usuario>>() {
    override fun doInBackground(vararg params: Void): List<Usuario> {
        return api.obtenerUsuarios()  // red en background thread
    }
    override fun onPostExecute(result: List<Usuario>) {
        adapter.submitList(result)    // UI en main thread
    }
}
// Uso: ObtenerUsuariosTask().execute()
```

```kotlin
// MODERNO — Coroutines
viewModelScope.launch {
    val usuarios = withContext(Dispatchers.IO) {
        api.obtenerUsuarios()          // red en IO thread
    }
    _usuarios.value = usuarios         // UI en main thread (automático con StateFlow)
}
```

> **Metáfora:** AsyncTask era como mandar un empleado a hacer un recado y esperar a que volviera bloqueando la oficina. Las Coroutines son como un sistema de mensajería asíncrona: el trabajo se hace en paralelo, y cuando termina, te notifica sin bloquear nada.

### Dispatchers — dónde corre el código

```kotlin
// Dispatchers.Main     → hilo de UI (leer/escribir StateFlow, actualizar UI)
// Dispatchers.IO       → operaciones de I/O: red, base de datos, archivos
// Dispatchers.Default  → cómputo intensivo: ordenar listas grandes, compresión

viewModelScope.launch {
    // Este bloque corre en Main por defecto

    val datos = withContext(Dispatchers.IO) {
        // Este bloque corre en IO — seguro hacer llamadas de red aquí
        repositorio.obtenerDatosDeRed()
    }

    // De vuelta en Main — seguro actualizar el StateFlow
    _uiState.value = UiState.Success(datos)
}
```

### `viewModelScope` — el scope correcto para apps

```kotlin
class MiViewModel : ViewModel() {
    // viewModelScope se cancela automáticamente cuando el ViewModel se destruye
    // Nunca hay fugas de memoria como con AsyncTask
    fun cargarDatos() {
        viewModelScope.launch {
            // Si el usuario sale de la pantalla, esta coroutine se cancela
        }
    }
}
```

---

## 2. Retrofit: cliente HTTP moderno

Retrofit transforma una interfaz Kotlin en un cliente HTTP completamente funcional. Es el estándar de la industria para Android (usado por Instagram, Airbnb, Uber).

### Dependencias

```kotlin
// build.gradle.kts
dependencies {
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")  // para debug
}
```

### Definir el modelo de datos

```kotlin
@Serializable
data class Producto(
    val id: Int,
    val nombre: String,
    val precio: Double,
    @SerialName("imagen_url") val imagenUrl: String,  // mapea JSON "imagen_url" a Kotlin
    val categoria: String,
    val disponible: Boolean = true
)

@Serializable
data class RespuestaProductos(
    val productos: List<Producto>,
    val total: Int,
    val pagina: Int
)
```

### Definir la interfaz de la API

```kotlin
interface ProductosApi {

    @GET("products")
    suspend fun obtenerProductos(
        @Query("page") pagina: Int = 1,
        @Query("limit") limite: Int = 20,
        @Query("category") categoria: String? = null
    ): RespuestaProductos

    @GET("products/{id}")
    suspend fun obtenerProducto(@Path("id") id: Int): Producto

    @POST("products")
    suspend fun crearProducto(@Body producto: Producto): Producto

    @PUT("products/{id}")
    suspend fun actualizarProducto(
        @Path("id") id: Int,
        @Body producto: Producto
    ): Producto

    @DELETE("products/{id}")
    suspend fun eliminarProducto(@Path("id") id: Int): Response<Unit>
}
```

### Configurar Retrofit (con interceptor de autenticación)

```kotlin
object NetworkModule {

    private val json = Json {
        ignoreUnknownKeys = true    // campos extra en JSON no rompen la app
        isLenient = true
    }

    private val authInterceptor = Interceptor { chain ->
        val request = chain.request().newBuilder()
            .addHeader("Authorization", "Bearer ${TokenManager.getToken()}")
            .addHeader("Accept", "application/json")
            .build()
        chain.proceed(request)
    }

    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) HttpLoggingInterceptor.Level.BODY
                else HttpLoggingInterceptor.Level.NONE
    }

    private val okHttpClient = OkHttpClient.Builder()
        .addInterceptor(authInterceptor)
        .addInterceptor(loggingInterceptor)
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .build()

    val productosApi: ProductosApi = Retrofit.Builder()
        .baseUrl("https://api.mitienda.com/v1/")
        .client(okHttpClient)
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()
        .create(ProductosApi::class.java)
}
```

---

## 3. Manejo de errores en red

### Sealed class para resultados

```kotlin
sealed class Resultado<out T> {
    data class Exito<T>(val datos: T) : Resultado<T>()
    data class Error(val mensaje: String, val codigo: Int? = null) : Resultado<Nothing>()
    object Cargando : Resultado<Nothing>()
}
```

### Repositorio con manejo de errores

```kotlin
class ProductosRepositorio(private val api: ProductosApi) {

    suspend fun obtenerProductos(pagina: Int = 1): Resultado<List<Producto>> {
        return try {
            val respuesta = api.obtenerProductos(pagina = pagina)
            Resultado.Exito(respuesta.productos)
        } catch (e: HttpException) {
            when (e.code()) {
                401 -> Resultado.Error("Sesión expirada. Inicia sesión nuevamente.", 401)
                404 -> Resultado.Error("Productos no encontrados.", 404)
                429 -> Resultado.Error("Demasiadas solicitudes. Intenta más tarde.", 429)
                else -> Resultado.Error("Error del servidor: ${e.code()}", e.code())
            }
        } catch (e: IOException) {
            Resultado.Error("Sin conexión a internet. Verifica tu red.")
        } catch (e: Exception) {
            Resultado.Error("Error inesperado: ${e.message}")
        }
    }
}
```

### ViewModel con estados de carga

```kotlin
data class ProductosUiState(
    val productos: List<Producto> = emptyList(),
    val cargando: Boolean = false,
    val error: String? = null,
    val paginaActual: Int = 1,
    val hayMasPaginas: Boolean = true
)

class ProductosViewModel(private val repositorio: ProductosRepositorio) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductosUiState(cargando = true))
    val uiState: StateFlow<ProductosUiState> = _uiState.asStateFlow()

    init { cargarProductos() }

    fun cargarProductos() {
        viewModelScope.launch {
            _uiState.update { it.copy(cargando = true, error = null) }

            when (val resultado = repositorio.obtenerProductos()) {
                is Resultado.Exito -> _uiState.update {
                    it.copy(productos = resultado.datos, cargando = false)
                }
                is Resultado.Error -> _uiState.update {
                    it.copy(error = resultado.mensaje, cargando = false)
                }
                Resultado.Cargando -> { /* estado transitorio */ }
            }
        }
    }

    fun reintentar() = cargarProductos()
}
```

### UI con estados de carga/error/éxito

```kotlin
@Composable
fun PantallaProductos(viewModel: ProductosViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Box(modifier = Modifier.fillMaxSize()) {
        when {
            uiState.cargando && uiState.productos.isEmpty() -> {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }

            uiState.error != null && uiState.productos.isEmpty() -> {
                EstadoError(
                    mensaje = uiState.error!!,
                    onReintentar = viewModel::reintentar,
                    modifier = Modifier.align(Alignment.Center)
                )
            }

            else -> {
                LazyColumn {
                    items(uiState.productos) { producto ->
                        TarjetaProducto(producto)
                    }
                    if (uiState.cargando) {
                        item { CircularProgressIndicator(modifier = Modifier.padding(16.dp)) }
                    }
                }
            }
        }
    }
}

@Composable
fun EstadoError(mensaje: String, onReintentar: () -> Unit, modifier: Modifier = Modifier) {
    Column(
        modifier = modifier.padding(32.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Icon(Icons.Default.Warning, contentDescription = null, modifier = Modifier.size(48.dp))
        Spacer(Modifier.height(16.dp))
        Text(mensaje, textAlign = TextAlign.Center)
        Spacer(Modifier.height(16.dp))
        Button(onClick = onReintentar) { Text("Reintentar") }
    }
}
```

---

## 4. Estrategia offline-first

La arquitectura offline-first cachea datos en Room y siempre muestra datos locales mientras actualiza en segundo plano. Esto es lo que hacen apps como Spotify (música offline), Gmail (correos sin conexión) y Twitter (timeline cacheado).

```kotlin
class ProductosRepositorio(
    private val api: ProductosApi,
    private val dao: ProductoDao
) {
    // Siempre emite datos locales primero, luego actualiza desde la red
    val productos: Flow<List<Producto>> = dao.obtenerTodos()

    suspend fun sincronizar() {
        try {
            val productosRemotos = api.obtenerProductos().productos
            dao.reemplazarTodos(productosRemotos)  // actualiza la caché local
        } catch (e: IOException) {
            // Sin red — el Flow sigue emitiendo datos locales
        }
    }
}
```

---

## 5. Content Providers modernos

Los `ContentProvider` propios (para compartir datos entre apps) son raros hoy en día. El uso más común es acceder a datos del sistema: galería, contactos, archivos.

### MediaStore — acceder a imágenes de la galería

```kotlin
// Seleccionar imagen con ActivityResultContracts (moderno)
val seleccionarImagen = rememberLauncherForActivityResult(
    contract = ActivityResultContracts.PickVisualMedia()
) { uri: Uri? ->
    uri?.let { viewModel.procesarImagenSeleccionada(it) }
}

Button(onClick = {
    seleccionarImagen.launch(
        PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
    )
}) {
    Text("Seleccionar foto de perfil")
}
```

### FileProvider — compartir archivos con otras apps

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```kotlin
// Compartir un archivo generado por la app
fun compartirArchivo(context: Context, archivo: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        archivo
    )
    val intent = Intent(Intent.ACTION_SEND).apply {
        type = "application/pdf"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
    }
    context.startActivity(Intent.createChooser(intent, "Compartir reporte"))
}
```

---

## 6. Flujo completo: API REST → Room → Compose

```
       ┌─────────────────────────────────────────────────┐
       │                  RED (Retrofit)                  │
       │  GET /products → List<ProductoDto>               │
       └───────────────────────┬─────────────────────────┘
                               │ suspend fun
                               ▼
       ┌─────────────────────────────────────────────────┐
       │              REPOSITORIO                         │
       │  1. Mapear Dto → Entity                         │
       │  2. Guardar en Room (dao.insertAll)             │
       │  3. Exponer Flow<List<Producto>> desde Room     │
       └───────────────────────┬─────────────────────────┘
                               │ Flow<List<Producto>>
                               ▼
       ┌─────────────────────────────────────────────────┐
       │              VIEWMODEL                           │
       │  productos.stateIn(viewModelScope)              │
       │  → StateFlow<List<Producto>>                    │
       └───────────────────────┬─────────────────────────┘
                               │ collectAsStateWithLifecycle
                               ▼
       ┌─────────────────────────────────────────────────┐
       │              COMPOSE UI                          │
       │  LazyColumn { items(productos) { TarjetaX() } } │
       └─────────────────────────────────────────────────┘
```

---

## Resumen de equivalencias

| Concepto antiguo                     | Equivalente moderno                    |
|--------------------------------------|----------------------------------------|
| `AsyncTask`                          | Coroutines + `viewModelScope.launch`   |
| `HttpURLConnection` manual           | Retrofit                               |
| `JSONObject`/`JSONArray` manual      | `kotlinx.serialization` / Gson         |
| `ContentProvider` propio             | Room + FileProvider (según caso)       |
| `CursorLoader`                       | `Flow<T>` desde Room                   |
| `AsyncTask<Void, Void, Result>`      | `suspend fun(): Result`                |
| `onPostExecute`                      | Código después del `suspend fun`       |

---

## Recursos oficiales

- [Retrofit](https://square.github.io/retrofit/) — documentación oficial
- [Coroutines en Android](https://developer.android.com/kotlin/coroutines)
- [Kotlin serialization](https://github.com/Kotlin/kotlinx.serialization)
- [ActivityResult APIs](https://developer.android.com/training/basics/intents/result)
- [Guía offline-first](https://developer.android.com/topic/architecture/data-layer/offline-first)
