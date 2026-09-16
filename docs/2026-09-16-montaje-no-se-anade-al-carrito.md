# Incidencia: el montaje no llega al carrito

**Tienda:** Carez Designs (www.carezdesigns.com)
**Detectado:** 16/09/2026, por quejas de clientes (caso concreto reportado: montaje **Trail**)
**Estado:** causa localizada, arreglo del tema pendiente

## Síntoma

Clientes que quieren el montaje acaban con el kit en el carrito sin él, o dicen
directamente que "no les deja añadirlo".

## Causa real: el montaje solo existe dentro del configurador

La ficha de producto muestra **dos botones de compra**, y el nativo del tema va
primero:

| Posición (móvil 390×844) | Botón | ¿Añade montaje? |
|---|---|---|
| y ≈ 587 | "Agregar al carrito" (nativo del tema) | **No** |
| y ≈ 740–800 | "AÑADIR AL CARRITO · Base, acabado y ficha de tu moto" (rojo, abre el configurador `bike_customizer`) | Sí |

Comprobado igual en BMW 1200 GS, Yamaha MT-07 y CFMOTO 1000MTX.

Los dos dicen prácticamente lo mismo ("añadir al carrito"), pero el montaje —y
el resto de extras— solo se puede elegir dentro del configurador, en el paso
"3. Extras y confirmación". Quien pulsa el botón nativo, que aparece antes al
hacer scroll, se lleva el kit sin montaje y sin ver en ningún momento que el
montaje existía.

## Lo que NO era

- **Las variantes de montaje están bien.** `Montaje (extra)` (producto
  `montaje-extra`, ID 16219533377923) tiene las 8 opciones disponibles, sin
  seguimiento de stock y con política `CONTINUE`.
- **Trail funciona de punta a punta hoy.** Verificado el 16/09:
  - Se añade por API en todos los mercados: España 85 €, Reino Unido 75,
    Estados Unidos 101.
  - El configurador lo añade correctamente en escritorio y en móvil
    (dos llamadas `/cart/add.js`, ambas HTTP 200, sin error visible).
  - El carrito lo conserva: kit 188 € + Montaje Trail 85 € = 273 €.
- **Cada moto ofrece su opción correcta**: trail → Trail, naked → Naked,
  scooter → Scooter, cross/supermotard → Enduro/MX/Supermoto. Ningún producto
  se queda sin opción de montaje.

## Corrección de un diagnóstico anterior

En una primera pasada se arreglaron 23 variantes de los productos ocultos
**"Montaje Kit Adhesivos"** de la app Globo Product Options (opciones NAKED,
SCOOTER y MOTO DE AGUA creadas el 13/08/2026 con seguimiento de inventario
activado, stock 0 y política `DENY`, por lo que Shopify las daba por agotadas).

El arreglo es correcto en sí mismo, pero **no afecta a lo que ve el cliente**:

- La app Globo no aparece en la ficha de producto (0 referencias en el HTML).
- Esos 8 productos ocultos no han vendido nada en 180 días.

Es decir, es una ruta muerta que quedó del sistema anterior. Trail, además,
nunca estuvo entre esas variantes bloqueadas: ya estaba disponible.

## Dato que respalda el problema

Ventas de `Montaje (extra)` por semana frente al total de pedidos:

| Semana | Pedidos | Montajes |
|---|---|---|
| 10/08 | 8 | 1 (Carretera) |
| 17/08 | 3 | 0 |
| 24/08 | 6 | 1 (Trail) |
| 31/08 | 18 | 3 (Naked, Trail, Carretera) |
| 07/09 | 10 | **0** |
| 14/09 | 5 | **0** |

15 pedidos seguidos sin un solo montaje, después de semanas con ~15% de
adjunción. Los kits se siguen vendiendo con normalidad: encaja con que los
clientes estén comprando por el botón nativo, que no pasa por el configurador.

## Arreglo propuesto (tema)

En las fichas que usan el configurador, dejar **un solo** camino de compra:

1. Ocultar el bloque nativo de compra (`product-form-component` /
   `buy-buttons-block`) cuando la sección `bike_customizer` esté activa, **o**
2. dejar el botón nativo solo como "Comprar sin personalizar" y subir el botón
   del configurador por encima, con un texto que lo distinga de verdad.

La opción 1 es la que menos ambigüedad deja. Requiere editar el tema; conviene
hacerlo sobre una copia no publicada y previsualizarla antes de publicar.

## Limpieza recomendada aparte

- Los 8 productos ocultos "Montaje Kit Adhesivos" de Globo ya no se usan: si la
  app está desinstalada, conviene archivarlos para que no confundan en informes
  ni en búsquedas del admin.
- La variante NAKED de `option-set-1305088-dropdown-1`
  (ProductVariant 57354084254083) sigue con seguimiento de inventario activado;
  irrelevante mientras esos productos no se usen.

## Cómo reproducirlo

1. Abrir cualquier ficha de kit en móvil.
2. Hacer scroll: aparece antes "Agregar al carrito" que el botón rojo del
   configurador.
3. Pulsar el nativo → el kit entra en el carrito y el montaje no se ofrece en
   ningún momento.
