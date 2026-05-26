# S4 — Compose UI Avanzado: Animaciones, Scaffold y Componentes Complejos

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Intermedio-Avanzado  
> **Reemplaza:** "S3 IU.pdf" (Basic Views, Picker Views, List Views, Fragments especializados)

---

## Agenda

1. Scaffold: la estructura de pantalla estándar
2. TopAppBar, BottomNavigationBar, NavigationDrawer
3. Animaciones en Compose
4. Componentes de selección: DatePicker, TimePicker
5. Paginación con Pager
6. Pull-to-refresh
7. Componentes personalizados con Canvas

---

## 1. Scaffold: la estructura estándar de una pantalla

`Scaffold` implementa el layout de Material Design: barra superior, barra inferior, FAB, snackbar, y contenido principal, todos en sus posiciones correctas.

```kotlin
@Composable
fun PantallaProductos(navController: NavController) {
    val snackbarHostState = remember { SnackbarHostState() }

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Catálogo") },
                navigationIcon = {
                    IconButton(onClick = { navController.popBackStack() }) {
                        Icon(Icons.AutoMirrored.Filled.ArrowBack, contentDescription = "Atrás")
                    }
                },
                actions = {
                    IconButton(onClick = { /* buscar */ }) {
                        Icon(Icons.Default.Search, contentDescription = "Buscar")
                    }
                    IconButton(onClick = { /* filtrar */ }) {
                        Icon(Icons.Default.FilterList, contentDescription = "Filtrar")
                    }
                }
            )
        },
        floatingActionButton = {
            FloatingActionButton(onClick = { /* agregar producto */ }) {
                Icon(Icons.Default.Add, contentDescription = "Agregar")
            }
        },
        snackbarHost = { SnackbarHost(snackbarHostState) }
    ) { paddingValues ->
        // El contenido principal debe respetar paddingValues
        // para no quedar debajo de la TopAppBar o el FAB
        LazyColumn(
            modifier = Modifier.padding(paddingValues),
            contentPadding = PaddingValues(16.dp)
        ) {
            // items...
        }
    }
}
```

> **Por qué `paddingValues`?** Scaffold sabe dónde está la TopAppBar y el FAB. Si no aplicas `paddingValues` al contenido, tus elementos quedarán debajo de estas barras — un error visual muy común al empezar con Compose.

---

## 2. Navegación inferior y lateral

### BottomNavigationBar

```kotlin
// Definir los destinos
sealed class DestinoNavInferior(
    val ruta: String,
    val etiqueta: String,
    val icono: ImageVector
) {
    object Inicio : DestinoNavInferior("inicio", "Inicio", Icons.Default.Home)
    object Buscar : DestinoNavInferior("buscar", "Buscar", Icons.Default.Search)
    object Perfil : DestinoNavInferior("perfil", "Perfil", Icons.Default.Person)
    object Carrito : DestinoNavInferior("carrito", "Carrito", Icons.Default.ShoppingCart)
}

val destinos = listOf(
    DestinoNavInferior.Inicio,
    DestinoNavInferior.Buscar,
    DestinoNavInferior.Carrito,
    DestinoNavInferior.Perfil
)

// Scaffold con BottomBar
Scaffold(
    bottomBar = {
        NavigationBar {
            val navBackStackEntry by navController.currentBackStackEntryAsState()
            val rutaActual = navBackStackEntry?.destination?.route

            destinos.forEach { destino ->
                NavigationBarItem(
                    icon = { Icon(destino.icono, contentDescription = destino.etiqueta) },
                    label = { Text(destino.etiqueta) },
                    selected = rutaActual == destino.ruta,
                    onClick = {
                        navController.navigate(destino.ruta) {
                            // Evitar múltiples copias del mismo destino en el backstack
                            popUpTo(navController.graph.findStartDestination().id) {
                                saveState = true
                            }
                            launchSingleTop = true
                            restoreState = true
                        }
                    }
                )
            }
        }
    }
) { ... }
```

### NavigationDrawer (panel lateral)

```kotlin
val drawerState = rememberDrawerState(initialValue = DrawerValue.Closed)
val scope = rememberCoroutineScope()

ModalNavigationDrawer(
    drawerState = drawerState,
    drawerContent = {
        ModalDrawerSheet {
            Spacer(Modifier.height(16.dp))
            Text("Mi App", modifier = Modifier.padding(16.dp), style = MaterialTheme.typography.titleLarge)
            HorizontalDivider()
            NavigationDrawerItem(
                label = { Text("Inicio") },
                selected = false,
                icon = { Icon(Icons.Default.Home, contentDescription = null) },
                onClick = {
                    scope.launch { drawerState.close() }
                    navController.navigate("inicio")
                }
            )
            // más items...
        }
    }
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Mi App") },
                navigationIcon = {
                    IconButton(onClick = { scope.launch { drawerState.open() } }) {
                        Icon(Icons.Default.Menu, contentDescription = "Menú")
                    }
                }
            )
        }
    ) { ... }
}
```

---

## 3. Animaciones en Compose

Compose tiene un sistema de animaciones declarativo. Las más comunes:

### `AnimatedVisibility` — mostrar/ocultar con transición

```kotlin
var visible by remember { mutableStateOf(true) }

AnimatedVisibility(
    visible = visible,
    enter = fadeIn() + slideInVertically(),
    exit = fadeOut() + slideOutVertically()
) {
    Card {
        Text("Este elemento se anima", modifier = Modifier.padding(16.dp))
    }
}

Button(onClick = { visible = !visible }) {
    Text(if (visible) "Ocultar" else "Mostrar")
}
```

### `animateContentSize` — animar cambios de tamaño

```kotlin
var expandido by remember { mutableStateOf(false) }

Card(
    modifier = Modifier
        .fillMaxWidth()
        .animateContentSize()  // ← anima automáticamente cuando cambia el tamaño
        .clickable { expandido = !expandido }
) {
    Column(modifier = Modifier.padding(16.dp)) {
        Text("Descripción del producto", style = MaterialTheme.typography.titleMedium)
        if (expandido) {
            Spacer(Modifier.height(8.dp))
            Text("Descripción detallada que puede ser muy larga y ocupa múltiples líneas cuando el usuario expande la tarjeta para leer más información sobre este producto.")
        }
    }
}
```

### `animate*AsState` — animar valores individuales

```kotlin
var seleccionado by remember { mutableStateOf(false) }

val escala by animateFloatAsState(
    targetValue = if (seleccionado) 1.1f else 1.0f,
    animationSpec = spring(dampingRatio = Spring.DampingRatioMediumBouncy),
    label = "escala_seleccion"
)

val color by animateColorAsState(
    targetValue = if (seleccionado) MaterialTheme.colorScheme.primaryContainer
                  else MaterialTheme.colorScheme.surface,
    label = "color_seleccion"
)

Card(
    modifier = Modifier
        .scale(escala)
        .clickable { seleccionado = !seleccionado },
    colors = CardDefaults.cardColors(containerColor = color)
) {
    // contenido
}
```

> **Caso real:** La animación del ícono del corazón en Instagram (escala con rebote al dar like) se logra con este patrón. Twitter/X usa `AnimatedVisibility` para mostrar/ocultar opciones de respuesta.

---

## 4. DatePicker y TimePicker (Material 3)

```kotlin
// DatePicker Modal
var mostrarDatePicker by remember { mutableStateOf(false) }
var fechaSeleccionada by remember { mutableStateOf<Long?>(null) }

if (mostrarDatePicker) {
    val datePickerState = rememberDatePickerState()

    DatePickerDialog(
        onDismissRequest = { mostrarDatePicker = false },
        confirmButton = {
            TextButton(onClick = {
                fechaSeleccionada = datePickerState.selectedDateMillis
                mostrarDatePicker = false
            }) {
                Text("Aceptar")
            }
        },
        dismissButton = {
            TextButton(onClick = { mostrarDatePicker = false }) {
                Text("Cancelar")
            }
        }
    ) {
        DatePicker(state = datePickerState)
    }
}

// Botón para mostrar el picker
OutlinedButton(onClick = { mostrarDatePicker = true }) {
    Icon(Icons.Default.DateRange, contentDescription = null)
    Spacer(Modifier.width(8.dp))
    Text(
        text = fechaSeleccionada?.let {
            SimpleDateFormat("dd/MM/yyyy", Locale.getDefault()).format(Date(it))
        } ?: "Seleccionar fecha"
    )
}
```

---

## 5. Pager: carruseles y pantallas deslizables

```kotlin
// Carrusel de imágenes (estilo banners de e-commerce)
val pagerState = rememberPagerState(pageCount = { imagenes.size })

Column {
    HorizontalPager(
        state = pagerState,
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    ) { pagina ->
        AsyncImage(
            model = imagenes[pagina],
            contentDescription = "Imagen $pagina",
            modifier = Modifier.fillMaxSize(),
            contentScale = ContentScale.Crop
        )
    }

    // Indicador de página (dots)
    Row(
        modifier = Modifier
            .fillMaxWidth()
            .padding(top = 8.dp),
        horizontalArrangement = Arrangement.Center
    ) {
        repeat(imagenes.size) { index ->
            val color by animateColorAsState(
                targetValue = if (pagerState.currentPage == index)
                    MaterialTheme.colorScheme.primary
                else MaterialTheme.colorScheme.onSurface.copy(alpha = 0.3f),
                label = "dot_color"
            )
            Box(
                modifier = Modifier
                    .padding(4.dp)
                    .size(8.dp)
                    .clip(CircleShape)
                    .background(color)
            )
        }
    }
}
```

---

## 6. Pull-to-refresh

```kotlin
// Dependencia
// implementation("androidx.compose.material:material:1.7.x")
// o desde Accompanist (community library)

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun ListaConRefresh(viewModel: ProductosViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    val pullToRefreshState = rememberPullToRefreshState()

    PullToRefreshBox(
        isRefreshing = uiState.cargando,
        onRefresh = { viewModel.recargar() },
        state = pullToRefreshState
    ) {
        LazyColumn(modifier = Modifier.fillMaxSize()) {
            items(uiState.productos) { producto ->
                TarjetaProducto(producto)
            }
        }
    }
}
```

---

## 7. Componente personalizado con Canvas

Para cuando los componentes estándar no son suficientes:

```kotlin
// Indicador de progreso circular personalizado
@Composable
fun IndicadorCircular(
    progreso: Float,             // 0.0 a 1.0
    modifier: Modifier = Modifier,
    color: Color = MaterialTheme.colorScheme.primary
) {
    val animatedProgress by animateFloatAsState(
        targetValue = progreso,
        animationSpec = tween(durationMillis = 1000),
        label = "progreso"
    )

    Canvas(modifier = modifier.size(80.dp)) {
        val ancho = 8.dp.toPx()
        val diametro = size.minDimension - ancho

        // Círculo de fondo (gris)
        drawArc(
            color = color.copy(alpha = 0.2f),
            startAngle = -90f,
            sweepAngle = 360f,
            useCenter = false,
            style = Stroke(width = ancho, cap = StrokeCap.Round),
            size = Size(diametro, diametro),
            topLeft = Offset(ancho / 2, ancho / 2)
        )

        // Círculo de progreso (color principal)
        drawArc(
            color = color,
            startAngle = -90f,
            sweepAngle = 360f * animatedProgress,
            useCenter = false,
            style = Stroke(width = ancho, cap = StrokeCap.Round),
            size = Size(diametro, diametro),
            topLeft = Offset(ancho / 2, ancho / 2)
        )
    }
}

// Uso
IndicadorCircular(
    progreso = 0.75f,
    modifier = Modifier.size(80.dp)
)
```

---

## Resumen de equivalencias

| Concepto antiguo                   | Equivalente moderno Compose              |
|------------------------------------|------------------------------------------|
| `ActionBar` / `Toolbar`            | `TopAppBar`                              |
| `BottomNavigationView`             | `NavigationBar`                          |
| `NavigationDrawer` XML             | `ModalNavigationDrawer`                  |
| `ViewPager2`                       | `HorizontalPager` / `VerticalPager`      |
| `SwipeRefreshLayout`               | `PullToRefreshBox`                       |
| `DatePickerDialog` Java            | `DatePickerDialog` Composable (M3)       |
| Animaciones con `ObjectAnimator`   | `animate*AsState`, `AnimatedVisibility`  |
| Canvas/custom View                 | `Canvas` composable                      |

---

## Recursos oficiales

- [Animaciones en Compose](https://developer.android.com/jetpack/compose/animation/introduction)
- [Pager](https://developer.android.com/jetpack/compose/layouts/pager)
- [Material 3 components](https://developer.android.com/jetpack/compose/components)
- [Canvas en Compose](https://developer.android.com/jetpack/compose/graphics/draw/overview)
