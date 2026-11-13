# PLAN — Redimensionar piezas arrastrando vértices

- **Proyecto:** Simulador de depósito (`simulador-deposito.html`)
- **Estado:** implementado, verificado y publicado en `main` (commit `e5d637c`)
- **Creado:** 2026-11-12
- **Última actualización:** 2026-11-12

## Objetivo

Permitir que, con la pieza seleccionada, se cambie su tamaño arrastrando con el
mouse:

- los **4 vértices superiores**, que ajustan largo y ancho;
- un **handle en el centro de la cara superior**, que ajusta el alto.

El vértice opuesto y la base de la pieza quedan fijos. Si el nuevo tamaño
colisiona o se sale del piso, se conserva el último estado válido.

## Alcance

### Entra

- Handles visibles solo para la pieza seleccionada.
- Resize de largo/ancho desde los 4 vértices superiores.
- Resize de alto desde el handle central del techo.
- Anclaje en el vértice opuesto, base fija.
- Snap a la grilla de 10 cm cuando está activo.
- Mínimo de 10 cm con grilla y 1 cm sin grilla.
- Freno en el último estado válido ante colisión o borde del piso, con feedback
  visual rojo.
- Actualización en vivo de la geometría, la tabla y las estadísticas.
- Cancelación con **Esc**.
- Documentación en el README.

### No entra

- Vértices inferiores.
- Resize simultáneo de las 3 dimensiones con un solo vértice.
- Historial/undo global.
- Edición de la posición desde los handles (mover sigue siendo arrastrar el
  cuerpo).
- Cambios en el cálculo de superficie, redondeo o precio.

## Decisiones de diseño ya tomadas

| ID | Decisión | Motivo |
|---|---|---|
| D1 | 4 vértices superiores (XZ) + 1 handle central (alto) | Evita acoplar las 3 dimensiones en un puntero 2D y hace predecible el arrastre. |
| D2 | Ante colisión o borde: frenar en el último estado válido | Nunca deja piezas superpuestas ni fuera del piso. |
| D3 | El vértice opuesto y la base quedan fijos | Es el comportamiento esperado de un resize desde una esquina. |
| D4 | El alto se mueve por desplazamiento vertical en píxeles | Funciona igual en vista 3D y cenital; no depende de planos casi rasantes. |
| D5 | Snap del vértice arrastrado a la grilla **absoluta** (10 cm); alto a 10 cm con grilla y 1 cm sin grilla | Mantiene la coherencia con el arrastre de piezas y permite medidas finas sin grilla. |
| D6 | Mínimo: 10 cm con grilla, 1 cm sin grilla | Evita cajas de tamaño cero o negativo. |
| D7 | No se llama a `settle()` durante el arrastre; sí al soltar | Evita saltos y recalculos de apoyo en cada `pointermove`. |
| D8 | **Esc** cancela y restaura medidas, posición y tabla | La app no tiene undo; es la válvula de escape mínima. |
| D9 | Handles de tamaño constante en pantalla, azul, `depthTest:false` | Visibles y agarrables en cualquier zoom y con la pieza tapada por otra. |
| D10 | El resize escribe `fx/fz` y de ahí `l/w` según `rot` | Mantiene compatibilidad con el modelo actual y la rotación de 90°. |
| D11 | Al agarrar un vértice se compensa el offset entre el handle (en el techo) y el punto del rayo sobre el plano de la base | El handle está a `p.y + p.h` y el arrastre se calcula sobre la base; sin la compensación la pieza salta al empezar el gesto. |
| D12 | `syncHandles()` y `syncMeshes()` llaman a `updateMatrixWorld(true)`; `pickHandle()` también lo hace antes de lanzar el rayo | Sin matrices actualizadas, un clic inmediato después de mover un handle por código podía fallar. |
| D13 | Si varios handles se superponen en pantalla, gana el centro proyectado más cercano al puntero | En vistas chicas los handles se solapan y el orden del rayo elegía el equivocado (por ejemplo, el de alto al hacer hover en un vértice). |
| D14 | La rotación de 90° no cambia las columnas Largo/Ancho de la tabla: solo intercambia `fx/fz` | Aclara la semántica de D10 y evita expectativas erróneas en la documentación y en las pruebas. |

## Archivos a modificar

| Archivo | Cambio |
|---|---|
| `simulador-deposito.html` | Handles, picking, resize XZ, handle de alto, cierre/cancelación, cursor. |
| `README.md` | Nueva sección de resize, controles, reglas, limitaciones y mapa de funciones. |
| `PLAN.md` | Este archivo: checkboxes y registro de avances/desvíos. |

No se agregan dependencias ni archivos de test al repositorio: las pruebas se
hacen con páginas y scripts temporales fuera del proyecto.

## Cómo se mantiene este plan

1. Marcar `- [x]` en una subtarea **solo después** de verificarla.
2. Por cada tarea terminada, agregar una fila al final en
   [Registro de avances y desvíos](#registro-de-avances-y-desvíos).
3. Si una decisión cambia, **no borrar** la original: agregar una fila nueva con
   la decisión tomada, el motivo y el impacto.
4. Mantener la fecha de "Última actualización".

---

## Tareas

### T1 — Handles visuales

- [x] 1.1 Crear un grupo `handles` en la escena con 5 mallas: 4 esquinas
      superiores y 1 centro del techo. Material azul, `depthTest: false`,
      `renderOrder` alto y `frustumCulled: false`.
- [x] 1.2 Guardar en `userData` qué esquina representa cada malla (`sx`, `sz`) y
      marcar cuál es el handle de alto.
- [x] 1.3 Posicionar los handles en `syncMeshes()`: `p.x ± p.fx/2`,
      `p.y + p.h`, `p.z ± p.fz/2`; el central en `p.x`, `p.y + p.h`, `p.z`.
- [x] 1.4 Mostrar solo si `selected === p && p.valid`; ocultar al
      deseleccionar, eliminar o invalidar.
- [x] 1.5 Escala constante en pantalla (~7 px de radio) recalculada en
      `animate()` según `fov`, alto del canvas y distancia a la cámara.

**Criterios de aceptación**

- Al seleccionar una pieza aparecen exactamente 5 handles en las posiciones
  correctas; al cambiar de pieza se mueven; sin selección no hay ninguno.
- El tamaño visual se mantiene estable al hacer zoom y al cambiar el tamaño del
  piso.
- No hay errores en consola ni cambios en el render de las piezas.

### T2 — Picking e interacción

- [x] 2.1 Implementar `pickHandle(clientX, clientY)` con `raycaster` contra las
      mallas visibles.
- [x] 2.2 En `onDown`, prioridad **handle > pieza > vacío**; seleccionar la
      pieza y capturar el puntero al agarrar un handle.
- [x] 2.3 Extender `guard()` para que OrbitControls no orbite al agarrar un
      handle.
- [x] 2.4 Deshabilitar `controls` al iniciar el resize y restaurarlo al
      terminar.
- [x] 2.5 Cursor de hover: `nwse-resize` en esquinas, `ns-resize` en el handle
      de alto, vacío en el resto.

**Criterios de aceptación**

- Agarrar un handle nunca orbita la cámara ni mueve la pieza.
- Arrastrar el cuerpo de la pieza sigue funcionando como hoy.
- Clic en vacío deselecciona y oculta los handles.
- Touch y mouse se comportan igual (mismo flujo de pointer events).

### T3 — Resize de largo y ancho

- [x] 3.1 Guardar un snapshot al iniciar (`l`, `w`, `fx`, `fz`, `h`, `x`, `z`,
      `y`) y calcular el ancla como el vértice opuesto sobre la base.
- [x] 3.2 Calcular la intersección del rayo con el plano horizontal que pasa por
      `anchor.y`.
- [x] 3.3 Clampear el vértice al interior del piso; aplicar mínimo y evitar que
      la caja se invierta al cruzar el ancla.
- [x] 3.4 Snap del vértice a la grilla absoluta de 10 cm si está activo; si no,
      resolución de 1 cm.
- [x] 3.5 Recalcular el centro y `fx/fz`, y traducir a `l/w` según `rot`.
- [x] 3.6 Validar con `collides(p, x, p.y, z)`; si falla, revertir dimensiones y
      centro al último estado válido y pintar el handle de rojo.
- [x] 3.7 Actualizar `rebuildMesh`, `syncMeshes` (incluye handles),
      `updateStats` y los inputs de la fila sin disparar eventos `input`.

**Criterios de aceptación**

- El vértice opuesto no se mueve y la base permanece en `p.y`.
- Las medidas resultantes son las esperadas con snap activado y desactivado, y
  respetan el mínimo.
- Nunca se superpone con otra pieza: al bloquear, las medidas no cambian y el
  handle se ve rojo.
- Tabla, geometría y estadísticas coinciden.
- Funciona con la pieza rotada 90°.

### T4 — Resize de alto

- [x] 4.1 Al agarrar el handle central, guardar `h` inicial y `clientY` inicial.
- [x] 4.2 Convertir píxeles a centímetros con
      `worldPerPixel = 2 * tan(fov/2) * distancia / altoCanvas` y calcular el
      nuevo alto.
- [x] 4.3 Aplicar mínimo y snap (10 cm con grilla; 1 cm sin grilla).
- [x] 4.4 Validar colisión con `collides`; frenar en el último válido y marcar
      rojo.
- [x] 4.5 Actualizar geometría, handles, tabla y estadísticas.

**Criterios de aceptación**

- Arrastrar hacia arriba agranda y hacia abajo achica, tanto en vista 3D como
  cenital.
- La base queda fija y la pieza no atraviesa una pieza que tenga arriba.
- Tabla y estadísticas reflejan el nuevo alto.
- Al soltar, las piezas apiladas encima se reacomodan con `settle`.

### T5 — Cierre y cancelación

- [x] 5.1 En `endDrag`, si había resize: liberar captura, `settle(null)`,
      `syncMeshes`, `updateStats`, restaurar color del handle y reactivar
      `controls`.
- [x] 5.2 Con **Esc** durante el resize: restaurar el snapshot (medidas y
      posición), reconstruir, sincronizar, actualizar estadísticas y cancelar
      sin `settle`.
- [x] 5.3 Tratar `pointercancel` igual que el cierre normal.
- [x] 5.4 Evitar estados colgados: segundo `pointerdown` durante un resize,
      puntero liberado fuera del canvas y pieza eliminada durante el arrastre.

**Criterios de aceptación**

- Al soltar no queda ningún handle rojo ni `controls` deshabilitado.
- Esc devuelve exactamente el estado previo, incluida la tabla.
- Cancelar con el mouse fuera del canvas funciona.
- No se puede iniciar un resize con otra pieza mientras hay uno activo.

### T6 — README

- [x] 6.1 Nueva sección **Redimensionar con el mouse** con el esquema de
      handles.
- [x] 6.2 Actualizar la tabla de controles (vértices XZ y handle de alto).
- [x] 6.3 Documentar anclaje, base fija, mínimo, snap, rojo = bloqueado y Esc.
- [x] 6.4 Aclarar en limitaciones que no hay undo global.
- [x] 6.5 Agregar las funciones nuevas a la tabla "Para modificarlo".

**Criterios de aceptación**

- El README describe exactamente el comportamiento implementado.
- No promete funcionalidades que no existen.

### T7 — Verificación

- [x] 7.1 Probar la matemática del resize (función pura si se extrae) en Edge
      headless con al menos 12 vectores: exacto, mínimo, cruce del ancla, borde
      del piso, snap on/off y pieza rotada.
- [x] 7.2 Cargar la página real en Edge headless: 5 handles solo al seleccionar,
      sin errores de consola y con estadísticas correctas.
- [x] 7.3 Intentar un arrastre sintético con `PointerEvent` replicando la
      proyección de cámara; si resulta frágil, documentarlo y cubrirlo con
      prueba manual.
- [x] 7.4 Regresión: mover, apilar, rotar, cambiar el piso y facturación
      (incluido 2,1 m² → 3 m²).
- [x] 7.5 Prueba manual: zoom extremo, touch, pieza rotada, piso máximo y
      solapamiento con vecinas.

**Criterios de aceptación**

- Todas las pruebas pasan o quedan documentadas con el motivo de la excepción.
- No hay regresiones en mover, apilar, rotar, piso ni facturación.

### T8 — Git

- [x] 8.1 Revisar el diff completo y confirmar que no hay archivos temporales ni
      cambios ajenos al plan.
- [x] 8.2 Commit y push a `main` en `takacondaga/simulador-deposito`, incluido
      este `PLAN.md`.

**Criterios de aceptación**

- Árbol limpio después del push.
- El remoto queda actualizado y con el mismo contenido que el local.

---

## Riesgos y mitigaciones

| Riesgo | Nivel | Mitigación |
|---|---|---|
| Ambigüedad puntero 2D → 3D | Medio | Vértices solo para XZ; alto en handle separado (D1, D4). |
| Conflicto con OrbitControls | Bajo | Picking de handles primero y `guard()` ampliado (T2). |
| Piezas apiladas que pierden apoyo | Medio | Sin `settle` durante el arrastre; `settle(null)` al soltar (D7, T5). |
| Salirse del piso al resizear | Bajo | Clamp del vértice al borde interior antes de calcular (T3.3). |
| Rebote por píxeles | Bajo | Snap, mínimo y freno en el último válido (D2, D5, D6). |
| Estado colgado tras cancelar | Bajo | Esc, `pointercancel` y limpieza en `endDrag` (T5). |
| Proyección de cámara frágil en tests | Medio | Probar la matemática por separado y aceptar verificación manual del gesto (T7.3). |

## Definición de terminado

- Los 5 handles funcionan con mouse y touch.
- Resize XZ y de alto con anclaje correcto, mínimo, snap y freno en colisión.
- Tabla, geometría y estadísticas siempre consistentes.
- Esc cancela sin efectos secundarios.
- Sin errores de consola y sin regresiones.
- README actualizado.
- Commit subido a `main` y este plan con todas las tareas verificadas.

## Registro de avances y desvíos

| Fecha | Tarea | Decisión / desvío | Motivo |
|---|---|---|---|
| 2026-11-12 | Plan | Creado el plan; implementación no iniciada | Pedido explícito de no escribir código todavía |
| 2026-11-12 | T1 | Handles implementados: grupo con 4 vértices superiores y 1 centro del techo, azul `0x1f6feb`, `depthTest:false`, `renderOrder` 1000 y radio constante de 7 px. | Verificado en el test principal (visibilidad, ocultamiento y cursores) y en la captura de producción. |
| 2026-11-12 | T2 | Picking con prioridad handle > pieza > vacío, `guard()` ampliado y cursores `nwse-resize` / `ns-resize`. Se agregó D13 para desempatar handles superpuestos. | Verificado con mouse y con punteros táctiles sintéticos. |
| 2026-11-12 | T3 | Resize XZ anclado en el vértice opuesto, snap absoluto, mínimo y freno en el último estado válido. Se agregaron D11 (compensación de offset) y D12 (matrices al día). | Casos A–E, G, K y prueba aparte de pieza rotada: 12/12. |
| 2026-11-12 | T4 | Resize de alto por desplazamiento vertical en píxeles, snap de 10/1 cm, mínimo y colisión. | Caso F, Esc, touch, vista cenital y piso máximo. |
| 2026-11-12 | T5 | Esc, `pointercancel`, segundo `pointerdown` ignorado, pieza eliminada durante el arrastre y cierre con el puntero fuera del canvas. | 36/36 en el test principal. |
| 2026-11-12 | T6 | README actualizado con la sección de resize, controles, limitaciones y mapa de funciones. Se aclaró la semántica de rotación (D14). | Revisión del diff. |
| 2026-11-12 | T7 | 70 checks en Edge headless: 36 principales, 12 de rotación, 13 extra (zoom, cenital y piso máximo) y 9 de producción. Desvío: no se extrajo una función pura; la matemática se validó con un oráculo independiente y arrastres sintéticos. | Todas las suites en verde, sin errores JS. |
| 2026-11-12 | T7 | El arrastre sintético con `PointerEvent` resultó estable, así que T7.3 no requirió prueba manual. La cobertura táctil se hizo con `pointerType: "touch"` (vértice y alto, incluida vista cenital); no se probó en un dispositivo táctil físico. | Limitación del entorno headless, documentada. |
| 2026-11-12 | T8 | Diff revisado: solo `simulador-deposito.html`, `README.md` y `PLAN.md`; sin hook `__dbg` ni archivos temporales en el repo. | `git status` con solo los cambios previstos; los tests viven en `%TEMP%`. |
| 2026-11-12 | T8 | Push a `main` verificado: `d83637a..e5d637c` en `takacondaga/simulador-deposito`; el remoto quedó con el mismo contenido que el local. | `git push` sin errores y `git status -sb` sin diferencias con `origin/main`. |
