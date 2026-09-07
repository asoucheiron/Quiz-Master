# Quiz Master — Document de requeriments

> Estat: **esborrany en iteració**. Aquest document recull les decisions preses fins ara i les qüestions que encara queden obertes.

## 1. Visió general

Quiz Master és una aplicació multiplataforma (Android/iOS) per jugar a quizzes d'opció múltiple, per a un sol jugador. El contingut s'organitza en **categories temàtiques** (Països, Ciutats, Cultura general, Trivial). Per aquesta primera versió, el joc té només dos **submodes**, iguals per a totes les categories: **Normal** i **60 seconds**. No té servidor: tot el contingut i les dades es guarden localment al dispositiu. No requereix compte d'usuari ni login.

> Aquesta v1 busca deliberadament ser senzilla: submodes més elaborats (pistes, imatges, rècords, etc.) es deixen per a versions posteriors un cop l'app ja sigui funcional — veure secció 13 (Backlog).

## 2. Plataformes i tecnologia

- **Kotlin Multiplatform (KMP)**: lògica de negoci i model de dades compartits.
- **Compose Multiplatform**: interfície d'usuari compartida entre plataformes.
- **Targets**: Android i iOS.
- Desenvolupament principal en Windows (Android Studio); la compilació i les proves del target iOS es faran en un Mac disponible per aquest propòsit (requisit d'Apple/Xcode, no bloqueja el desenvolupament).
- **Persistència**: base de dades local amb **Room** (suport multiplataforma des de la versió 2.7, Android + iOS + Desktop, via `BundledSQLiteDriver`). Decisió tancada — veure flux complet a la secció 4.1.
- Sense backend/servidor. Sense autenticació.

## 3. Cicle de vida de les dades

- Totes les dades (contingut de quizzes, historial, puntuacions) es guarden **només** al dispositiu.
- Si l'usuari desinstal·la l'app, es perden totes les dades. És un comportament acceptat i volgut (simplicitat per sobre de la persistència al núvol).
- No hi ha sincronització entre dispositius.

## 4. Gestió de contingut

### 4.1 Origen del contingut

- **El contingut ve integrat a l'app** (Països, Ciutats, Cultura general, Trivial). L'usuari obre l'app i juga directament — **no hi ha cap pas d'importació de cara a l'usuari**.
- L'esquema **JSON** (secció 8) no és una funcionalitat d'usuari, sinó el **format intern d'autoria/empaquetat del contingut**: és com es defineixen les preguntes dins el projecte (per qui desenvolupa l'app), no una acció que faci qui hi juga.
- ✅ Decisió tancada: no hi ha importació de quizzes per part de l'usuari en la v1. El botó "Importa" que havíem tret de la pantalla Home (ES-1.0) es queda fora definitivament, no és un pendent.
- **Idioma**: per aquesta v1, tot l'app és **en castellà** — tant la interfície (botons, menús, textos fixos) com el contingut de les preguntes (títols, enunciats, opcions). Es descarta el multi-idioma a la interfície per unificar-ho amb l'idioma del contingut i evitar muntar un sistema de traduccions que ara mateix no aporta res (el contingut seguiria sent només en castellà). Veure secció 13 (Backlog) si en el futur es vol ampliar a més idiomes (interfície i/o contingut).

**Flux de càrrega del contingut (JSON → Room)**:

1. El contingut es defineix en un **fitxer JSON de contingut** empaquetat amb l'app, amb un camp `versioContingut` a nivell global i la llista de quizzes (veure estructura a la secció 8).
2. En cada arrencada de l'app, es compara `versioContingut` del fitxer empaquetat amb la versió guardada a la base de dades (una petita taula/registre de metadades a Room).
3. Si la versió del fitxer és **més nova**, es buida i es torna a carregar el contingut de quizzes a Room amb el nou JSON, i s'actualitza la versió guardada. Si és igual, no es toca res.
4. **A partir d'aquí, tota lectura de l'app (biblioteca, partida, etc.) es fa sempre via Room** — el JSON no es torna a llegir en temps d'execució normal, només en aquesta comprovació d'arrencada.

Amb aquest mecanisme, afegir contingut nou o corregir preguntes és tan senzill com pujar la `versioContingut` del JSON en una actualització de l'app; la migració de dades locals és automàtica en obrir-la.

### 4.2 Organització / biblioteca

- El contingut es llista per **categoria** (Països, Ciutats, Cultura general, Trivial).
- La **dificultat s'aplica només a nivell de quiz sencer** (no per pregunta individual).

## 5. Estructura del joc: categories i submodes

Flux: Pantalla d'inici → **Jugar** → contenidor de categories → **submode** de la categoria triada → partida.

### 5.1 Categories (nivell 1, en prémer "Jugar")

- **Països**
- **Ciutats**
- **Cultura general**
- **Trivial** (barrejat entre categories)

### 5.2 Submodes (v1)

Per aquesta primera versió, **tots els submodes són els mateixos per a totes les categories** — no hi ha variació entre Països, Ciutats, Cultura general o Trivial de moment:

| Submode | Format de la pregunta | Notes |
|---|---|---|
| **Normal** | Enunciat de text + 4 opcions | Sense límit de temps |
| **60 seconds** | Igual que Normal | Temporitzador fix de **60 segons totals** per tot el quiz — reutilitza el camp `tempsLimitSegons` de l'esquema amb valor 60 |

Formats més elaborats que s'havien explorat (pistes numerades tipus "país/ciutat amagat", banderes com a enunciat, proximitat geogràfica, rècords) queden **descartats per aquesta v1** i es reprenen en una versió posterior — veure secció 13 (Backlog).

Cada submode del joc tindrà assignat un color diferent, en aquest cas:

Països: Verd(a decidir tonalitats)
Ciutats: Blau(a definir tonalitats)
Cultura general: Lila(a definir tonalitats)
Trivial: Degradat multicolor

## 6. Tipus de preguntes

- **Opció múltiple** (una sola resposta correcta d'entre diverses opcions) — únic format necessari per als submodes Normal i 60 seconds.
- Altres tipus (veritat/fals, resposta oberta, multi-resposta) queden **fora d'abast per ara**; es podran afegir més endavant si cal.

## 7. Joc

- Mode **un sol jugador**.
- **Temporitzador opcional**: si el quiz defineix un temps límit, aquest és un **temps total per a tot el quiz** (no per pregunta individual). Si el camp no hi és, el quiz es juga sense límit de temps.
- **Feedback immediat en respondre**: en tocar una opció, es marca a l'instant en verd (correcta) o vermell (incorrecta) abans de passar a la pregunta següent — igual que al disseny de referència de Ciutats. No hi ha cap text d'explicació addicional (només el color), decisió presa per simplicitat.
- En acabar un quiz, es mostra la **puntuació final** (encerts sobre el total).
- No es guarda historial de partides anteriors ni estadístiques agregades (només la partida actual) — decisió presa per simplicitat en aquesta primera iteració.

## 8. Model de dades — esquema JSON definitiu

El fitxer de contingut té un nivell superior amb la versió global i la llista de quizzes:

```json
{
  "versioContingut": 1,
  "quizzes": [
    {
      "titol": "Capitales del mundo",
      "categoria": "Países",
      "dificultat": "mitjana",
      "tempsLimitSegons": 120,
      "preguntes": [
        {
          "enunciat": "¿Cuál es la capital de Francia?",
          "imatge": "https://exemple.com/paris.jpg",
          "opcions": ["Madrid", "París", "Roma", "Berlín"],
          "respostaCorrecta": 1
        }
      ]
    }
  ]
}
```

| Camp | Tipus | Obligatori | Descripció |
|---|---|---|---|
| `versioContingut` | enter | Sí | Versió global del contingut. En pujar-la, l'app recarrega tots els quizzes a Room (veure secció 4.1). |
| `quizzes` | array | Sí | Llista de quizzes. |
| `quizzes[].titol` | string | Sí | Nom del quiz, mostrat a la biblioteca. |
| `quizzes[].categoria` | string | No | Etiqueta per agrupar el quiz a la biblioteca. |
| `quizzes[].dificultat` | string (`facil` \| `mitjana` \| `dificil`) | No | Nivell del quiz sencer. |
| `quizzes[].tempsLimitSegons` | enter | No | Temps total en segons per completar el quiz. Absent = sense temporitzador. |
| `quizzes[].preguntes` | array | Sí (mín. 1) | Llista de preguntes del quiz. |
| `quizzes[].preguntes[].enunciat` | string | Sí | Text de la pregunta (pot ser multilínia, p. ex. pistes numerades). |
| `quizzes[].preguntes[].imatge` | string (URL o ruta) | No | Imatge opcional associada a la pregunta (p. ex. una bandera). |
| `quizzes[].preguntes[].opcions` | array de strings | Sí (mín. 2) | Opcions de resposta. |
| `quizzes[].preguntes[].respostaCorrecta` | enter | Sí | Índex (base 0) de l'opció correcta dins `opcions`. |

**Duplicats**: gràcies al mecanisme de wipe-and-reload per `versioContingut` (secció 4.1), no hi ha acumulació de duplicats entre actualitzacions. L'única regla necessària és a nivell d'autoria: `titol` ha de ser **únic dins de tot el fitxer de contingut**; dos quizzes amb el mateix `titol` es considera contingut invàlid.

**Validació**: el contingut és autoria pròpia del projecte (mai ve de l'usuari ni d'internet), així que no cal un sistema de validació de cara al públic:
- Es verifica **en desenvolupament** (un test que carrega el fitxer i comprova l'esquema: camps obligatoris, `titol` únics, `respostaCorrecta` dins de rang de `opcions`, etc.) abans de publicar cap versió de l'app.
- Xarxa de seguretat en temps d'execució: si el JSON empaquetat no es pot parsejar o no compleix l'esquema, **l'app no toca el contingut ja carregat a Room** (es queda amb l'últim contingut vàlid) i registra l'error — mai es deixa la biblioteca buida per un fitxer trencat.
- Comportament tot o res per fitxer: si hi ha un error, no es carrega res d'aquell fitxer (sense càrrega parcial saltant els quizzes trencats) — es manté simple per la v1.

## 9. Fora d'abast (per ara)

- Servidor / backend / API externa.
- Login o comptes d'usuari.
- Sincronització al núvol o entre dispositius.
- Multijugador (local o en xarxa).
- Generació de quizzes amb IA.
- Tipus de pregunta diferents d'opció múltiple.
- Historial de partides i estadístiques agregades.

## 10. Decisions pendents / a iterar

*(De moment no queda cap decisió de producte pendent — tot el que hi havia s'ha anat resolent a mesura que avançava el document.)*

## 11. Identitat visual

✅ **Sistema visual pràcticament definitiu** (maquetat i iterat com a artifact — veure conversa). Estil "sticker": contorn de tinta fosca i ombra sòlida desplaçada (sense difuminar) a totes les targetes, botons i el propi marc de pantalla, en lloc del típic flat + ombra suau. Lleugeres rotacions alternades a targetes i botons per un toc més manual/juganer.

**Colors**:
- Fons de totes les pantalles neutres (Home, selecció de categoria, resultat): **paper crema** `#FBF3E3`, amb un degradat suau de fons barrejant els colors de marca (radial-gradients molt difuminats) més taques orgàniques addicionals — substitueix la idea inicial de fons menta pla.
- Tinta / contorn / text principal: `#211F3D`.
- **Accent** (botó Jugar, Tornar a jugar, submode destacat, anella de puntuació): **coral** `#FF6B4D`.
- **Països**: verd `#2FB380`. **Ciutats**: turquesa `#1CA3A3` (no blau — descartat per massa "SaaS genèric"). **Cultura general**: rosa `#E0447C`. **Trivial**: degradat multicolor reutilitzant els colors del logo.
- Correcte: verd `#1E8F4E`. Incorrecte: vermell `#E63946`.
- Logo ja definit (roda de colors en forma d'interrogant + text "quiz Master"), acceptat com a base de la marca — no necessita cap redisseny; és la font dels colors de marca reutilitzats arreu.

**Tipografia**: Fredoka (títols, botons, wordmark) + Plus Jakarta Sans (text de cos).

**Nota de producció** (no és una decisió, és feina d'exportació quan toqui): en generar els fitxers d'icona de l'app per Android/iOS, cal preparar el logo en un **canvas quadrat** amb el disseny sagnant fins als marges (iOS aplica la seva pròpia màscara arrodonida sobre un fitxer quadrat, no sobre un cercle ja retallat).

## 12. Especificació de pantalles (ES-x.x)

Cada pantalla es descriu amb un codi `ES-x.x`, una descripció breu del seu propòsit, els elements que conté i cap a on navega. ✅ **El disseny visual de la v1 es considera pràcticament definitiu** (estil sticker + paleta de la secció 11, maquetat i iterat com a artifact — veure conversa); canviar-lo després d'implementar-lo seria costós, per això s'ha iterat a fons abans de tocar codi.

### ES-1.0 — Pantalla Home

**Propòsit**: punt d'entrada de l'app. Ha de portar l'usuari a jugar com més ràpid millor, sense soroll.

**Elements**:
- Logo / icona de Quiz Master (roda de colors + interrogant).
- Wordmark "Quiz Master".
- Botó primari **Jugar**.
- Fons de paper crema amb degradat/taques de marca (secció 11), sense elements addicionals.

**Navegació**:
- `Jugar` → **ES-2.0** (selecció de categoria).

**Estat**: ✅ tancat, contingut i disseny visual definitius (sense Ajustos, veure conversa).

---

### ES-2.0 — Selecció de categoria

**Propòsit**: en prémer "Jugar" des de la Home, es navega a una pantalla pròpia amb les categories disponibles perquè l'usuari triï amb què vol jugar.

**Elements**:
- Pantalla completa (no modal) amb les 4 categories com a targetes apilades, cadascuna amb el seu color (secció 5.2 i 11):
  - Països (verd)
  - Ciutats (turquesa)
  - Cultura general (rosa)
  - Trivial (degradat multicolor)
- Títol "Elige categoría" centrat i gran, en una fila pròpia sota la fletxa d'enrere (no a la mateixa alçada).

**Navegació**:
- Tocar una categoria → obre **ES-2.1** com a modal superposat (no navega a una pantalla nova pròpia).
- Enrere → torna a **ES-1.0** (Home).

**Estat**: ✅ tancat, contingut i disseny visual definitius.

---

### ES-2.1 — Selecció de submode (modal)

**Propòsit**: un cop triada la categoria, un **modal centrat** (targeta crema amb fons fosc semitransparent al darrere que enfosqueix tot ES-2.0) per triar el submode — **no és un bottom-sheet que puja des de baix**, apareix centrat verticalment i horitzontalment a la pantalla. Com que a la v1 els submodes són els mateixos per a totes les categories (secció 5.2), és el mateix modal reutilitzat per Països, Ciutats, Cultura general i Trivial.

**Elements**:
- Normal
- 60 seconds

**Navegació**:
- Tocar un submode → tanca el modal i comença **ES-3.0** (partida).
- Tocar fora / enrere → tanca el modal, es queda a ES-2.0.

**Estat**: ✅ tancat, contingut i disseny visual definitius.

---

### ES-3.0 — Pantalla de partida

**Propòsit**: mostrar les preguntes d'una en una i recollir la resposta de l'usuari.

**Elements**:
- Comptador de progrés (p. ex. "3/10").
- **Barra de temps**: només visible si el submode és "60 seconds"; mostra el temps total restant (secció 5.2/7).
- Enunciat de la pregunta.
- 4 opcions de resposta com a botons.
- **Feedback immediat**: en tocar una opció, aquesta es marca a l'instant en verd (si és correcta) o vermell (si és incorrecta); si l'usuari s'equivoca, l'opció correcta també es marca en verd al mateix moment, perquè es vegi quina era (com al disseny de referència de Ciutats).
- Després d'una **breu pausa automàtica** (~1 segon) es passa sola a la pregunta següent — sense botó "Següent".

**Navegació**:
- En respondre l'última pregunta (i mostrar el seu feedback) → **ES-4.0** (resultat).

**Estat**: 🟡 contingut definit; ja maquetada amb el sistema visual definitiu (secció 11), però pendent una última revisió explícita (les últimes iteracions de color/estil s'han validat sobretot a ES-1.0/2.0/2.1).

---

### ES-4.0 — Pantalla de resultat final

**Propòsit**: mostrar com ha anat la partida i oferir continuar jugant.

**Elements**:
- Puntuació final (encerts sobre el total).
- Botó **Tornar a jugar** (repeteix el mateix categoria + submode).
- Botó **Sortir a l'inici**.
- Sense "New Record" ni cap dada d'historial (backlog, secció 13).

**Navegació**:
- `Tornar a jugar` → torna a **ES-3.0** amb la mateixa categoria i submode.
- `Sortir a l'inici` → **ES-1.0** (Home).

**Estat**: 🟡 contingut definit; ja maquetada amb el sistema visual definitiu (secció 11), però pendent una última revisió explícita (les últimes iteracions de color/estil s'han validat sobretot a ES-1.0/2.0/2.1).

---

Totes les pantalles de la v1 (ES-1.0 a ES-4.0) tenen el contingut i comportament definits. Només queda pendent el disseny visual final de cadascuna (Canva).

## 13. Backlog (versions futures)

Funcionalitats explorades però **descartades deliberadament d'aquesta primera versió**, per mantenir-la senzilla i evolucionar l'app un cop ja sigui funcional:

- Submode "amagat/amagada" (pistes numerades) per Països i Ciutats.
- Submode "Banderes" (imatge com a enunciat).
- Format de pregunta de "proximitat geogràfica" (p. ex. "quina ciutat està més a prop de X?").
- Submodes diferenciats per categoria (ara mateix Normal i 60 seconds són iguals per a totes).
- Mode "Música" (contingut d'àudio) com a nova categoria.
- Pantalla de pausa durant la partida.
- Sistema de "rècord" / millor puntuació per submode — reconsiderar la decisió actual de no guardar historial ni estadístiques (secció 7).
- Importació de quizzes per part de l'usuari (descartada per la v1, secció 4.1) — es podria revisitar si en el futur es vol permetre contingut extern.
- Multi-idioma (interfície i/o contingut) — per la v1 tot és només en castellà (secció 4.1).
- **Pantalla d'Ajustos** — descartada del tot per la v1 (sense idioma per triar, sense sentit un tema fosc amb la identitat visual actual de colors saturats, i cap altra opció real a configurar). Es revisita quan hi hagi alguna cosa concreta a oferir-hi (p. ex. so/música si s'afegeix, o un replantejament del sistema visual que faci sentir un tema fosc).

### 13.1 Sistema de progressió / gamificació (v2)

Proposta completa treballada per fer que el joc "enganxi" més enllà de la puntuació d'una partida — **conscientment fora de la v1** perquè reobre la decisió de no guardar historial/estadístiques (secció 7) i multiplica molt l'abast a dissenyar i construir abans de tenir res jugable. Es documenta sencera aquí per no perdre-la de cara a la v2.

**⭐ XP — progressió permanent**
- Es guanya XP jugant: completar un quiz, respostes correctes, bonus per 100%, bonus per reptes.
- No es gasta; serveix per pujar de nivell (p. ex. "Nivell 7 → 820/1.000 XP").

**🪙 Monedes — recurs gastable**
- S'aconsegueixen jugant, es gasten en pistes. Connecta la recompensa directament amb el gameplay (no és cosmètic).

**💡 Pistes** (ajuden, no donen la resposta directa) — cost en monedes, exemples a validar:

| Pista | Efecte | Cost (exemple) |
|---|---|---|
| 50/50 | Elimina dues opcions incorrectes | 50 🪙 |
| Pista | Dona informació relacionada amb la resposta | 75 🪙 |
| +10 segons | Només té sentit al submode 60 seconds | 100 🪙 |
| Segona oportunitat | Permet tornar a intentar la pregunta | 150 🪙 |

⚠️ La pista "Pista" (informació relacionada) necessitaria un **camp nou al contingut de cada pregunta** al JSON (secció 8) — mateix cost d'autoria que vam descartar amb el multi-idioma del contingut, multiplicat per cada pregunta.

**🔓 Nivells amb desbloquejos**: cada nivell podria desbloquejar una pista nova (p. ex. Nivell 1 → 50/50, Nivell 3 → Pista, Nivell 5 → +10 segons, Nivell 8 → Segona oportunitat), de manera que hi ha dues progressions creuades: nivell (què pots fer servir) i monedes (quantes vegades t'ho pots permetre).

**🎯 Reptes diaris**: 3 reptes senzills que es renoven cada dia (p. ex. "Completa 1 quiz", "Aconsegueix 7/10", "Completa un 60 seconds"), cadascun amb recompensa d'XP.

**🔥 Ratxa**: dies consecutius jugant, sense ser excessivament castigador si un dia no es juga.

**🏅 Assoliments**: col·lecció d'objectius secundaris (primera partida, 10 preguntes correctes, primer 100%, 10 quizzes completats, 7 dies de ratxa, 100 preguntes correctes, etc.).

**Canvis d'UX previstos**:
- Home (ES-1.0): afegiria nivell + barra d'XP, ratxa, i progrés dels reptes del dia.
- Resultat (ES-4.0): deixaria de ser només "puntuació + tornar a jugar" per mostrar XP guanyat, monedes guanyades, progrés de nivell i reptes completats — el resultat esdevé la recompensa en si mateixa.

**Model de dades nou necessari** (Room, tot local, sense servidor ni compte — això no canvia):
```
UserProgress
 ├── totalXp
 ├── level
 ├── coins
 ├── currentStreak
 ├── lastPlayedDate
 ├── totalQuizzes
 ├── totalCorrectAnswers
 └── unlockedRewards

CategoryProgress
 ├── category
 ├── xp
 ├── quizzesCompleted
 └── correctAnswers
```
No caldria guardar l'historial complet de cada partida, només l'estat agregat necessari per a la progressió.

**Explícitament descartat, també per a la v2** (per no convertir-ho en un monstre): botiga complexa, avatars, battle pass, energia/vides, loot boxes, leaderboards, social/multijugador, backend, moneda real.
