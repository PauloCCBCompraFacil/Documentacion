> [!NOTE]
Este procedure porq no esta actualizado al nuevo formato  de inventario lima quitar estos 2 campos ==precio_venta_minimo== y ==esta_subido_mercado_libre== si se desea implemtar devuelta este metodo limpiar y adaptar el sp

Obtiene las unidades físicas (paginado) de un producto en inventario Lima, con filtros opcionales por estado del producto y estado general.

### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `p_id_producto_inventario` | int | ID del producto en `inventario_lima_factory` cuyas unidades se listan. |
| `page` | int | Número de página a consultar. |
| `page_size` | int | Cantidad de registros por página. |
| `p_estado_producto` | varchar | Filtro opcional por condición física (`nuevo`, `open_box`, etc.). NULL = sin filtro. |
| `p_estado_general` | varchar | Filtro opcional por estado general (`disponible`, `reservado`, etc.). NULL = sin filtro. |
| `content` (OUT) | json | Array con las unidades encontradas. |
| `pagination` (OUT) | json | Metadatos de paginación. |

### Lógica
1. Calcula el `offset` según `page` y `page_size`.
2. Cuenta el total de registros que cumplen los filtros (para armar la paginación).
3. Trae las unidades filtradas por `id_inventario_lima_factory`, `estado_producto` y `estado_general`, ordenadas por `id_producto_fisico_inventario` descendente (más recientes primero).
4. Por cada unidad devuelve: ubicación, guía/factura, precio venta mínimo, precio compra, fecha de registro, motivo, estado producto, estado general, usuario que reservó (`reservador`), procedencia, si está subido a MELI (booleano) y número de pedido.
5. Arma `pagination` con página actual, tamaño de página, total de páginas, total de ítems y flags de si hay página siguiente/anterior.

### Tablas utilizadas
- `unidad_inventario_lima` — unidades físicas de Lima (tabla principal)
- `sc_user` — resuelve el nombre del usuario que reservó la unidad (`id_usuario_reserva`)

```sql
create  procedure sp_get_unidades_inventario_lima(IN p_id_producto_inventario int, IN page int,  
                                                                   IN page_size int, IN p_estado_producto varchar(50),  
                                                                   IN p_estado_general varchar(50), OUT content json,  
                                                                   OUT pagination json)  
BEGIN  
    DECLARE offset_value INT;  
    DECLARE total_records INT;  
    DECLARE total_pages INT;  
    DECLARE has_next_page BOOLEAN;  
    DECLARE has_previous_page BOOLEAN;  
  
    SET offset_value = (page - 1) * page_size;  
  
    SELECT COUNT(*)  
    INTO total_records  
    FROM unidad_inventario_lima  
    WHERE id_inventario_lima_factory = p_id_producto_inventario  
      AND (p_estado_producto IS NULL OR estado_producto = p_estado_producto)  
      AND (p_estado_general IS NULL OR estados_general_inventario_lima = p_estado_general);  
  
    SET total_pages = CEILING(total_records / page_size);  
    SET has_next_page = (page < total_pages);  
    SET has_previous_page = (page > 1);  
  
    SELECT COALESCE(  
                   JSON_ARRAYAGG(JSON_OBJECT(  
                           'id_producto_fisico_inventario', id_producto_fisico_inventario,  
                           'descripcion_ubicacion', descripcion_ubicacion,  
                           'numero_guia_factura', numero_guia_factura,  
                           'precio_venta_minimo', precio_venta_minimo,  
                           'precio_compra', precio_compra,  
                           'fecha_registro', fecha_registro,  
                           'motivo', motivo,  
                           'estado_producto', estado_producto,  
                           'estado_general', estado_general,  
                           'reservador', reservador,  
                           'procedencia_inventario', procedencia_inventario,  
                           'subido_mercado_libre', subido_mercado_libre,  
                           'numero_pedido', numero_pedido  
                                 )),  
                   JSON_ARRAY()  
           )  
    INTO content  
    FROM (SELECT uil.id_producto_fisico_inventario,  
                 COALESCE(uil.descripcion_ubicacion, '')                        AS descripcion_ubicacion,  
                 COALESCE(uil.guia_factura, '')                                 AS numero_guia_factura,  
                 CAST(COALESCE(uil.precio_venta_minimo, 0.00) AS CHAR)          AS precio_venta_minimo,  
                 CAST(COALESCE(uil.precio_compra, 0.00) AS CHAR)                AS precio_compra,  
                 DATE_FORMAT(uil.fecha_registro, '%Y-%m-%d %H:%i:%s')           AS fecha_registro,  
                 uil.motivo,  
                 uil.estado_producto,  
                 uil.estados_general_inventario_lima                            AS estado_general,  
                 u.user_name                                                    AS reservador,  
                 uil.procedencia                                                AS procedencia_inventario,  
                 IF(uil.esta_subido_mercado_libre = 1, CAST('true' AS JSON),  
                    CAST('false' AS JSON))                                      AS subido_mercado_libre,  
                 uil.numero_pedido  
          FROM unidad_inventario_lima uil  
                   LEFT JOIN sc_user u ON uil.id_usuario_reserva = u.id  
          WHERE uil.id_inventario_lima_factory = p_id_producto_inventario  
            AND (p_estado_producto IS NULL OR uil.estado_producto = p_estado_producto)  
            AND (p_estado_general IS NULL OR uil.estados_general_inventario_lima = p_estado_general)  
          ORDER BY uil.id_producto_fisico_inventario DESC  
          LIMIT page_size OFFSET offset_value) AS unidades;  
  
    SET pagination = JSON_OBJECT(  
            'currentPage', page,  
            'perPage', page_size,  
            'totalPages', total_pages,  
            'totalItems', total_records,  
            'hasNextPage', has_next_page,  
            'hasPreviousPage', has_previous_page  
                     );  
END;
```
