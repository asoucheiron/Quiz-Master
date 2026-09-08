# Quiz Master

Esquelet inicial del projecte KMP + Compose Multiplatform. Veure [REQUISITS.md](REQUISITS.md) i [ARQUITECTURA.md](ARQUITECTURA.md) per tot el context de producte i disseny tècnic.

## Estat

Esquelet mínim que compila i arrenca (mostra només "Quiz Master" en pantalla), amb:

- Estructura de paquets d'`ARQUITECTURA.md` creada dins `composeApp/src/commonMain/kotlin/com/quizmaster/`.
- Models de domini (`Quiz`, `Pregunta`, `Categoria`, `Submode`, `Dificultat`, `RespostaJugador`, `ResultatPartida`).
- Dependències configurades: Compose Multiplatform, Room (KMP), Koin, kotlinx.serialization.

Encara **no** hi ha cap pantalla (ES-1.0 i següents), ni Finders, ni UseCases, ni la base de dades Room muntada — es van afegint a mesura que s'implementa cada pantalla.

## Obrir el projecte

Obre la carpeta arrel amb Android Studio (amb el plugin de Kotlin Multiplatform instal·lat) i deixa que sincronitzi. **Coses a vigilar en aquesta primera sincronització**, ja que les versions de `gradle/libs.versions.toml` s'han triat sense poder compilar el projecte des d'aquí:

- Falta `gradle/wrapper/gradle-wrapper.jar` (és un binari; Android Studio l'hauria de regenerar sol en sincronitzar, però si no ho fa, `File → Sync Project with Gradle Files` o obrir el projecte i acceptar l'actualització del wrapper que proposi l'IDE ho hauria de resoldre).
- Si AGP/Kotlin/KSP es queixen d'incompatibilitat de versions, Android Studio sol suggerir la versió correcta automàticament — accepta l'actualització suggerida.
- No hi ha icona d'app (`android:icon`) configurada encara — pendent (veure nota de producció a REQUISITS.md §11).

## iOS

El target iOS està declarat a `composeApp/build.gradle.kts` (genera un framework `ComposeApp`), però **no hi ha cap projecte Xcode (`iosApp/`) creat** — cal generar-lo des del Mac quan toqui provar-hi (veure conversa sobre per què no es crea des de Windows).
