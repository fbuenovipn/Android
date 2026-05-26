# Capítulo 1 — Persistencia de Datos Moderna

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Avanzado  
> **Reemplaza:** "Capítulo 1 Avanzados Móviles" (SharedPreferences, SQLite manual, ORMLite)

---

## Agenda

1. Room: la base de datos oficial (reemplaza SQLite manual + ORMLite)
2. DataStore: preferencias modernas (reemplaza SharedPreferences)
3. Integración Room + ViewModel + Compose
4. Migraciones de esquema
5. Consultas complejas con Flow

---

## ¿Por qué cambió la persistencia?

| Antiguo                  | Moderno                        | Razón del cambio                        |
|--------------------------|--------------------------------|-----------------------------------------|
| `SharedPreferences`      | `DataStore (Preferences)`      | No es thread-safe, API síncrona         |
| SQLite manual            | `Room`                         | Verboso, propenso a errores, sin type safety |
| `ORMLite`                | `Room` (built-in)              | Room es oficial, con mejor soporte Kotlin |
| `AsyncTask`              | Coroutines                     | AsyncTask está **deprecated** desde API 30 |

---

## 1. Room: base de datos con type safety

Room es una capa de abstracción sobre SQLite. Genera SQL en tiempo de compilación (detección de errores antes de ejecutar).

> **Metáfora:** Room es como un ORM empresarial (Hibernate en Java, SQLAlchemy en Python), pero diseñado específicamente para móviles: ligero, generador de código, y con soporte nativo de Coroutines y Flow. Uber usa Room para cachear rutas y precios localmente cuando hay mala señal.

### Dependencias

```kotlin
// build.gradle.kts
plugins {
    id("com.google.devtools.ksp") version "2.0.21-1.0.26"
}

dependencies {
    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion")  // soporte Coroutines
    ksp("androidx.room:room-compiler:$roomVersion")        // generador de código
}
```

### Los tres componentes de Room

```
@Entity          →  Define la tabla (clase de datos)
@Dao             →  Define las operaciones SQL (interfaz)
@Database        →  Punto de entrada, conecta Entity + Dao
```

### @Entity — la tabla

```kotlin
@Entity(tableName = "tareas")
data class Tarea(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,

    @ColumnInfo(name = "titulo")
    val titulo: String,

    @ColumnInfo(name = "completada")
    val completada: Boolean = false,

    @ColumnInfo(name = "fecha_creacion")
    val fechaCreacion: Long = System.currentTimeMillis()
)
```

> Room convierte esta clase en el SQL equivalente:
> ```sql
> CREATE TABLE tareas (
>     id INTEGER PRIMARY KEY AUTOINCREMENT,
>     titulo TEXT NOT NULL,
>     completada INTEGER NOT NULL DEFAULT 0,
>     fecha_creacion INTEGER NOT NULL
> )
> ```

### @Dao — las operaciones

```kotlin
@Dao
interface TareaDao {

    // Flow emite automáticamente cuando los datos cambian
    @Query("SELECT * FROM tareas ORDER BY fecha_creacion DESC")
    fun obtenerTodas(): Flow<List<Tarea>>

    @Query("SELECT * FROM tareas WHERE completada = 0")
    fun obtenerPendientes(): Flow<List<Tarea>>

    @Query("SELECT * FROM tareas WHERE id = :tareaId")
    suspend fun obtenerPorId(tareaId: Int): Tarea?

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertar(tarea: Tarea)

    @Update
    suspend fun actualizar(tarea: Tarea)

    @Delete
    suspend fun eliminar(tarea: Tarea)

    // Consulta con parámetros
    @Query("SELECT * FROM tareas WHERE titulo LIKE '%' || :busqueda || '%'")
    fun buscar(busqueda: String): Flow<List<Tarea>>
}
```

> **`Flow<List<Tarea>>`** es el punto clave: cada vez que se inserta, actualiza o elimina una tarea, Room emite automáticamente la nueva lista. El UI se actualiza sin código adicional — como una hoja de cálculo en tiempo real.

### @Database — el punto de entrada

```kotlin
@Database(
    entities = [Tarea::class],
    version = 1,
    exportSchema = true          // guarda el esquema para migraciones
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun tareaDao(): TareaDao

    companion object {
        @Volatile
        private var instancia: AppDatabase? = null

        fun obtener(context: Context): AppDatabase {
            return instancia ?: synchronized(this) {
                Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                ).build().also { instancia = it }
            }
        }
    }
}
```

---

## 2. Repositorio: la capa intermediaria

```kotlin
class TareasRepositorio(private val dao: TareaDao) {

    val todasLasTareas: Flow<List<Tarea>> = dao.obtenerTodas()
    val tareasPendientes: Flow<List<Tarea>> = dao.obtenerPendientes()

    suspend fun guardar(tarea: Tarea) = dao.insertar(tarea)
    suspend fun marcarCompletada(tarea: Tarea) = dao.actualizar(tarea.copy(completada = true))
    suspend fun eliminar(tarea: Tarea) = dao.eliminar(tarea)
    fun buscar(query: String): Flow<List<Tarea>> = dao.buscar(query)
}
```

---

## 3. ViewModel con Room

```kotlin
class TareasViewModel(private val repositorio: TareasRepositorio) : ViewModel() {

    val tareas: StateFlow<List<Tarea>> = repositorio.todasLasTareas
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    fun agregarTarea(titulo: String) {
        viewModelScope.launch {
            repositorio.guardar(Tarea(titulo = titulo))
        }
    }

    fun completarTarea(tarea: Tarea) {
        viewModelScope.launch {
            repositorio.marcarCompletada(tarea)
        }
    }

    fun eliminarTarea(tarea: Tarea) {
        viewModelScope.launch {
            repositorio.eliminar(tarea)
        }
    }
}
```

---

## 4. UI Compose con Room

```kotlin
@Composable
fun PantallaTareas(viewModel: TareasViewModel = viewModel()) {
    val tareas by viewModel.tareas.collectAsStateWithLifecycle()
    var nuevaTarea by remember { mutableStateOf("") }

    Scaffold(
        floatingActionButton = {
            FloatingActionButton(onClick = {
                if (nuevaTarea.isNotBlank()) {
                    viewModel.agregarTarea(nuevaTarea)
                    nuevaTarea = ""
                }
            }) {
                Icon(Icons.Default.Add, contentDescription = "Agregar")
            }
        }
    ) { padding ->
        Column(modifier = Modifier.padding(padding)) {
            OutlinedTextField(
                value = nuevaTarea,
                onValueChange = { nuevaTarea = it },
                label = { Text("Nueva tarea") },
                modifier = Modifier
                    .fillMaxWidth()
                    .padding(16.dp),
                trailingIcon = {
                    if (nuevaTarea.isNotBlank()) {
                        IconButton(onClick = { nuevaTarea = "" }) {
                            Icon(Icons.Default.Clear, contentDescription = "Limpiar")
                        }
                    }
                }
            )

            LazyColumn {
                items(tareas, key = { it.id }) { tarea ->
                    ItemTarea(
                        tarea = tarea,
                        onCompletar = { viewModel.completarTarea(tarea) },
                        onEliminar = { viewModel.eliminarTarea(tarea) }
                    )
                }
            }
        }
    }
}

@Composable
fun ItemTarea(
    tarea: Tarea,
    onCompletar: () -> Unit,
    onEliminar: () -> Unit
) {
    ListItem(
        headlineContent = {
            Text(
                text = tarea.titulo,
                textDecoration = if (tarea.completada) TextDecoration.LineThrough else null
            )
        },
        leadingContent = {
            Checkbox(checked = tarea.completada, onCheckedChange = { onCompletar() })
        },
        trailingContent = {
            IconButton(onClick = onEliminar) {
                Icon(Icons.Default.Delete, contentDescription = "Eliminar")
            }
        }
    )
}
```

---

## 5. DataStore: reemplaza SharedPreferences

`SharedPreferences` tenía dos problemas críticos: no era thread-safe y su API era síncrona (bloqueaba el hilo principal). `DataStore` los resuelve con Coroutines y Flow.

### Preferences DataStore (clave-valor)

```kotlin
// Definir claves tipadas
object ConfiguracionKeys {
    val TEMA_OSCURO = booleanPreferencesKey("tema_oscuro")
    val IDIOMA = stringPreferencesKey("idioma")
    val NOTIFICACIONES = booleanPreferencesKey("notificaciones_activas")
}

// Repositorio de configuración
class ConfiguracionRepositorio(private val dataStore: DataStore<Preferences>) {

    val temaOscuro: Flow<Boolean> = dataStore.data
        .map { prefs -> prefs[ConfiguracionKeys.TEMA_OSCURO] ?: false }

    val idioma: Flow<String> = dataStore.data
        .map { prefs -> prefs[ConfiguracionKeys.IDIOMA] ?: "es" }

    suspend fun guardarTemaOscuro(activo: Boolean) {
        dataStore.edit { prefs ->
            prefs[ConfiguracionKeys.TEMA_OSCURO] = activo
        }
    }

    suspend fun guardarIdioma(codigo: String) {
        dataStore.edit { prefs ->
            prefs[ConfiguracionKeys.IDIOMA] = codigo
        }
    }
}
```

### Configurar DataStore en la app

```kotlin
// En Application o inyección de dependencias
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "configuracion")
```

### Usar en ViewModel + Compose

```kotlin
@Composable
fun PantallaConfiguracion(viewModel: ConfiguracionViewModel = viewModel()) {
    val temaOscuro by viewModel.temaOscuro.collectAsStateWithLifecycle(initialValue = false)
    val idioma by viewModel.idioma.collectAsStateWithLifecycle(initialValue = "es")

    Column(modifier = Modifier.padding(16.dp)) {
        Text("Configuración", style = MaterialTheme.typography.headlineSmall)

        Spacer(Modifier.height(24.dp))

        ListItem(
            headlineContent = { Text("Tema oscuro") },
            trailingContent = {
                Switch(
                    checked = temaOscuro,
                    onCheckedChange = { viewModel.cambiarTema(it) }
                )
            }
        )

        ListItem(
            headlineContent = { Text("Idioma") },
            supportingContent = { Text(idioma) },
            trailingContent = {
                Icon(Icons.Default.ArrowForward, contentDescription = null)
            },
            modifier = Modifier.clickable { /* navegar a selector de idioma */ }
        )
    }
}
```

---

## 6. Migraciones de esquema en Room

Cuando cambias la estructura de la base de datos (agregar columna, renombrar tabla), Room requiere una **migración** para no perder datos de usuarios existentes.

```kotlin
// Versión 1 → 2: agregar columna "prioridad"
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL(
            "ALTER TABLE tareas ADD COLUMN prioridad INTEGER NOT NULL DEFAULT 0"
        )
    }
}

// Registrar la migración
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

> **Por qué importan las migraciones?** Si un usuario de tu app tiene datos guardados y lanzas una actualización que cambia el esquema sin migración, Room lanzará una excepción y la app puede crashear o perder todos los datos del usuario. Google Play tiene políticas estrictas sobre pérdida de datos — esto puede resultar en reseñas negativas y reportes de bugs críticos.

---

## 7. Flujo de datos completo: Room → Flow → ViewModel → Compose

```
Base de datos Room
      │
      │  Flow<List<Tarea>>  (emite automáticamente al cambiar datos)
      ▼
  Repositorio
      │
      │  Flow<List<Tarea>>
      ▼
  ViewModel (.stateIn → StateFlow)
      │
      │  StateFlow<List<Tarea>>
      ▼
  Composable (.collectAsStateWithLifecycle → State<List<Tarea>>)
      │
      │  recomposición automática
      ▼
  UI actualizada
```

Este flujo unidireccional garantiza que:
- La fuente de verdad es siempre la base de datos
- El UI refleja exactamente el estado de los datos
- No hay inconsistencias entre lo que se muestra y lo que está guardado

> **Caso real:** La app de Google Tasks usa exactamente este patrón. Cuando marcas una tarea como completada, el cambio se persiste en Room, el Flow emite la lista actualizada, el ViewModel la propaga, y el `LazyColumn` elimina la tarea de la lista — todo automáticamente.

---

## Resumen de equivalencias

| Concepto antiguo                 | Equivalente moderno                     |
|----------------------------------|-----------------------------------------|
| `SharedPreferences`              | `DataStore<Preferences>`                |
| SQLite manual con `Cursor`       | `Room` con `@Dao`                       |
| `ORMLite`                        | `Room` (soporte nativo)                 |
| `AsyncTask` para DB              | `suspend fun` + `viewModelScope.launch` |
| Callback para resultados async   | `Flow<T>` / `suspend fun`               |
| `ContentValues` + `rawQuery`     | `@Insert`, `@Update`, `@Query` en Room  |

---

## Recursos oficiales

- [Room codelab](https://developer.android.com/codelabs/android-room-with-a-view-kotlin)
- [DataStore documentation](https://developer.android.com/topic/libraries/architecture/datastore)
- [Room with Flow](https://developer.android.com/training/data-storage/room/async-queries)
- [Guía de migración SQLite → Room](https://developer.android.com/training/data-storage/room/migrating-db-versions)
