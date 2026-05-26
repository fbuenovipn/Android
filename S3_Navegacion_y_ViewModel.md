# S3 — Navegación, ViewModel y Arquitectura MVVM

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Intermedio  
> **Reemplaza:** "S1/S2 Activities, Fragments e Intents" + "S3 IU" (Fragments, backstack manual, startActivity)

---

## Agenda

1. De Activities/Fragments a Navigation Compose
2. ViewModel: estado que sobrevive a la rotación
3. Arquitectura MVVM en Compose
4. Navigation Compose: rutas, argumentos y backstack
5. Intents modernos: acciones del sistema
6. `LazyColumn` y `LazyRow`: listas eficientes
7. Diálogos, bottom sheets y snackbars con Compose

---

## 1. De Activities/Fragments a Navigation Compose

### El modelo antiguo

```
┌──────────────┐   startActivity(Intent)   ┌──────────────┐
│  ActivityA   │ ────────────────────────► │  ActivityB   │
└──────────────┘                           └──────────────┘
       │
       │   FragmentTransaction
       ▼
┌──────────────┐
│  FragmentX   │
└──────────────┘
```

Problemas del modelo antiguo:
- El back stack era difícil de manejar con múltiples Fragments
- Pasar datos entre Activities requería `Serializable`/`Parcelable` en Intents
- Cada Fragment tenía su propio ciclo de vida complejo
- Las transiciones entre pantallas requerían código verboso

### El modelo moderno (Navigation Compose)

```
┌─────────────────────────────────────────────────────┐
│                  NavHost                             │
│                                                     │
│  composable("home") { PantallaInicio() }           │
│  composable("detalle/{id}") { DetalleProducto() }  │
│  composable("carrito") { Carrito() }               │
└─────────────────────────────────────────────────────┘
         ▲
         │ navController.navigate("detalle/42")
         │
   NavController (un solo back stack centralizado)
```

> **Metáfora:** El sistema antiguo era como un edificio donde cada piso (Activity) era un edificio separado al que debías salir y entrar con equipaje. Navigation Compose es como un edificio único con un directorio: el ascensor (NavController) te lleva al piso correcto y el botón "atrás" siempre sabe adónde llevarte.

---

## 2. ViewModel: estado que sobrevive

### El problema que resuelve

Cuando el usuario rota el teléfono, Android **destruye y recrea** la Activity. El estado local (`remember`) se pierde.

```kotlin
// PROBLEMA: el texto se borra al rotar
@Composable
fun FormularioFragilCompose() {
    var nombre by remember { mutableStateOf("") } // ← se pierde al rotar
    TextField(value = nombre, onValueChange = { nombre = it }, label = { Text("Nombre") })
}
```

### La solución: ViewModel

```kotlin
// ViewModel — sobrevive rotaciones y cambios de configuración
class RegistroViewModel : ViewModel() {

    private val _nombre = MutableStateFlow("")
    val nombre: StateFlow<String> = _nombre.asStateFlow()

    fun actualizarNombre(nuevo: String) {
        _nombre.value = nuevo
    }
}

// Composable
@Composable
fun FormularioRegistro(viewModel: RegistroViewModel = viewModel()) {
    val nombre by viewModel.nombre.collectAsStateWithLifecycle()

    OutlinedTextField(
        value = nombre,
        onValueChange = viewModel::actualizarNombre,
        label = { Text("Nombre") }
    )
}
```

> **Metáfora real:** El ViewModel es como la memoria RAM de un servidor web: persiste mientras la sesión (proceso) existe, pero es independiente de cada request (Activity recreada). Google Forms guarda tu progreso aunque recargues la página — mismo concepto.

---

## 3. Arquitectura MVVM completa

```
┌─────────────────────────────────────────────────────────────┐
│  UI LAYER                                                    │
│  ┌────────────────┐     observa StateFlow                   │
│  │  Composables   │ ◄────────────────────────────────────┐  │
│  │  (Pantallas)   │                                      │  │
│  └────────┬───────┘                                      │  │
│           │ eventos (clicks, inputs)                     │  │
│           ▼                                              │  │
│  ┌────────────────┐                                      │  │
│  │   ViewModel    │ ──── actualiza UiState ──────────────┘  │
│  └────────┬───────┘                                         │
└───────────┼─────────────────────────────────────────────────┘
            │ llama a suspend fun / Flow
┌───────────┼─────────────────────────────────────────────────┐
│  DATA LAYER                                                  │
│           ▼                                                  │
│  ┌────────────────┐                                         │
│  │   Repository   │                                         │
│  └────────┬───────┘                                         │
│           │                                                  │
│    ┌──────┴──────┐                                          │
│    ▼             ▼                                          │
│  Room DB      Retrofit API                                   │
└─────────────────────────────────────────────────────────────┘
```

### Ejemplo completo: lista de productos

```kotlin
// Modelo de datos
data class Producto(val id: Int, val nombre: String, val precio: Double)

// Estado de UI — todo lo que la pantalla necesita saber
data class ProductosUiState(
    val productos: List<Producto> = emptyList(),
    val cargando: Boolean = false,
    val error: String? = null
)

// ViewModel
class ProductosViewModel(private val repositorio: ProductosRepositorio) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductosUiState(cargando = true))
    val uiState: StateFlow<ProductosUiState> = _uiState.asStateFlow()

    init {
        cargarProductos()
    }

    private fun cargarProductos() {
        viewModelScope.launch {
            try {
                val productos = repositorio.obtenerProductos()
                _uiState.update { it.copy(productos = productos, cargando = false) }
            } catch (e: Exception) {
                _uiState.update { it.copy(error = e.message, cargando = false) }
            }
        }
    }
}

// Pantalla
@Composable
fun PantallaProductos(viewModel: ProductosViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when {
        uiState.cargando -> CircularProgressIndicator()
        uiState.error != null -> Text("Error: ${uiState.error}")
        else -> ListaProductos(productos = uiState.productos)
    }
}
```

---

## 4. Navigation Compose

### Configuración

```kotlin
// build.gradle.kts
implementation("androidx.navigation:navigation-compose:2.8.3")
```

### Configuración del NavHost

```kotlin
// NavigationApp.kt
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            PantallaInicio(
                onProductoClick = { id -> navController.navigate("detalle/$id") }
            )
        }

        composable(
            route = "detalle/{productoId}",
            arguments = listOf(navArgument("productoId") { type = NavType.IntType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getInt("productoId") ?: return@composable
            PantallaDetalle(
                productoId = id,
                onAtras = { navController.popBackStack() }
            )
        }

        composable("carrito") {
            PantallaCarrito(onComprar = { navController.navigate("confirmacion") })
        }
    }
}
```

### Pantalla de inicio con navegación

```kotlin
@Composable
fun PantallaInicio(onProductoClick: (Int) -> Unit) {
    LazyColumn {
        items(productos) { producto ->
            TarjetaProducto(
                producto = producto,
                onClick = { onProductoClick(producto.id) }
            )
        }
    }
}
```

> **Beneficio clave:** La pantalla `PantallaInicio` no sabe nada de `NavController`. Solo expone callbacks (`onProductoClick`). Esto la hace 100 % testeable de forma aislada — una práctica estándar en apps como la app de Google Play Store.

---

## 5. Intents modernos: acciones del sistema

Los `Intent` explícitos (abrir otra Activity propia) los reemplazó Navigation Compose. Los `Intent` implícitos (abrir el marcador, compartir, abrir URL) siguen siendo válidos:

```kotlin
@Composable
fun BotonesAccionSistema() {
    val context = LocalContext.current

    Column {
        // Abrir URL en el navegador
        Button(onClick = {
            val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://developer.android.com"))
            context.startActivity(intent)
        }) {
            Text("Abrir documentación")
        }

        // Compartir texto
        Button(onClick = {
            val intent = Intent(Intent.ACTION_SEND).apply {
                type = "text/plain"
                putExtra(Intent.EXTRA_TEXT, "Mira este producto increíble")
            }
            context.startActivity(Intent.createChooser(intent, "Compartir vía"))
        }) {
            Text("Compartir")
        }

        // Llamar por teléfono
        Button(onClick = {
            val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:+525500000000"))
            context.startActivity(intent)
        }) {
            Text("Llamar a soporte")
        }
    }
}
```

---

## 6. LazyColumn y LazyRow: listas eficientes

`LazyColumn` es el reemplazo de `RecyclerView`. Renderiza solo los elementos visibles en pantalla.

```kotlin
// Lista de artículos de noticias (estilo Google News)
@Composable
fun ListaNoticias(noticias: List<Noticia>) {
    LazyColumn(
        contentPadding = PaddingValues(16.dp),
        verticalArrangement = Arrangement.spacedBy(12.dp)
    ) {
        // Encabezado
        item {
            Text("Últimas noticias", style = MaterialTheme.typography.headlineSmall)
        }

        // Lista dinámica
        items(
            items = noticias,
            key = { noticia -> noticia.id }  // clave estable para animaciones
        ) { noticia ->
            TarjetaNoticia(noticia)
        }

        // Pie de página
        item {
            Spacer(modifier = Modifier.height(80.dp))
        }
    }
}

@Composable
fun TarjetaNoticia(noticia: Noticia) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        onClick = { /* navegar al detalle */ }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(noticia.titulo, style = MaterialTheme.typography.titleMedium)
            Spacer(modifier = Modifier.height(4.dp))
            Text(noticia.resumen, style = MaterialTheme.typography.bodyMedium, maxLines = 2)
        }
    }
}
```

**LazyRow** para listas horizontales (carruseles):

```kotlin
// Carrusel de categorías (estilo YouTube o Netflix)
LazyRow(
    horizontalArrangement = Arrangement.spacedBy(8.dp),
    contentPadding = PaddingValues(horizontal = 16.dp)
) {
    items(categorias) { categoria ->
        FilterChip(
            selected = categoriaSeleccionada == categoria,
            onClick = { categoriaSeleccionada = categoria },
            label = { Text(categoria.nombre) }
        )
    }
}
```

> **Por qué LazyColumn y no Column?** `Column` crea *todos* los elementos en memoria. Si tienes 10,000 tweets en un feed, `Column` intenta renderizarlos todos. `LazyColumn` (como `RecyclerView`) solo renderiza los visibles + algunos de buffer — exactamente cómo funciona Twitter/X en Android.

---

## 7. Diálogos, bottom sheets y snackbars

### AlertDialog

```kotlin
if (mostrarDialogoEliminar) {
    AlertDialog(
        onDismissRequest = { mostrarDialogoEliminar = false },
        title = { Text("Eliminar producto") },
        text = { Text("¿Estás seguro? Esta acción no se puede deshacer.") },
        confirmButton = {
            TextButton(onClick = {
                viewModel.eliminarProducto(productoId)
                mostrarDialogoEliminar = false
            }) {
                Text("Eliminar", color = MaterialTheme.colorScheme.error)
            }
        },
        dismissButton = {
            TextButton(onClick = { mostrarDialogoEliminar = false }) {
                Text("Cancelar")
            }
        }
    )
}
```

### ModalBottomSheet (reemplaza DialogFragment)

```kotlin
var mostrarBottomSheet by remember { mutableStateOf(false) }

if (mostrarBottomSheet) {
    ModalBottomSheet(
        onDismissRequest = { mostrarBottomSheet = false }
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text("Filtrar por", style = MaterialTheme.typography.titleMedium)
            Spacer(Modifier.height(16.dp))
            // Opciones de filtro...
            Button(
                onClick = { mostrarBottomSheet = false },
                modifier = Modifier.fillMaxWidth()
            ) {
                Text("Aplicar filtros")
            }
        }
    }
}
```

### Snackbar

```kotlin
val snackbarHostState = remember { SnackbarHostState() }
val scope = rememberCoroutineScope()

Scaffold(
    snackbarHost = { SnackbarHost(snackbarHostState) }
) { paddingValues ->
    Button(
        onClick = {
            scope.launch {
                val resultado = snackbarHostState.showSnackbar(
                    message = "Producto añadido al carrito",
                    actionLabel = "Ver carrito",
                    duration = SnackbarDuration.Short
                )
                if (resultado == SnackbarResult.ActionPerformed) {
                    navController.navigate("carrito")
                }
            }
        }
    ) {
        Text("Añadir al carrito")
    }
}
```

---

## Resumen de equivalencias

| Concepto antiguo                           | Equivalente moderno Compose              |
|--------------------------------------------|------------------------------------------|
| `startActivity(Intent)`                    | `navController.navigate("ruta")`         |
| `FragmentTransaction` / back stack manual  | `NavHost` + `NavController`              |
| Datos en Intent con `putExtra`             | Argumentos de ruta `"detalle/{id}"`      |
| `FragmentDialog` / `DialogFragment`        | `AlertDialog` composable                 |
| `BottomSheetDialogFragment`                | `ModalBottomSheet` composable            |
| `RecyclerView` + Adapter + ViewHolder      | `LazyColumn` / `LazyRow`                 |
| `Snackbar.make(view, ...).show()`          | `snackbarHostState.showSnackbar(...)`     |
| `onSaveInstanceState` para rotación        | `ViewModel` + `StateFlow`                |

---

## Recursos oficiales

- [Navigation Compose](https://developer.android.com/jetpack/compose/navigation)
- [ViewModel con Compose](https://developer.android.com/jetpack/compose/state#viewmodel-state)
- [Guía de arquitectura de apps](https://developer.android.com/topic/architecture)
- [LazyColumn](https://developer.android.com/jetpack/compose/lists)
- [Now in Android — navegación real](https://github.com/android/nowinandroid/tree/main/app/src/main/kotlin/com/google/samples/apps/nowinandroid/navigation)
