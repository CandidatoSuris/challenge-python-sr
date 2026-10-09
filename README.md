# Challenge técnico: Sr. Python Developer

**Suris Code**

## Contexto

Vas a trabajar en una plataforma internacional que protege el catálogo de la industria musical en internet. Hace dos cosas:

1. **Recibe el catálogo de varias discográficas y lo consolida.** El catálogo llega en XML estándar de la industria (DDEX ERN), en más de una versión, y también en planillas. Una misma grabación puede llegar de varias fuentes y tiene que quedar una sola vez.
2. **Identifica links que ofrecen esa música sin licencia.** Los links traen títulos sucios. Para cada uno hay que decidir qué grabación del catálogo es, porque con eso se envían avisos de baja.

Un aviso de baja enviado por error tiene costo legal. Por eso **un falso positivo es mucho peor que mandar un caso a revisión humana**.

Este challenge es una versión chica de ese problema, con datos 100% ficticios.

## Reglas de juego

| | |
|---|---|
| **Tiempo** | **5 horas.** No esperamos que termines todo. Anotá en `NOTES.md` cuánto tiempo te llevó en realidad: no penaliza. |
| **Entrega** | Repo Git privado con acceso para nosotros. Cloná este repo y subilo a un repo **privado** tuyo. **No hagas fork**: un fork de un repo público es público y otros candidatos verían tu solución. |
| **Idioma** | Código y `NOTES.md` en **inglés**: el cliente es internacional. |
| **IA** | **Podés usar las herramientas de IA que quieras.** Contanos cuáles usaste y para qué en `NOTES.md`. En la entrevista vas a explicar y modificar tu código en vivo, así que asegurate de entender lo que entregás. |
| **Dudas** | Si algo no está claro, decidí vos, anotalo en `NOTES.md` y seguí. |
| **Extras** | **No suman puntos.** Preferimos que el tiempo vaya a lo pedido. Temas como base de datos, escala, embeddings y LLM los charlamos en la entrevista. |

**Prioridades.** Si no llegás con todo, priorizá en este orden:

1. Ingesta correcta.
2. Matching sin falsos positivos.
3. Tests.
4. Endpoint de la API.

## Qué te damos

```
data/
  inbox/                       21 archivos de catálogo, para procesar en orden de nombre
    *_andina_ern383_*.xml      Andina Records: DDEX ERN 3.8.3
    *_pacific_ern43_*.xml      Pacific Sound Group: DDEX ERN 4.3
    *_indie_catalogo.csv       Sello independiente: CSV
  infringements.csv            ~100 links a matchear
  infringements_expected.csv   respuestas esperadas de esos links, para que midas tu solución
```

Sobre el formato de los datos:

- **XML de Andina y Pacific.** Usan versiones distintas de DDEX ERN y la estructura cambia entre versiones: no es solo el namespace. Abrí los archivos antes de escribir el parser.
- **CSV.** No asumas nada sobre su encoding ni su separador.
- **`infringements_expected.csv`** tiene estas columnas:

  | Columna | Qué es |
  |---|---|
  | `preferred_decision` | La decisión ideal. |
  | `acceptable_decisions` | Otras decisiones que también se aceptan. |
  | `target_isrcs` | Los ISRC válidos cuando la decisión es `MATCH`. |

> También vamos a evaluar tu solución con **otro set de links que no te damos**, sobre el mismo catálogo. Ajustar reglas a mano para cada fila del set público no te sirve.

## Reglas de negocio

### Ingesta

**R1. Orden e idempotencia.**
- Los archivos se procesan en orden lexicográfico de nombre.
- Un archivo con el mismo contenido (SHA-256) que otro ya procesado se saltea.

**R2. Registros y normalización.**
- Un registro es cada `SoundRecording` del XML, o cada fila no vacía del CSV.
- **Texto normalizado:** NFKD, sin diacríticos, `casefold`, con lo no alfanumérico reemplazado por espacio y los espacios colapsados. Ejemplo: `"Tití Me Preguntó!"` → `"titi me pregunto"`.
- **ISRC normalizado:** en mayúsculas y solo con letras y dígitos. Es válido si cumple `^[A-Z]{2}[A-Z0-9]{3}\d{7}$`. Ejemplo: `"us-ab1-24-00001"` → `"USAB12400001"`.

**R3. Admisión.** Se aplica en este orden:

1. **`REJECTED`**: el ISRC falta o es inválido.
2. **`PARKED`** (se guarda, pero no se consolida ni se matchea), en cualquiera de estos casos:
   - el título normalizado es un placeholder: matchea `^(track|song|audio|pista|tema|title) ?\d+$` o `^untitled$`;
   - el artista normalizado es `unknown`, `unknown artist`, `various artists` o `tbc`.
3. **Sigue a consolidación** (R5) en cualquier otro caso.

**R4. Versiones.**
- Se buscan en lo que va entre paréntesis o corchetes del título, y en el subtítulo de versión si el formato lo trae.
- **Título base:** el título sin esas partes.
- **Versiones duras** (son grabaciones distintas): `live`, `en vivo`, `remix`, `acoustic`, `instrumental`, `radio edit`, `sped up`.
- **Versiones blandas** (es la misma grabación, se ignoran): `remaster`, `remastered`, `deluxe`, `explicit`, `clean`.

**R5. Consolidación en grabaciones canónicas.** El *artista principal* es el artista con rol principal, sin los featured artists.

1. **El ISRC ya pertenece a una canónica:**
   - Si `rapidfuzz.fuzz.token_set_ratio` entre los artistas principales normalizados es **≥ 75**, el registro se une a esa canónica.
   - Si no, es un ISRC reutilizado por otro artista: el registro queda `PARKED`, con motivo `ISRC_ARTIST_DISSIMILAR`.
2. **El ISRC es nuevo:** si el artista principal, el título base y el conjunto de versiones duras coinciden con los de una canónica (normalizados), el registro se une a ella. Así, una canónica puede tener varios ISRC.
3. **Ninguno de los dos casos:** se crea una canónica nueva.

### Matching de links

**M1. `DISCARDED`.** Se descarta el link si se cumple cualquiera de estas condiciones:
- La URL es inválida: el esquema no es `http`/`https` o falta el host.
- El dominio es de un servicio licenciado: `open.spotify.com`, `music.apple.com`, `www.youtube.com` o `www.deezer.com`.
- La extensión del archivo, en la URL o en `raw_text`, no es de audio. Son de audio: `mp3`, `m4a`, `flac`, `wav`, `ogg` y `aac`.

**M2. ISRC exacto.** Si en la URL o en el texto aparece el ISRC de una canónica, la decisión es `MATCH`.

**M3. Matching aproximado.**
- **`MATCH`**: solo con alta confianza **en el artista y en el título**, y con la misma versión dura. Por ejemplo, un *live* no matchea la versión de estudio.
- **`REVIEW`**: hay un candidato plausible, pero es dudoso o ambiguo.
- **`NO_MATCH`**: ningún candidato alcanza.

Solo se matchean canónicas. Los registros `PARKED` y `REJECTED` no cuentan.

**Puntaje por link:**

| Resultado | Puntos |
|---|---|
| Decisión preferida | +1 |
| Otra decisión aceptable | +0,5 |
| `MATCH` equivocado | **−3** |

## Qué tenés que construir

- **Stack:** Python ≥ 3.12, **Pydantic v2**, **FastAPI**, **pytest** y **lxml**. Podés usar rapidfuzz, pandas y Polars.
- **Persistencia:** **No hace falta base de datos.** Alcanza con memoria o SQLite.
- **Ingesta y matching:** según las reglas de arriba. Cada decisión de matching lleva una explicación breve: qué candidato eligió, con qué puntaje y por qué.
- **Endpoint `POST /api/v1/matches`:**
  - **Body:** `{"items": [{"infringement_id": "...", "url": "...", "raw_text": "..."}]}`, con 1 a 500 items.
  - **Más de 500 items:** responde **413**.
  - **Respuesta:** una decisión por item, con los mismos campos que una fila de `matches.csv`.
- **Tests:** los que te parezcan críticos. No buscamos cobertura total, sino ver qué elegís testear.
- **`NOTES.md`**, en inglés y como máximo una página:
  1. cómo correrlo;
  2. decisiones tomadas y qué dejaste afuera;
  3. qué herramientas de IA usaste y para qué;
  4. tiempo real que te llevó;
  5. en 5 a 10 líneas, cómo lo llevarías a producción con **PostgreSQL**: qué tablas crearías y cómo manejarías millones de registros.

## Contrato

Esta parte la corremos de forma automática. **Respetala al pie de la letra.**

```bash
make setup                                   # instala las dependencias
make run IN=data/infringements.csv OUT=out   # ingiere data/inbox y matchea IN
make test                                    # corre los tests
make api                                     # levanta la API en http://localhost:8000
```

**`make run`** escribe dos archivos.

`OUT/stats.json` describe la ingesta de `data/inbox`:

```json
{
  "files_processed": 0,
  "files_skipped_duplicate": 0,
  "records_accepted": 0,
  "records_parked": 0,
  "records_rejected": 0,
  "isrc_conflicts": 0,
  "canonical_recordings": 0
}
```

| Campo | Qué cuenta |
|---|---|
| `files_processed` | Archivos procesados, sin contar los duplicados. |
| `records_accepted` | Registros que se unieron a una canónica o crearon una. |
| `records_parked` | Registros `PARKED`, incluidos los `ISRC_ARTIST_DISSIMILAR`. |
| `isrc_conflicts` | Solo los registros `PARKED` con motivo `ISRC_ARTIST_DISSIMILAR`. |

`OUT/matches.csv` va en UTF-8, con encabezado y una fila por link:

```
infringement_id,decision,matched_isrc,matched_title,score,reason
```

| Columna | Contenido |
|---|---|
| `decision` | `MATCH`, `REVIEW`, `NO_MATCH` o `DISCARDED`. |
| `matched_isrc` | Cualquier ISRC de la canónica elegida, o vacío. |
| `score` | Número entre 0 y 1. |
| `reason` | Texto breve que explica la decisión. |

## Qué sigue

Hay una **entrevista técnica de unos 60 minutos**. Repasamos tu solución, hacés un cambio chico en vivo y charlamos de cómo lo llevarías a producción.

¡Éxitos! Si algo de los archivos no te funciona, escribinos.
