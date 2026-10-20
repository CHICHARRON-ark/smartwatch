# FocusWatch

Aplicación de temporizador **Pomodoro para Wear OS** escrita en **Kotlin** con **Compose for Wear OS**. Mantiene el reloj corriendo en un servicio en primer plano con notificación persistente, vibra al terminar cada fase, muestra la frecuencia cardiaca y guarda el historial de pomodoros completados.

- Paquete / applicationId: `com.example.focuswatch`
- Min SDK 30 (Wear OS 3+), target SDK 34, compile SDK 37
- APK debug: `app/build/outputs/apk/debug/app-debug.apk`

## Funcionalidad

- **Ciclo Pomodoro**: 25 min de enfoque → vibra y arranca automático el descanso de 5 min → vibra y vuelve al enfoque esperando que el usuario lo inicie.
- **Pantalla principal**: anillo de progreso con fase, tiempo restante (`mm:ss`), frecuencia cardiaca y tres botones (Iniciar/Pausar, Reiniciar, Historial).
- **Servicio en primer plano**: notificación persistente con la fase y el tiempo restante, actualizada cada tick; se retira solo 30 s después de pausar.
- **Frecuencia cardiaca**: muestra las muestras reales de Health Services (solo durante la fase de enfoque) con respaldo simulado si no hay sensor/datos.
- **Historial**: contadores "Hoy" y "Total" de pomodoros completados, persistidos con DataStore; pantalla con `ScalingLazyColumn` y `TimeText`.
- **Navegación Wear**: `SwipeDismissableNavHost` (atrás por botón o deslizando la pantalla a la derecha).

## Arquitectura

```
UI (Compose)  ──lee──►  PomodoroEngine (singleton, StateFlow)  ◄──escribe──  TimerService (único)
FocusWatchViewModel  ──lee──►  PomodoroRepository (DataStore)  ◄──incrementa── TimerService
                 └──lee──►  HeartRateRepository (Health Services)
```

- **`timer/PomodoroEngine`**: motor puro del ciclo. El tiempo se calcula contra `SystemClock.elapsedRealtime()` (reloj monotono) y se publica cada 200 ms; emite eventos de fase completada. Es un singleton para sobrevivir a rotaciones de la Activity.
- **`timer/TimerService`**: servicio `FOREGROUND_SERVICE_TYPE_SPECIAL_USE` que es el **único escritor** del contador (evita doble conteo), construye/actualiza la notificación, vibra al completar cada fase y registra el pomodoro en DataStore. Arranca solo con la acción `ACCION_INICIAR`.
- **`data/PomodoroRepository`**: persiste `pomodoros_hoy`, `pomodoros_total` y la fecha del último incremento; "Hoy" se reinicia solo al cambiar la fecha local.
- **`health/HeartRateRepository`**: `callbackFlow` sobre `MeasureClient`; si no llegan muestras reales en 12 s (o no hay sensor), un vigilante emite valores simulados hasta que aparezca una muestra real.
- **`ui/FocusWatchViewModel`**: expone los tres flujos (estado, historial, pulso) y las acciones; inicia el servicio al pulsar "Iniciar".
- **`ui/FocusWatchApp`**: navegación con `SwipeDismissableNavHost` y `androidx.wear.compose.navigation.composable` (imprescindible: los destinos deben ser de `WearNavigator`, si no la pantalla queda vacía).

### Ciclo del temporizador

| Evento | Comportamiento |
|---|---|
| Fin de fase de enfoque | Vibración larga (600 ms), +1 pomodoro en historial, descanso de 5 min arranca **automático** |
| Fin de fase de descanso | Vibración corta (150 ms), vuelve a enfoque de 25 min **en pausa** |
| Pausar | El servicio programa su parada a los 30 s (si se reanuda antes, se cancela) |

## Tecnologías

| Componente | Versión |
|---|---|
| Android Gradle Plugin | 9.4.1 |
| Gradle wrapper | 9.6.0 |
| Kotlin (embebido en AGP) / plugin compose | 2.2.10 |
| Wear Compose (material, foundation, navigation) | 1.7.1 |
| Health Services Client | 1.1.0 |
| DataStore Preferences | 1.2.1 |
| Lifecycle / Activity Compose | 2.10.0 / 1.10.0 |
| Coroutines / Concurrent Futures | 1.10.1 / 1.2.0 |

Versiones centralizadas en `gradle/libs.versions.toml`.

## Compilar y ejecutar

Requisitos: JDK 17+ (probado con JDK 26), Android SDK con `platforms;android-37` y `build-tools;37.0.0`.

```powershell
# Windows (PowerShell) — JAVA_HOME no está definido por defecto en esta máquina
$env:JAVA_HOME = "C:\Program Files\Java\jdk-26.0.1"
$env:ANDROID_HOME = "C:\Users\kevin\AppData\Local\Android\Sdk"
.\gradlew.bat assembleDebug --no-daemon
```

Instalar y probar en el emulador de Wear OS:

```powershell
# Crear/arrancar el AVD "Wear_OS_Small_Round" (384x384 px, 320 dpi ≈ 192 dp, API 37)
& "$env:ANDROID_HOME\emulator\emulator.exe" -avd Wear_OS_Small_Round
# En otra terminal
& "$env:ANDROID_HOME\platform-tools\adb.exe" install -r app\build\outputs\apk\debug\app-debug.apk
& "$env:ANDROID_HOME\platform-tools\adb.exe" shell am start -n com.example.focuswatch/.MainActivity
```

Conceder los permisos que pide la app la primera vez (o por línea de comandos):

```powershell
adb shell pm grant com.example.focuswatch android.permission.POST_NOTIFICATIONS
adb shell pm grant com.example.focuswatch android.permission.BODY_SENSORS
```

> En el emulador la pantalla se apaga a los pocos segundos; para pruebas automatizadas conviene `adb shell svc power stayon true` o tocar la pantalla antes de cada `uiautomator dump`.

## Estructura del proyecto

```
app_smartwatch/
├── gradle/libs.versions.toml        # catálogo de versiones
├── gradle/wrapper/                  # wrapper 9.6.0
├── settings.gradle.kts, build.gradle.kts, gradle.properties
└── app/
    ├── build.gradle.kts             # compileSdk 37, minSdk 30, targetSdk 34
    └── src/main/
        ├── AndroidManifest.xml      # permisos, uses-feature watch, FGS specialUse
        ├── res/                     # strings, themes, drawables, icono adaptativo
        └── java/com/example/focuswatch/
            ├── MainActivity.kt              # setContent + solicitud de permisos
            ├── data/PomodoroRepository.kt   # contadores en DataStore
            ├── health/HeartRateRepository.kt
            ├── timer/
            │   ├── PomodoroEngine.kt        # motor del ciclo (singleton)
            │   ├── TimerService.kt          # FGS + notificación + vibración + persistencia
            │   └── Vibrador.kt
            └── ui/
                ├── FocusWatchApp.kt         # navegación Wear
                ├── FocusWatchViewModel.kt   # estado reactivo para la UI
                ├── theme/Theme.kt
                ├── screens/TimerScreen.kt, HistoryScreen.kt
                └── components/ProgressRing.kt, Iconos.kt, KeepScreenOn.kt
```

## Supuestos y decisiones de diseño

1. **Descanso automático**: al terminar el enfoque el descanso arranca solo; al terminar el descanso el enfoque queda en pausa y el usuario debe pulsar "Iniciar".
2. **Solo cuenta el enfoque**: solo la fase de enfoque completa incrementa el historial; un descanso interrumpido no cuenta.
3. **Escritor único del contador**: el servicio es quien incrementa DataStore, para no duplicar si UI y servicio observan el mismo evento.
4. **La sesión en curso no persiste**: el estado del reloj vive en memoria (`PomodoroEngine` + `elapsedRealtime`); si el proceso muere, al reabrir la app el temporizador vuelve a 25:00 en pausa (`START_NOT_STICKY`, sin guardado de la sesión).
5. **Frecuencia cardiaca de respaldo**: sin muestras reales en 12 s (emulador, sensor ausente, muñeca sin contacto) se emite un valor simulado de 62–96 lpm; solo se consulta durante la fase de enfoque.
6. **Permisos opcionales**: si se deniegan `POST_NOTIFICATIONS` o `BODY_SENSORS` la app sigue funcionando (la notificación puede no mostrarse y el pulso queda simulado).
7. **FGS de tipo `specialUse`**: desde API 34 los servicios en primer plano deben declarar un tipo; se usa `specialUse` con la propiedad `PROPERTY_SPECIAL_USE_FGS_SUBTYPE` (temporizador persistente).
8. **Auto-parada del servicio**: 30 s después de pausar el servicio se retira del primer plano para no consumir batería; reanudar antes cancela la parada.
9. **Sin entrada rotatoria**: no se implementa scroll con la corona (`RotaryScrollable`); la interacción es táctil (botones `CompactButton` y scroll estándar de la lista).
10. **Sin `TimeText` en la pantalla del temporizador** para no tapar el anillo; solo se usa en la pantalla de historial.
11. **Diseño adaptativo**: el anillo ocupa el alto disponible (`weight` + `aspectRatio(1f)`) para caber en "small round" de 192 dp y crecer en pantallas mayores.
12. **Sin pruebas automatizadas**: la verificación fue manual en el emulador (ver abajo).

## Verificación realizada en el emulador

Con el AVD `Wear_OS_Small_Round` (API 37) y `uiautomator dump`/`dumpsys`:

- La pantalla principal renderiza: `Enfoque`, `25:00`, `80 lpm` y los botones `Iniciar`/`Reiniciar`/`Historial`.
- **Iniciar** → `TimerService` en primer plano (`isForeground=true`, `types=0x40000000`), notificación `Enfoque / Quedan 24:48` que decrece junto con la UI (`24:47`), botón cambia a `Pausar`.
- **Pausar** → botón vuelve a `Iniciar`, el tiempo se congela.
- **Reiniciar** → el tiempo vuelve a `25:00`.
- **Historial** → pantalla con `Hoy 0`, `Total 0`, `TimeText` y botón `Volver` (regresa al temporizador).
- Permisos concedidos y ejecución sin cierres inesperados (buffer de crashes vacío).

Limitaciones: la transición automática enfoque→descanso no se observó end-to-end (requiere 25 min de ejecución; la lógica está en `PomodoroEngine.completarFase`), el emulador no tiene sensor de pulso real (siempre muestra el valor simulado) y no existen pruebas unitarias.
