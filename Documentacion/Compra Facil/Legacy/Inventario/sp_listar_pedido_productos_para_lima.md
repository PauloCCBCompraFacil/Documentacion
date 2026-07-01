
Lista los productos (`pedidos_productos`) de un pedido específico, para selección en el registro de inventario Lima.

### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `p_numero_pedido` | varchar | Número de pedido a consultar. |
| `content` (OUT) | json | Array con los productos del pedido. |
| `p_resultado` (OUT) | int | 1 = éxito, 0 = error (pedido no encontrado). |
| `p_mensaje` (OUT) | varchar | Mensaje descriptivo del resultado. |

### Lógica
Busca el pedido por `numero_pedido`. Si no existe, retorna `content` vacío y `p_resultado = 0`. Si existe, retorna un JSON por cada línea de producto con: `id_pedido_producto`, `sku_producto`, `nombre_producto`, `cantidad`, `imagen_producto`, `logistic_status` y `id_logistic_status`.

### Tablas utilizadas
- `pedidos` — resuelve `id_pedido` por número de pedido
- `pedidos_productos` — líneas de producto del pedido (tabla principal)
- `logistic_status` — nombre del estado logístico
- `productos` — imagen del producto (por `sku_cf`)

```sql
create  
    definer = anders@`%` procedure sp_listar_pedido_productos_para_lima(IN p_numero_pedido varchar(100),  
                                                                        OUT content json, OUT p_resultado int,  
                                                                        OUT p_mensaje varchar(500))  
BEGIN  
    DECLARE v_id_pedido INT;  
  
    SELECT id_pedido  
    INTO v_id_pedido  
    FROM pedidos  
    WHERE numero_pedido COLLATE utf8mb4_unicode_ci = p_numero_pedido COLLATE utf8mb4_unicode_ci  
    LIMIT 1;  
  
    IF v_id_pedido IS NULL THEN  
        SET p_resultado = 0;  
        SET p_mensaje = CONCAT('Pedido ', p_numero_pedido, ' no encontrado');  
        SET content = JSON_ARRAY();  
    ELSE  
        SELECT COALESCE(  
                       JSON_ARRAYAGG(JSON_OBJECT(  
                               'id_pedido_producto', pp.id_pedido_producto,  
                               'sku_producto', pp.sku_producto,  
                               'nombre_producto', pp.nombre_producto,  
                               'cantidad', pp.cantidad,  
                               'imagen_producto', COALESCE(pr.main_image_url, pr.image_0_webp, ''),  
                               'logistic_status', ls.status,  
                               'id_logistic_status', pp.id_logistic_status  
                                     )),  
                       JSON_ARRAY()  
               )  
        INTO content  
        FROM pedidos_productos pp  
                 LEFT JOIN logistic_status ls ON ls.id_logistic_status = pp.id_logistic_status  
                 LEFT JOIN productos pr  
                           ON pr.sku COLLATE utf8mb4_unicode_ci = pp.sku_cf COLLATE utf8mb4_unicode_ci  
        WHERE pp.id_pedido = v_id_pedido  
        ORDER BY pp.id_pedido_producto ASC;  
  
        SET p_resultado = 1;  
        SET p_mensaje = 'OK';  
    END IF;  
END;

```