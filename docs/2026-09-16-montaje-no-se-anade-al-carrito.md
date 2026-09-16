# Incidencia: el montaje no llega al carrito

**Tienda:** Carez Designs (www.carezdesigns.com)
**Detectado:** 16/09/2026, por quejas de clientes (caso concreto reportado: montaje **Trail**)
**Estado:** arreglo hecho y probado en el borrador **V78**; falta publicarlo

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

## Arreglo aplicado

El conector de Shopify bloquea la escritura sobre el tema publicado, así que el
cambio se ha hecho sobre un duplicado exacto del tema en vivo:

- **Tema en vivo:** V77 (`gid://shopify/OnlineStoreTheme/199615840643`)
- **Borrador con el arreglo:** **V78 — un solo boton de compra en fichas con
  configurador** (`gid://shopify/OnlineStoreTheme/199679803779`), duplicado de
  V77 el 16/09/2026.

Dos archivos:

1. **`snippets/carez-un-boton-compra.liquid`** (nuevo, 1876 bytes). Oculta el
   bloque de compra nativo, pero solo en las páginas donde existe el
   configurador:

   ```css
   body:has([data-cz-dialog]) .buy-buttons-block,
   body:has([data-cz-dialog]) sticky-add-to-cart {
     display: none !important;
   }
   ```

   Incluye un respaldo en JS para navegadores sin `:has()`, que hace lo mismo y
   nada más.

2. **`layout/theme.liquid`**: una sola línea añadida, junto al resto de snippets
   `carez-*`:

   ```diff
      {% render 'carez-montaje-tipo' %}
   +  {% render 'carez-un-boton-compra' %}
      {% render 'carez-tipografia' %}
   ```

   La copia local desde la que se generó el cambio se verificó byte a byte
   contra el archivo en vivo (MD5 `7e56f154628e8c8fa5b2af1c31aeef1e`, 8086
   bytes) antes de tocar nada, y los dos archivos subidos devolvieron el MD5
   esperado.

### Por qué se oculta y no se elimina

El configurador añade el kit haciendo `submitButton.click()` sobre el formulario
nativo del tema (`product-form-component form[data-type="add-to-cart-form"]`), y
ese formulario es también el que sube las fotos en multipart. Si se quita el
bloque, el configurador cae a un añadido directo que **pierde las fotos**. Un
`display:none` no impide el click programático, así que el formulario sigue
haciendo su trabajo desde dentro del configurador.

No se ha cambiado el texto de ningún botón.

### Comprobaciones hechas

| Comprobación | Resultado |
|---|---|
| Botón "Agregar al carrito" dentro de `.buy-buttons-block` | Sí (52×264 px) |
| Con el CSS aplicado | 0 px, no visible |
| Formulario y botón submit siguen en el DOM y habilitados | Sí |
| Flujo completo en móvil (kit + montaje Trail) | 2 × `/cart/add` HTTP 200 |
| Carrito resultante | Kit 188 € + Montaje Trail 85 € = **273 €** |
| Fichas **con** configurador (BMW GS, MT-07, kit personalizado) | Botón nativo oculto |
| Fichas **sin** configurador (montaje-extra, vinilos-llanta-yamaha-gt) | Botón nativo visible, intactas |

Esa última fila es la que evita el riesgo grave: la regla está anclada a
`[data-cz-dialog]`, que solo existe donde hay configurador, así que ningún
producto se queda sin forma de comprarse.

### Cómo publicarlo

Previsualización:
`https://www.carezdesigns.com/products/adhesivos-para-moto-bmw-1200-gs-2014-2016?preview_theme_id=199679803779`

Opción A — publicar V78 desde el admin (Tienda online → Temas → V78 →
Publicar). El borrador es copia de V77 del 16/09: cualquier cambio hecho en V77
después de esa fecha no está incluido.

Opción B — mantener V77 como tema en vivo y aplicar los dos cambios a mano en
Tienda online → Temas → V77 → Editar código: crear
`snippets/carez-un-boton-compra.liquid` con el contenido de arriba y añadir la
línea `{% render 'carez-un-boton-compra' %}` en `layout/theme.liquid`.

Para revertirlo, basta con quitar esa línea de `layout/theme.liquid`.

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
