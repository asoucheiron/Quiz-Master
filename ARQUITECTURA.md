# Quiz Master — Arquitectura

> Complementa [REQUISITS.md](REQUISITS.md). Aquest document descriu com s'implementa el que allà es defineix: arquitectura hexagonal (ports & adapters) amb quatre capes — **domain**, **application**, **infrastructure** i **view**.

## 1. Principis

- **Un sol mòdul Gradle** (KMP): les capes se separen per paquets dins de `commonMain`, no per mòduls Gradle. `androidMain`/`iosMain` només contenen detalls específics de plataforma (p. ex. com es construeix la base de dades Room a cada SO).
- **Regla de dependència** (la clau de tot plegat): les capes internes mai coneixen les externes.
  - `domain` no depèn de ningú (Kotlin pur, sense Room, sense Compose, sense Koin).
  - `application` depèn només de `domain`.
  - `infrastructure` depèn de `domain` (hi implementa els ports).
  - `view` depèn només de `application`.
  - **Mai al revés**, i `view` mai accedeix a `infrastructure` directament.
- **Injecció de dependències**: Koin (estàndard a KMP/Compose Multiplatform, sense generació de codi — a diferència de Dagger/Hilt, que no funcionen a iOS).
- **Noms de paquet en anglès** (`domain`, `application`, `infrastructure`, `view`), tot i que la resta de documentació del projecte és en català.

## 2. Estructura de paquets

```
com.quizmaster
├── domain
│   ├── model        → Quiz, Pregunta, Opcio, Categoria, Submode, Dificultat, RespostaJugador, ResultatPartida
│   └── port         → interfícies dels Finders (i qualsevol altre port que calgui)
├── application
│   └── usecase      → un cas d'ús per acció que demana una pantalla
├── infrastructure
│   ├── database     → Room: entities, DAOs, RoomDatabase (amb actual/expect a androidMain/iosMain)
│   ├── content      → DTOs del JSON de contingut (kotlinx.serialization) + lògica de seed/versioContingut
│   ├── mapper       → DTO → Entity (seed) i Entity → model de domini (lectura)
│   └── finder       → implementacions concretes dels ports Finder, amb Room per sota
├── view
│   ├── screen       → Composables per pantalla (una per ES-x.x)
│   ├── viewmodel    → un ViewModel per pantalla, exposa StateFlow<UiState>
│   └── state        → UiState de cada pantalla
└── di               → mòduls de Koin (bindings de Finders, UseCases, ViewModels)
```

## 3. Domain

Kotlin pur, cap dependència externa. Conté:

- **Models**: `Quiz`, `Pregunta`, `Opcio`, `Categoria` (enum: Països/Ciutats/CulturaGeneral/Trivial), `Submode` (enum: Normal/SeixantaSegons), `Dificultat` (enum: Facil/Mitjana/Dificil), `RespostaJugador`, `ResultatPartida`.
- **Ports**: una interfície **Finder** per cada consulta necessària (decisió presa: Finders com a classes independents, no mètodes genèrics d'un repositori). Exemples: `QuizzesPerCategoriaFinder`, `QuizAmbPreguntesFinder`.
- El mecanisme de seed (JSON → Room, REQUISITS.md §4.1) **no té port al domini**: és un detall intern d'`infrastructure` que no representa cap acció de negoci que `application`/`view` necessitin invocar directament.

## 4. Application

- Un **cas d'ús** per cada acció que una pantalla necessita fer, orquestrant un o més Finders del domini. Exemples: `ObtenirCategoriesUseCase`, `ObtenirSubmodesUseCase`, `IniciarPartidaUseCase`, `RespondrePreguntaUseCase`, `CalcularResultatUseCase`.
- Depèn només dels models i ports de `domain` — mai sap que Room existeix.

## 5. Infrastructure

- **database**: entities i DAOs de Room, i la classe `RoomDatabase` de Quiz Master. La construcció de la base de dades (que necessita `Context` a Android però no a iOS) es resol amb `expect`/`actual`.
- **content**: DTOs que reflecteixen l'esquema JSON (REQUISITS.md §8: `ContingutDto`, `QuizDto`, `PreguntaDto`), i la lògica que compara `versioContingut` i decideix si cal recarregar Room (REQUISITS.md §4.1).
- **mapper**: `QuizDto → QuizEntity` (en el seed) i `QuizEntity → Quiz` de domini (en les lectures).
- **finder**: implementacions concretes dels ports declarats a `domain.port`, per exemple `RoomQuizzesPerCategoriaFinder`, que usen els DAOs de Room per sota.

## 6. View

- Un **Composable de pantalla** per cada `ES-x.x` de REQUISITS.md §12 (`HomeScreen`, `SeleccioCategoriaScreen`, `SeleccioSubmodeScreen`, `PartidaScreen`, `ResultatScreen`).
- Un **ViewModel** per pantalla, que crida casos d'ús d'`application` i exposa un `StateFlow<UiState>` propi de la pantalla (no exposa models de domini directament a la UI).
- `view` mai importa res d'`infrastructure`.

## 7. Injecció de dependències (Koin)

- Mòduls de Koin que registren: implementacions de Finder (`infrastructure` → `domain.port`), casos d'ús (`application`), i ViewModels (`view`), amb `viewModel { }` de Koin per als ViewModels.
- La inicialització de Koin es fa des del punt d'entrada de cada plataforma (Application d'Android, entry point d'iOS).

## 8. Relació amb REQUISITS.md

| Concepte a REQUISITS.md | On viu a l'arquitectura |
|---|---|
| Esquema JSON (§8) | `infrastructure.content` (DTOs) |
| Flux JSON → Room (§4.1) | `infrastructure.content` + `infrastructure.database` |
| Categories i submodes (§5) | `domain.model` (`Categoria`, `Submode`) |
| Pantalles ES-x.x (§12) | `view.screen` + `view.viewmodel` (una parella per pantalla) |

## 9. Pendent de definir

- [ ] Noms exactes de cada Finder/UseCase — es concretaran a mesura que s'implementi cada pantalla.
- [ ] Estratègia de tests (unitaris a `domain`/`application`, instrumentats a `infrastructure`?) — no discutit encara.
- [ ] Com propaguen els errors els UseCases cap a `view` (excepcions, `Result<T>`, sealed classes d'error?).
