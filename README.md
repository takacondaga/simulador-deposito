# Simulador de depósito

Página única para probar cómo entran cajas o piezas en un depósito rectangular:
se define el piso, se cargan las piezas con sus medidas y la app las ubica en
una vista 3D, calculando la superficie ocupada y el precio.

Todo vive en `simulador-deposito.html`: no hay instalación, build ni
dependencias locales.

## Abrir

Doble click en `simulador-deposito.html`.

La primera carga necesita **internet**, porque three.js se baja de un CDN
(jsDelivr). Si no hay conexión, la vista muestra un mensaje de error. También se
puede servir con cualquier servidor estático, por ejemplo:

```bat
python -m http.server 8000
```

y abrir `http://localhost:8000/simulador-deposito.html`.

## Interfaz

La pantalla se divide en dos:

- **Panel izquierdo:** medidas del piso, precio, resultados, opciones y la
  tabla de piezas.
- **Vista derecha:** escena 3D con el piso y las piezas.

En pantallas angostas (≤ 820 px) la vista 3D pasa arriba (46 % del alto) y el
panel abajo, a todo el ancho.

## Uso rápido

1. Poné el **frente** y el **fondo** del depósito, en metros (1 a 500).
2. Cargá el **precio por m²** si querés ver el total.
3. Agregá filas con **Agregar fila** y completá nombre, largo, ancho y alto de
   cada pieza, en centímetros.
4. Arrastrá las piezas en la vista 3D para acomodarlas.
5. Para apilar, activá **Modo apilar** o mantené **Shift** mientras arrastrás.
6. Seleccioná una pieza y usá **Rotar 90°** o la tecla **R** para girarla.

Al arrancar ya hay un piso de 6 × 8 m y una pieza de 100 × 100 × 100 cm.

## Unidades

- **Piso:** metros (se convierte internamente a centímetros).
- **Piezas:** centímetros, con decimales.
- **Superficie:** m². **Precio:** pesos argentinos, formato `es-AR` con dos
  decimales.

## Resultados

| Campo | Qué muestra |
|---|---|
| **Piso** | Frente × fondo en metros y la superficie total en m² |
| **Piezas (apiladas)** | Cantidad de piezas válidas y, entre paréntesis, cuántas están elevadas |
| **Superficie necesaria** | Suma real de la huella de las piezas que apoyan en el piso, en m² con hasta 3 decimales |
| **Superficie a facturar** | Esa superficie redondeada hacia arriba al m² entero |
| **Precio total** | Superficie a facturar × precio por m² |

Solo las piezas que apoyan en el piso suman superficie: las apiladas no ocupan
m² adicionales. Las medidas inválidas tampoco suman (ver abajo).

## Regla de facturación

No se cobran fracciones de metro cuadrado: cualquier excedente se redondea
**siempre hacia el entero superior** antes de multiplicar por el precio. El
redondeo se hace una sola vez sobre el total, no pieza por pieza.

| Superficie ocupada | Se factura |
|---|---|
| 1,00 m² | 1 m² |
| 1,01 m² | 2 m² |
| 2,10 m² | 3 m² |
| 15,00 m² | 15 m² |
| 0 m² | 0 m² |

Sin superficie ocupada el total es $0.

Por ejemplo, con un precio de $1.000/m², una superficie de 2,1 m² se factura
como 3 m²: **$3.000**. La fila **Superficie necesaria** sigue mostrando la
superficie real, para poder comparar.

## Tabla de piezas

Columnas: **Nombre**, **Largo**, **Ancho**, **Alto** (cm) y el botón `×` para
eliminar la fila. Cada pieza tiene un color propio, que se ve en la tabla y en
la escena.

- El nombre se usa en la etiqueta que flota sobre la pieza en la vista 3D.
- Una fila queda en rojo y la pieza se oculta si largo, ancho o alto no son
  números mayores que cero. Al corregirla vuelve a aparecer y se reubica sola.
- Cambiar las medidas de una pieza ya ubicada puede hacer que se mueva al
  primer lugar libre si la nueva huella no entra donde estaba.
- Las piezas nuevas se ubican automáticamente en el primer lugar libre sobre el
  piso; si no hay lugar, se apilan sobre lo que encuentren.

## Vista 3D y controles

| Acción | Cómo |
|---|---|
| Mover una pieza | Arrastrarla con el botón izquierdo |
| Redimensionar largo/ancho | Arrastrar uno de los 4 vértices superiores |
| Redimensionar alto | Arrastrar el handle central del techo |
| Cancelar el redimensionado | **Esc** |
| Apilar al mover | Modo apilar activado, o **Shift** mientras arrastrás |
| Orbitar | Arrastrar el fondo (fuera de una pieza) |
| Zoom | Rueda del mouse |
| Desplazar la vista | Botón derecho y arrastrar |
| Vista 3D / cenital | Botones arriba a la izquierda |
| Rotar 90° | Botón **Rotar 90°** o tecla **R** |

La tecla R no actúa mientras estás escribiendo en un campo de texto o número.
Rotar intercambia la orientación de la huella (el largo y el ancho en el piso);
las columnas **Largo** y **Ancho** de la tabla no cambian. Si la pieza no entra
rotada, se revierte y aparece un aviso.

## Redimensionar con el mouse

Con la pieza seleccionada aparecen **5 handles azules** sobre ella:

| Handle | Qué cambia |
|---|---|
| Los **4 vértices superiores** | Largo y ancho |
| El **centro de la cara superior** | Alto |

- El **vértice opuesto** y la base de la pieza quedan fijos.
- Con **Ajustar a grilla** activo, la esquina arrastrada cae en la grilla de
  10 cm. Sin grilla, la resolución es de 1 cm.
- El tamaño mínimo es de 10 cm con grilla y 1 cm sin grilla.
- Si el nuevo tamaño choca o se sale del piso, se conserva el último tamaño
  válido y el handle se pone **rojo**.
- Mientras arrastrás no se reacomoda nada: al soltar, la pieza y las que estén
  encima se asientan según las reglas de apoyo.
- **Esc** cancela el redimensionado en curso y devuelve las medidas anteriores.
- La tabla, la etiqueta y las estadísticas se actualizan en vivo.
- El cursor cambia a `nwse-resize` en los vértices y a `ns-resize` en el alto.

## Reglas de acomodo

- La grilla de ajuste es de **10 cm** y está activada por defecto
  (**Ajustar a grilla de 10 cm**).
- Las piezas se mantienen dentro del piso cuando entran. Si una pieza es más
  grande que el piso, se centra y puede sobresalir (no hay colisión con
  paredes).
- No se pueden superponer: se compara la huella en planta y, además, que las
  alturas se crucen.
- Al soltar una pieza, todas las que quedaron sin apoyo caen hasta el piso o
  hasta la próxima pieza debajo.
- Para apilar, el **centro** de la pieza de arriba tiene que caer dentro de la
  huella de la de abajo. Se permite que sobresalga del borde.
- No se puede colocar una pieza debajo de otra que se apoya en ella: la app
  descarta esos apoyos para evitar ciclos.
- Si el destino choca, se intenta el mismo movimiento en un solo eje (primero
  X, después Z) antes de dejarla donde estaba.
- Al cambiar el tamaño del piso, las piezas se reacomodan y la cámara se
  reencuadra.

## Requisitos técnicos

- Navegador con WebGL y `ResizeObserver` (cualquier Chrome, Edge, Firefox o
  Safari actual).
- Internet para cargar three.js r128 y `OrbitControls` desde jsDelivr.
- Sin dependencias, sin base de datos y sin guardado: **al recargar se pierde
  todo**.

### Usarlo sin internet

Descargá `three.min.js` y `OrbitControls.js` de la versión **0.128.0** (la que
usa el archivo), guardalos junto al HTML y reemplazá los dos `<script src=...>`
del final del `<head>` por:

```html
<script src="three.min.js"></script>
<script src="OrbitControls.js"></script>
```

Tiene que ser la 0.128: el código usa el `OrbitControls` clásico
(`THREE.OrbitControls`), que en versiones nuevas ya no existe con esa forma.

## Limitaciones conocidas

- El piso es siempre rectangular: no hay pasillos, estanterías, columnas ni
  obstáculos.
- La física es deliberadamente simple: no hay peso, estabilidad ni vuelco; solo
  se evita la superposición y se calcula el apoyo.
- El giro es solo de 90° sobre el eje vertical.
- El precio se calcula únicamente sobre la huella en el piso; no incluye
  volumen, altura ni peso. Siempre se factura en m² enteros (redondeo hacia
  arriba).
- No hay deshacer ni historial. Borrar una fila es inmediato. **Esc** solo
  cancela el redimensionado que está en curso.
- Con muchas piezas la edición puede ponerse lenta, porque cada cambio
  reconstruye geometrías y etiquetas.
- No se guarda nada: ni piezas, ni medidas, ni precio.

## Para modificarlo

Todo está en un único `<script>` al final del archivo. Puntos útiles:

| Qué | Dónde |
|---|---|
| Tolerancia de colisión (cm) | `EPS` |
| Grilla de ajuste (cm) | `GRID` |
| Medidas iniciales del piso | `floorW` / `floorD` y los `value` de los inputs |
| Valor inicial de una pieza | `makePiece()` (100 × 100 × 100) |
| Colisión y apoyo | `ov()`, `collides()`, `supportY()`, `stackedOn()` |
| Caída y reacomodo | `settle()`, `tryPlace()`, `fit()` |
| Ubicación automática | `placeNew()` |
| Handles de redimensionado | `handleMeshes`, `syncHandles()`, `sizeHandles()` |
| Resize de largo/ancho y de alto | `resizeVertex()`, `resizeHeight()`, `snapshot()`, `restore()` |
| Cancelación con Esc | `cancelResize()` |
| Cálculo de superficie, redondeo y precio | `updateStats()` |
| Cámara | `fitCamera()` |
| Estilos y layout responsive | bloque `<style>` (`#panel`, `#view`, media query) |

Si cambiás la versión de three.js, revisá que sigan existiendo
`THREE.OrbitControls` y las APIs usadas (`BoxGeometry`, `EdgesGeometry`,
`CanvasTexture`, `Raycaster`).
