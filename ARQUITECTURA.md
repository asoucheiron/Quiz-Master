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
- **Reutilització de components de vista, obligatòria**: el sistema visual (REQUISITS.md §11) és consistent entre pantalles a propòsit (mateix estil "sticker" de botons, targetes, badges...), i el codi ho ha de reflectir. **Cap Composable amb estil propi es defineix dins d'una pantalla concreta** — si el fa servir (o és previsible que el faci servir) més d'una pantalla, es defineix una sola vegada a `view.shared` i totes les pantalles l'importen d'allà. No es duplica mai el mateix botó/targeta/badge reescrivint-ne els estils a cada `screen`.

## 2. Estructura de paquets

```
com.quizmaster
├── domain
│   ├── model        → Quiz, Pregunta, Opcio, Categoria, Submode, Dificultat, RespostaJugador, ResultatPartida
│   └── port         → interfícies dels *DataFinder (i qualsevol altre port que calgui)
├── application
│   └── usecase      → un cas d'ús (*UC) per acció que demana una pantalla
├── infrastructure
│   ├── database     → Room: entities, DAOs, RoomDatabase (amb actual/expect a androidMain/iosMain)
│   ├── content      → DTOs del JSON de contingut (kotlinx.serialization) + lògica de seed/versioContingut
│   ├── mapper       → DTO → Entity (seed) i Entity → model de domini (lectura)
│   └── finder       → implementacions concretes (*Finder) dels ports *DataFinder, amb Room per sota
├── view
│   ├── shared
│   │   ├── components → Composables reutilitzats entre pantalles (StickerButton, StickerCard, CategoryTile...)
│   │   └── theme       → colors, tipografia, mides i radis centralitzats (tokens de REQUISITS.md §11)
│   ├── screen       → Composables per pantalla (una per ES-x.x), compostos a partir de view.shared
│   ├── viewmodel    → un ViewModel per pantalla, exposa StateFlow<UiState>
│   └── state        → UiState de cada pantalla
└── di               → mòduls de Koin (bindings de Finders, UseCases, ViewModels)
```

## 3. Convenció de noms

Regla fixa per a tot el projecte, perquè el nom d'una classe digui per si sol de quin tipus és i què fa:

- **UseCase**: `<Verb + complement>` + sufix **`UC`**. El verb és sempre l'acció que fa. Exemples: `ObtenirCategoriesUC`, `ObtenirSubmodesUC`, `IniciarPartidaUC`, `RespondrePreguntaUC`, `CalcularResultatUC`.
- **Finder**: el nom sempre és **el de què es busca**, mai un nom genèric (`Finder` sol, `Repository`, etc.):
  - **Interfície (port, a `domain.port`)**: `<Entitat>` + sufix **`DataFinder`**. Exemples: `QuizDataFinder`, `CategoriaDataFinder`, `SubmodeDataFinder`.
  - **Implementació concreta (a `infrastructure.finder`)**: `<Entitat>` + sufix **`Finder`** (sense "Data"). Exemples: `QuizFinder`, `CategoriaFinder`. És la classe que implementa el `DataFinder` corresponent fent servir Room per sota.

Aquesta convenció **substitueix** els noms d'exemple usats en versions anteriors d'aquest document (p. ex. `QuizzesPerCategoriaFinder` com a interfície, o `ObtenirCategoriesUseCase` com a cas d'ús) — els noms correctes ara són `QuizDataFinder` (interfície) i `ObtenirCategoriesUC` (cas d'ús).

## 4. Domain

Kotlin pur, cap dependència externa. Conté:

- **Models**: `Quiz`, `Pregunta`, `Opcio`, `Categoria` (enum: Països/Ciutats/CulturaGeneral/Trivial), `Submode` (enum: Normal/SeixantaSegons), `Dificultat` (enum: Facil/Mitjana/Dificil), `RespostaJugador`, `ResultatPartida`.
- **Ports**: una interfície `*DataFinder` per cada consulta necessària (decisió presa: Finders com a classes independents, no mètodes genèrics d'un repositori — veure convenció de noms, secció 3). Exemples: `QuizDataFinder`, `CategoriaDataFinder`.
- El mecanisme de seed (JSON → Room, REQUISITS.md §4.1) **no té port al domini**: és un detall intern d'`infrastructure` que no representa cap acció de negoci que `application`/`view` necessitin invocar directament.

## 5. Application

- Un **cas d'ús** (`*UC`) per cada acció que una pantalla necessita fer, orquestrant un o més `*DataFinder` del domini. Exemples: `ObtenirCategoriesUC`, `ObtenirSubmodesUC`, `IniciarPartidaUC`, `RespondrePreguntaUC`, `CalcularResultatUC`.
- Depèn només dels models i ports de `domain` — mai sap que Room existeix.
- **Gestió d'errors: excepcions normals, sense embolicar cada `UC` en un `Result<T>` propi.** Aquesta app gairebé no té camins d'error "esperats" al domini/aplicació: el contingut ve empaquetat i validat en desenvolupament (REQUISITS.md §8), no hi ha xarxa, no hi ha input d'usuari a validar en temps real, i els `*Finder` gairebé sempre tindran èxit perquè les dades ja hi són. Modelar cada possible fallada amb sealed classes a totes les capes seria sobreenginyeria per a un risc que amb prou feines existeix — veure secció 7 per on sí que es modela l'error (a la frontera amb la UI).

## 6. Infrastructure

- **database**: entities i DAOs de Room, i la classe `RoomDatabase` de Quiz Master. La construcció de la base de dades (que necessita `Context` a Android però no a iOS) es resol amb `expect`/`actual`.
- **content**: DTOs que reflecteixen l'esquema JSON (REQUISITS.md §8: `ContingutDto`, `QuizDto`, `PreguntaDto`), i la lògica que compara `versioContingut` i decideix si cal recarregar Room (REQUISITS.md §4.1).
- **mapper**: `QuizDto → QuizEntity` (en el seed) i `QuizEntity → Quiz` de domini (en les lectures).
- **finder**: implementacions concretes (`*Finder`) dels ports `*DataFinder` declarats a `domain.port` — per exemple `QuizFinder` implementa `QuizDataFinder` usant els DAOs de Room per sota (veure convenció de noms, secció 3).

## 7. View

- Un **Composable de pantalla** per cada `ES-x.x` de REQUISITS.md §12 (`HomeScreen`, `SeleccioCategoriaScreen`, `SeleccioSubmodeScreen`, `PartidaScreen`, `ResultatScreen`).
- Un **ViewModel** per pantalla, que crida casos d'ús d'`application` i exposa un `StateFlow<UiState>` propi de la pantalla (no exposa models de domini directament a la UI).
- `view` mai importa res d'`infrastructure`.
- **Gestió d'errors: sealed class `UiState` (p. ex. `Loading` / `Content` / `Error`), només en aquesta frontera.** El `ViewModel` és qui fa `try/catch` (o `runCatching`) al voltant de la crida al `UC` dins la coroutine, i mapeja qualsevol excepció inesperada a `UiState.Error` — un únic punt de conversió, en lloc d'escampar sealed classes d'error per totes les capes (veure secció 5). Compose renderitza sempre un `when` exhaustiu sobre `UiState`, així que mai es pot oblidar de gestionar l'estat d'error a la UI.

### 7.1 `view.shared` — components reutilitzables (obligatori)

El disseny (REQUISITS.md §11) repeteix deliberadament els mateixos elements visuals a totes les pantalles (contorn de tinta + ombra sòlida, rotació lleugera, mateixa paleta). Cada un d'aquests elements es defineix **una sola vegada** a `view.shared.components` i les pantalles només el composen — mai es reimplementa l'estil dins d'un `screen`. A partir del que ja s'ha maquetat, com a mínim calen:

| Component | Ús a les pantalles |
|---|---|
| `StickerButton` | Jugar (ES-1.0), Tornar a jugar (ES-4.0), submodes del modal (ES-2.1) |
| `StickerCard` | Targeta de pregunta (ES-3.0), el modal centrat (ES-2.1) |
| `CategoryTile` | Cada categoria (ES-2.0) |
| `AnswerOption` | Opcions de resposta amb estats idle/correcte/incorrecte (ES-3.0) |
| `ProgressBadge` | Comptador "3/10" (ES-3.0) |
| `OrganicBackdrop` | Degradat + taques de fons (ES-1.0, ES-2.0, ES-4.0) |
| `CenteredModal` | Contenidor de modal amb scrim (ES-2.1) |

Els valors de la paleta (colors, radis, gruix de contorn, mida de l'ombra) viuen a `view.shared.theme`, no com a valors repetits a cada Composable — si canvia un color de marca, es canvia en un sol lloc.

## 8. Injecció de dependències (Koin)

- Mòduls de Koin que registren: implementacions de `*Finder` (`infrastructure` → `domain.port`), casos d'ús `*UC` (`application`), i ViewModels (`view`), amb `viewModel { }` de Koin per als ViewModels.
- La inicialització de Koin es fa des del punt d'entrada de cada plataforma (Application d'Android, entry point d'iOS).

## 9. Relació amb REQUISITS.md

| Concepte a REQUISITS.md | On viu a l'arquitectura |
|---|---|
| Esquema JSON (§8) | `infrastructure.content` (DTOs) |
| Flux JSON → Room (§4.1) | `infrastructure.content` + `infrastructure.database` |
| Categories i submodes (§5) | `domain.model` (`Categoria`, `Submode`) |
| Pantalles ES-x.x (§12) | `view.screen` + `view.viewmodel` (una parella per pantalla) |

## 10. Pendent de definir

- [ ] Estratègia de tests (unitaris a `domain`/`application`, instrumentats a `infrastructure`?) — no discutit encara.
