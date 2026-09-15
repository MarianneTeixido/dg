# capturas/ — la interfaz, capturada del sistema corriendo

**Esto no es registro del estreno.** Lo del 14 de agosto está en `fotos/` y son tres
fotografías y dieciséis segundos de video. Aquí está la misma interfaz corriendo el **15 de
septiembre de 2026** contra el corpus de pruebas grabado el 6 de agosto: `web/index.html` en
modo obra, a 1920×1080, sin nadie en la sala. Sirve para ver de cerca lo que en las fotos del
auditorio se ve chico, torcido y con el grano del proyector encima.

Las produce `captura/capturas.py`, que vive en el repo del motor porque necesita el motor. No
hay composición ni retoque: cada PNG es un cuadro que la página dibujó. Lo que sí hay es un
método para no elegirlas a ojo: el script espera a que el estado de las dos manos sea el que
dice el nombre del archivo, dispara, y después comprueba. Cómo comprueba depende del estado, y
la diferencia costó cuatro tandas: los estados que aguantan quietos se verifican parando el
video, y la pérdida de detección se verifica **leyendo los píxeles del PNG**, porque es el
único testigo que no se puede deshacer. Está explicado abajo.

| Archivo | Qué es |
|---|---|
| `interfaz-agarrada.png` | La pieza completa. Las dos manos `AGARRADA`, el anillo con las ocho bocinas numeradas, la cinta de ganancia, el cuerpo en la malla de cromo y las trazas a los lados. Es el equivalente nítido de `fotos/2026-08-14-placa-detalle.jpg`. |
| `interfaz-perdida.png` | Las dos manos `PERDIDA`, en ceniza, y el anillo encendido entero: las fuentes quedaron ancladas y siguen sonando. En `fotos/2026-08-14-sala-fondo.jpg` pasa lo mismo con una sola. |
| `morfeo-1.png` … `morfeo-3.png` | Un solo trayecto del morfeo visto a los 0.3, 1.0 y 1.8 segundos del intervalo de 2 s. No son tres poses: es la interpolación entre dos capturas, que es por qué el cuerpo se ve líquido. |
| `interfaz-cuerpo.png` | La capa 3D sola, con `[p]` y `[t]`. Sin la placa encima se ve el tamaño real de la figura dentro del cuadro. |
| `desarrollo-planta.png` | `planta.html`, el andamio. La marca hueca es una fuente anclada y la llena una agarrada; la línea punteada entre las dos rotula su separación. |
| `desarrollo-monitor.png` | `monitor.html`. Las mismas series que las trazas de la obra, rotuladas, con crudo contra filtrado y el mínimo y máximo de cada rasgo. |

`bitacora.json` guarda, por captura, el estado de las dos manos que reportaba el puente en ese
momento. Es lo que respalda cada pie de foto.

## Cómo se corren

```bash
cd ../decoding-gestures
.venv/bin/python captura/capturas.py            # todo
.venv/bin/python captura/capturas.py --solo morfeo
```

Necesita el corpus, que pesa 345 MB y no está en git. Los parámetros se quedan donde salieron a
escena: umbral a 1.50 m, captura cada 2.00 s.

## Lo que no está, y por qué

**Las dos fuentes sobre la misma bocina.** Conviene precisar cuál es el hueco, porque magenta
sí hay: en `interfaz-agarrada.png` la parte de arriba de la cinta es magenta, que es donde las
ganancias de las dos manos alcanzan a las mismas bocinas. Lo que falta es el caso extremo, las
dos fuentes en el mismo punto del anillo, y **no ocurre en este corpus**: medido sobre una
pasada entera, con las dos manos detectadas nunca bajan de 38° de separación, y una bocina
abarca 45°. Tiene sentido, la performer trabaja con una mano de cada lado del cuerpo. El único
caso por debajo de 14° tenía una mano `PERDIDA`, o sea un ancla vieja y no un encimado. La
escena está escrita en el script y va a salir el día que haya un corpus con las manos cruzadas.

**Una sola mano perdida**, que es lo que muestra la foto desde el fondo de la sala. En este
corpus dura uno o dos cuadros y no se puede capturar: cualquier cosa que se haga para congelar
el instante tarda más que eso. Pausar el video no ayuda y además engaña — la página no congela
la detección, la vuelve a correr sobre el cuadro quieto, y un cuadro quieto se detecta bien
aunque el anterior, movido, se hubiera perdido—, o sea que el acto de parar para capturar
deshace justo el estado que se venía a documentar. Las pérdidas que sí duran son las de las dos
manos a la vez, y ésa es la que está.

**`FUERA DE UMBRAL`**, el segundo escalón del aviso, el ámbar. Se consigue en diez segundos
bajando el umbral a 1.00 m, y por eso mismo no entra: sería escenificar una avería que no pasó
en la función. Si algún día hay que documentarlo, que salga de una función.

**`enlace.html`**, que mide la cadencia y el jitter del stream WiFi. Sin el iPhone enfrente no
tiene nada que medir y la captura sería la plantilla vacía.

**`manos.html`**, que era la más explicativa de las tres del andamio: enseña la mitad RGB del
cuadro con las cajas de MediaPipe encima y, al lado, lo que el contrato entiende. Se generó, se
revisó y se descartó el 15 de septiembre de 2026: la mitad RGB es Marianne de cuerpo entero en
su casa, y la página no la necesita para explicarse.

## Tres cosas que conviene saber antes de volver a correrlo

**El rótulo de la captura no es el que dijo el puente.** Entre que se decide disparar y que el
PNG está escrito pasan un par de décimas, y ahí cabe entero un parpadeo de la pinza: salieron
dos imágenes rotulando `ANCLADA` donde el registro decía `AGARRADA`, y al revés. Para los
estados que aguantan, el arreglo es parar el video antes de disparar —llamando a `pause()` del
elemento, no pulsando `[espacio]`, que alterna y tarda tres viajes del protocolo—, con un tope
de un segundo porque a los 1.3 s la placa escribe `SIN IMAGEN`. Para la pérdida de detección no
hay arreglo por ese lado y la comprobación se hace sobre el archivo: si el punto de identidad
junto al rótulo está en ceniza, la mano se perdió; si está en su color, no.

**El puente que conecta no es el puente.** La placa dibuja `SIN PUENTE` en blanco mientras el
WebSocket no conecte, y el blanco está reservado a la avería, así que sin algo escuchando en el
8765 toda captura reportaría una falla que no existe. El script levanta un sumidero que recibe
los paquetes y no hace nada con ellos. La placa sólo lee de él el estado de la conexión, así
que la imagen es la misma; lo que no pasa por ahí es el número de escena, que viene de
SuperCollider.

**Si la ventana de Chrome queda tapada, la captura miente y no se nota.** Chrome deja de darle
cuadros a la página, pero el video sigue avanzando: la comprobación obvia dice que todo va bien
mientras la malla se congeló hace dos minutos. Se vio como 0.4 cuadros por segundo y las dos
manos clavadas en el origen. El script lo evita con `Emulation.setFocusEmulationEnabled` y
además se planta si el bucle de dibujo va por debajo de 2 fps. Con GPU van 9, que es menos que
los 27 de la máquina del escenario y de sobra para sacar imágenes fijas.
