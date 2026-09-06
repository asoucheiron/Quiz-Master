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

- Logo ja definit (roda de colors en forma d'interrogant + text "quiz Master"), acceptat com a base de la marca — no necessita cap redisseny.
- Paleta de colors principal: arc de Sant Martí / multicolor, estil juganer.
- Fons de pantalla clar: to **menta** (`#E1F5EE` aprox.) triat per la pantalla d'inici.
- Pendent aplicar-ho a la resta de la UI quan es dissenyin les pantalles.
- **Nota de producció** (no és una decisió, és feina d'exportació quan toqui): en generar els fitxers d'icona de l'app per Android/iOS, cal preparar el logo en un **canvas quadrat** amb el disseny sagnant fins als marges (iOS aplica la seva pròpia màscara arrodonida sobre un fitxer quadrat, no sobre un cercle ja retallat).

## 12. Especificació de pantalles (ES-x.x)

Cada pantalla es descriu amb un codi `ES-x.x`, una descripció breu del seu propòsit, els elements que conté i cap a on navega. L'objectiu és tancar aquí el contingut i el comportament de cada pantalla abans de passar-les a Canva per al disseny visual final.

### ES-1.0 — Pantalla Home

**Propòsit**: punt d'entrada de l'app. Ha de portar l'usuari a jugar com més ràpid millor, sense soroll.

**Elements**:
- Logo / icona de Quiz Master (roda de colors + interrogant).
- Wordmark "Quiz Master".
- Botó primari **Jugar**.
- Fons de color menta clar (`#E1F5EE` aprox.), sense elements addicionals.

**Navegació**:
- `Jugar` → **ES-2.0** (selecció de categoria).

**Estat**: ✅ tancat a nivell de contingut i maquetat com a artifact (sense Ajustos, veure conversa).

---

### ES-2.0 — Selecció de categoria

**Propòsit**: en prémer "Jugar" des de la Home, es mostra un contenidor amb les categories disponibles perquè l'usuari triï amb què vol jugar.

**Elements**:
- Contenidor (pantalla completa o modal — pendent de decidir) amb les categories:
  - Països
  - Ciutats
  - Cultura general
  - Trivial (barrejat)

**Navegació**:
- Cada categoria → la seva pantalla de selecció de submode (p. ex. **ES-2.1** per Països).

**Estat**: 🟡 definides les categories, pendent decidir si és pantalla completa o modal, i el disseny visual.

---

### ES-2.1 — Selecció de submode

**Propòsit**: dins de qualsevol categoria, triar amb quin submode de joc es vol jugar. Com que a la v1 els submodes són els mateixos per a totes les categories (secció 5.2), aquesta pantalla és **el mateix patró reutilitzat per Països, Ciutats, Cultura general i Trivial** — no calen quatre disenys diferents.

**Elements**:
- Normal
- 60 seconds

**Navegació**:
- Cada submode → pantalla de partida (**ES-3.x**, pendent d'especificar).

**Estat**: 🟡 contingut/funció definits, pendent el disseny visual.

---

### Pendents d'especificar

- [ ] ES-3.x — Pantalla de partida (pregunta de text + 4 opcions).
- [ ] ES-4.x — Pantalla de resultat final (puntuació).

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
