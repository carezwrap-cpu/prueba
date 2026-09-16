# Incidencia: la opción de montaje no se añadía al carrito

**Tienda:** Carez Designs (www.carezdesigns.com)
**Detectado:** 16/09/2026, por quejas de clientes
**Estado:** resuelto

## Síntoma

Los clientes elegían la opción "Montaje" en la ficha de producto, pero el montaje
no llegaba al carrito. El kit de adhesivos entraba solo, sin el extra de montaje.

## Causa

El montaje se cobra mediante la app **Globo Product Options**, que lo implementa
con productos ocultos (`status: UNLISTED`, etiqueta `globo-product-options`)
llamados "Montaje Kit Adhesivos": uno por cada set de opciones. Cada opción del
desplegable es una variante de ese producto oculto, y al añadir al carrito la app
mete esa variante como línea adicional.

El 13/08/2026 (17:04–17:05 UTC) se añadieron tres opciones nuevas —**NAKED**,
**SCOOTER** y **MOTO DE AGUA**— a los 8 sets de opciones. Esas variantes se
crearon con la configuración por defecto de Shopify:

- seguimiento de inventario **activado** (`inventoryItem.tracked: true`)
- política **`DENY`** (no vender sin stock)
- stock **0**

Resultado: `availableForSale: false`. Shopify rechaza añadirlas al carrito por
estar agotadas, así que la línea de montaje se cae silenciosamente.

Las opciones antiguas (MOTO CARRETERA, ENDURO/MX/SUPERMOTARD, TRAIL, QUAD,
creadas el 20/01/2026) tienen el seguimiento desactivado, por eso esas sí
funcionaban. De ahí que el fallo pareciera aleatorio: dependía del tipo de moto
que eligiera el cliente.

## Alcance

23 variantes bloqueadas en los 8 productos "Montaje Kit Adhesivos":

| Producto (handle) | Product ID | Opciones afectadas |
|---|---|---|
| option-set-1311594-dropdown-1 | 15589713674627 | NAKED, SCOOTER, MOTO DE AGUA |
| template-54683-dropdown-1 | 15589716164995 | NAKED, SCOOTER, MOTO DE AGUA |
| template-54684-dropdown-1 | 15589716328835 | NAKED, SCOOTER, MOTO DE AGUA |
| option-set-1311596-dropdown-1 | 15589729534339 | NAKED, SCOOTER, MOTO DE AGUA |
| option-set-1311597-dropdown-1 | 15589742936451 | NAKED, SCOOTER, MOTO DE AGUA |
| option-set-1329073-dropdown-1 | 15589765513603 | NAKED, SCOOTER, MOTO DE AGUA |
| option-set-1305088-dropdown-1 | 15589773246851 | SCOOTER, MOTO DE AGUA |
| option-set-1305305-dropdown-1 | 15589780128131 | NAKED, SCOOTER, MOTO DE AGUA |

NAKED afecta a buena parte del catálogo (MT-07, Tmax, etc.), así que el impacto
en ventas de montaje fue alto durante ese mes.

El resto de opciones de pago (Base del Kit Gráfico, Acabado, Espesor del
Material, Tipo De Vehículo, Adaptación Diseño, Modelo De Moto) y los productos
extra sueltos (Montaje (extra), Llantas personalizadas, Cambio de diseño,
Stickers de Instagram) estaban bien configurados y no se han tocado.

## Arreglo aplicado

Para las 23 variantes: seguimiento de inventario **desactivado** y política
**`CONTINUE`**, igual que las opciones que ya funcionaban. El montaje es un
servicio, no tiene stock que controlar.

```graphql
mutation FixMontaje($productId: ID!, $variants: [ProductVariantsBulkInput!]!) {
  productVariantsBulkUpdate(productId: $productId, variants: $variants) {
    productVariants {
      id
      title
      availableForSale
      inventoryPolicy
      inventoryItem { tracked }
    }
    userErrors { field message }
  }
}
```

Con `variants: [{ id, inventoryPolicy: CONTINUE, inventoryItem: { tracked: false } }]`
por cada variante afectada.

Verificado tras el cambio: las 56 variantes de los 8 sets (7 opciones × 8)
devuelven `availableForSale: true`.

## Cómo evitar que vuelva a pasar

Al añadir una opción de pago nueva en Globo Product Options, Shopify crea la
variante con seguimiento de inventario activado y `DENY` por defecto. Hay que
desactivar el seguimiento en esa variante nada más crearla, o la opción nacerá
agotada y no se podrá añadir al carrito.

Comprobación rápida de que no hay ninguna opción bloqueada:

```graphql
query {
  products(first: 50, query: "tag:globo-product-options") {
    edges { node { handle variants(first: 20) { edges { node { title availableForSale } } } } }
  }
}
```

Cualquier variante con `availableForSale: false` es una opción que el cliente
puede elegir pero no comprar.

## Pendiente (menor)

La variante NAKED de `option-set-1305088-dropdown-1`
(ProductVariant 57354084254083) sigue con el seguimiento de inventario activado,
aunque con política `CONTINUE`, así que funciona. Conviene desactivarle el
seguimiento para dejar los 8 sets con la misma configuración y que no se rompa
si alguien vuelve a ponerla en `DENY`.
