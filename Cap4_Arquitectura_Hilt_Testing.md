# Capítulo 4 — Arquitectura Limpia, Hilt y Testing

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Avanzado  
> **Complementa:** Todos los capítulos anteriores — cómo ensamblarlos en una app real

---

## Agenda

1. Clean Architecture en Android: capas y responsabilidades
2. Hilt: inyección de dependencias (el estándar actual)
3. Testing de ViewModels con Coroutines
4. Testing de Composables con Compose Testing
5. Testing de Room con base de datos en memoria
6. Estructura de una app de producción

---

## 1. Clean Architecture en Android

La arquitectura limpia divide la app en capas con dependencias unidireccionales. Esto garantiza que puedes cambiar la base de datos o la API sin tocar el UI, y viceversa.

```
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER (Compose + ViewModel)                             │
│  • Solo conoce UiState y envía eventos al ViewModel         │
│  • No sabe si los datos vienen de red o base de datos       │
├─────────────────────────────────────────────────────────────┤
│  DOMAIN LAYER (Use Cases)  — opcional para apps medianas    │
│  • Reglas de negocio puras (sin Android)                    │
│  • "ObtenerProductosEnOfertaUseCase"                        │
├─────────────────────────────────────────────────────────────┤
│  DATA LAYER (Repository + DataSources)                      │
│  • Coordina fuentes de datos (Room, Retrofit, DataStore)    │
│  • Implementa la interfaz definida en el Domain             │
└─────────────────────────────────────────────────────────────┘
         ↑ las capas superiores no conocen las inferiores
```

### Interfaces en el Domain, implementaciones en Data

```kotlin
// domain/ProductosRepositorio.kt — contrato (sin Android)
interface ProductosRepositorio {
    fun obtenerProductos(): Flow<List<Producto>>
    suspend fun sincronizarConServidor()
    suspend fun obtenerProductoPorId(id: Int): Producto?
}

// data/ProductosRepositorioImpl.kt — implementación real
class ProductosRepositorioImpl(
    private val dao: ProductoDao,
    private val api: ProductosApi
) : ProductosRepositorio {

    override fun obtenerProductos(): Flow<List<Producto>> = dao.obtenerTodos()

    override suspend fun sincronizarConServidor() {
        withContext(Dispatchers.IO) {
            val productos = api.obtenerProductos().productos
            dao.insertarTodos(productos.map { it.toEntity() })
        }
    }

    override suspend fun obtenerProductoPorId(id: Int) = dao.obtenerPorId(id)
}
```

### Use Cases (capa de dominio)

```kotlin
// Encapsula una operación de negocio específica
class ObtenerProductosEnOfertaUseCase(
    private val repositorio: ProductosRepositorio
) {
    operator fun invoke(): Flow<List<Producto>> {
        return repositorio.obtenerProductos()
            .map { productos -> productos.filter { it.descuento > 0 } }
    }
}

// En el ViewModel
class OfertasViewModel(
    private val obtenerOfertas: ObtenerProductosEnOfertaUseCase
) : ViewModel() {
    val ofertas = obtenerOfertas()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

---

## 2. Hilt: inyección de dependencias

Hilt es la librería oficial de DI de Android (basada en Dagger). Elimina el boilerplate de crear instancias manualmente.

> **Metáfora:** Sin DI, crear una app es como construir un coche fabricando tú mismo cada tornillo, el motor, las ruedas... Hilt es como tener una fábrica que te entrega los componentes ensamblados cuando los necesitas.

### Configuración

```kotlin
// build.gradle.kts (proyecto)
plugins {
    id("com.google.dagger.hilt.android") version "2.52" apply false
}

// build.gradle.kts (módulo app)
plugins {
    id("com.google.dagger.hilt.android")
    id("com.google.devtools.ksp")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.52")
    ksp("com.google.dagger:hilt-android-compiler:2.52")
    implementation("androidx.hilt:hilt-navigation-compose:1.2.0")
}
```

### Configurar la Application

```kotlin
@HiltAndroidApp
class MiApp : Application() {
    override fun onCreate() {
        super.onCreate()
        crearCanalesNotificacion(this)
    }
}
```

### Módulo de dependencias

```kotlin
@Module
@InstallIn(SingletonComponent::class)  // ← instancia única en toda la app
object AppModule {

    @Provides
    @Singleton
    fun provideAppDatabase(app: Application): AppDatabase {
        return AppDatabase.obtener(app)
    }

    @Provides
    fun provideProductoDao(db: AppDatabase): ProductoDao = db.productoDao()

    @Provides
    @Singleton
    fun provideProductosApi(): ProductosApi = NetworkModule.productosApi

    @Provides
    @Singleton
    fun provideProductosRepositorio(
        dao: ProductoDao,
        api: ProductosApi
    ): ProductosRepositorio = ProductosRepositorioImpl(dao, api)
}
```

### Usar en ViewModel

```kotlin
@HiltViewModel
class ProductosViewModel @Inject constructor(
    private val repositorio: ProductosRepositorio
) : ViewModel() {
    val productos = repositorio.obtenerProductos()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5_000), emptyList())
}
```

### Usar en Composables

```kotlin
// Hilt inyecta el ViewModel automáticamente — no necesitas crear instancias
@Composable
fun PantallaProductos(
    viewModel: ProductosViewModel = hiltViewModel()  // ← en lugar de viewModel()
) {
    val productos by viewModel.productos.collectAsStateWithLifecycle()
    // ...
}
```

---

## 3. Testing de ViewModels

### Configuración

```kotlin
// build.gradle.kts
testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.8.1")
testImplementation("io.mockk:mockk:1.13.12")
testImplementation("app.cash.turbine:turbine:1.1.0")  // para testear Flow
```

### Test de un ViewModel con StateFlow

```kotlin
@ExtendWith(MockKExtension::class)
class ProductosViewModelTest {

    @MockK
    private lateinit var repositorio: ProductosRepositorio

    private lateinit var viewModel: ProductosViewModel

    @Before
    fun setup() {
        // Reemplaza el dispatcher de Main por uno de test
        Dispatchers.setMain(StandardTestDispatcher())
    }

    @After
    fun tearDown() {
        Dispatchers.resetMain()
    }

    @Test
    fun `cuando se cargan productos, uiState contiene la lista`() = runTest {
        // Arrange
        val productosEsperados = listOf(
            Producto(1, "Laptop", 999.99),
            Producto(2, "Mouse", 29.99)
        )
        coEvery { repositorio.obtenerProductos() } returns flowOf(productosEsperados)

        viewModel = ProductosViewModel(repositorio)

        // Act + Assert con Turbine
        viewModel.uiState.test {
            val estado = awaitItem()
            assertThat(estado.productos).isEqualTo(productosEsperados)
            assertThat(estado.cargando).isFalse()
            cancelAndIgnoreRemainingEvents()
        }
    }

    @Test
    fun `cuando la red falla, uiState contiene el mensaje de error`() = runTest {
        coEvery { repositorio.sincronizar() } throws IOException("Sin red")

        viewModel = ProductosViewModel(repositorio)
        viewModel.recargar()
        advanceUntilIdle()  // ejecutar todas las coroutines pendientes

        assertThat(viewModel.uiState.value.error).isNotNull()
        assertThat(viewModel.uiState.value.cargando).isFalse()
    }
}
```

---

## 4. Testing de Composables

```kotlin
// build.gradle.kts
androidTestImplementation("androidx.compose.ui:ui-test-junit4")
debugImplementation("androidx.compose.ui:ui-test-manifest")

class PantallaProductosTest {

    @get:Rule
    val composeTestRule = createComposeRule()

    @Test
    fun `cuando la lista está vacía, muestra mensaje de estado vacío`() {
        composeTestRule.setContent {
            MiAppTheme {
                ListaProductos(productos = emptyList())
            }
        }

        composeTestRule
            .onNodeWithText("No hay productos disponibles")
            .assertIsDisplayed()
    }

    @Test
    fun `al hacer click en un producto, llama al callback`() {
        var productoClickeado: Producto? = null
        val productos = listOf(Producto(1, "Laptop", 999.99))

        composeTestRule.setContent {
            MiAppTheme {
                ListaProductos(
                    productos = productos,
                    onProductoClick = { productoClickeado = it }
                )
            }
        }

        composeTestRule
            .onNodeWithText("Laptop")
            .performClick()

        assertThat(productoClickeado?.id).isEqualTo(1)
    }

    @Test
    fun `el indicador de carga es visible durante la carga`() {
        composeTestRule.setContent {
            MiAppTheme {
                EstadoCargando()
            }
        }

        composeTestRule
            .onNodeWithContentDescription("Cargando")
            .assertIsDisplayed()
    }
}
```

---

## 5. Testing de Room

```kotlin
@RunWith(AndroidJUnit4::class)
class TareaDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var dao: TareaDao

    @Before
    fun crearBaseDeDatos() {
        // Base de datos en memoria — no persiste entre tests
        db = Room.inMemoryDatabaseBuilder(
            InstrumentationRegistry.getInstrumentation().targetContext,
            AppDatabase::class.java
        ).allowMainThreadQueries().build()

        dao = db.tareaDao()
    }

    @After
    fun cerrarBaseDeDatos() = db.close()

    @Test
    fun insertarYRecuperarTarea() = runTest {
        val tarea = Tarea(titulo = "Comprar leche")
        dao.insertar(tarea)

        val tareas = dao.obtenerTodas().first()
        assertThat(tareas).hasSize(1)
        assertThat(tareas[0].titulo).isEqualTo("Comprar leche")
    }

    @Test
    fun marcarTareaComoCompletada() = runTest {
        val tarea = Tarea(titulo = "Hacer ejercicio")
        dao.insertar(tarea)

        val insertada = dao.obtenerTodas().first().first()
        dao.actualizar(insertada.copy(completada = true))

        val actualizada = dao.obtenerTodas().first().first()
        assertThat(actualizada.completada).isTrue()
    }
}
```

---

## 6. Estructura de una app de producción

```
app/
├── src/
│   ├── main/
│   │   ├── kotlin/com/empresa/miapp/
│   │   │   ├── MiApp.kt                          ← @HiltAndroidApp
│   │   │   ├── MainActivity.kt                   ← punto de entrada
│   │   │   │
│   │   │   ├── domain/                           ← sin Android imports
│   │   │   │   ├── model/
│   │   │   │   │   ├── Producto.kt
│   │   │   │   │   └── Usuario.kt
│   │   │   │   ├── repository/
│   │   │   │   │   └── ProductosRepositorio.kt   ← interfaces
│   │   │   │   └── usecase/
│   │   │   │       └── ObtenerOfertasUseCase.kt
│   │   │   │
│   │   │   ├── data/                             ← implementaciones
│   │   │   │   ├── local/
│   │   │   │   │   ├── AppDatabase.kt
│   │   │   │   │   ├── ProductoDao.kt
│   │   │   │   │   └── ProductoEntity.kt
│   │   │   │   ├── remote/
│   │   │   │   │   ├── ProductosApi.kt
│   │   │   │   │   └── ProductoDto.kt
│   │   │   │   └── repository/
│   │   │   │       └── ProductosRepositorioImpl.kt
│   │   │   │
│   │   │   ├── ui/
│   │   │   │   ├── navigation/
│   │   │   │   │   └── AppNavigation.kt
│   │   │   │   ├── theme/
│   │   │   │   │   ├── Theme.kt
│   │   │   │   │   ├── Color.kt
│   │   │   │   │   └── Type.kt
│   │   │   │   ├── screens/
│   │   │   │   │   ├── productos/
│   │   │   │   │   │   ├── ProductosScreen.kt    ← Composable de pantalla
│   │   │   │   │   │   └── ProductosViewModel.kt
│   │   │   │   │   └── detalle/
│   │   │   │   │       ├── DetalleScreen.kt
│   │   │   │   │       └── DetalleViewModel.kt
│   │   │   │   └── components/                   ← componentes reutilizables
│   │   │   │       ├── TarjetaProducto.kt
│   │   │   │       └── EstadoError.kt
│   │   │   │
│   │   │   └── di/
│   │   │       └── AppModule.kt                  ← módulos Hilt
│   │   │
│   │   └── res/
│   │       └── values/strings.xml
│   │
│   ├── test/                                     ← unit tests (JVM)
│   │   └── kotlin/.../
│   │       ├── ProductosViewModelTest.kt
│   │       └── ObtenerOfertasUseCaseTest.kt
│   │
│   └── androidTest/                              ← instrumented tests (dispositivo)
│       └── kotlin/.../
│           ├── TareaDaoTest.kt
│           └── PantallaProductosTest.kt
```

---

## Apps de referencia en producción

| App                  | Patrón                              |
|----------------------|-------------------------------------|
| **Now in Android**   | MVVM + Clean + Hilt + Room + Compose |
| **Google Play Store**| MVVM + Navigation Compose            |
| **Google Maps**      | MVVM + Compose + FusedLocation       |
| **Spotify**          | Clean Architecture + offline-first   |
| **Twitter/X**        | Clean + WorkManager + offline cache  |

> **[Now in Android](https://github.com/android/nowinandroid)** es la app de referencia oficial de Google. Es open source y representa exactamente cómo Google recomienda estructurar una app en 2024.

---

## Recursos oficiales

- [Guía de arquitectura de apps Android](https://developer.android.com/topic/architecture)
- [Hilt documentation](https://developer.android.com/training/dependency-injection/hilt-android)
- [Testing en Compose](https://developer.android.com/jetpack/compose/testing)
- [Testing de coroutines](https://developer.android.com/kotlin/coroutines/test)
- [Now in Android — código fuente](https://github.com/android/nowinandroid)
