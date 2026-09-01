# Decoding Gesture

## Descripción técnica y conceptual

Performance audiovisual octofónico con código al vuelo · 8 minutos  
Composición sonora: Marianne Teixido · Composición visual: Emilio Ocelotl y Marianne Teixido  
Auditorio del CMMAS, Morelia — 14 de agosto de 2026

---

## 1. La obra

*Decoding Gesture* es un performance audiovisual octofónico con código al vuelo en el que el
cuerpo de la intérprete —seguido gestual y facialmente con una cámara de profundidad— controla
la síntesis neuronal de una voz asémica que se granula y se distribuye en el espacio sonoro. La
obra explora el gesto como umbral entre cuerpo y voz sintética, donde el género se codifica,
decodifica y queeriza en tiempo real.

La pieza dura ocho minutos y ese arco fijo es lo que organiza las escenas: no hay estructura
generativa abierta, hay una partitura que se live-codea sobre una duración conocida.

### 1.1 El diferenciador

Existe un linaje reciente de trabajo audiovisual donde el cuerpo capturado en 3D es el material
visual en tiempo real (Sinjin Hawke & Zora Jones / Fractal Fantasy es la referencia más
cercana). Aquí el cuerpo no es el espectáculo: es el **instrumento de una voz sintética**. Lo
que la proyección muestra al público no es sólo el cuerpo sino el **control** —cómo el cuerpo
escribe la voz—, y por eso la interfaz proyectada no es decoración sino evidencia de esa
escritura.

### 1.2 Los dos públicos

Toda decisión de interfaz se juzga contra dos lectores simultáneos y con necesidades opuestas:

- **La intérprete**, que lee la pantalla a 2–4 metros y en ángulo (va al costado del escenario)
  y que **no toca teclado ni ratón durante los ocho minutos**. Para ella la interfaz tiene que
  estar callada cuando todo va bien e imposible de ignorar cuando algo se rompe: nadie va a
  reparar nada en vivo, así que su función es que ella lo *sepa* a tiempo para dejar de
  apoyarse en las manos y seguir tocando.
- **El público**, sentado a 8–15 metros en una sala larga, para el que la proyección es el fondo
  de escena y la única figura visible. Lo que se lee a esa distancia son áreas y grosores, no
  texto ni puntos.

Esa doble lectura tiene una consecuencia medible: los grosores de línea de la interfaz son
proporcionales al radio del dibujo y no píxeles absolutos, porque una línea de un píxel a doce
metros no se ve fina — no se ve.

---

## 2. Arquitectura

```
iPhone (TrueDepth / LiDAR)
   │  Record3D — streaming WiFi por WebRTC en red local dedicada
   ▼
Navegador (Chrome)
   ├─ decodifica el campo de profundidad (WebGL)
   ├─ segmenta la silueta y extrae rasgos gestuales
   ├─ rastrea manos (y rostro, v2) con MediaPipe sobre la mitad RGB
   ├─ renderiza la interfaz: cuerpo cromado en malla 3D + «la placa»
   └─► WebSocket ──► Puente Python
                       ├─ capa de mapeo: suavizado, calibración, escenas
                       │     ▲ escenas conmutadas por OSC desde el código al vuelo
                       └─► OSC ──► SuperCollider
                                     ├─ voz neuronal (RAVE vía nn.ar)
                                     └─ granulación + espacialización octofónica
```

El cambio de escenas viaja en sentido inverso: la pieza se live-codea desde SuperCollider, así
que quien escribe código en vivo decide en qué escena de mapeo está el puente (OSC de SC →
puente) en lugar de automatizarlo.

El proyecto es colaborativo con dos lados y una interfaz explícita entre ellos: el sonido
(`sc/`, Marianne) y lo visual más la captura (`web/`, Emilio), con el puente (`bridge/`) como
zona compartida. La interfaz formal entre ambos es el contrato OSC.

---

## 3. Captura y visión por computadora

### 3.1 La fuente

El iPhone transmite por WebRTC un cuadro partido: **mitad izquierda profundidad codificada en
el matiz** (rango 0–3 m), **mitad derecha RGB**, pixel-alineadas entre sí. El dispositivo expone
`GET /getOffer`, `POST /answer` y `GET /metadata` (de donde sale la matriz intrínseca de la
cámara), y **sólo admite un cliente a la vez** — restricción que determina que una sola página
del navegador sostenga el stream.

Como fuente alternativa para desarrollo y contingencia se acepta un **mp4 RGBD** exportado desde
Record3D, con el mismo cuadro partido y la matriz intrínseca embebida como JSON al final del
archivo. Ni el shader ni la extracción de rasgos distinguen la fuente.

Restricciones que impone esta vía: la profundidad llega con compresión con pérdida y el rango
útil son ~3 metros, así que la intérprete trabaja cerca del sensor. Plan B previsto: migrar la
captura a USB (librería `record3d` de Python) moviendo la extracción de rasgos al puente, sin
tocar ni la interfaz web ni el lado de SuperCollider.

### 3.2 De píxel a metros

La malla 3D no almacena posiciones: la geometría lleva sólo índices de vértice y el vertex
shader desproyecta cada píxel leyendo dos texturas (el cuadro y la máscara de silueta) con la
matriz intrínseca. Eso hace la desproyección **lineal en la profundidad**, con dos consecuencias
que el proyecto aprovecha:

- una «captura» del cuerpo son dos texturas (~2,4 MB), no una nube de puntos;
- interpolar profundidad *es* interpolar posición 3D, así que la transición entre capturas es un
  `mix()` en el shader.

### 3.3 Manos

MediaPipe Hand Landmarker corre **sólo sobre la mitad RGB** y **reducida a ~360×480**. Reducir
la imagen antes del modelo la mejora y la abarata (91,4 % de cuadros con las dos manos a 360×480
contra 85,2 % a resolución nativa): el modelo reescala internamente a 192×192 pase lo que pase,
así que bajar no le quita información, pero abarata el blit y el modo VIDEO conserva el
seguimiento entre cuadros.

De ahí la cadena: landmark de la palma → píxel → mismo píxel en la mitad de profundidad →
decodificar matiz → metros con la matriz intrínseca → restar el centro del cuerpo (marco
relativo, invariante a dónde esté parada la intérprete) → corregir la inclinación del tripié →
normalizar por el alcance del brazo.

Cuatro detalles que costaron medición y que definen la implementación:

- **La lateralidad se asigna por geometría, no por la etiqueta del modelo.** La etiqueta de
  MediaPipe resulta anatómicamente correcta tal cual (el video no viene espejeado), pero
  parpadea ~6 % de los cuadros; asignar el `id` por etiqueta intercambiaría las dos fuentes en
  el anillo a media pieza.
- **La vertical de la sala se saca del propio cuerpo**, por análisis de componentes principales
  sobre la nube 3D en una pose de calibración (de pie, quieta, brazos abajo). El eje principal
  de la silueta en 2D da el *roll* de la cámara, pero el que estorba es el *pitch*. Sin esta
  corrección, extender el brazo mueve el sonido en diagonal.
- **El ancla del cuerpo se filtra con τ = 3 s.** Es el centroide de la silueta con brazos
  incluidos: extender un brazo lo corre y movería la otra mano sola. Con tres segundos un gesto
  no lo toca, pero sí lo sigue si la intérprete camina.
- **La profundidad de la palma se lee por mediana de un parche interior y a resolución nativa**,
  nunca sobre la imagen reducida: el matiz es circular, así que promediar píxeles antes de
  decodificar inventa profundidades en los bordes, y el borde de la mano sangra 10–15 px.

### 3.4 Lo que se midió antes de escribir la tubería

A la distancia de trabajo (~1,2 m), sobre un corpus de video de 30 s grabado con el dispositivo:

| Qué | Cuánto |
|---|---|
| Ruido temporal de la profundidad, escena quieta | 4–5 mm (1σ) |
| Lo mismo, con el cuerpo en movimiento | ~10 mm |
| Paso de cuantización de la decodificación | ~2 mm |
| Ancho de la mano en la mitad RGB (1440×1920) | 160–200 px |
| Sangrado de profundidad en el borde de la mano | 10–15 px |

Tres hallazgos con consecuencia directa: (1) el eje de profundidad aguanta —10 mm son ~3,5° de
temblor en un anillo con bocinas cada 45°—, así que se canceló el estimador de respaldo por
tamaño aparente de la palma; (2) MediaPipe ve la mano de sobra y no hace falta una cascada
Pose→recorte→Hands; (3) **promediar la palma no compra precisión**, porque el códec ya entrega
el ruido correlacionado en el espacio, y además el ruido no es gaussiano sino una escalera
(43–90 % de los píxeles son idénticos entre cuadros consecutivos por los bloques *skip* de
H.264). El único recurso es el filtro temporal.

---

## 4. El contrato OSC

Es la interfaz formal entre las dos personas del proyecto y se trata como un acuerdo, no como un
detalle de implementación. Está cerrado en su versión 1.

### 4.1 La decisión que lo ordena

**La posición en el anillo octofónico la mandan las manos, no el centroide del cuerpo.** Dos
manos con posiciones independientes, cada una con su propio estado de agarre. El agarre es una
**pinza** pulgar-índice: la posición sólo se actualiza mientras hay pinza y, al soltar, la fuente
queda anclada donde estaba. El ancho del frente sonoro no se mapea: es emergente, la distancia
entre las dos manos agarradas.

La pinza hace dos cosas a la vez: le devuelve a la intérprete el control de *cuándo* el sistema
responde, y vuelve inofensivos los fallos de detección, porque una mano que se pierde no tiene
pinza y su fuente simplemente se queda donde estaba.

### 4.2 El disco es el plano frontal

El disco del anillo es el **plano frontal** del cuerpo (lateral × altura): el anillo girado 90° y
puesto de pie frente a la intérprete. Mano a la izquierda = sonido a su izquierda; mano en alto =
sonido atrás; mano abajo = sonido al frente. La **profundidad** —acercar o alejar la mano del
cuerpo— controla la **dispersión** (puntual ↔ envolvente).

Dos razones lo deciden. Primera: el eje cerca-lejos era el único sucio de los tres, iba al doble
de sensibilidad que el lateral y arrastraba toda la cadena de calibración vertical; ahora
desemboca en la dispersión, que es lenta y perdona. Segunda: coincide con lo que se ve — como el
cuerpo se proyecta dentro del anillo, la marca de la mano cae donde se ve la mano, y la metáfora
se aprende mirándola en vez de memorizándola.

Lo que se paga y hay que saberlo: se pierde el isomorfismo egocéntrico. «Empujo y el sonido se
aleja» pasa a «levanto y el sonido se va atrás», que es convención de mapa en vez de
correspondencia física.

### 4.3 Direcciones

```
# Manos (~30–60 Hz). Un mensaje por mano y por cuadro: posición y agarre viajan
# juntos a propósito, para que nunca se lean de cuadros distintos.
/mano   i f f f f i     # id, x, y, dispersión, pinza, agarre
                        #   id      1 = izquierda de la intérprete, 2 = derecha
                        #   x, y    posición en el plano de la sala, −1..1
                        #   disp.   0..1 (0 puntual, 1 envolvente)
                        #   pinza   0..1 continua (la medida cruda)
                        #   agarre  1 = agarrada (se mueve), 0 = suelta (anclada)
/mano/activa  i i       # id, 1 = la mano está a la vista del sensor

# Rasgos del cuerpo (~30–60 Hz, normalizados 0..1)
/gesto/centroide   f f  # x, y del cuerpo (encuadre y presencia, ya no posición)
/gesto/tamano      f f  # alto, ancho de silueta
/gesto/area        f    # cuánto cuerpo se presenta al sensor
/gesto/prof        f    # profundidad media (cercanía)
/gesto/energia     f    # energía de movimiento (cambio entre cuadros)

# Rostro (v2, direcciones reservadas)
/rostro/boca       f
/rostro/cejas      f
/rostro/cabeza     f f

# SuperCollider → puente (cambio de escena, desde el código al vuelo)
/escena            i
```

**Ejes de la sala**: `+x` a la derecha de la intérprete, `+y` alejándose de ella. Origen en el
centro del anillo. El campo es el **disco unitario**: `x²+y²` se recorta a 1, así que las
esquinas no existen y cualquier valor válido cae dentro del anillo.

La posición va **cartesiana y no en acimut a propósito**: cómo se renderiza a ocho canales es
decisión del lado sonoro y el contrato no debe presuponerla. Y va en rango −1..1, no 0..1 como
los rasgos escalares, porque un campo centrado necesita centro con signo.

Los rasgos del cuerpo son **geometría de silueta más energía de movimiento**, y esa elección es
deliberada: son robustos con iluminación escénica y suficientes para un mapeo expresivo. El
esqueleto —articulaciones específicas— queda como exploración posterior, bajo el criterio de que
**la expresividad de la v1 depende del mapeo y no de la cantidad de datos**.

La vía del rostro para la v2 ya está resuelta técnicamente aunque los rasgos no estén elegidos: el
stream de Record3D **no expone los blendshapes de ARKit**, pero la mitad RGB del cuadro sí llega
al navegador, y ahí MediaPipe Face Landmarker extrae landmarks y blendshapes sin cambiar la
arquitectura. Los candidatos, que son los que las direcciones reservan: apertura de boca y
mandíbula —los obvios para una voz—, cejas e inclinación de cabeza.

### 4.4 Comportamiento, no sólo direcciones

- Al arrancar, las dos fuentes están en el centro, sueltas e inactivas.
- Si se pierde una mano no se manda posición nueva: la fuente sostiene su último valor.
- **Si el navegador se desconecta, el puente no se calla**: sigue emitiendo la última posición
  con agarre y actividad en cero. El silencio se oiría como un salto.
- El suavizado y la calibración viven en el puente, con una excepción anotada: el alcance del
  brazo se fija en el navegador —que lo necesita para dibujar la vista en planta— y viaja al
  puente, no al revés.

---

## 5. El puente

Proceso Python que recibe los rasgos por WebSocket (puerto 8765) y los reenvía por OSC a
SuperCollider (57120 por omisión), emitiendo a **tasa fija de 60 Hz** independientemente de la
tasa a la que entren los datos. Escucha `/escena` en un puerto propio (57121) para recibir el
cambio de escena que manda el código al vuelo.

El suavizado es un **filtro de un euro**, no una media exponencial: la espacialización necesita
quietud y golpe a la vez, y un euro da las dos con dos perillas (corte mínimo y beta).

La capa de mapeo vive en `bridge/mapeo.json` y **se recarga sola al guardar**. Eso no es
comodidad sino requisito: reiniciar el puente se *oye*, porque las manos nacen en (0,0) y las dos
fuentes saltan al centro del anillo, así que afinar con la intérprete enfrente sin interrumpir el
sonido sólo es posible por recarga en caliente. La **escena 0 es la identidad** y funciona como
red de seguridad: reproduce bit a bit el comportamiento previo a que existiera el archivo. Las
escenas actuales van del extremo «quieta» (la fuente casi no tiembla, a cambio de llegar tarde)
al extremo «filo» (el golpe llega con filo, a cambio de temblar con la mano quieta), más una
escena «solo manos» que corta los rasgos del cuerpo para aislar culpas.

Lo que no vive en el mapeo, a propósito: la geometría del contrato (está en un único módulo del
navegador) y el recorrido del brazo.

---

## 6. El lado sonoro

### 6.1 Forma de trabajo

El patch de SuperCollider **se live-codea**: no es una aplicación monolítica sino bloques que se
evalúan de arriba abajo y se reemplazan en vivo, sobre `ProxySpace`. Todo es un node proxy y la
mezcla es una suma de proxies.

El orden de declaración es estructura y no estilo: el maestro octofónico va antes que cualquier
capa, el reloj antes que cualquier `Demand`, y los buffers antes que las capas que los indexan.
Las tres cosas fallan **en silencio** si se evalúan al revés — anillo colapsado a estéreo,
patrones congelados, o índice sin resolver.

### 6.2 Las tres capas

1. **Dron** — cama armónica continua que sostiene afinación y registro grave. Poco o nada
   gestual, espacializada lento.
2. **Granulación semifija, live-codeable** — el cuerpo del set: granulación de buffers de voz
   (`PlayBuf` con modulación por `Demand` para el glitch, más `Warp1` con control de puntero,
   tamaño de ventana y solapamientos), con patrones rítmicos que se reescriben en vivo. Sobre
   ella viven variantes «primas» que exponen distintos parámetros al gesto.
3. **Sólo gesto** — capa que únicamente suena cuando el cuerpo la produce. Es donde el gesto se
   percibe como *causa* de la voz.

### 6.3 Voz neuronal

La voz es un modelo RAVE corriendo dentro de SuperCollider mediante la UGen `nn.ar` —no por un
puente Python, lo que evita la latencia que rompería la percepción de causa-efecto—. El modelo
queda fijo al construir el grafo del SynthDef: para alternar modelos se duplica el bloque, no se
parametriza el nombre. El destino del mapeo no es un banco de parámetros de síntesis tradicional
sino el **espacio latente** del modelo, más los parámetros de granulación; ése es exactamente el
mapeo no-1:1 que la pieza busca.

### 6.4 Octofonía

Toda la espacialización pasa por `PanAz` a través de una sola función, y el principio que la
ordena es **separar quién genera la posición de quién la reparte**: la posición vive en su propio
proxy y una única función la distribuye a los ocho canales. Por eso cambiar un movimiento es
redefinir un proxy, y por eso la mano de la intérprete entra sin tocar ninguna capa.

Como `PanAz` gana energía al ensanchar el frente, hay una compensación de ley medida —no
adivinada— para que abrir la dispersión no se oiga como un crescendo.

**El anillo del CMMAS no está numerado en orden.** Las ocho bocinas van en pares
izquierda/derecha de adelante hacia atrás, así que el recorrido físico horario es 1, 2, 4, 6, 8,
7, 5, 3. Como `PanAz` reparte en canales consecutivos, sin corregir esto una fuente girando suave
saldría saltando de un lado a otro de la sala. No se resuelve girando ni espejeando el anillo:
hace falta una **permutación** explícita, que existe tanto en el patch como en el dibujo de la
interfaz. Verificada oyéndola en el auditorio: las ocho suenan donde dicen.

---

## 7. La interfaz proyectada

La proyección tiene dos mitades con referentes distintos y funciones distintas: **el cuerpo**, en
malla 3D con material de cromo líquido, y **el dibujo** —«la placa»—, un diagrama de línea fina y
concéntrico dibujado en canvas 2D encima. Figura y diagrama. El cuerpo va **dentro** del anillo,
que es lo que literalmente pasa en la sala.

Tres decisiones que salieron de restricciones físicas y no de gusto:

- **El fondo es negro por obligación.** El proyector es aditivo: negro es ausencia de luz. Un
  fondo claro deja el cañón a tope, ilumina la sala y le quita los negros al cromo. Y como la
  sala lleva luz frontal difusa (que MediaPipe exige, mientras la profundidad preferiría
  contraluz), el contraste ya viene comprometido: nada de grises medios, mueren en sala.
- **Todo en líneas y nada en post-proceso.** Cada milisegundo compite con MediaPipe en el hilo
  principal. Resulta que esa restricción coincide con el lenguaje visual elegido.
- **El color es semántica, no adorno.** La línea del diagrama es acromática; el color dentro del
  disco es siempre un dato de mano (violeta la izquierda, rojo la derecha, y **magenta emergente
  donde se encima**, porque se dibuja en modo aditivo igual que suma el proyector); el color
  arriba, en el bisel, significa avería. El blanco puro está reservado a un solo suceso: el
  puente caído.

El anillo funciona como **medidor** y no como diagrama: las ganancias de las ocho bocinas se
dibujan como una cinta cerrada de grosor variable alrededor del cuerpo, interpolada de forma que
los picos caen exactamente sobre las bocinas que suenan. Eso hace visible la octofonía —el
público está sentado dentro de ocho bocinas sin ver nunca qué se mueve—, y el grosor es el canal
que sobrevive a doce metros.

El único movimiento de la interfaz es el **agarre** —un pulso breve al cerrar la pinza—, de modo
que el público aprende la regla (cierra la mano y el sonido se queda ahí) sin leyenda. Los avisos
de avería laten a 0,5 Hz con un suelo que nunca llega a cero: latir y no parpadear, porque un
rótulo que parpadea en la proyección se lee como fallo de la proyección.

Los avisos de estado se ordenan por prioridad —`SIN PUENTE`, `SIN IMAGEN`, `FUERA DE UMBRAL`— y
esa prioridad es la mitad del diseño: sin cuadros de video, «fuera de umbral» es una tautología y
mandaría a la intérprete a mover la única perilla que en ese caso no arregla nada. **Nombrar la
causa equivocada es peor que no nombrar ninguna.**

### 7.1 Morfeo entre capturas

La interfaz no muestra el cuerpo en tiempo real: congela una captura cada cierto intervalo
(0,25–6 s, ajustable) e interpola continuamente de la anterior a la más reciente. El cuerpo se ve
con un retardo líquido, entre poses, en vez de seguir al gesto cuadro a cuadro. La
correspondencia es por píxel y no por punto del cuerpo: no es un morfismo «mano → mano» sino un
derretimiento del campo de profundidad, y los píxeles que sólo existen en una de las dos capturas
se desvanecen en su sitio en vez de volar desde el fondo.

---

## 8. Rendimiento

El cuello del sistema **no es la GPU**. Medido con reloj de GPU real, la malla de 2,7 millones de
vértices con sus ~8 lecturas de textura por vértice cuesta 6,5 ms, y MediaPipe encima no la
mueve: son 0,4–8,8 ms de un cuadro de 50–100 ms. Lo que se satura es el **hilo principal**, y
dentro de él manda la lectura de píxeles: cada `drawImage` + `getImageData` fuerza una
sincronía GPU→CPU, y llegó a haber cuatro por cuadro, dos de ellas trabajo duplicado entre las
dos ramas del programa.

Compartir una sola lectura de profundidad por cuadro entre todos los consumidores, más bajar las
dos palmas en un solo recorte, redujo las sincronías de cuatro a dos y quitó 21 ms de hilo
principal.

En la máquina del escenario (i9-14900HX con GPU discreta), MediaPipe cuesta 7,3 ms y el sistema
completo corre a **27 Hz**, un 10 % por debajo de los 30–60 Hz que pide el contrato. La lectura
es que no vale la pena abrir el contrato por 3 Hz: el puente emite a 60 Hz fijos con un euro, así
que SuperCollider recibe suave igual y lo que se pierde es filo en gestos rápidos —juicio de la
intérprete, no del código. Queda un asterisco: esa medición se hizo con **una** mano, y la obra
usa dos.

Nota de método que el proyecto se aplica a sí mismo: el banco se corre siempre **pareado** (las
dos versiones seguidas, misma sesión), porque la máquina deriva ~12 % entre sesiones y Chrome
suspende el bucle de dibujo en pestañas de fondo. Comparar contra una tabla de otro día no vale.

---

## 9. Requisitos de montaje

- iPhone o iPad con FaceID o LiDAR, con Record3D y streaming WiFi habilitado, en red local
  dedicada con la computadora.
- Computadora con GPU capaz, navegador Chrome (el streaming de Record3D reporta problemas con
  Firefox). Todas las dependencias del navegador están vendorizadas: no se depende de red en
  escena.
- Python 3 con `websockets` y `python-osc`, para el puente.
- SuperCollider nativo (no WSL: el audio de WSL no da la latencia del vivo), con la extensión
  `nn.ar` y un modelo RAVE de voz.
- Interfaz de audio de **8 salidas** y anillo octofónico. En ensayo casero basta estéreo.
- Iluminación: **luz frontal difusa y neutra**. La profundidad preferiría contraluz y MediaPipe
  exige luz frontal; probado en sala, las dos conviven.
- Proyección: la pantalla es el fondo de escena y la pieza se juzga desde la última fila, no
  desde la laptop.

---

## 10. Criterios de revisión

En lugar de una lista de roles, el proyecto se revisa con preguntas concretas al cerrar cada
paso. Si alguna no se puede responder que sí, ese eje se está descuidando.

- **Expresividad** — ¿la intérprete puede producir contraste: silencio, sutileza, clímax? ¿O
  todo gesto suena igual de intenso?
- **Legibilidad** — ¿sabe, sin mirar un manual, qué está controlando en este momento?
- **Sonido** — ¿los parámetros que llegan a SuperCollider mueven algo musicalmente relevante, o
  sólo mueven números?
- **Voz** — ¿lo que suena se percibe como voz, aunque sea asémica, y el gesto se percibe como su
  causa? ¿O es textura genérica que casualmente sigue al cuerpo?
- **Espectáculo** — ¿la proyección se entiende y atrae a distancia, o sólo funciona de cerca?
- **Dirección** — ¿esto sirve a la pieza, o es una demo técnica bonita?

Hay decisiones que no se delegan al código: cómo se siente el sistema en el cuerpo, qué es la
pieza, y cuándo un mapeo «responde» son juicios de la intérprete.

---

## 11. Estado

La cadena completa —iPhone → navegador → puente → OSC → SuperCollider con el material real—
corrió en el auditorio del CMMAS con las ocho bocinas. Quedaron confirmados en sala el mapa de
bocinas, la correspondencia entre la proyección y el cuerpo, y la convivencia de la iluminación
con los dos sistemas de visión.

Lo que sigue abierto: el rango de altura calibrado con el recorrido real de los brazos, la tasa
del sistema medida con las dos manos a la vista, qué material sostiene cada mano (hoy el mismo en
las dos; la intención es granulación en una y voz neuronal en la otra), los rasgos faciales de la
v2 —cuyas direcciones OSC ya están reservadas pero cuyos rasgos concretos no se han elegido— y la
integración del agarre con la imagen, que es lo que terminaría de convertir la proyección en una
visualización en vez de dos capas que comparten pantalla.
