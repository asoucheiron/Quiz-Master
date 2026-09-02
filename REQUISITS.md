# Quiz Master — Document de requeriments

> Estat: **esborrany en iteració**. Aquest document recull les decisions preses fins ara i les qüestions que encara queden obertes.

## 1. Visió general

Quiz Master és una aplicació multiplataforma (Android/iOS) per jugar a quizzes d'opció múltiple, per a un sol jugador. No té servidor: tot el contingut i les dades es guarden localment al dispositiu. No requereix compte d'usuari ni login.

## 2. Plataformes i tecnologia

- **Kotlin Multiplatform (KMP)**: lògica de negoci i model de dades compartits.
- **Compose Multiplatform**: interfície d'usuari compartida entre plataformes.
- **Targets**: Android i iOS.
- Desenvolupament principal en Windows (Android Studio); la compilació i les proves del target iOS es faran en un Mac disponible per aquest propòsit (requisit d'Apple/Xcode, no bloqueja el desenvolupament).
- **Persistència**: base de dades local (a valorar SQLDelight, estàndard a l'ecosistema KMP — veure decisions pendents).
- Sense backend/servidor. Sense autenticació.

## 3. Cicle de vida de les dades

- Totes les dades (quizzes importats, historial, puntuacions) es guarden **només** al dispositiu.
- Si l'usuari desinstal·la l'app, es perden totes les dades. És un comportament acceptat i volgut (simplicitat per sobre de la persistència al núvol).
- No hi ha sincronització entre dispositius.

## 4. Gestió de quizzes

### 4.1 Importació

- Els quizzes s'incorporen a l'app mitjançant **importació de fitxers en format JSON**.
- Cal definir l'esquema JSON exacte (veure secció 7, proposta inicial).
- Pendent de decidir: com arriba el fitxer a l'app (compartir fitxer des del sistema, selector de fitxers, escanejar QR, etc. — veure decisions pendents).

### 4.2 Organització / biblioteca

- Els quizzes importats es llisten en una biblioteca dins l'app.
- Es poden agrupar per **categoria** (p. ex. "Història", "Ciència").
- La **dificultat s'aplica només a nivell de quiz sencer** (no per pregunta individual).

## 5. Tipus de preguntes

- **Opció múltiple** (una sola resposta correcta d'entre diverses opcions).
- Cada pregunta pot incloure opcionalment una **imatge** (URL o ruta).
- Altres tipus (veritat/fals, resposta oberta, multi-resposta) queden **fora d'abast per ara**; es podran afegir més endavant si cal.

## 6. Joc

- Mode **un sol jugador**.
- **Temporitzador opcional**: si el quiz defineix un temps límit, aquest és un **temps total per a tot el quiz** (no per pregunta individual). Si el camp no hi és, el quiz es juga sense límit de temps.
- No hi ha camp d'explicació/feedback per pregunta (només enunciat, opcions i resposta correcta) — decisió presa per simplicitat en aquesta primera iteració.
- En acabar un quiz, es mostra la **puntuació final** (encerts sobre el total).
- No es guarda historial de partides anteriors ni estadístiques agregades (només la partida actual) — decisió presa per simplicitat en aquesta primera iteració.

## 7. Model de dades — esquema JSON definitiu

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
| `preguntes[].enunciat` | string | Sí | Text de la pregunta. |
| `preguntes[].imatge` | string (URL o ruta) | No | Imatge opcional associada a la pregunta. |
| `preguntes[].opcions` | array de strings | Sí (mín. 2) | Opcions de resposta. |
| `preguntes[].respostaCorrecta` | enter | Sí | Índex (base 0) de l'opció correcta dins `opcions`. |

*(Pendent: com identificar/gestionar quizzes duplicats en re-importar un fitxer amb el mateix `titol` — veure decisions pendents. Tampoc s'ha definit encara el comportament exacte de validació quan el JSON és invàlid o li falten camps obligatoris.)*

## 8. Fora d'abast (per ara)

- Servidor / backend / API externa.
- Login o comptes d'usuari.
- Sincronització al núvol o entre dispositius.
- Multijugador (local o en xarxa).
- Generació de quizzes amb IA.
- Tipus de pregunta diferents d'opció múltiple.
- Historial de partides i estadístiques agregades.

## 9. Decisions pendents / a iterar

- [ ] Validació i gestió d'errors quan el fitxer JSON és invàlid o li falten camps obligatoris.
- [ ] Com s'importa el fitxer a l'app (selector de fitxers del sistema, compartir des d'una altra app, etc.) a cada plataforma (Android/iOS).
- [ ] Llibreria de persistència local concreta (SQLDelight vs. altres alternatives KMP).
- [ ] Idioma de la interfície (només català? multi-idioma?).
- [ ] Gestió de duplicats o actualització de quizzes ja importats amb el mateix títol.
- [ ] Comportament en cas de resposta incorrecta (mostrar la correcta? feedback immediat o al final de la pregunta?).
- [ ] Retocs tècnics pendents al logo (contrast del text, versió en canvas quadrat per iOS) — disseny original acceptat, veure conversa.

## 10. Identitat visual

- Logo ja definit (roda de colors en forma d'interrogant + text "quiz Master"), acceptat com a base de la marca.
- Paleta de colors principal: arc de Sant Martí / multicolor, estil juganer.
- Pendent aplicar-ho a la resta de la UI quan es dissenyin les pantalles.
