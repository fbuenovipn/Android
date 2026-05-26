# Capítulo 3 — Tareas en Background y Servicios Modernos

> **Curso:** Desarrollo Android con Jetpack Compose  
> **Nivel:** Avanzado  
> **Reemplaza:** "Capítulo 3 Avanzados Móviles" (AsyncTask, Service, IntentService, Localización con API antigua)

---

## Agenda

1. WorkManager: el reemplazo oficial de Services y AsyncTask para background
2. Coroutines en background (vs IntentService)
3. Servicios de primer plano (Foreground Services)
4. Localización con las APIs modernas (FusedLocationProvider)
5. Notificaciones modernas (Notification Channels)
6. Firebase Cloud Messaging (push notifications)

---

## El panorama del trabajo en background (2024)

| Tipo de tarea                          | Solución moderna         | Antiguo (deprecated)     |
|----------------------------------------|--------------------------|--------------------------|
| Trabajo diferible en background        | **WorkManager**          | Service, IntentService   |
| Operación asíncrona corta en UI        | **Coroutines**           | AsyncTask                |
| Tarea continua visible al usuario      | **Foreground Service**   | Service                  |
| Trabajo inmediato fuera de la UI       | **viewModelScope/launch**| Threads manuales         |

> **Metáfora general:** Antes, tú gestionabas manualmente los hilos como un chef que también limpia, compra y atiende a los clientes. Hoy, tienes un equipo especializado: WorkManager es el planificador de tareas de la cocina, las Coroutines son el equipo de cocina que trabaja en paralelo, y el Foreground Service es el chef que trabaja a la vista del cliente.

---

## 1. WorkManager: trabajo garantizado en background

WorkManager garantiza que el trabajo se ejecute **incluso si el usuario cierra la app o reinicia el dispositivo**. Es ideal para:

- Sincronización periódica de datos
- Subir fotos al servidor
- Comprimir archivos
- Enviar logs de analítica

### Dependencias

```kotlin
// build.gradle.kts
implementation("androidx.work:work-runtime-ktx:2.9.1")
```

### Crear un Worker

```kotlin
// Worker para sincronizar datos con el servidor
class SincronizacionWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {  // ← CoroutineWorker, no Worker

    override suspend fun doWork(): Result {
        return try {
            val repositorio = (applicationContext as MiApp).repositorio

            // Sincronizar con la API
            repositorio.sincronizarConServidor()

            // Trabajo completado exitosamente
            Result.success()

        } catch (e: IOException) {
            // Sin red — WorkManager reintentará automáticamente
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()

        } catch (e: Exception) {
            // Error fatal — no reintentar
            Result.failure(
                workDataOf("error_mensaje" to (e.message ?: "Error desconocido"))
            )
        }
    }
}
```

> **`CoroutineWorker`** vs **`Worker`**: usa siempre `CoroutineWorker` con `suspend fun doWork()`. El `Worker` básico bloquea un hilo — `CoroutineWorker` usa el pool de Coroutines de manera eficiente.

### Programar trabajo

```kotlin
// Trabajo único (ejecutar una vez)
val sincronizacionRequest = OneTimeWorkRequestBuilder<SincronizacionWorker>()
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)  // solo con red
            .setRequiresBatteryNotLow(true)                 // no con batería baja
            .build()
    )
    .setBackoffCriteria(
        BackoffPolicy.EXPONENTIAL,
        WorkRequest.MIN_BACKOFF_MILLIS,
        TimeUnit.MILLISECONDS
    )
    .build()

WorkManager.getInstance(context).enqueue(sincronizacionRequest)

// Trabajo periódico (repetir cada 6 horas)
val sincronizacionPeriodica = PeriodicWorkRequestBuilder<SincronizacionWorker>(
    repeatInterval = 6,
    repeatIntervalTimeUnit = TimeUnit.HOURS
)
    .setConstraints(
        Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()
    )
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "sincronizacion_periodica",
    ExistingPeriodicWorkPolicy.UPDATE,  // reemplaza si ya existe
    sincronizacionPeriodica
)
```

### Observar el estado del trabajo

```kotlin
@Composable
fun PantallaSync(context: Context) {
    val workManager = WorkManager.getInstance(context)

    val estadoSync by workManager
        .getWorkInfosForUniqueWorkLiveData("sincronizacion_periodica")
        .observeAsState()

    val info = estadoSync?.firstOrNull()

    when (info?.state) {
        WorkInfo.State.RUNNING -> {
            LinearProgressIndicator(modifier = Modifier.fillMaxWidth())
            Text("Sincronizando...")
        }
        WorkInfo.State.SUCCEEDED -> {
            Icon(Icons.Default.CheckCircle, contentDescription = null, tint = Color.Green)
            Text("Sincronizado")
        }
        WorkInfo.State.FAILED -> {
            val error = info.outputData.getString("error_mensaje")
            Text("Error: $error", color = MaterialTheme.colorScheme.error)
        }
        else -> { /* encolado o cancelado */ }
    }
}
```

---

## 2. Cadenas de trabajo (Work Chains)

WorkManager permite encadenar workers para procesos complejos:

```kotlin
// Ejemplo: subir foto → comprimir → generar miniatura → notificar
val comprimir = OneTimeWorkRequestBuilder<ComprimirImagenWorker>()
    .setInputData(workDataOf("uri" to imagenUri.toString()))
    .build()

val subir = OneTimeWorkRequestBuilder<SubirImagenWorker>().build()

val notificar = OneTimeWorkRequestBuilder<NotificarSubidaWorker>().build()

WorkManager.getInstance(context)
    .beginWith(comprimir)
    .then(subir)
    .then(notificar)
    .enqueue()
```

> **Caso real:** Instagram usa este patrón cuando subes una foto: comprime en background → sube con reintentos automáticos → notifica cuando está disponible. Funciona aunque cierres la app.

---

## 3. Foreground Service: trabajo visible

Para tareas largas que el usuario debe saber que están ocurriendo (reproducción de música, descarga de archivos, navegación GPS), se usa un **Foreground Service** con una notificación persistente.

```kotlin
class DescargaService : Service() {

    private val notificationId = 1

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notificacion = NotificationCompat.Builder(this, "descargas")
            .setContentTitle("Descargando archivo")
            .setContentText("En progreso...")
            .setSmallIcon(R.drawable.ic_download)
            .setProgress(0, 0, true)  // progreso indeterminado
            .setOngoing(true)          // no se puede descartar
            .build()

        // Iniciar como foreground (muestra notificación, sobrevive en background)
        startForeground(notificationId, notificacion)

        // Hacer el trabajo en una coroutine
        CoroutineScope(Dispatchers.IO).launch {
            realizarDescarga()
            stopSelf()  // detener el servicio al terminar
        }

        return START_NOT_STICKY  // no recrear si el sistema lo mata
    }

    override fun onBind(intent: Intent?) = null
}
```

**Declarar en AndroidManifest:**

```xml
<service
    android:name=".DescargaService"
    android:foregroundServiceType="dataSync"
    android:exported="false" />

<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC" />
```

---

## 4. Localización moderna con FusedLocationProvider

La API antigua (`LocationManager`) era difícil de usar y consumía mucha batería. `FusedLocationProviderClient` de Google Play Services maneja automáticamente el balance entre precisión y batería.

### Dependencias

```kotlin
implementation("com.google.android.gms:play-services-location:21.3.0")
```

### Solicitar permisos

```kotlin
// Declarar en AndroidManifest
// <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
// <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />

// En el Composable — solicitar permiso con el launcher moderno
val solicitarPermiso = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permisos ->
    val tienePermiso = permisos[Manifest.permission.ACCESS_FINE_LOCATION] == true ||
                       permisos[Manifest.permission.ACCESS_COARSE_LOCATION] == true
    if (tienePermiso) {
        viewModel.iniciarSeguimientoUbicacion()
    }
}

Button(onClick = {
    solicitarPermiso.launch(arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION
    ))
}) {
    Text("Activar ubicación")
}
```

### Obtener ubicación en ViewModel

```kotlin
class UbicacionViewModel(application: Application) : AndroidViewModel(application) {

    private val fusedLocationClient = LocationServices.getFusedLocationProviderClient(application)

    private val _ubicacion = MutableStateFlow<Location?>(null)
    val ubicacion: StateFlow<Location?> = _ubicacion.asStateFlow()

    @SuppressLint("MissingPermission")
    fun obtenerUbicacionActual() {
        viewModelScope.launch {
            val ubicacion = fusedLocationClient.awaitCurrentLocation(Priority.PRIORITY_HIGH_ACCURACY)
            _ubicacion.value = ubicacion
        }
    }

    @SuppressLint("MissingPermission")
    fun iniciarSeguimientoUbicacion() {
        val request = LocationRequest.Builder(
            Priority.PRIORITY_BALANCED_POWER_ACCURACY,
            10_000L  // actualizar cada 10 segundos
        ).build()

        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult) {
                _ubicacion.value = result.lastLocation
            }
        }

        fusedLocationClient.requestLocationUpdates(
            request, callback, Looper.getMainLooper()
        )
    }
}
```

### UI con mapa (Google Maps Compose)

```kotlin
// build.gradle.kts
implementation("com.google.maps.android:maps-compose:6.1.2")

// Composable
@Composable
fun PantallaMapas(viewModel: UbicacionViewModel = viewModel()) {
    val ubicacion by viewModel.ubicacion.collectAsStateWithLifecycle()

    val cameraPositionState = rememberCameraPositionState {
        position = CameraPosition.fromLatLngZoom(
            LatLng(19.4326, -99.1332),  // Ciudad de México por defecto
            12f
        )
    }

    // Centrar cámara cuando cambia la ubicación
    LaunchedEffect(ubicacion) {
        ubicacion?.let {
            cameraPositionState.animate(
                CameraUpdateFactory.newLatLngZoom(LatLng(it.latitude, it.longitude), 15f)
            )
        }
    }

    GoogleMap(
        modifier = Modifier.fillMaxSize(),
        cameraPositionState = cameraPositionState
    ) {
        ubicacion?.let {
            Marker(
                state = MarkerState(LatLng(it.latitude, it.longitude)),
                title = "Mi ubicación"
            )
        }
    }
}
```

---

## 5. Notificaciones modernas (Notification Channels)

Desde Android 8.0, las notificaciones requieren **canales** (grupos lógicos que el usuario puede controlar individualmente).

```kotlin
// Crear canales al iniciar la app (en Application o MainActivity)
fun crearCanalesNotificacion(context: Context) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
        val canalPedidos = NotificationChannel(
            "pedidos",
            "Pedidos y entregas",
            NotificationManager.IMPORTANCE_HIGH
        ).apply {
            description = "Notificaciones sobre el estado de tus pedidos"
            enableLights(true)
            lightColor = Color.GREEN
        }

        val canalOfertas = NotificationChannel(
            "ofertas",
            "Ofertas y promociones",
            NotificationManager.IMPORTANCE_DEFAULT
        ).apply {
            description = "Descuentos y ofertas especiales para ti"
        }

        val manager = context.getSystemService(NotificationManager::class.java)
        manager.createNotificationChannels(listOf(canalPedidos, canalOfertas))
    }
}

// Mostrar notificación
fun mostrarNotificacionPedido(context: Context, numeroPedido: String) {
    val intent = Intent(context, MainActivity::class.java).apply {
        putExtra("pedido_id", numeroPedido)
        flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
    }

    val pendingIntent = PendingIntent.getActivity(
        context, 0, intent,
        PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
    )

    val notificacion = NotificationCompat.Builder(context, "pedidos")
        .setSmallIcon(R.drawable.ic_shopping_bag)
        .setContentTitle("Pedido #$numeroPedido enviado")
        .setContentText("Tu pedido está en camino. Llegará en 30-45 minutos.")
        .setPriority(NotificationCompat.PRIORITY_HIGH)
        .setContentIntent(pendingIntent)
        .setAutoCancel(true)  // desaparece al tocarla
        .addAction(
            R.drawable.ic_track,
            "Rastrear pedido",
            pendingIntent
        )
        .build()

    NotificationManagerCompat.from(context).notify(numeroPedido.hashCode(), notificacion)
}
```

---

## 6. Solicitar permiso de notificaciones (Android 13+)

```kotlin
// Desde Android 13 (API 33), se requiere permiso explícito
val solicitarPermisoNotif = rememberLauncherForActivityResult(
    ActivityResultContracts.RequestPermission()
) { otorgado ->
    if (otorgado) {
        // Mostrar mensaje de confirmación
    }
}

LaunchedEffect(Unit) {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        if (ContextCompat.checkSelfPermission(
                context, Manifest.permission.POST_NOTIFICATIONS
            ) != PackageManager.PERMISSION_GRANTED
        ) {
            solicitarPermisoNotif.launch(Manifest.permission.POST_NOTIFICATIONS)
        }
    }
}
```

---

## Resumen de equivalencias

| Concepto antiguo                         | Equivalente moderno                         |
|------------------------------------------|---------------------------------------------|
| `AsyncTask`                              | `CoroutineWorker` / `viewModelScope.launch` |
| `Service` / `IntentService`              | `WorkManager` (diferible) o Foreground Service |
| `LocationManager` directo                | `FusedLocationProviderClient`               |
| `requestLocationUpdates` con Listener    | `LocationCallback` con Flow                 |
| Notificaciones sin canal                 | `NotificationChannel` + `NotificationCompat` |
| `AlarmManager` para trabajo periódico    | `PeriodicWorkRequest` en WorkManager        |

---

## Recursos oficiales

- [WorkManager guide](https://developer.android.com/topic/libraries/architecture/workmanager)
- [Background work overview](https://developer.android.com/guide/background)
- [FusedLocationProvider](https://developer.android.com/training/location/retrieve-current)
- [Notification overview](https://developer.android.com/develop/ui/views/notifications)
- [Google Maps Compose](https://developers.google.com/maps/documentation/android-sdk/maps-compose)
