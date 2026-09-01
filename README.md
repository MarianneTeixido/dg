# Decoding Gesture — documentación

Sitio y registro de *Decoding Gesture*, performance audiovisual octofónico con código al
vuelo. **Estrenado el viernes 14 de agosto de 2026 en el auditorio del CMMAS, Morelia.**

- Composición sonora: **Marianne Teixido** — [teixido.cc](https://teixido.cc)
- Composición visual: **Emilio Ocelotl** y Marianne Teixido — [ocelotl.cc](https://ocelotl.cc)

El sitio se publica en **<https://marianneteixido.github.io/dg/>**.

## Qué hay aquí, y qué no

Este repositorio es **la documentación pública** de la obra: las notas de programa
bilingües, el registro fotográfico y en video del estreno, y la descripción técnica del
sistema. **No contiene el motor de la pieza** —el patch de SuperCollider, la interfaz web y
el puente OSC viven en un repositorio privado aparte— y por eso no se ejecuta nada desde
aquí: no hay build, ni dependencias, ni red.

```
index.html                 el sitio, una sola página, con los dos idiomas dentro
estilo.css                 la paleta y las tipografías DE LA OBRA, no del sitio
fotos/                     el registro del estreno — ver fotos/LEEME.md
fuentes/                   las caras vendorizadas, más IBM Plex Sans para texto
descripcion-tecnica.md     cómo funciona el sistema completo, en español
```

Se ve con cualquier servidor estático:

```bash
python3 -m http.server 8077
```

## Tres cosas que no son obvias

**El anillo de la cabecera no es un dibujo de un anillo: es el anillo.** El SVG se generó
con los ocho ángulos reales del arreglo de bocinas del CMMAS y con la ley de ganancia real
del sistema, con las dos manos puestas en el estado exacto de
`fotos/2026-08-14-placa-detalle.jpg`. Si cambia la geometría del anillo, este SVG queda
desfasado y hay que regenerarlo.

**La paleta y las tipografías son las de la obra.** Con una excepción declarada en el CSS:
en texto corrido el violeta y el rojo van aclarados, porque el violeta de la obra da 4.08:1
sobre negro y no llega al 4.5 que pide WCAG AA. En el anillo van los hex verdaderos, porque
ahí el color es el dato.

**El idioma se oculta con un selector que empieza por `body`**, y eso no es adorno: sin el
`body` delante empata en especificidad con cualquier regla que fije `display` y pierde por
ir antes en el archivo. Pasó: salían los dos idiomas a la vez en las semblanzas.

## Lo que falta

**La documentación audiovisual completa.** Lo que hay hoy del estreno son tres fotografías y
dieciséis segundos de video, sin registro de audio y sin ninguna toma con las manos a la
vista. Cuando exista una documentación en forma entra aquí, y **la dirección del sitio no
cambia**.

**La sección «Qué se oye» la escribe Marianne Teixido**, que compone el sonido. Lo que está
publicado es el mínimo verificable sacado de la descripción técnica; falta con qué corpus se
entrenó el modelo de voz, por qué la voz es asémica, y qué pasa musicalmente a lo largo de
los ocho minutos.
