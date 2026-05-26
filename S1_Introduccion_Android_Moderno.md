# S1 — Introducción al Desarrollo Android Moderno

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Introductorio  
> **Stack:** Kotlin · Jetpack Compose · Android Studio Hedgehog+

---

## Agenda

1. ¿Qué es Android hoy? Plataforma, ecosistema y cuota de mercado
2. Arquitectura moderna de Android (capas 2024)
3. Kotlin como lenguaje oficial
4. Android Studio: herramientas y configuración
5. Tu primera app con Jetpack Compose
6. Emuladores y dispositivos virtuales (AVD)
7. Estructura de un proyecto Compose moderno

---

## 1. ¿Qué es Android hoy?

Android es el sistema operativo móvil más usado del mundo, con **>70 % de cuota global** (StatCounter 2024). Corre en teléfonos, tablets, televisores (Android TV), autos (Android Automotive), relojes (Wear OS) y más.

**Cronología clave:**

| Año  | Evento                                                      |
|------|-------------------------------------------------------------|
| 2008 | Primera versión pública (Android 1.0, HTC Dream)           |
| 2014 | Material Design — lenguaje visual unificado                 |
| 2017 | Kotlin declarado lenguaje oficial en Google I/O             |
| 2021 | Jetpack Compose 1.0 — UI declarativa estable               |
| 2022 | Material 3 (Material You) — theming dinámico                |
| 2024 | Android 15 · Compose BOM 2024.x                            |

> **Metáfora industrial:** Si Android fuera una fábrica, el **kernel Linux** sería los cimientos y la maquinaria pesada; las **apps** serían los productos terminados; y **Jetpack Compose** sería la línea de ensamblaje moderna que reemplazó las viejas cadenas de montaje XML.

---

## 2. Arquitectura de Android (2024)

```
┌─────────────────────────────────────────┐
│           TU APLICACIÓN                 │  ← Compose UI, ViewModel, Room
├─────────────────────────────────────────┤
│        JETPACK / ANDROIDX               │  ← Navigation, DataStore, WorkManager
├─────────────────────────────────────────┤
│     ANDROID FRAMEWORK (Java API)        │  ← Activity, Service, ContentResolver
├─────────────────────────────────────────┤
│     ART  (Android Runtime)              │  ← Compila Kotlin/Java a bytecode nativo
├─────────────────────────────────────────┤
│     HAL  (Hardware Abstraction Layer)   │  ← Cámara, GPS, sensores
├─────────────────────────────────────────┤
│          KERNEL LINUX                   │  ← Drivers, memoria, procesos
└─────────────────────────────────────────┘
```

**ART vs Dalvik (historia rápida):**  
Antes de Android 5.0, cada app corría en una instancia de **Dalvik VM** (compilación JIT). Desde Android 5.0, **ART** compila el código *antes* de ejecutarlo (AOT), lo que reduce el tiempo de arranque y el consumo de batería. Es como pasar de traducir un libro palabra por palabra mientras lo lees (JIT) a haberlo traducido completo antes (AOT).

---

## 3. Kotlin: el lenguaje oficial

Desde 2019, Google declaró Android **Kotlin-first**. Kotlin es:

- **Conciso:** menos código para lo mismo que Java
- **Seguro:** null safety integrado en el tipo de sistema
- **Interoperable:** 100 % compatible con código Java existente
- **Moderno:** coroutines, extension functions, data classes

**Ejemplo comparativo — un modelo de datos:**

```java
// Java (antiguo)
public class Usuario {
    private String nombre;
    private int edad;
    public Usuario(String nombre, int edad) {
        this.nombre = nombre;
        this.edad = edad;
    }
    public String getNombre() { return nombre; }
    public int getEdad() { return edad; }
}
```

```kotlin
// Kotlin (moderno)
data class Usuario(val nombre: String, val edad: Int)
```

> Una `data class` de Kotlin genera automáticamente `equals()`, `hashCode()`, `toString()` y `copy()`. Es el equivalente a usar un generador de código, pero nativo en el lenguaje.

---

## 4. Android Studio

Android Studio es el IDE oficial, basado en IntelliJ IDEA. La versión mínima recomendada para Compose es **Hedgehog (2023.1.1)** o posterior.

**Herramientas clave:**

| Herramienta          | Para qué sirve                                       |
|----------------------|------------------------------------------------------|
| Compose Preview      | Ver el UI en tiempo de diseño sin ejecutar la app    |
| Layout Inspector     | Inspeccionar el árbol de composables en runtime      |
| Profiler             | Medir CPU, memoria, red y energía                    |
| Logcat               | Ver logs del dispositivo/emulador                    |
| Device Manager       | Crear y gestionar emuladores (AVD)                   |

### Configuración mínima del proyecto (`build.gradle.kts`)

```kotlin
android {
    compileSdk = 35

    defaultConfig {
        minSdk = 24          // cubre ~97% de dispositivos activos
        targetSdk = 35
    }
    buildFeatures {
        compose = true       // habilitar Compose
    }
    composeOptions {
        kotlinCompilerExtensionVersion = "1.5.14"
    }
}

dependencies {
    val composeBom = platform("androidx.compose:compose-bom:2024.09.00")
    implementation(composeBom)
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.ui:ui-tooling-preview")
    implementation("androidx.activity:activity-compose:1.9.2")
}
```

> **¿Qué es el BOM (Bill of Materials)?** Es como un menú del día: defines una versión del BOM y todas las librerías Compose adoptan versiones compatibles entre sí. Sin BOM, tendrías que gestionar manualmente que `material3:1.2.0` sea compatible con `ui:1.6.0`.

---

## 5. Tu primera app con Jetpack Compose

### El cambio de paradigma: imperativo → declarativo

**Antiguo enfoque (Views + XML):** "Toma este botón, cámbialo a rojo, ponle el texto 'Hola'." El desarrollador *mutar* el estado del UI.

**Nuevo enfoque (Compose):** "Si el estado es X, el UI debe verse así." Compose *reconstruye* automáticamente el UI cuando el estado cambia.

> **Metáfora:** El sistema XML es como moldear arcilla: tocas la pieza directamente para cambiar su forma. Compose es como imprimir en 3D: describes el resultado final y la máquina lo construye cada vez que cambia el diseño.

### Código completo — Hola Mundo con estado

```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                ContadorApp()
            }
        }
    }
}

@Composable
fun ContadorApp() {
    var contador by remember { mutableStateOf(0) }

    Column(
        modifier = Modifier.fillMaxSize(),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Contador: $contador",
            style = MaterialTheme.typography.headlineMedium
        )
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = { contador++ }) {
            Text("Incrementar")
        }
    }
}

// Vista previa en Android Studio (sin ejecutar el emulador)
@Preview(showBackground = true)
@Composable
fun ContadorAppPreview() {
    MaterialTheme {
        ContadorApp()
    }
}
```

**Conceptos nuevos aquí:**

- `@Composable`: anotación que convierte una función en un elemento de UI
- `remember { mutableStateOf(0) }`: estado local que Compose rastraea
- `by`: delegado de Kotlin que permite leer/escribir `contador` directamente
- Cuando `contador` cambia, Compose **recompone** solo las partes afectadas

---

## 6. Emuladores y AVD

**Crear un AVD en Android Studio:**

1. `Device Manager` → `Create Virtual Device`
2. Seleccionar hardware: `Pixel 8` (recomendado para probar Material 3)
3. Seleccionar imagen del sistema: **API 35** (Android 15), imagen `x86_64`
4. Verificar configuración y finalizar

**Consejo de rendimiento:** En Mac con Apple Silicon, usa imágenes `arm64-v8a` — corren nativamente y son significativamente más rápidas que las x86_64 con traducción Rosetta.

---

## 7. Estructura de un proyecto Compose moderno

```
MiApp/
├── app/
│   ├── src/main/
│   │   ├── java/com/ejemplo/miapp/
│   │   │   ├── MainActivity.kt          ← Punto de entrada
│   │   │   ├── ui/
│   │   │   │   ├── theme/
│   │   │   │   │   ├── Theme.kt         ← MaterialTheme, colores
│   │   │   │   │   ├── Color.kt
│   │   │   │   │   └── Type.kt          ← Tipografía
│   │   │   │   └── screens/
│   │   │   │       └── HomeScreen.kt    ← Pantalla principal
│   │   │   └── data/                    ← Capa de datos (Room, Retrofit)
│   │   └── res/
│   │       └── values/
│   │           └── strings.xml          ← Strings localizables
│   └── build.gradle.kts
└── build.gradle.kts
```

> **Comparación con el pasado:** En el modelo XML, `res/layout/` era el corazón del UI. En Compose, `res/layout/` desaparece casi por completo — el UI vive en archivos `.kt`. Esto unifica lógica y presentación en el mismo lenguaje, eliminando la fricción de sincronizar XML y Java/Kotlin.

---

## Resumen de cambios respecto al material anterior

| Concepto antiguo          | Equivalente moderno              |
|---------------------------|----------------------------------|
| `Activity` + XML Layout   | `ComponentActivity` + `setContent {}` |
| Java                      | Kotlin                           |
| Dalvik VM                 | ART                              |
| `R.layout.activity_main`  | Función `@Composable`            |
| `TextView` en XML         | `Text()` composable              |
| `Button` en XML           | `Button()` composable            |

---

## Recursos oficiales

- [developer.android.com/compose](https://developer.android.com/compose) — Documentación oficial Compose
- [Jetpack Compose Pathway](https://developer.android.com/courses/pathways/compose) — Ruta de aprendizaje oficial Google
- [Now in Android](https://github.com/android/nowinandroid) — App de referencia real de Google con arquitectura completa
