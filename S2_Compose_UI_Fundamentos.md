# S2 — Fundamentos de UI con Jetpack Compose

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Intermedio  
> **Reemplaza:** "S2 - Elementos Base de las IU" (Views XML, RelativeLayout, setContentView)

---

## Agenda

1. El modelo mental de Compose: composables, recomposición y estado
2. Componentes base (Text, Button, TextField, Image, Checkbox…)
3. Layouts: Column, Row, Box
4. Modifier: el sistema de decoración universal
5. Estado y `remember` / `mutableStateOf`
6. Theming con Material 3
7. Adaptación a orientación y tamaños de pantalla

---

## 1. El modelo mental: de Views a Composables

### Antes (sistema de Views)

La UI se construía en XML y se *mutaba* desde Kotlin/Java:

```xml
<!-- res/layout/activity_main.xml -->
<LinearLayout ...>
    <TextView android:id="@+id/tvSaludo" android:text="Hola" />
    <Button android:id="@+id/btnCambiar" android:text="Cambiar" />
</LinearLayout>
```

```kotlin
// MainActivity.kt
val tv = findViewById<TextView>(R.id.tvSaludo)
val btn = findViewById<Button>(R.id.btnCambiar)
btn.setOnClickListener { tv.text = "Mundo" }
```

### Ahora (Compose)

El UI es una **función** que describe *cómo debe verse* dado un estado. No hay mutación directa.

```kotlin
@Composable
fun Saludo() {
    var texto by remember { mutableStateOf("Hola") }

    Column {
        Text(text = texto)
        Button(onClick = { texto = "Mundo" }) {
            Text("Cambiar")
        }
    }
}
```

> **Metáfora:** El sistema de Views es como una pizarra: escribes y borras directamente. Compose es como una función matemática: dado un input (estado), siempre produce el mismo output (UI). Google Maps usa este mismo principio — dado un estado de mapa (zoom, posición, tráfico), renderiza exactamente el UI correcto, sin gestionar "qué cambió".

---

## 2. Componentes base

### Text

```kotlin
Text(
    text = "Precio: $99.99",
    style = MaterialTheme.typography.bodyLarge,
    color = MaterialTheme.colorScheme.onSurface,
    maxLines = 2,
    overflow = TextOverflow.Ellipsis
)
```

**Ejemplo real:** Netflix usa `Text` con `overflow = Ellipsis` en títulos de películas para que no desborden su tarjeta.

### Button / OutlinedButton / TextButton

```kotlin
// Botón primario — acción principal
Button(onClick = { /* comprar */ }) {
    Text("Comprar ahora")
}

// Botón secundario — acción alternativa
OutlinedButton(onClick = { /* cancelar */ }) {
    Text("Cancelar")
}

// Botón terciario — acción menor
TextButton(onClick = { /* ver detalles */ }) {
    Text("Ver más detalles")
}
```

> **Caso real:** En una app de e-commerce como Amazon, el "Añadir al carrito" sería un `Button` (primario), "Guardar para después" sería `OutlinedButton`, y "Ver políticas de devolución" sería `TextButton`.

### TextField

```kotlin
var correo by remember { mutableStateOf("") }

OutlinedTextField(
    value = correo,
    onValueChange = { correo = it },
    label = { Text("Correo electrónico") },
    leadingIcon = { Icon(Icons.Default.Email, contentDescription = null) },
    keyboardOptions = KeyboardOptions(keyboardType = KeyboardType.Email),
    singleLine = true
)
```

### Image

```kotlin
// Imagen desde recursos
Image(
    painter = painterResource(id = R.drawable.logo),
    contentDescription = "Logo de la empresa",
    modifier = Modifier.size(80.dp),
    contentScale = ContentScale.Fit
)

// Imagen desde URL (con Coil — librería estándar de industria)
AsyncImage(
    model = "https://ejemplo.com/foto.jpg",
    contentDescription = "Foto de perfil",
    modifier = Modifier
        .size(48.dp)
        .clip(CircleShape)
)
```

> Para imágenes remotas, **Coil** (`io.coil-kt:coil-compose`) es el estándar de la industria en apps Android modernas (usado por Twitter/X, Airbnb, Mercado Libre).

### Checkbox / Switch / RadioButton

```kotlin
var aceptado by remember { mutableStateOf(false) }

Row(verticalAlignment = Alignment.CenterVertically) {
    Checkbox(
        checked = aceptado,
        onCheckedChange = { aceptado = it }
    )
    Text("Acepto los términos y condiciones")
}

// Switch — para configuraciones on/off
var notificaciones by remember { mutableStateOf(true) }
Switch(
    checked = notificaciones,
    onCheckedChange = { notificaciones = it }
)
```

---

## 3. Layouts: Column, Row, Box

Los layouts en Compose son composables normales, no tipos especiales de XML.

### Column — apila elementos verticalmente

```kotlin
Column(
    modifier = Modifier
        .fillMaxWidth()
        .padding(16.dp),
    verticalArrangement = Arrangement.spacedBy(8.dp),  // espacio entre hijos
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Encabezado")
    Text("Cuerpo del contenido")
    Button(onClick = {}) { Text("Acción") }
}
```

> **Equivalente antiguo:** `LinearLayout` con `orientation="vertical"`

### Row — distribuye elementos horizontalmente

```kotlin
Row(
    modifier = Modifier
        .fillMaxWidth()
        .padding(horizontal = 16.dp),
    horizontalArrangement = Arrangement.SpaceBetween,
    verticalAlignment = Alignment.CenterVertically
) {
    Text("Nombre de usuario", style = MaterialTheme.typography.titleMedium)
    Icon(Icons.Default.ArrowForward, contentDescription = "Ver perfil")
}
```

> **Caso real:** Una fila de contacto en WhatsApp es un `Row`: foto de perfil a la izquierda, nombre y último mensaje en el centro (otro `Column`), timestamp y badge de notificaciones a la derecha.

### Box — superposición de elementos

```kotlin
Box(
    modifier = Modifier.size(80.dp),
    contentAlignment = Alignment.BottomEnd
) {
    // Imagen de fondo
    Image(
        painter = painterResource(R.drawable.avatar),
        contentDescription = "Avatar",
        modifier = Modifier.fillMaxSize()
    )
    // Badge encima
    Badge {
        Text("3")
    }
}
```

> **Equivalente antiguo:** `FrameLayout`

### Comparación visual

```
Column          Row            Box
┌───┐           ┌─┬─┬─┐        ┌─────┐
│ A │           │A│B│C│        │  A  │
├───┤           └─┴─┴─┘        │ ┌─┐ │
│ B │                          │ │B│ │
├───┤                          │ └─┘ │
│ C │                          └─────┘
└───┘
```

---

## 4. Modifier: el sistema de decoración universal

`Modifier` es la forma de agregar tamaño, padding, comportamiento, clics, etc. Es **encadenable** (order matters):

```kotlin
Text(
    text = "Producto destacado",
    modifier = Modifier
        .fillMaxWidth()          // ancho completo del padre
        .padding(16.dp)          // espacio interno
        .background(             // fondo con esquinas redondeadas
            color = MaterialTheme.colorScheme.primaryContainer,
            shape = RoundedCornerShape(12.dp)
        )
        .clickable { /* abrir detalle */ }
)
```

**El orden importa:**

```kotlin
// Padding afuera del borde → el área clickeable incluye el padding
Modifier.clickable { }.padding(16.dp)

// Padding adentro del borde → el área clickeable NO incluye el padding
Modifier.padding(16.dp).clickable { }
```

> **Metáfora:** `Modifier` es como las capas de Photoshop aplicadas en orden. Si agregas sombra después de recortar, la sombra se recorta también. El orden de las capas importa.

**Modificadores más usados:**

| Modifier                      | Efecto                                  |
|-------------------------------|-----------------------------------------|
| `fillMaxSize()`               | Ocupa todo el espacio disponible        |
| `fillMaxWidth()`              | Ancho completo                          |
| `size(48.dp)`                 | Tamaño fijo                             |
| `padding(16.dp)`              | Espacio interno (o externo según orden) |
| `clip(CircleShape)`           | Recorta en círculo                      |
| `background(Color.Red)`       | Color de fondo                          |
| `clickable { }`               | Hace el elemento clickeable             |
| `weight(1f)`                  | Dentro de Row/Column: reparte espacio   |

---

## 5. Estado: `remember` y `mutableStateOf`

**Estado** es cualquier dato que puede cambiar y que afecta el UI.

```kotlin
// Estado local — solo visible dentro de este composable
var contador by remember { mutableStateOf(0) }

// remember asegura que el valor sobrevive a recomposiciones
// mutableStateOf notifica a Compose cuando el valor cambia
```

### Elevación de estado (State Hoisting)

Cuando dos composables necesitan compartir estado, se *eleva* al ancestro común:

```kotlin
// CORRECTO: Estado elevado al padre
@Composable
fun FormularioPago() {
    var monto by remember { mutableStateOf("") }

    CampoMonto(
        value = monto,
        onValueChange = { monto = it }     // callback hacia arriba
    )
    ResumenPago(monto = monto)             // pasa estado hacia abajo
}

@Composable
fun CampoMonto(value: String, onValueChange: (String) -> Unit) {
    OutlinedTextField(value = value, onValueChange = onValueChange, label = { Text("Monto") })
}

@Composable
fun ResumenPago(monto: String) {
    Text("Total: $$monto")
}
```

> **Regla de oro:** Los datos fluyen *hacia abajo*, los eventos fluyen *hacia arriba*. Igual que en una empresa: las decisiones bajan de directivos a empleados, los reportes suben de empleados a directivos.

---

## 6. Theming con Material 3

Material 3 (Material You) permite theming dinámico basado en el fondo de pantalla del usuario (Android 12+).

### Estructura del tema

```kotlin
// ui/theme/Theme.kt
@Composable
fun MiAppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true,  // theming dinámico Android 12+
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography,
        content = content
    )
}
```

### Tokens de color de Material 3

```kotlin
// Usando tokens semánticos (NO colores hardcodeados)
Card(
    colors = CardDefaults.cardColors(
        containerColor = MaterialTheme.colorScheme.surfaceVariant
    )
) {
    Text(
        text = "Título",
        color = MaterialTheme.colorScheme.onSurfaceVariant
    )
}
```

> **Por qué tokens y no `Color.Blue`?** Si hardcodeas `Color.Blue`, en modo oscuro tu app se ve rota. Con tokens como `primary` o `onSurface`, el sistema adapta automáticamente los colores al modo claro/oscuro. Spotify, Google Maps y Gmail usan esta estrategia para su soporte de dark mode.

---

## 7. Adaptación a orientación y pantallas

### `WindowSizeClass` — el reemplazo moderno de las orientaciones

```kotlin
// En MainActivity
val windowSizeClass = calculateWindowSizeClass(this)

setContent {
    MiAppTheme {
        AppConAdaptacion(windowSizeClass)
    }
}

// En el composable
@Composable
fun AppConAdaptacion(windowSizeClass: WindowSizeClass) {
    when (windowSizeClass.widthSizeClass) {
        WindowWidthSizeClass.Compact -> {
            // Teléfono en portrait → layout de una columna
            LayoutMovil()
        }
        WindowWidthSizeClass.Medium -> {
            // Tablet pequeña o teléfono en landscape → dos paneles
            LayoutDosColumnas()
        }
        WindowWidthSizeClass.Expanded -> {
            // Tablet grande o desktop → tres paneles
            LayoutTresColumnas()
        }
    }
}
```

> **Antes:** Se usaban calificadores de recursos XML (`res/layout-land/`, `res/layout-sw600dp/`). Con Compose, la lógica de adaptación está en Kotlin, junto con el resto del código. Google Docs mobile usa exactamente esta estrategia para mostrar el panel de comentarios en tablets.

---

## Resumen de equivalencias

| Concepto antiguo (Views)      | Concepto moderno (Compose)              |
|-------------------------------|-----------------------------------------|
| `LinearLayout vertical`       | `Column`                                |
| `LinearLayout horizontal`     | `Row`                                   |
| `FrameLayout`                 | `Box`                                   |
| `RelativeLayout`              | `Box` + `Modifier.align()`              |
| `TextView`                    | `Text()`                                |
| `EditText`                    | `TextField()` / `OutlinedTextField()`   |
| `ImageView`                   | `Image()` / `AsyncImage()`              |
| `Button`                      | `Button()` / `OutlinedButton()`         |
| `android:padding="16dp"`      | `Modifier.padding(16.dp)`               |
| `android:layout_weight="1"`   | `Modifier.weight(1f)`                   |
| Orientación con XML alternos  | `WindowSizeClass` + lógica Kotlin       |

---

## Recursos oficiales

- [Jetpack Compose basics codelab](https://developer.android.com/codelabs/jetpack-compose-basics)
- [Material 3 para Compose](https://developer.android.com/jetpack/compose/designsystems/material3)
- [Layouts en Compose](https://developer.android.com/jetpack/compose/layouts/basics)
- [Adaptive layouts](https://developer.android.com/jetpack/compose/layouts/adaptive)
