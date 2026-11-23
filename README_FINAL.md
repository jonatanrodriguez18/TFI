# Teclado Seguro — Prevención de Phishing y Fraude Financiero

**Trabajo Final Integrador (PTFI) · Licenciatura en Ciberseguridad · Universidad Nacional Scalabrini Ortiz (UNSO)**

Teclado virtual para Android, accesible y con un motor **DLP (Data Loss Prevention) de entrada** que analiza en tiempo real y de forma local lo que el usuario escribe. Si detecta datos financieros sensibles o señales de ingeniería social, avisa en pantalla y envía una alerta por SMS a un contacto de confianza.

> Título de la propuesta: *Desarrollo de un Teclado Seguro y Accesible en Android (DLP de Entrada) con Sistema de Alerta Temprana para la Mitigación de Fraudes Financieros e Ingeniería Social Multietaria.*

---

## Problema que resuelve

La fuga involuntaria de credenciales y datos financieros durante el uso cotidiano de mensajería (por ejemplo, pasarle el CBU o los datos de la tarjeta a un falso "asesor" que llamó por teléfono) es una de las vías más comunes del fraude. La evidencia citada en la propuesta muestra que la ingeniería social no afecta solo a adultos mayores: los jóvenes de 18 a 30 años también presentan tasas elevadas de victimización por su hiperconectividad.

El proyecto cubre ambos extremos:

- **Factor de impulsividad:** una alerta inmediata en pantalla y un aviso a un tercero de confianza.
- **Barreras ergonómicas:** teclas de tamaño configurable, contraste alto y respuesta háptica, pensados para adultos mayores.

## Funcionalidades

### Motor DLP local (`MotorDlp.kt`)
- Detección de **tarjetas** (13–19 dígitos) validadas con el **algoritmo de Luhn**, tolerando espacios y guiones.
- Detección de **CBU/CVU** (22 dígitos) y **DNI** (7–8 dígitos).
- Detección de **CVV/PIN**, por contexto ("código", "cvv", "pin", "clave", "tarjeta") o por bloques cortos de 3–4 dígitos.
- Detección de **vencimientos** (`MM/AA`, `MM/AAAA`).
- **Diccionario de palabras clave de ingeniería social**: *urgente, bloqueo, soporte, transferí, validar, cuenta, suspensión, asesor, fraude*.
- Evaluación **en tiempo real**, tras cada pulsación, sobre un buffer de memoria de corto plazo (150 caracteres).

### Alertas y mitigación
- **Barra de alerta roja** dentro del propio teclado (visible y accesible).
- **SMS de alerta temprana** a un contacto principal y uno de respaldo, enviado por red celular (funciona **sin internet**). El mensaje usa solo caracteres GSM-7 para evitar fragmentación y bloqueos antispam de las operadoras.
- **Nivel de riesgo:** *Crítico* cuando se combina ingeniería social con un dato sensible; *Alto* en los demás casos.
- **Registro de auditoría local** (Room/SQLite) con el texto **ofuscado** (`[TARJETA OFUSCADA]`, `[CBU OFUSCADO]`, `[DNI OFUSCADO]`, `[CVV OFUSCADO]`).

### Accesibilidad y usabilidad
- Tamaño de teclado configurable: **Normal / Grande / Extra Grande**.
- Retroalimentación háptica en cada tecla.
- Borrado continuo manteniendo presionada la tecla.
- Modo mayúsculas con bloqueo (pulsación larga) y modo símbolos.
- **Español (Argentina):** incluye la **Ñ**, vocales con tilde por **pulsación larga** (`á é í ó ú`) y los signos `¿` y `¡`.
- Interfaz oscura de alto contraste (azul `#0A1929` con acentos verde cian).

### Aplicación de configuración
- Pantalla de bienvenida (Splash) y panel de configuración.
- Carga de contactos de emergencia y selección del tamaño del teclado.
- Tarjeta de estado que indica si el teclado está habilitado y seleccionado.
- **Divulgación destacada (Prominent Disclosure)** previa a solicitar el permiso `SEND_SMS`.

## Arquitectura

```
┌────────────────────────────── Dispositivo (100 % local) ──────────────────────────────┐
│                                                                                        │
│  ServicioTeclado  ──texto──▶  GestorPrivacidad  ──▶  MotorDlp  (regex + Luhn + claves) │
│  (InputMethodService)              │                                                   │
│        ▲                           ├──▶ AppDatabase / LogFraudeDao  (Room, ofuscado)   │
│        │ alerta visual             └──▶ SmsManager ──▶ contactos de confianza (SMS)    │
│        │                                                                               │
│  MainActivity / SplashActivity ──▶ GestorConfiguracion (SharedPreferences privadas)    │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

| Componente | Responsabilidad |
|---|---|
| `ServicioTeclado.kt` | Teclado (`InputMethodService`): layout, eventos de teclas, buffer, tamaño dinámico, barra de alerta |
| `MotorDlp.kt` | Detección de datos sensibles y palabras clave; validación de Luhn |
| `GestorPrivacidad.kt` | Orquesta motor, nivel de riesgo, ofuscación, log y envío de SMS |
| `GestorConfiguracion.kt` | Contactos y tamaño del teclado (`SharedPreferences`, modo privado) |
| `AppDatabase.kt`, `LogFraude.kt`, `LogFraudeDao.kt` | Persistencia local con Room |
| `MainActivity.kt`, `SplashActivity.kt` | Onboarding, configuración y estado del teclado |

### Principios de diseño
- **Offline-first / privacy by design:** el análisis se hace en el dispositivo. El texto del usuario no se envía a ningún servidor.
- **Sin permisos privilegiados:** no requiere root; usa la API oficial `InputMethodService`.
- **Ofuscación antes de persistir:** los datos sensibles nunca se guardan en claro.
- **El SMS no incluye el texto escrito**, solo el tipo de riesgo detectado.

## Estructura del repositorio

```
.
├── TecladoSeguro/              Proyecto Android (Kotlin, Gradle)
│   ├── app/src/main/java/com/tfi/tecladoseguro/
│   └── Bitácora del desarrollo Backend Tec.txt   Bitácora de desarrollo por fases
├── motor_dlp.py                Prototipo inicial del motor DLP (Python)
├── base_datos.py               Prototipo de auditoría en SQLite (Python)
└── simulador_teclado.py        Simulador de teclado para probar el motor
```

> Los archivos `.py` son el **prototipo** en el que se validó la lógica antes de migrarla a Kotlin. El motor vigente es `MotorDlp.kt`; las expresiones regulares de ambos ya no son equivalentes.

## Requisitos

- Android Studio (versión reciente) con un SDK de Android instalado.
- JDK compatible con la versión de Gradle del proyecto.
- Dispositivo o emulador con **Android 7.0 (API 24)** o superior. `targetSdk` = 37.
- Para el prototipo Python: Python 3.8+ (solo biblioteca estándar).

## Compilar y ejecutar

1. Abrí la carpeta `TecladoSeguro/` en Android Studio y esperá la sincronización de Gradle.
2. Si hace falta, verificá que exista `local.properties` con `sdk.dir=<ruta al SDK>` (Android Studio lo genera solo).
3. Ejecutá la app con **▶ Run** sobre un dispositivo o emulador.

Desde la terminal:

```bash
cd TecladoSeguro
sh ./gradlew :app:assembleDebug      # APK de depuración
sh ./gradlew :app:testDebugUnitTest  # tests unitarios del motor DLP
```

Si la terminal no encuentra el SDK, definí `ANDROID_HOME` (por ejemplo `export ANDROID_HOME=~/Library/Android/sdk` en macOS).

## Activar el teclado

Instalar la app no activa el teclado. Hay que habilitarlo y seleccionarlo:

1. Abrí **Teclado Seguro** y seguí las tarjetas de estado, o andá a *Ajustes → Sistema → Idioma y entrada → Teclado en pantalla → Administrar teclados* y activá **Teclado Seguro Accesible**.
2. En cualquier campo de texto, tocá el ícono de teclado de la barra de navegación y elegí **Teclado Seguro Accesible**.
3. Cargá los contactos de emergencia y otorgá el permiso de SMS cuando la app lo solicite.

Por línea de comandos (útil en emulador):

```bash
adb shell ime enable com.tfi.tecladoseguro/.ServicioTeclado
adb shell ime set com.tfi.tecladoseguro/.ServicioTeclado
```

> **Emulador:** si el emulador tiene teclado físico habilitado, Android puede ocultar teclados en pantalla que no lo fuerzan. Activá *Ajustes → Teclado físico → Usar el teclado en pantalla* o ejecutá `adb shell settings put secure show_ime_with_hard_keyboard 1`.

## Pruebas

El test `MotorDlpTest` verifica la detección de un escenario completo de fraude (CBU, tarjeta con Luhn, vencimiento, CVV y palabras de ingeniería social).

Datos de prueba ficticios usados por el equipo: tarjeta `4242 4242 4242 4242` y CBU `0140000000000000000000`.

## Estado frente a la propuesta

| Objetivo de la propuesta | Estado |
|---|---|
| UI/UX accesible: tamaño de teclas, respuesta táctil | ✅ Implementado (contraste fijo oscuro; sin selector de contraste) |
| Motor DLP local con regex + Luhn + CBU/CVU | ✅ Implementado |
| Análisis de **portapapeles** (pegado) antes de consolidar el texto | ⏳ Resuelto en el prototipo Python; **pendiente en Kotlin** |
| Canal de alerta desacoplado y **cifrado** (bot/webhook) | ⚠️ Se implementó **SMS** (funciona sin internet). El canal cifrado queda como mejora |
| Metadatos únicamente en la alerta (sin texto) | ✅ El SMS solo informa el tipo de riesgo |
| **Sin retención en disco** del texto | ⚠️ Se guarda un registro de auditoría local con texto **ofuscado**; conviene revisarlo contra este objetivo |
| Aviso persistente si el teclado es desactivado o reemplazado | ⏳ Pendiente (hoy solo hay tarjeta de estado dentro de la app) |
| Desarrollo seguro (OWASP Mobile Top 10) | 🔄 En curso |
| CI/CD con GitHub Actions y escaneo MobSF | ⏳ Pendiente |
| Pentesting y verificación de telemetría con Burp Suite | ⏳ Pendiente |
| Marco normativo (Ley 25.326) y consentimiento informado | ✅ Política de privacidad redactada y *Prominent Disclosure* integrado |
| Distribución: Google Play / F-Droid / APK institucional | 🔄 `.aab` firmado generado; APK para pruebas de campo |

## Limitaciones conocidas

- La detección usa expresiones regulares y palabras clave, por lo que puede generar **falsos positivos** y **falsos negativos**. Por ejemplo, "suspensión" con tilde no coincide con la palabra clave "suspension".
- Una **sola palabra clave** de ingeniería social ya dispara la alerta y el SMS, aunque no haya datos sensibles.
- El **DNI** se detecta en el motor, pero todavía no dispara alerta ni figura en el tipo de riesgo.
- El teclado **avisa y notifica**; no impide que el texto se escriba en el campo de destino.
- Layout disponible solo en **español**. No hay sugerencias, autocorrección ni emojis.
- Google Play restringe el uso del permiso `SEND_SMS` y los métodos de entrada; por eso la propuesta prevé F-Droid y distribución institucional como alternativa.

## Privacidad

- El análisis es 100 % local; no hay llamadas a APIs externas.
- Los registros guardados están ofuscados y se almacenan solo en el dispositivo.
- Los contactos de emergencia se guardan en `SharedPreferences` privadas de la app.
- El SMS enviado contiene únicamente el tipo de riesgo detectado, nunca el texto del usuario.

## Equipo — Grupo 13

Integrantes y pilares según la propuesta del PTFI:

| Pilar | Responsable |
|---|---|
| 1. Gestión de flujos y marco normativo | Gladys Romina Ponce |
| 2. Lógica de datos y motor DLP | Ignacio Graneros |
| 3. Desarrollo móvil y usabilidad (UX/UI) | Jonatan Daniel Rodriguez |
| 4. DevSecOps y canal de comunicación | Paula Olivia Rapetti |
| 5. Auditoría ofensiva y pentesting | Pablo Daniel Alberto Gargiulo |

## Referencias

- Anderson, C., Benton, C., Biddle, R., & Chiasson, S. (2020). Who falls for phishing? A study of susceptibility and age differences in online fraud. *Computers in Human Behavior, 108*, 106316. https://doi.org/10.1016/j.chb.2020.106316
- Harrison, B., Vishwanath, A., & Rao, H. R. (2018). A survey of phishing susceptibility among young adults. *Proceedings of the 51st HICSS* (pp. 4038–4047). http://hdl.handle.net/10125/50394
- Open Web Application Security Project. (2024). *OWASP Mobile Top 10 Security Risks*. https://owasp.org/www-project-mobile-top-10/
- Stallings, W. (2020). *Cryptography and network security: Principles and practice* (8.ª ed.). Pearson.
- Zhou, Y., & Jiang, X. (2012). Dissecting Android malware: Characterization and evolution. *2012 IEEE Symposium on Security and Privacy* (pp. 95–109). https://doi.org/10.1109/SP.2012.16

## Declaración de uso de IA

Según la propuesta, se utilizó un asistente de lenguaje LLM en la redacción del documento de propuesta (menos del 15 %). La arquitectura, la lógica de validación y el análisis de riesgos fueron definidos por el equipo. Durante el desarrollo del código también se empleó asistencia de IA, que el equipo revisó y validó.
