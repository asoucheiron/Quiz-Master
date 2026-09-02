# Quiz Master — Document de requeriments

> Estat: **esborrany en iteració**. Aquest document recull les decisions preses fins ara i les qüestions que encara queden obertes.

## 1. Visió general

Quiz Master és una aplicació multiplataforma (Android/iOS) per jugar a quizzes d'opció múltiple, per a un sol jugador. El contingut s'organitza en **categories temàtiques** (Països, Ciutats, Cultura general, Trivial), cadascuna amb els seus propis **submodes de joc** (p. ex. Normal, 60 seconds, País amagat, Banderes). No té servidor: tot el contingut i les dades es guarden localment al dispositiu. No requereix compte d'usuari ni login.

## 2. Plataformes i tecnologia

- **Kotlin Multiplatform (KMP)**: lògica de negoci i model de dades compartits.
- **Compose Multiplatform**: interfície d'usuari compartida entre plataformes.
- **Targets**: Android i iOS.
- Desenvolupament principal en Windows (Android Studio); la compilació i les proves del target iOS es faran en un Mac disponible per aquest propòsit (requisit d'Apple/Xcode, no bloqueja el desenvolupament).
- **Persistència**: base de dades local (a valorar SQLDelight, estàndard a l'ecosistema KMP — veure decisions pendents).
- Sense backend/servidor. Sense autenticació.

## 3. Cicle de vida de les dades

- Totes les dades (contingut de quizzes, historial, puntuacions) es guarden **només** al dispositiu.
- Si l'usuari desinstal·la l'app, es perden totes les dades. És un comportament acceptat i volgut (simplicitat per sobre de la persistència al núvol).
- No hi ha sincronització entre dispositius.

## 4. Gestió de contingut

### 4.1 Origen del contingut

- **El contingut ve integrat a l'app** (Països, Ciutats, Cultura general, Trivial). L'usuari obre l'app i juga directament — **no hi ha cap pas d'importació de cara a l'usuari**.
- L'esquema **JSON** (secció 8) no és una funcionalitat d'usuari, sinó el **format intern d'autoria/empaquetat del contingut**: és com es defineixen les preguntes dins el projecte (per qui desenvolupa l'app), no una acció que faci qui hi juga.
- En el primer arrencament (o en build), aquest contingut JSON inclòs a l'app es carrega a la base de dades local, i a partir d'aquí totes les consultes es fan contra la base de dades (no es torna a llegir el JSON en cada ús) — mateix raonament d'eficiència que ja havíem parlat.
- ✅ Decisió tancada: no hi ha importació de quizzes per part de l'usuari en la v1. El botó "Importa" que havíem tret de la pantalla Home (ES-1.0) es queda fora definitivament, no és un pendent.

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
- *Idea en exploració, no confirmada*: mode **Música** — implicaria contingut d'àudio, fora de l'abast actual si s'acaba afegint (veure decisions pendents).

### 5.2 Submodes per categoria

Cada categoria té els seus propis submodes — **no són compartits entre categories**. ⚠️ **Tot aquest apartat és un esborrany obert, no una decisió tancada** — tant Països com Ciutats tenen un disseny previ de referència de l'usuari, però el mateix usuari no n'està convençut, especialment del submode "amagat/amagada". No es dona per bo cap format fins que no es rediscuteixi.

**Països** (disseny previ de referència):

| Submode | Format de la pregunta | Notes |
|---|---|---|
| **Normal** | Enunciat de text simple + 4 opcions | Sense límit de temps |
| **60 seconds** | Igual que Normal | Temporitzador fix de **60 segons totals** per tot el quiz, amb barra de progrés visible — reutilitza el camp `tempsLimitSegons` de l'esquema amb valor 60 |
| **País amagat** ⚠️ | Llista numerada de pistes com a enunciat (p. ex. "1. Paella / 2. Siesta / 3. Barça") + 4 opcions amb noms de país | **En dubte** — l'usuari no acaba de veure clar aquest submode, pendent de repensar o descartar |
| **Banderes** | Una imatge (la bandera) com a enunciat + 4 opcions amb noms de país | Reutilitza el camp opcional `imatge` de l'esquema |

**Ciutats** (disseny previ de referència, encara menys definitiu que Països):

| Submode | Format de la pregunta | Notes |
|---|---|---|
| **Normal** | Enunciat de text simple + 4 opcions | Sense límit de temps |
| **60 seconds** | Igual que Normal | Temporitzador fix de 60 segons totals |
| **Ciutat amagada** ⚠️ | Llista numerada de pistes (p. ex. "1. Platja / 2. Muntanya / 3. Sagrada Família") + 4 opcions | **En dubte**, mateix motiu que "País amagat" |
| *(sense nom encara)* | "Quina ciutat està més a prop de [ciutat X]?" + 4 opcions | Format nou: pregunta de **proximitat geogràfica**, no una simple pregunta directa — no encaixa amb els altres submodes ni amb l'esquema JSON actual, cal pensar-hi |

El disseny de referència de Ciutats també mostra una **pantalla de pausa** i una **pantalla de resultat final amb "New Record" / puntuació més alta**. Això últim xoca amb la decisió de la secció 7 ("no es guarda historial ni estadístiques agregades") — guardar un rècord implica persistir com a mínim la millor puntuació per submode. Cal decidir si s'accepta aquesta petita excepció o es descarta el "rècord".

Els submodes de **Cultura general** i **Trivial** encara **no estan definits** — pendent d'iterar.

## 6. Tipus de preguntes

- **Opció múltiple** (una sola resposta correcta d'entre diverses opcions) — cobreix tots els submodes definits fins ara (Normal, 60 seconds, País amagat, Banderes), variant només el contingut de l'enunciat (text, pistes, o imatge).
- Altres tipus (veritat/fals, resposta oberta, multi-resposta) queden **fora d'abast per ara**; es podran afegir més endavant si cal.

## 7. Joc

- Mode **un sol jugador**.
- **Temporitzador opcional**: si el quiz defineix un temps límit, aquest és un **temps total per a tot el quiz** (no per pregunta individual). Si el camp no hi és, el quiz es juga sense límit de temps.
- No hi ha camp d'explicació/feedback per pregunta (només enunciat, opcions i resposta correcta) — decisió presa per simplicitat en aquesta primera iteració.
- En acabar un quiz, es mostra la **puntuació final** (encerts sobre el total).
- No es guarda historial de partides anteriors ni estadístiques agregades (només la partida actual) — decisió presa per simplicitat en aquesta primera iteració.

## 8. Model de dades — esquema JSON definitiu

```json
{
  "titol": "Capitals del món",
  "categoria": "Geografia",
  "dificultat": "mitjana",
  "tempsLimitSegons": 120,
  "preguntes": [
    {
      "enunciat": "Quina és la capital de França?",
      "imatge": "https://exemple.com/paris.jpg",
      "opcions": ["Madrid", "París", "Roma", "Berlín"],
      "respostaCorrecta": 1
    }
  ]
}
```

| Camp | Tipus | Obligatori | Descripció |
|---|---|---|---|
| `titol` | string | Sí | Nom del quiz, mostrat a la biblioteca. |
| `categoria` | string | No | Etiqueta per agrupar el quiz a la biblioteca. |
| `dificultat` | string (`facil` \| `mitjana` \| `dificil`) | No | Nivell del quiz sencer. |
| `tempsLimitSegons` | enter | No | Temps total en segons per completar el quiz. Absent = sense temporitzador. |
| `preguntes` | array | Sí (mín. 1) | Llista de preguntes del quiz. |
| `preguntes[].enunciat` | string | Sí | Text de la pregunta (pot ser multilínia, p. ex. pistes numerades). |
| `preguntes[].imatge` | string (URL o ruta) | No | Imatge opcional associada a la pregunta (p. ex. una bandera). |
| `preguntes[].opcions` | array de strings | Sí (mín. 2) | Opcions de resposta. |
| `preguntes[].respostaCorrecta` | enter | Sí | Índex (base 0) de l'opció correcta dins `opcions`. |

*(Pendent: com identificar/gestionar duplicats si es reprocessa contingut amb el mateix `titol`. Tampoc s'ha definit encara el comportament exacte de validació quan el JSON és invàlid o li falten camps obligatoris.)*

## 9. Fora d'abast (per ara)

- Servidor / backend / API externa.
- Login o comptes d'usuari.
- Sincronització al núvol o entre dispositius.
- Multijugador (local o en xarxa).
- Generació de quizzes amb IA.
- Tipus de pregunta diferents d'opció múltiple.
- Historial de partides i estadístiques agregades.

## 10. Decisions pendents / a iterar

- [ ] Repensar (o descartar) el submode "amagat/amagada" (pistes numerades) a Països i Ciutats — l'usuari no n'està convençut.
- [ ] Definir el format de pregunta "proximitat geogràfica" (Ciutats) i si encaixa amb l'esquema JSON actual.
- [ ] Decidir si es manté algun tipus de "rècord"/millor puntuació per submode (vist al disseny de Ciutats), tot i la decisió de no guardar historial ni estadístiques.
- [ ] Definir del tot els submodes de Països i Ciutats (ara mateix són esborranys oberts), i els de Cultura general i Trivial (encara no definits).
- [ ] Explorar la viabilitat d'un mode "Música" (contingut d'àudio).
- [ ] Validació i gestió d'errors quan el JSON intern és invàlid o li falten camps obligatoris.
- [ ] Llibreria de persistència local concreta (SQLDelight vs. altres alternatives KMP).
- [ ] Idioma de la interfície (només català? multi-idioma?).
- [ ] Comportament en cas de resposta incorrecta (mostrar la correcta? feedback immediat o al final de la pregunta?).
- [ ] Retocs tècnics pendents al logo (contrast del text, versió en canvas quadrat per iOS) — disseny original acceptat, veure conversa.
- [ ] Contingut i abast exacte dels **Ajustos** (pantalla afegida a ES-1.0 Home — idioma? tema clar/fosc? res més per ara?).

## 11. Identitat visual

- Logo ja definit (roda de colors en forma d'interrogant + text "quiz Master"), acceptat com a base de la marca.
- Paleta de colors principal: arc de Sant Martí / multicolor, estil juganer.
- Fons de pantalla clar: to **menta** (`#E1F5EE` aprox.) triat per la pantalla d'inici.
- Pendent aplicar-ho a la resta de la UI quan es dissenyin les pantalles.

## 12. Especificació de pantalles (ES-x.x)

Cada pantalla es descriu amb un codi `ES-x.x`, una descripció breu del seu propòsit, els elements que conté i cap a on navega. L'objectiu és tancar aquí el contingut i el comportament de cada pantalla abans de passar-les a Canva per al disseny visual final.

### ES-1.0 — Pantalla Home

**Propòsit**: punt d'entrada de l'app. Ha de portar l'usuari a jugar com més ràpid millor, sense soroll.

**Elements**:
- Logo / icona de Quiz Master (roda de colors + interrogant).
- Wordmark "Quiz Master".
- Botó primari **Jugar**.
- Botó secundari d'**Ajustos** (icona d'engranatge, cantonada superior dreta) — contingut de la pantalla d'ajustos encara pendent de definir.
- Fons de color menta clar (`#E1F5EE` aprox.), sense elements addicionals.

**Navegació**:
- `Jugar` → **ES-2.0** (selecció de categoria).
- `Ajustos` → pantalla d'ajustos (encara no especificada).

**Estat**: ✅ tancat a nivell de contingut i maquetat com a artifact (veure conversa).

---

### ES-2.0 — Selecció de categoria

**Propòsit**: en prémer "Jugar" des de la Home, es mostra un contenidor amb les categories disponibles perquè l'usuari triï amb què vol jugar.

**Elements**:
- Contenidor (pantalla completa o modal — pendent de decidir) amb les categories:
  - Països
  - Ciutats
  - Cultura general
  - Trivial (barrejat)
- *Idea en exploració*: possible mode "Música" — no confirmat, implicaria contingut d'àudio fora de l'abast actual.

**Navegació**:
- Cada categoria → la seva pantalla de selecció de submode (p. ex. **ES-2.1** per Països).

**Estat**: 🟡 definides les categories, pendent decidir si és pantalla completa o modal, i el disseny visual.

---

### ES-2.1 — Selecció de submode: Països

**Propòsit**: dins la categoria Països, triar amb quin submode de joc es vol jugar.

**Elements** (basat en el disseny previ de l'usuari):
- Normal
- 60 seconds
- País amagat
- Banderes

**Navegació**:
- Cada submode → pantalla de partida (**ES-3.x**, pendent d'especificar) amb el format de pregunta corresponent (veure secció 5.2).

**Estat**: 🟡 contingut/funció definits (a partir del disseny propi existent de l'usuari), pendent portar-ho al sistema visual de Quiz Master (tipografia Fredoka/Plus Jakarta Sans, colors propis) en lloc del disseny de referència.

---

### Pendents d'especificar

- [ ] ES-2.x — Selecció de submode per Ciutats, Cultura general i Trivial (depèn de definir primer els seus submodes a la secció 5.2).
- [ ] ES-3.x — Pantalla de partida (una per format de pregunta: Normal/text, pistes numerades, imatge/bandera).
- [ ] ES-4.x — Pantalla de resultat final (puntuació).
- [ ] ES-5.x — Pantalla d'Ajustos.
