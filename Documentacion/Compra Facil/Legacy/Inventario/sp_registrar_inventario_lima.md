Registra unidades físicas en el inventario de Lima. El flujo ejecutado depende de `p_procedencia`.

### Parámetros

| Parámetro                 | Tipo    | Descripción                                                                             |     |
| ------------------------- | ------- | --------------------------------------------------------------------------------------- | --- |
| `p_sku_producto`          | varchar | SKU del producto. Usado en `inventario_usa`, `compra_peru`, `libre`.                    |     |
| `p_procedencia`           | enum    | Flujo a ejecutar: `pedido`, `pedido_erroneo`, `inventario_usa`, `compra_peru`, `libre`. |     |
| `p_numero_detalle`        | varchar | Guía (USA), factura/boleta (Perú) o referencia (libre).                                 |     |
| `p_descripcion_ubicacion` | varchar | Ubicación física en almacén Lima.                                                       |     |
| `p_estado_producto`       | enum    | Condición: `nuevo`, `open_box`, `caja_golpeada`, `sin_empaque_original`.                |     |
| `p_id_usuario`            | int     | Usuario que ejecuta la acción.                                                          |     |
| `p_cantidad`              | int     | Cantidad de unidades a crear (obligatorio en `inventario_usa`, `compra_peru`, `libre`). |     |
| `p_comentario`            | text    | Comentario libre, opcional. Si vacío, se autogenera uno.                                |     |
| `p_datos`                 | json    | Array con detalle por línea. Obligatorio en `pedido` y `pedido_erroneo`.                |     |
| `p_resultado` (OUT)       | int     | 1 = éxito, 0 = error.                                                                   |     |
| `p_mensaje` (OUT)         | varchar | Mensaje descriptivo del resultado o error.                                              |     |

### Flujos

**1. `inventario_usa`** — Recibe unidades desde una transición USA→Lima ya `recepcionado` (tabla `tb_tranciciones`, filtrada por guía + SKU). Descuenta unidades USA disponibles y crea sus equivalentes en Lima.

**2. `compra_peru`** — Crea unidades nuevas sin origen USA, asociadas a una factura/boleta local.

**3. `pedido`** — Recibe `p_datos` con lista de `id_pedido_producto`. Por cada uno, busca la unidad USA vinculada al pedido y la traslada a Lima, marcándola `enviado_lima`.

**4. `pedido_erroneo`** — Maneja errores de SKU en pedidos, con 3 sub-opciones combinables vía flags en `p_datos`:

- **Opción 1** (`descontar_unidades_usa`): descuenta del stock disponible de `sku_llegado` (SKU incorrecto que llegó).
- **Opción 2** (`cambiar_sku_unidad`): cambia el SKU/inventario de la unidad ya vinculada al pedido, sin tocar stock.
- **Opción 3** (`regresar_a_procesado`): desvincula la unidad del pedido y regresa `logistic_status` a `PROCESADO` (1). Corre después de la opción 2.

Opciones 1 y 2 son mutuamente excluyentes.

**5. `libre`** — Crea unidades sin vínculo a pedido ni a USA. Precio de compra opcional vía `p_datos[0].precio_compra`.

### Tablas utilizadas

**Lectura/escritura principal:**

- `productos` — catálogo, resuelve `id_producto` por SKU
- `inventario_lima_factory` — catálogo de productos registrados en Lima
- `unidad_inventario_lima` — unidad física creada en Lima (tabla destino principal)
- `historial_unidad_inventario_lima` — historial de movimientos de cada unidad Lima

**Origen USA:**

- `tb_unidad_inventario_usa` — unidades físicas en USA
- `tb_inventario_usa` — catálogo de inventario USA
- `tb_historial_acciones_unidad_inventario` — historial de acciones sobre unidades USA
- `tb_tranciciones` — transiciones de envío USA→Lima
- `inventario_proveedor_info` — factura y precio unitario de compra

**Pedidos:**

- `pedidos` — cabecera del pedido
- `pedidos_productos` — línea de producto del pedido
- `logistica_usa` — logística de asignación en USA
- `logistica_lima` — logística de distribución en Lima
- `logistica_lima_documentos`, `logistica_lima_inv_doc`, `logistica_lima_ticket_doc` — documentos asociados (solo en limpieza de op3)
- `historial_estado_pedido` — historial de cambios de estado del pedido
- `logistic_status` — catálogo de estados logísticos

**Otros:**

- `sc_user` — datos del usuario ejecutor

### Procedimientos externos invocados

- `sp_es_producto_espejo` — resuelve si un SKU es "espejo" de otro y redirige al SKU primario.

```SQL
DELIMITER $$
DROP PROCEDURE IF EXISTS sp_registrar_inventario_lima;
create  procedure sp_registrar_inventario_lima(IN p_sku_producto varchar(255),  
                                                                IN p_procedencia enum ('pedido', 'pedido_erroneo', 'inventario_usa', 'compra_peru', 'libre'),  
                                                                IN p_numero_detalle varchar(100),  
                                                                IN p_descripcion_ubicacion varchar(500),  
                                                                IN p_estado_producto enum ('nuevo', 'open_box', 'caja_golpeada', 'sin_empaque_original'),  
                                                                IN p_id_usuario int, IN p_cantidad int,  
                                                                IN p_comentario text, IN p_datos json,  
                                                                OUT p_resultado int, OUT p_mensaje varchar(500))  
sp_block: BEGIN  
  
    -- ── Variables generales ───────────────────────────────────────────────────  
    DECLARE v_id_producto               INT;  
    DECLARE v_id_inventario_factory     INT;  
    DECLARE v_nombre_usuario            VARCHAR(40);  
    DECLARE v_error_handler             INT DEFAULT 0;  
    DECLARE v_comentario_final          TEXT;  
    DECLARE v_id_unidad_creada          BIGINT UNSIGNED;  
    DECLARE v_index                     INT DEFAULT 0;  
    DECLARE v_unidades_a_crear          INT DEFAULT 0;  
  
    -- ── Espejo ────────────────────────────────────────────────────────────────  
    DECLARE v_es_espejo                 TINYINT(1) DEFAULT 0;  
    DECLARE v_sku_primario_espejo       VARCHAR(255);  
    DECLARE v_id_primario_espejo        INT;  
    DECLARE v_sku_original_espejo       VARCHAR(255);  
  
    -- ── Transición (inventario_usa) ───────────────────────────────────────────  
    DECLARE v_id_transicion             BIGINT UNSIGNED;  
    DECLARE v_estado_transicion         VARCHAR(50);  
    DECLARE v_cantidad_en_movimiento    INT;  
    DECLARE v_cantidad_llegada          INT;  
    DECLARE v_cantidad_faltante         INT;  
  
    -- ── Iteración datos[] ────────────────────────────────────────────────────  
    DECLARE v_total_datos               INT DEFAULT 0;  
    DECLARE v_idx                       INT DEFAULT 0;  
    DECLARE v_id_pp                     BIGINT UNSIGNED;  
    DECLARE v_sku_llegado               VARCHAR(255);  
    DECLARE v_sku_cambio                VARCHAR(255);  
  
    DECLARE v_sku_original              VARCHAR(255);  
    DECLARE v_sku_usar                  VARCHAR(255);  
    DECLARE v_cantidad_pp               INT;  
    DECLARE v_id_inv_factory_item       INT;  
    DECLARE v_id_producto_item          INT;  
    DECLARE v_id_unidad_lima_item       BIGINT UNSIGNED;  
    DECLARE v_id_unidad_usa_item        BIGINT UNSIGNED;  
    DECLARE v_i                         INT DEFAULT 0;  
  
    -- ── OPCIÓN 1: descontar_unidades_usa ─────────────────────────────────────  
    DECLARE v_id_inventario_usa_llegado BIGINT;  
    DECLARE v_stock_usa_disponible      INT DEFAULT 0;  
    DECLARE v_id_unidad_usa_llegado     BIGINT UNSIGNED;  
  
    -- ── OPCIÓN 2: cambiar_sku_unidad (variable dedicada, no comparte con op1) ─  
    DECLARE v_id_inv_usa_cambio         BIGINT;  
    DECLARE v_sku_final_unidad          VARCHAR(255);   -- SKU real de la unidad tras el cambio  
  
    DECLARE v_ls_pedido_check           INT;  
    DECLARE v_ls_fuente_actual          INT;  
  
    DECLARE v_numero_pedido_pp          VARCHAR(100);  
    DECLARE v_id_pedido                 BIGINT UNSIGNED;  
    DECLARE v_id_logistica_lima         BIGINT UNSIGNED;  
    DECLARE v_id_logistica_lima_docs    BIGINT UNSIGNED;  
    DECLARE v_factura_unidad_usa        VARCHAR(100);  
    DECLARE v_precio_unidad_usa         DECIMAL(16,4);  
    DECLARE v_id_origen_unidad          BIGINT UNSIGNED;  
    DECLARE v_id_unidad_original        BIGINT UNSIGNED;  
  
    DECLARE v_precio_compra_libre       DECIMAL(16,4) DEFAULT NULL;  
  
    DECLARE v_regresar_procesado        TINYINT(1) DEFAULT 0;  
    DECLARE v_descontar_usa             TINYINT(1) DEFAULT 0;  
    DECLARE v_cantidad_descontar        INT DEFAULT 0;  
    DECLARE v_cambiar_sku_unidad        TINYINT(1) DEFAULT 0;  
  
    -- ── Captura de error SQL ──────────────────────────────────────────────────  
    DECLARE v_sql_state                 CHAR(5)      DEFAULT '00000';  
    DECLARE v_error_code                INT          DEFAULT 0;  
    DECLARE v_error_msg                 VARCHAR(500) DEFAULT '';  
  
    -- ── Handler: captura estado/código/mensaje exacto del error SQL ───────────  
    DECLARE CONTINUE HANDLER FOR SQLEXCEPTION  
    BEGIN        GET DIAGNOSTICS CONDITION 1  
            v_sql_state  = RETURNED_SQLSTATE,  
            v_error_code = MYSQL_ERRNO,  
            v_error_msg  = MESSAGE_TEXT;  
        SET v_error_handler = 1;  
        ROLLBACK;  
    END;  
  
    START TRANSACTION;  
  
    SELECT user_name INTO v_nombre_usuario FROM sc_user WHERE id = p_id_usuario LIMIT 1;  
  
    -- ══════════════════════════════════════════════════════════════════════════  
    -- FLUJO: INVENTARIO USA    -- ══════════════════════════════════════════════════════════════════════════    IF p_procedencia = 'inventario_usa' THEN  
  
        IF p_cantidad IS NULL OR p_cantidad <= 0 THEN  
            SET p_resultado = 0; SET p_mensaje = 'inventario_usa: la cantidad debe ser mayor a 0';  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SET v_es_espejo = 0; SET v_sku_primario_espejo = NULL; SET v_id_primario_espejo = NULL;  
        CALL sp_es_producto_espejo(p_sku_producto, v_es_espejo, v_sku_primario_espejo, v_id_primario_espejo);  
  
        IF v_es_espejo = 1 THEN  
            SET v_sku_original_espejo = p_sku_producto;  
            SET p_sku_producto        = v_sku_primario_espejo;  
        ELSE  
            SET v_sku_original_espejo = NULL;  
        END IF;  
  
        SELECT id INTO v_id_producto FROM productos  
        WHERE  sku COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_producto IS NULL THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('inventario_usa: producto SKU "', p_sku_producto, '" no encontrado en catálogo');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SELECT id_producto_inventario INTO v_id_inventario_factory  
        FROM   inventario_lima_factory  
        WHERE  id_producto = v_id_producto  
          AND  sku_registrado COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_inventario_factory IS NULL THEN  
            INSERT INTO inventario_lima_factory (id_producto, sku_registrado) VALUES (v_id_producto, p_sku_producto);  
            SET v_id_inventario_factory = LAST_INSERT_ID();  
        END IF;  
  
        SELECT id_transicion, estado, cantidad_en_movimiento, COALESCE(cantidad_llegada, 0)  
        INTO   v_id_transicion, v_estado_transicion, v_cantidad_en_movimiento, v_cantidad_llegada  
        FROM   tb_tranciciones  
        WHERE  guia         COLLATE utf8mb4_unicode_ci = p_numero_detalle COLLATE utf8mb4_unicode_ci  
          AND  sku_producto COLLATE utf8mb4_unicode_ci = p_sku_producto   COLLATE utf8mb4_unicode_ci  
        LIMIT  1;  
  
        IF v_id_transicion IS NULL THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('inventario_usa: no se encontró transición con guía "',  
                p_numero_detalle, '" y SKU "', p_sku_producto, '"');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        IF v_estado_transicion != 'recepcionado' THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('inventario_usa: la transición #', v_id_transicion,  
                ' (guía "', p_numero_detalle, '") está en estado "', v_estado_transicion,  
                '", se requiere "recepcionado"');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SET v_cantidad_faltante = v_cantidad_en_movimiento - v_cantidad_llegada;  
  
        IF p_cantidad > v_cantidad_faltante THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('inventario_usa: intentas registrar ', p_cantidad,  
                ' unidades pero solo faltan ', v_cantidad_faltante,  
                ' de ', v_cantidad_en_movimiento, ' totales (ya llegaron: ', v_cantidad_llegada, ')');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        UPDATE tb_tranciciones  
        SET    cantidad_llegada = v_cantidad_llegada + p_cantidad  
        WHERE  id_transicion = v_id_transicion;  
  
        SET v_unidades_a_crear = p_cantidad;  
        SET v_comentario_final = COALESCE(NULLIF(p_comentario, ''),  
            CONCAT('Recepción de ', p_cantidad, ' unidad(es) de inventario USA con guía ',  
                   p_numero_detalle, ' (SKU: ', p_sku_producto,  
                   IF(v_sku_original_espejo IS NOT NULL,  
                      CONCAT(', redirigido desde espejo: ', v_sku_original_espejo), ''),  
                   ') en fecha ', DATE_FORMAT(NOW(), '%d/%m/%Y'),  
                   ' | Llegadas: ', v_cantidad_llegada + p_cantidad, '/', v_cantidad_en_movimiento));  
  
        SET v_index = 0;  
        WHILE v_index < v_unidades_a_crear DO  
  
            SET v_id_origen_unidad   = NULL;  
            SET v_factura_unidad_usa = NULL;  
            SET v_precio_unidad_usa  = NULL;  
  
            SELECT h.id_inventario_usa_unidad, ipi.codigo_factura, ipi.precio_unitario  
            INTO   v_id_origen_unidad, v_factura_unidad_usa, v_precio_unidad_usa  
            FROM   tb_historial_acciones_unidad_inventario h  
            INNER JOIN tb_unidad_inventario_usa ui ON ui.id_inventario_unidad = h.id_inventario_usa_unidad  
            LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
            WHERE  ui.id_transicion = v_id_transicion  
              AND  h.accion = 'enviado_lima'  
              AND  NOT EXISTS (  
                  SELECT 1 FROM unidad_inventario_lima uil WHERE uil.id_origen = h.id_inventario_usa_unidad  
              )  
            ORDER  BY h.fecha_registro ASC LIMIT 1;  
  
            IF v_id_origen_unidad IS NULL THEN  
                SELECT ui.id_inventario_unidad, ipi.codigo_factura, ipi.precio_unitario  
                INTO   v_id_origen_unidad, v_factura_unidad_usa, v_precio_unidad_usa  
                FROM   tb_unidad_inventario_usa ui  
                LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                WHERE  ui.id_transicion  = v_id_transicion  
                  AND  ui.estado_general = 'enviado_lima'  
                  AND  NOT EXISTS (  
                      SELECT 1 FROM unidad_inventario_lima uil WHERE uil.id_origen = ui.id_inventario_unidad  
                  )  
                ORDER  BY ui.fecha_registro ASC LIMIT 1;  
            END IF;  
  
            IF v_id_origen_unidad IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('inventario_usa: no se encontró unidad USA disponible para',  
                    ' registrar en Lima. Transición: #', v_id_transicion,  
                    ' | Iteración: ', v_index + 1, '/', v_unidades_a_crear,  
                    ' | Puede que todas las unidades ya estén registradas en Lima');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            INSERT INTO unidad_inventario_lima (  
                id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                motivo, numero_pedido, numero_pedido_origen  
            ) VALUES (  
                v_id_inventario_factory, NULL, v_id_origen_unidad,  
                p_descripcion_ubicacion, v_factura_unidad_usa, v_precio_unidad_usa, p_estado_producto,  
                'disponible', 'inventario_usa', NULL, v_comentario_final, NULL, NULL  
            );  
            SET v_id_unidad_creada = LAST_INSERT_ID();  
  
            INSERT INTO historial_unidad_inventario_lima (  
                id_unidad_inventario, id_usuario_responsable, tipo_accion, descripcion  
            ) VALUES (  
                v_id_unidad_creada, p_id_usuario, 'entrada',  
                CONCAT('Registro inicial. ', v_comentario_final,  
                       ' | Unidad USA origen: #', v_id_origen_unidad,  
                       IF(v_factura_unidad_usa IS NOT NULL, CONCAT(' | Factura: ', v_factura_unidad_usa), ''),  
                       IF(v_precio_unidad_usa  IS NOT NULL, CONCAT(' | Precio compra: $', v_precio_unidad_usa), ''))  
            );  
  
            SET v_index = v_index + 1;  
        END WHILE;  
  
        IF v_error_handler = 0 THEN  
            COMMIT; SET p_resultado = 1;  
            SET p_mensaje = CONCAT(p_cantidad, ' unidad(es) registrada(s) en Lima. Total llegadas: ',  
                v_cantidad_llegada + p_cantidad, '/', v_cantidad_en_movimiento, ' | Guía: ', p_numero_detalle);  
        ELSE  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('inventario_usa: error SQL al registrar unidades',  
                ' | Código: ', v_error_code, ' | Estado: ', v_sql_state, ' | ', v_error_msg);  
        END IF;  
        LEAVE sp_block;  
  
    -- ══════════════════════════════════════════════════════════════════════════  
    -- FLUJO: COMPRA PERÚ    -- ══════════════════════════════════════════════════════════════════════════    ELSEIF p_procedencia = 'compra_peru' THEN  
  
        IF p_cantidad IS NULL OR p_cantidad <= 0 THEN  
            SET p_resultado = 0; SET p_mensaje = 'compra_peru: la cantidad debe ser mayor a 0';  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SET v_es_espejo = 0; SET v_sku_primario_espejo = NULL; SET v_id_primario_espejo = NULL;  
        CALL sp_es_producto_espejo(p_sku_producto, v_es_espejo, v_sku_primario_espejo, v_id_primario_espejo);  
  
        IF v_es_espejo = 1 THEN  
            SET v_sku_original_espejo = p_sku_producto;  
            SET p_sku_producto        = v_sku_primario_espejo;  
        ELSE  
            SET v_sku_original_espejo = NULL;  
        END IF;  
  
        SELECT id INTO v_id_producto FROM productos  
        WHERE  sku COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_producto IS NULL THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('compra_peru: producto SKU "', p_sku_producto, '" no encontrado en catálogo');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SELECT id_producto_inventario INTO v_id_inventario_factory  
        FROM   inventario_lima_factory  
        WHERE  id_producto = v_id_producto  
          AND  sku_registrado COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_inventario_factory IS NULL THEN  
            INSERT INTO inventario_lima_factory (id_producto, sku_registrado) VALUES (v_id_producto, p_sku_producto);  
            SET v_id_inventario_factory = LAST_INSERT_ID();  
        END IF;  
  
        SET v_unidades_a_crear = p_cantidad;  
        SET v_comentario_final = COALESCE(NULLIF(p_comentario, ''),  
            CONCAT('Registro de ', p_cantidad, ' unidad(es) de compra en Perú con factura/boleta ',  
                   p_numero_detalle, ' (SKU: ', p_sku_producto,  
                   IF(v_sku_original_espejo IS NOT NULL,  
                      CONCAT(', redirigido desde espejo: ', v_sku_original_espejo), ''), ')'));  
  
        SET v_index = 0;  
        WHILE v_index < v_unidades_a_crear DO  
            INSERT INTO unidad_inventario_lima (  
                id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                motivo, numero_pedido, numero_pedido_origen  
            ) VALUES (  
                v_id_inventario_factory, NULL, NULL,  
                p_descripcion_ubicacion, p_numero_detalle, NULL, p_estado_producto,  
                'disponible', 'compra_peru', NULL, v_comentario_final, NULL, NULL  
            );  
            SET v_id_unidad_creada = LAST_INSERT_ID();  
  
            INSERT INTO historial_unidad_inventario_lima (  
                id_unidad_inventario, id_usuario_responsable, tipo_accion, descripcion  
            ) VALUES (v_id_unidad_creada, p_id_usuario, 'entrada', CONCAT('Registro inicial. ', v_comentario_final));  
  
            SET v_index = v_index + 1;  
        END WHILE;  
  
        IF v_error_handler = 0 THEN  
            COMMIT; SET p_resultado = 1;  
            SET p_mensaje = CONCAT(p_cantidad, ' unidad(es) registrada(s) exitosamente en inventario Lima');  
        ELSE  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('compra_peru: error SQL al registrar unidades',  
                ' | Código: ', v_error_code, ' | Estado: ', v_sql_state, ' | ', v_error_msg);  
        END IF;  
        LEAVE sp_block;  
  
    -- ══════════════════════════════════════════════════════════════════════════  
    -- FLUJO: PEDIDO CORRECTO    -- ══════════════════════════════════════════════════════════════════════════    ELSEIF p_procedencia = 'pedido' THEN  
  
        IF p_datos IS NULL OR JSON_LENGTH(p_datos) = 0 THEN  
            SET p_resultado = 0; SET p_mensaje = 'pedido: p_datos es requerido y no puede estar vacío';  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SET v_total_datos = JSON_LENGTH(p_datos);  
        SET v_idx = 0;  
  
        WHILE v_idx < v_total_datos DO  
  
            SET v_id_pp = CAST(JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].id_pedido_producto'))) AS UNSIGNED);  
            SET v_sku_llegado = JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].sku_llegado')));  
            IF v_sku_llegado = 'null' OR v_sku_llegado = '' THEN SET v_sku_llegado = NULL; END IF;  
  
            IF v_sku_llegado IS NOT NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido: datos[', v_idx, '] id_pedido_producto=', v_id_pp,  
                    ' — sku_llegado debe ser null en procedencia "pedido"');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            IF v_id_pp IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido: datos[', v_idx, '] — id_pedido_producto inválido o nulo');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT pp2.id_logistic_status INTO v_ls_pedido_check  
            FROM   pedidos_productos pp2 WHERE pp2.id_pedido_producto = v_id_pp LIMIT 1;  
  
            IF (v_ls_pedido_check IS NULL OR v_ls_pedido_check IN (1, 2, 3)) AND COALESCE(v_ls_pedido_check, 0) != 9 THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido: id_pedido_producto=', v_id_pp,  
                    ' — logistic_status debe ser ENVIADO(4) o posterior.',  
                    ' Estado actual: ', COALESCE(CAST(v_ls_pedido_check AS CHAR), 'NULL'));  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT pp.sku_cf, pp.cantidad INTO v_sku_original, v_cantidad_pp  
            FROM   pedidos_productos pp WHERE pp.id_pedido_producto = v_id_pp LIMIT 1;  
  
            IF v_sku_original IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido: id_pedido_producto=', v_id_pp, ' no encontrado en pedidos_productos');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT p.numero_pedido INTO v_numero_pedido_pp  
            FROM   pedidos_productos pp2  
            INNER JOIN pedidos p ON p.id_pedido = pp2.id_pedido  
            WHERE  pp2.id_pedido_producto = v_id_pp LIMIT 1;  
  
            SET v_sku_usar = v_sku_original;  
  
            SELECT id INTO v_id_producto_item FROM productos  
            WHERE  sku COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
            IF v_id_producto_item IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido: SKU "', v_sku_usar, '" del pedido_producto=', v_id_pp,  
                    ' no encontrado en catálogo de productos');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT id_producto_inventario INTO v_id_inv_factory_item  
            FROM   inventario_lima_factory  
            WHERE  id_producto = v_id_producto_item  
              AND  sku_registrado COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
            IF v_id_inv_factory_item IS NULL THEN  
                INSERT INTO inventario_lima_factory (id_producto, sku_registrado) VALUES (v_id_producto_item, v_sku_usar);  
                SET v_id_inv_factory_item = LAST_INSERT_ID();  
            END IF;  
  
            SET v_comentario_final = COALESCE(NULLIF(p_comentario, ''), NULL);  
            SET v_i = 0;  
  
            WHILE v_i < v_cantidad_pp DO  
  
                SET v_id_unidad_usa_item = NULL; SET v_factura_unidad_usa = NULL; SET v_precio_unidad_usa = NULL;  
  
                SELECT h.id_inventario_usa_unidad, ipi.codigo_factura, ipi.precio_unitario  
                INTO   v_id_unidad_usa_item, v_factura_unidad_usa, v_precio_unidad_usa  
                FROM   tb_historial_acciones_unidad_inventario h  
                INNER JOIN tb_unidad_inventario_usa ui ON ui.id_inventario_unidad = h.id_inventario_usa_unidad  
                LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                WHERE  h.id_pedido_producto = v_id_pp AND h.accion = 'enviado_lima'  
                  AND  ui.id_inventario_proveedor_info IS NOT NULL  
                ORDER  BY h.fecha_registro ASC LIMIT 1;  
  
                IF v_id_unidad_usa_item IS NULL THEN  
                    SELECT h.id_inventario_usa_unidad, ipi.codigo_factura, ipi.precio_unitario  
                    INTO   v_id_unidad_usa_item, v_factura_unidad_usa, v_precio_unidad_usa  
                    FROM   tb_historial_acciones_unidad_inventario h  
                    INNER JOIN tb_unidad_inventario_usa ui ON ui.id_inventario_unidad = h.id_inventario_usa_unidad  
                    LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                    WHERE  h.id_pedido_producto = v_id_pp AND ui.id_inventario_proveedor_info IS NOT NULL  
                    ORDER  BY h.fecha_registro DESC LIMIT 1;  
                END IF;  
  
                IF v_id_unidad_usa_item IS NULL THEN  
                    SELECT ui.id_inventario_unidad, ipi.codigo_factura, ipi.precio_unitario  
                    INTO   v_id_unidad_usa_item, v_factura_unidad_usa, v_precio_unidad_usa  
                    FROM   tb_unidad_inventario_usa ui  
                    LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                    WHERE  ui.id_pedido_producto_asignado = v_id_pp AND ui.id_inventario_proveedor_info IS NOT NULL  
                    ORDER  BY ui.fecha_registro ASC LIMIT 1;  
                END IF;  
  
                IF v_id_unidad_usa_item IS NULL THEN  
                    SELECT ui.id_inventario_unidad, ipi.codigo_factura, ipi.precio_unitario  
                    INTO   v_id_unidad_usa_item, v_factura_unidad_usa, v_precio_unidad_usa  
                    FROM   logistica_usa l  
                    INNER JOIN tb_unidad_inventario_usa ui ON ui.id_pedido_producto_asignado = l.id_pedido_producto  
                    LEFT  JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                    WHERE  l.id_pedido_producto = v_id_pp AND l.is_active = 1  
                      AND  ui.id_inventario_proveedor_info IS NOT NULL  
                    ORDER  BY ui.fecha_registro ASC LIMIT 1;  
                END IF;  
  
                INSERT INTO unidad_inventario_lima (  
                    id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                    descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                    estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                    motivo, numero_pedido, numero_pedido_origen  
                ) VALUES (  
                    v_id_inv_factory_item, NULL, v_id_unidad_usa_item,  
                    p_descripcion_ubicacion, v_factura_unidad_usa, v_precio_unidad_usa, p_estado_producto,  
                    'disponible', 'pedido', NULL, v_comentario_final, NULL, v_numero_pedido_pp  
                );  
                SET v_id_unidad_lima_item = LAST_INSERT_ID();  
  
                INSERT INTO historial_unidad_inventario_lima (  
                    id_unidad_inventario, id_usuario_responsable,  
                    tipo_accion, descripcion, numero_pedido, id_pedido_producto  
                ) VALUES (  
                    v_id_unidad_lima_item, p_id_usuario, 'entrada',  
                    CONCAT('Procedencia: pedido. Pedido origen: ', COALESCE(v_numero_pedido_pp, p_numero_detalle),  
                           IF(v_id_unidad_usa_item IS NOT NULL,  
                              CONCAT(' | Unidad USA origen: #', v_id_unidad_usa_item,  
                                     IF(v_factura_unidad_usa IS NOT NULL, CONCAT(' | Factura: ', v_factura_unidad_usa), ''),  
                                     IF(v_precio_unidad_usa  IS NOT NULL, CONCAT(' | Precio compra: ', v_precio_unidad_usa), '')),  
                              ' | Sin unidad USA vinculada')),  
                    NULL, NULL  
                );  
  
                IF v_id_unidad_usa_item IS NOT NULL THEN  
                    UPDATE tb_unidad_inventario_usa  
                    SET    estado_general = 'enviado_lima', id_pedido_producto_asignado = NULL, numero_pedido = NULL  
                    WHERE  id_inventario_unidad = v_id_unidad_usa_item  
                      AND  estado_general IN ('en_pedido','para_envio','recepcionado');  
  
                    INSERT INTO tb_historial_acciones_unidad_inventario (  
                        id_inventario_usa_unidad, id_usuario_accion, accion, tipo_accion,  
                        detalles, numero_pedido, id_pedido_producto  
                    ) VALUES (  
                        v_id_unidad_usa_item, p_id_usuario, 'enviado_lima', 'salida',  
                        CONCAT('Enviado a inventario Lima (pedido correcto). Pedido: ',  
                               COALESCE(v_numero_pedido_pp, p_numero_detalle),  
                               ' | pedido_producto: ', v_id_pp, ' | SKU: ', v_sku_original),  
                        COALESCE(v_numero_pedido_pp, p_numero_detalle), v_id_pp  
                    );  
                END IF;  
  
                SET v_i = v_i + 1;  
            END WHILE;  
  
            SET v_idx = v_idx + 1;  
        END WHILE;  
  
        IF v_error_handler = 0 THEN  
            COMMIT; SET p_resultado = 1;  
            SET p_mensaje = CONCAT(v_total_datos, ' pedido_producto(s) registrado(s) en Lima');  
        ELSE  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('pedido: error SQL al registrar',  
                ' | Código: ', v_error_code, ' | Estado: ', v_sql_state, ' | ', v_error_msg);  
        END IF;  
        LEAVE sp_block;  
  
    -- ══════════════════════════════════════════════════════════════════════════  
    -- FLUJO: PEDIDO ERRÓNEO    --    -- OPCIÓN 1 – descontar_unidades_usa = 1    --   Toma N unidades del stock DISPONIBLE (estado=recepcionado, sin asignación)    --   del SKU en sku_llegado y las registra como nuevas unidades en Lima.    --   Requiere: sku_llegado, cantidad_descontar.    --   INCOMPATIBLE con cambiar_sku_unidad = 1.    --    -- OPCIÓN 2 – cambiar_sku_unidad = 1    --   Cambia el id_inventario_usa de la unidad YA VINCULADA al pedido al    --   inventario de sku_cambio. NO toca stock disponible.    --   Requiere: sku_cambio.    --   INCOMPATIBLE con descontar_unidades_usa = 1.    --    -- OPCIÓN 3 – regresar_a_procesado = 1    --   Desvincula la unidad vinculada y revierte logistic_status = 1.    --   Compatible con OPCIÓN 2 (corre después).    -- ══════════════════════════════════════════════════════════════════════════    ELSEIF p_procedencia = 'pedido_erroneo' THEN  
  
        IF p_datos IS NULL OR JSON_LENGTH(p_datos) = 0 THEN  
            SET p_resultado = 0; SET p_mensaje = 'pedido_erroneo: p_datos es requerido y no puede estar vacío';  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SET v_total_datos = JSON_LENGTH(p_datos);  
        SET v_idx = 0;  
  
        WHILE v_idx < v_total_datos DO  
  
            SET v_id_pp = CAST(  
                JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].id_pedido_producto'))) AS UNSIGNED  
            );  
  
            SET v_sku_llegado = JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].sku_llegado')));  
            IF v_sku_llegado = 'null' OR v_sku_llegado = '' THEN SET v_sku_llegado = NULL; END IF;  
  
            SET v_sku_cambio = JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].sku_cambio')));  
            IF v_sku_cambio = 'null' OR v_sku_cambio = '' THEN SET v_sku_cambio = NULL; END IF;  
  
            -- NULLIF doble: filtra el string literal 'null' que devuelve JSON_UNQUOTE  
            -- cuando el campo JSON es null — CAST('null' AS UNSIGNED) explota en modo estricto            SET v_regresar_procesado = COALESCE(CAST(  
                NULLIF(NULLIF(JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].regresar_a_procesado'))),  'null'), '')  
                AS UNSIGNED), 0);  
            SET v_descontar_usa = COALESCE(CAST(  
                NULLIF(NULLIF(JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].descontar_unidades_usa'))), 'null'), '')  
                AS UNSIGNED), 0);  
            SET v_cantidad_descontar = COALESCE(CAST(  
                NULLIF(NULLIF(JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].cantidad_descontar'))),    'null'), '')  
                AS UNSIGNED), 0);  
            SET v_cambiar_sku_unidad = COALESCE(CAST(  
                NULLIF(NULLIF(JSON_UNQUOTE(JSON_EXTRACT(p_datos, CONCAT('$[', v_idx, '].cambiar_sku_unidad'))),    'null'), '')  
                AS UNSIGNED), 0);  
  
            -- ── Validaciones ─────────────────────────────────────────────────  
            IF v_id_pp IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido_erroneo: datos[', v_idx, '] — id_pedido_producto inválido o nulo');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            IF v_descontar_usa = 1 AND v_cambiar_sku_unidad = 1 THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT(  
                    'pedido_erroneo: datos[', v_idx, '] id_pedido_producto=', v_id_pp,  
                    ' — descontar_unidades_usa y cambiar_sku_unidad no pueden ser 1 simultáneamente.',  
                    ' descontar_unidades_usa toma una unidad del stock disponible de sku_llegado;',  
                    ' cambiar_sku_unidad reasigna el inventario de la unidad ya vinculada al pedido.');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            IF v_descontar_usa = 1 AND v_sku_llegado IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido_erroneo: datos[', v_idx, '] id_pedido_producto=', v_id_pp,  
                    ' — sku_llegado es obligatorio cuando descontar_unidades_usa = 1');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            IF v_descontar_usa = 1 AND v_cantidad_descontar <= 0 THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido_erroneo: datos[', v_idx, '] id_pedido_producto=', v_id_pp,  
                    ' — cantidad_descontar debe ser mayor a 0 cuando descontar_unidades_usa = 1');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            IF v_cambiar_sku_unidad = 1 AND v_sku_cambio IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido_erroneo: datos[', v_idx, '] id_pedido_producto=', v_id_pp,  
                    ' — sku_cambio es obligatorio cuando cambiar_sku_unidad = 1');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT pp.sku_cf, pp.cantidad, pp.id_logistic_status  
            INTO   v_sku_original, v_cantidad_pp, v_ls_fuente_actual  
            FROM   pedidos_productos pp WHERE pp.id_pedido_producto = v_id_pp LIMIT 1;  
  
            IF v_sku_original IS NULL THEN  
                SET p_resultado = 0;  
                SET p_mensaje = CONCAT('pedido_erroneo: id_pedido_producto=', v_id_pp,  
                    ' no encontrado en pedidos_productos');  
                ROLLBACK; LEAVE sp_block;  
            END IF;  
  
            SELECT p.numero_pedido, p.id_pedido INTO v_numero_pedido_pp, v_id_pedido  
            FROM   pedidos_productos pp2  
            INNER JOIN pedidos p ON p.id_pedido = pp2.id_pedido  
            WHERE  pp2.id_pedido_producto = v_id_pp LIMIT 1;  
  
            -- ────────────────────────────────────────────────────────────────  
            -- OPCIÓN 1: descontar_unidades_usa            -- Toma unidades del stock disponible de sku_llegado → Lima.            -- NO modifica la unidad vinculada al pedido.            -- ────────────────────────────────────────────────────────────────            IF v_descontar_usa = 1 THEN  
  
                SET v_es_espejo = 0; SET v_sku_primario_espejo = NULL; SET v_id_primario_espejo = NULL;  
                CALL sp_es_producto_espejo(v_sku_llegado, v_es_espejo, v_sku_primario_espejo, v_id_primario_espejo);  
                IF v_es_espejo = 1 THEN SET v_sku_usar = v_sku_primario_espejo;  
                ELSE SET v_sku_usar = v_sku_llegado;  
                END IF;  
  
                SELECT id INTO v_id_producto_item FROM productos  
                WHERE  sku COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                IF v_id_producto_item IS NULL THEN  
                    SET p_resultado = 0;  
                    SET p_mensaje = CONCAT('pedido_erroneo op1: sku_llegado "', v_sku_usar,  
                        '" no encontrado en catálogo | id_pedido_producto=', v_id_pp);  
                    ROLLBACK; LEAVE sp_block;  
                END IF;  
  
                SET v_id_inventario_usa_llegado = NULL;  
                SELECT inv.id_inventario INTO v_id_inventario_usa_llegado  
                FROM   tb_inventario_usa inv  
                WHERE  inv.sku COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                IF v_id_inventario_usa_llegado IS NULL THEN  
                    SET p_resultado = 0;  
                    SET p_mensaje = CONCAT('pedido_erroneo op1: no existe inventario USA para SKU "',  
                        v_sku_usar, '" | id_pedido_producto=', v_id_pp);  
                    ROLLBACK; LEAVE sp_block;  
                END IF;  
  
                SET v_stock_usa_disponible = 0;  
                SELECT COUNT(*) INTO v_stock_usa_disponible  
                FROM   tb_unidad_inventario_usa  
                WHERE  id_inventario_usa = v_id_inventario_usa_llegado  
                  AND  estado_general = 'recepcionado'  
                  AND  id_pedido_producto_asignado IS NULL;  
  
                IF v_stock_usa_disponible < v_cantidad_descontar THEN  
                    SET p_resultado = 0;  
                    SET p_mensaje = CONCAT('pedido_erroneo op1: stock insuficiente en USA para SKU "',  
                        v_sku_usar, '". Disponibles: ', v_stock_usa_disponible,  
                        ' | Requeridos: ', v_cantidad_descontar,  
                        ' | id_pedido_producto=', v_id_pp);  
                    ROLLBACK; LEAVE sp_block;  
                END IF;  
  
                SELECT id_producto_inventario INTO v_id_inv_factory_item  
                FROM   inventario_lima_factory  
                WHERE  id_producto = v_id_producto_item  
                  AND  sku_registrado COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                IF v_id_inv_factory_item IS NULL THEN  
                    INSERT INTO inventario_lima_factory (id_producto, sku_registrado) VALUES (v_id_producto_item, v_sku_usar);  
                    SET v_id_inv_factory_item = LAST_INSERT_ID();  
                END IF;  
  
                SET v_comentario_final = COALESCE(NULLIF(p_comentario, ''), NULL);  
                SET v_i = 0;  
  
                WHILE v_i < v_cantidad_descontar DO  
  
                    SET v_id_unidad_usa_llegado = NULL;  
                    SET v_factura_unidad_usa    = NULL;  
                    SET v_precio_unidad_usa     = NULL;  
  
                    SELECT ui.id_inventario_unidad, ipi.codigo_factura, ipi.precio_unitario  
                    INTO   v_id_unidad_usa_llegado, v_factura_unidad_usa, v_precio_unidad_usa  
                    FROM   tb_unidad_inventario_usa ui  
                    LEFT JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                    WHERE  ui.id_inventario_usa = v_id_inventario_usa_llegado  
                      AND  ui.estado_general = 'recepcionado'  
                      AND  ui.id_pedido_producto_asignado IS NULL  
                    ORDER  BY ui.fecha_registro ASC LIMIT 1;  
  
                    IF v_id_unidad_usa_llegado IS NULL THEN  
                        SET p_resultado = 0;  
                        SET p_mensaje = CONCAT('pedido_erroneo op1: no se encontró unidad USA disponible para SKU "',  
                            v_sku_usar, '" en iteración ', v_i + 1, '/',  v_cantidad_descontar,  
                            ' | id_pedido_producto=', v_id_pp);  
                        ROLLBACK; LEAVE sp_block;  
                    END IF;  
  
                    INSERT INTO unidad_inventario_lima (  
                        id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                        descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                        estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                        motivo, numero_pedido, numero_pedido_origen  
                    ) VALUES (  
                        v_id_inv_factory_item, NULL, v_id_unidad_usa_llegado,  
                        p_descripcion_ubicacion, v_factura_unidad_usa, v_precio_unidad_usa, p_estado_producto,  
                        'disponible', 'pedido_erroneo', NULL,  
                        v_comentario_final, NULL, v_numero_pedido_pp  
                    );  
                    SET v_id_unidad_lima_item = LAST_INSERT_ID();  
  
                    INSERT INTO historial_unidad_inventario_lima (  
                        id_unidad_inventario, id_usuario_responsable,  
                        tipo_accion, descripcion, numero_pedido, id_pedido_producto  
                    ) VALUES (  
                        v_id_unidad_lima_item, p_id_usuario, 'entrada',  
                        CONCAT('Procedencia: pedido_erroneo (op1-descontar). Pedido origen: ',  
                               COALESCE(v_numero_pedido_pp, 'N/A'),  
                               ' | SKU esperado: "', v_sku_original,  
                               '" | SKU real: "', v_sku_llegado, '"',  
                               IF(v_factura_unidad_usa IS NOT NULL, CONCAT(' | Factura: ', v_factura_unidad_usa), ''),  
                               IF(v_precio_unidad_usa  IS NOT NULL, CONCAT(' | Precio compra: $', v_precio_unidad_usa), ''),  
                               ' | Unidad USA origen: #', v_id_unidad_usa_llegado),  
                        NULL, NULL  
                    );  
  
                    UPDATE tb_unidad_inventario_usa  
                    SET    estado_general = 'enviado_lima', id_pedido_producto_asignado = NULL, numero_pedido = NULL  
                    WHERE  id_inventario_unidad = v_id_unidad_usa_llegado;  
  
                    INSERT INTO tb_historial_acciones_unidad_inventario (  
                        id_inventario_usa_unidad, id_usuario_accion, accion, tipo_accion,  
                        detalles, numero_pedido, id_pedido_producto  
                    ) VALUES (  
                        v_id_unidad_usa_llegado, p_id_usuario, 'enviado_lima', 'salida',  
                        CONCAT('Enviado a Lima por ERROR DE EMPRESA (op1-descontar).',  
                               ' SKU esperado: "', v_sku_original,  
                               '" | SKU real: "', v_sku_llegado, '".',  
                               ' Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A'),  
                               ' | pedido_producto: ', v_id_pp,  
                               ' | Unidad Lima: #', v_id_unidad_lima_item),  
                        COALESCE(v_numero_pedido_pp, p_numero_detalle), v_id_pp  
                    );  
  
                    SET v_i = v_i + 1;  
                END WHILE;  
  
            END IF;  
            -- ── FIN OPCIÓN 1 ────────────────────────────────────────────────  
  
            -- ────────────────────────────────────────────────────────────────            -- OPCIÓN 2: cambiar_sku_unidad            -- Cambia id_inventario_usa de la unidad VINCULADA al pedido.            -- Variable dedicada v_id_inv_usa_cambio (no comparte con op1).            -- ────────────────────────────────────────────────────────────────            IF v_cambiar_sku_unidad = 1 THEN  
  
                SET v_es_espejo = 0; SET v_sku_primario_espejo = NULL; SET v_id_primario_espejo = NULL;  
                CALL sp_es_producto_espejo(v_sku_cambio, v_es_espejo, v_sku_primario_espejo, v_id_primario_espejo);  
                IF v_es_espejo = 1 THEN SET v_sku_usar = v_sku_primario_espejo;  
                ELSE SET v_sku_usar = v_sku_cambio;  
                END IF;  
  
                SET v_id_inv_usa_cambio = NULL;  
                SELECT inv.id_inventario INTO v_id_inv_usa_cambio  
                FROM   tb_inventario_usa inv  
                WHERE  inv.sku COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                IF v_id_inv_usa_cambio IS NULL THEN  
                    SELECT id INTO v_id_producto_item FROM productos  
                    WHERE  sku COLLATE utf8mb4_unicode_ci = v_sku_usar COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                    IF v_id_producto_item IS NULL THEN  
                        SET p_resultado = 0;  
                        SET p_mensaje = CONCAT('pedido_erroneo op2: sku_cambio "', v_sku_usar,  
                            '" no encontrado en catálogo | id_pedido_producto=', v_id_pp);  
                        ROLLBACK; LEAVE sp_block;  
                    END IF;  
  
                    INSERT INTO tb_inventario_usa (sku, id_producto) VALUES (v_sku_usar, v_id_producto_item);  
                    SET v_id_inv_usa_cambio = LAST_INSERT_ID();  
                END IF;  
  
                -- Buscar EXCLUSIVAMENTE la unidad vinculada al pedido  
                SET v_id_unidad_original = NULL;  
                SELECT ui.id_inventario_unidad INTO v_id_unidad_original  
                FROM   tb_unidad_inventario_usa ui  
                WHERE  ui.id_pedido_producto_asignado = v_id_pp  
                  AND  ui.estado_general NOT IN ('anulado','devolucion_proveedor','devuelto_proveedor','merma','enviado_lima')  
                ORDER  BY ui.fecha_registro ASC LIMIT 1;  
  
                IF v_id_unidad_original IS NOT NULL THEN  
                    UPDATE tb_unidad_inventario_usa  
                    SET    id_inventario_usa = v_id_inv_usa_cambio  
                    WHERE  id_inventario_unidad = v_id_unidad_original;  
  
                    INSERT INTO tb_historial_acciones_unidad_inventario (  
                        id_inventario_usa_unidad, id_usuario_accion, accion, tipo_accion,  
                        detalles, numero_pedido, id_pedido_producto  
                    ) VALUES (  
                        v_id_unidad_original, p_id_usuario, 'recepcionado', 'sin_tipo',  
                        CONCAT('pedido_erroneo op2: SKU de unidad corregido.',  
                               ' "', v_sku_original, '" → "', v_sku_usar, '".',  
                               ' Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A'),  
                               ' | Motivo: Error en compra — SKU incorrecto recibido del proveedor',  
                               IF(v_regresar_procesado = 1,  
                                  ' | La unidad quedará desvinculada por op3-regresar', '')),  
                        v_numero_pedido_pp, v_id_pp  
                    );  
                -- Si no hay unidad vinculada: situación válida (pedido sin unidad asignada aún).  
                -- No se falla, se continúa.                END IF;  
  
            END IF;  
            -- ── FIN OPCIÓN 2 ────────────────────────────────────────────────  
  
            -- ────────────────────────────────────────────────────────────────            -- OPCIÓN 3: regresar_a_procesado            -- Corre DESPUÉS de op2: si se cambió el SKU de la unidad vinculada,            -- ésta queda correctamente categorizada antes de desvincularse.            -- ────────────────────────────────────────────────────────────────            IF v_regresar_procesado = 1  
               AND v_ls_fuente_actual IS NOT NULL  
               AND v_ls_fuente_actual NOT IN (1, 9) THEN  
  
                -- Re-buscar la unidad vinculada (op2 pudo haber cambiado su id_inventario_usa  
                -- pero id_pedido_producto_asignado sigue siendo v_id_pp)                SET v_id_unidad_original = NULL;  
                SELECT ui.id_inventario_unidad INTO v_id_unidad_original  
                FROM   tb_unidad_inventario_usa ui  
                WHERE  ui.id_pedido_producto_asignado = v_id_pp  
                  AND  ui.estado_general NOT IN ('anulado','devolucion_proveedor','devuelto_proveedor','merma','enviado_lima')  
                ORDER  BY ui.fecha_registro ASC LIMIT 1;  
  
                IF v_id_unidad_original IS NOT NULL THEN  
  
                    -- Resolver SKU final antes de cualquier UPDATE  
                    SET v_sku_final_unidad = NULL;  
                    SELECT inv.sku INTO v_sku_final_unidad  
                    FROM   tb_inventario_usa inv  
                    INNER  JOIN tb_unidad_inventario_usa u2 ON u2.id_inventario_usa = inv.id_inventario  
                    WHERE  u2.id_inventario_unidad = v_id_unidad_original LIMIT 1;  
                    IF v_sku_final_unidad IS NULL THEN SET v_sku_final_unidad = v_sku_original; END IF;  
  
                    IF v_cambiar_sku_unidad = 1 THEN  
                        -- La unidad está físicamente en Lima (pedido venía de EN_LIMA o similar).  
                        -- Registrarla en inventario Lima con el SKU correcto y marcarla enviado_lima.  
                        -- Obtener factura y precio de la unidad USA                        SET v_factura_unidad_usa = NULL;  
                        SET v_precio_unidad_usa  = NULL;  
                        SELECT ipi.codigo_factura, ipi.precio_unitario  
                        INTO   v_factura_unidad_usa, v_precio_unidad_usa  
                        FROM   tb_unidad_inventario_usa ui  
                        LEFT JOIN inventario_proveedor_info ipi ON ipi.id = ui.id_inventario_proveedor_info  
                        WHERE  ui.id_inventario_unidad = v_id_unidad_original LIMIT 1;  
  
                        -- Resolver o crear factory Lima para el SKU correcto  
                        SET v_id_producto_item    = NULL;  
                        SET v_id_inv_factory_item = NULL;  
  
                        SELECT id INTO v_id_producto_item FROM productos  
                        WHERE  sku COLLATE utf8mb4_unicode_ci = v_sku_final_unidad COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                        IF v_id_producto_item IS NOT NULL THEN  
                            SELECT id_producto_inventario INTO v_id_inv_factory_item  
                            FROM   inventario_lima_factory  
                            WHERE  id_producto = v_id_producto_item  
                              AND  sku_registrado COLLATE utf8mb4_unicode_ci = v_sku_final_unidad COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
                            IF v_id_inv_factory_item IS NULL THEN  
                                INSERT INTO inventario_lima_factory (id_producto, sku_registrado)  
                                VALUES (v_id_producto_item, v_sku_final_unidad);  
                                SET v_id_inv_factory_item = LAST_INSERT_ID();  
                            END IF;  
  
                            INSERT INTO unidad_inventario_lima (  
                                id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                                descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                                estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                                motivo, numero_pedido, numero_pedido_origen  
                            ) VALUES (  
                                v_id_inv_factory_item, NULL, v_id_unidad_original,  
                                p_descripcion_ubicacion, v_factura_unidad_usa, v_precio_unidad_usa,  
                                p_estado_producto, 'disponible', 'pedido_erroneo', NULL,  
                                CONCAT('pedido_erroneo op3+op2: SKU corregido de "', v_sku_original,  
                                       '" a "', v_sku_final_unidad, '" y registrado en Lima.',  
                                       ' Pedido origen: ', COALESCE(v_numero_pedido_pp, 'N/A')),  
                                NULL, v_numero_pedido_pp  
                            );  
                            SET v_id_unidad_lima_item = LAST_INSERT_ID();  
  
                            INSERT INTO historial_unidad_inventario_lima (  
                                id_unidad_inventario, id_usuario_responsable, tipo_accion, descripcion  
                            ) VALUES (  
                                v_id_unidad_lima_item, p_id_usuario, 'entrada',  
                                CONCAT('pedido_erroneo op3+op2: unidad registrada en Lima con SKU correcto "',  
                                       v_sku_final_unidad, '" (corregido de "', v_sku_original, '").',  
                                       ' Pedido origen: ', COALESCE(v_numero_pedido_pp, 'N/A'),  
                                       IF(v_factura_unidad_usa IS NOT NULL,  
                                          CONCAT(' | Factura: ', v_factura_unidad_usa), ''),  
                                       IF(v_precio_unidad_usa  IS NOT NULL,  
                                          CONCAT(' | Precio: $', v_precio_unidad_usa), ''))  
                            );  
                        END IF;  
  
                        -- Marcar la unidad USA como enviado_lima (está en Lima, no en stock USA)  
                        UPDATE tb_unidad_inventario_usa  
                        SET    estado_general              = 'enviado_lima',  
                               id_pedido_producto_asignado = NULL,  
                               numero_pedido               = NULL  
                        WHERE  id_inventario_unidad = v_id_unidad_original;  
  
                        INSERT INTO tb_historial_acciones_unidad_inventario (  
                            id_inventario_usa_unidad, id_usuario_accion, accion, tipo_accion,  
                            detalles, numero_pedido, id_pedido_producto  
                        ) VALUES (  
                            v_id_unidad_original, p_id_usuario, 'enviado_lima', 'salida',  
                            CONCAT('pedido_erroneo op3+op2: unidad enviada a inventario Lima.',  
                                   ' SKU corregido de "', v_sku_original, '" a "', v_sku_final_unidad, '".',  
                                   ' Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A')),  
                            v_numero_pedido_pp, v_id_pp  
                        );  
  
                    ELSE  
                        -- Sin cambio de SKU: desvincula y devuelve a recepcionado en USA  
                        UPDATE tb_unidad_inventario_usa  
                        SET    estado_general              = 'recepcionado',  
                               id_pedido_producto_asignado = NULL,  
                               numero_pedido               = NULL  
                        WHERE  id_inventario_unidad = v_id_unidad_original;  
  
                        INSERT INTO tb_historial_acciones_unidad_inventario (  
                            id_inventario_usa_unidad, id_usuario_accion, accion, tipo_accion,  
                            detalles, numero_pedido, id_pedido_producto  
                        ) VALUES (  
                            v_id_unidad_original, p_id_usuario, 'recepcionado', 'sin_tipo',  
                            CONCAT('pedido_erroneo op3: unidad desvinculada al revertir pedido a PROCESADO.',  
                                   ' SKU: "', v_sku_final_unidad, '".',  
                                   ' Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A'),  
                                   ' | Unidad disponible en stock recepcionado'),  
                            v_numero_pedido_pp, v_id_pp  
                        );  
                    END IF;  
  
                END IF;  
  
                UPDATE logistica_usa  
                SET    id_unidad_inventario_usa  = NULL,  
                       tipo_inventario_vinculado = NULL,  
                       tipo_asignacion           = NULL,  
                       recepcionado              = 0,  
                       fecha_recepcion           = NULL,  
                       observaciones             = CONCAT(COALESCE(observaciones, ''),  
                           ' | pedido_erroneo op3: revertido a PROCESADO.',  
                           ' SKU: "', v_sku_original, '"',  
                           ' | Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A'))  
                WHERE  id_pedido_producto = v_id_pp AND is_active = 1;  
  
                INSERT INTO historial_estado_pedido (  
                    estado_anterior, estado_nuevo, fecha_cambio,  
                    usuario_id, id_pedido_producto, observacion  
                )  
                SELECT ls.status, 'PROCESADO', NOW(), p_id_usuario, v_id_pp,  
                       CONCAT('pedido_erroneo op3: revertido a PROCESADO por ERROR DE EMPRESA.',  
                              ' SKU pedido: "', v_sku_original, '"',  
                              IF(v_sku_llegado IS NOT NULL,  
                                 CONCAT(' | SKU descontado de USA: "', v_sku_llegado, '"'), ''),  
                              IF(v_sku_cambio IS NOT NULL,  
                                 CONCAT(' | SKU cambiado en unidad: "', v_sku_cambio, '"'), ''),  
                              '. Pedido: ', COALESCE(v_numero_pedido_pp, 'N/A'),  
                              ' | Motivo: ', COALESCE(p_comentario, 'Sin motivo'))  
                FROM   logistic_status ls  
                WHERE  ls.id_logistic_status = v_ls_fuente_actual;  
  
                UPDATE pedidos_productos  
                SET    id_logistic_status = 1  
                WHERE  id_pedido_producto = v_id_pp;  
  
                -- Limpieza adicional cuando el pedido venía de estado > 5  
                -- (EN_CAMINO, ENTREGADO, etc.): resetea campos de logística Lima                IF v_ls_fuente_actual > 5 AND v_ls_fuente_actual != 9 THEN  
  
                    UPDATE pedidos  
                    SET    ready_to_send = FALSE  
                    WHERE  id_pedido = v_id_pedido;  
  
                    UPDATE pedidos_productos  
                    SET    fecha_recibido_cliente = NULL,  
                           fecha_recepcion        = NULL,  
                           fecha_entrega          = NULL  
                    WHERE  id_pedido_producto = v_id_pp;  
  
                    UPDATE logistica_lima  
                    SET    fecha_confirmacion  = NULL,  
                           fecha_distribucion  = NULL,  
                           correlative_invoice = NULL  
                    WHERE  id_pedido = v_id_pedido;  
  
                    -- Obtener IDs de documentos para desactivar  
                    SET v_id_logistica_lima      = NULL;  
                    SET v_id_logistica_lima_docs = NULL;  
  
                    SELECT id_logistica_lima INTO v_id_logistica_lima  
                    FROM   logistica_lima  
                    WHERE  id_pedido = v_id_pedido LIMIT 1;  
  
                    IF v_id_logistica_lima IS NOT NULL THEN  
                        SELECT id_logistica_lima_documento INTO v_id_logistica_lima_docs  
                        FROM   logistica_lima_documentos  
                        WHERE  id_logistica_lima = v_id_logistica_lima LIMIT 1;  
  
                        IF v_id_logistica_lima_docs IS NOT NULL THEN  
                            UPDATE logistica_lima_inv_doc  
                            SET    active = FALSE  
                            WHERE  id_logistica_lima_documentos = v_id_logistica_lima_docs;  
  
                            UPDATE logistica_lima_ticket_doc  
                            SET    active = FALSE  
                            WHERE  id_logistica_lima_documentos = v_id_logistica_lima_docs;  
                        END IF;  
                    END IF;  
  
                END IF;  
                -- ── FIN limpieza logística Lima ──────────────────────────────  
  
            END IF;  
            -- ── FIN OPCIÓN 3 ────────────────────────────────────────────────  
  
            SET v_idx = v_idx + 1;  
        END WHILE;  
  
        IF v_error_handler = 0 THEN  
            COMMIT; SET p_resultado = 1;  
            SET p_mensaje = CONCAT(v_total_datos, ' pedido_producto(s) procesado(s) correctamente (pedido_erroneo).');  
        ELSE  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('pedido_erroneo: error SQL al procesar',  
                ' | Código: ', v_error_code,  
                ' | Estado: ', v_sql_state,  
                ' | ', v_error_msg);  
        END IF;  
        LEAVE sp_block;  
  
    -- ══════════════════════════════════════════════════════════════════════════  
    -- FLUJO: LIBRE    -- ══════════════════════════════════════════════════════════════════════════    ELSEIF p_procedencia = 'libre' THEN  
  
        IF p_cantidad IS NULL OR p_cantidad <= 0 THEN  
            SET p_resultado = 0; SET p_mensaje = 'libre: la cantidad debe ser mayor a 0';  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        IF p_datos IS NOT NULL AND JSON_LENGTH(p_datos) > 0 THEN  
            SET v_precio_compra_libre = CAST(  
                JSON_UNQUOTE(JSON_EXTRACT(p_datos, '$[0].precio_compra')) AS DECIMAL(16,4)  
            );  
        END IF;  
  
        SET v_es_espejo = 0; SET v_sku_primario_espejo = NULL; SET v_id_primario_espejo = NULL;  
        CALL sp_es_producto_espejo(p_sku_producto, v_es_espejo, v_sku_primario_espejo, v_id_primario_espejo);  
  
        IF v_es_espejo = 1 THEN  
            SET v_sku_original_espejo = p_sku_producto;  
            SET p_sku_producto        = v_sku_primario_espejo;  
        ELSE  
            SET v_sku_original_espejo = NULL;  
        END IF;  
  
        SELECT id INTO v_id_producto FROM productos  
        WHERE  sku COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_producto IS NULL THEN  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('libre: producto SKU "', p_sku_producto, '" no encontrado en catálogo');  
            ROLLBACK; LEAVE sp_block;  
        END IF;  
  
        SELECT id_producto_inventario INTO v_id_inventario_factory  
        FROM   inventario_lima_factory  
        WHERE  id_producto = v_id_producto  
          AND  sku_registrado COLLATE utf8mb4_unicode_ci = p_sku_producto COLLATE utf8mb4_unicode_ci LIMIT 1;  
  
        IF v_id_inventario_factory IS NULL THEN  
            INSERT INTO inventario_lima_factory (id_producto, sku_registrado) VALUES (v_id_producto, p_sku_producto);  
            SET v_id_inventario_factory = LAST_INSERT_ID();  
        END IF;  
  
        SET v_comentario_final = COALESCE(  
            NULLIF(p_comentario, ''),  
            CONCAT('Registro libre de ', p_cantidad, ' unidad(es). SKU: ', p_sku_producto,  
                   IF(v_sku_original_espejo IS NOT NULL,  
                      CONCAT(' (espejo de: ', v_sku_original_espejo, ')'), ''),  
                   IF(p_numero_detalle IS NOT NULL AND p_numero_detalle != '',  
                      CONCAT(' | Ref: ', p_numero_detalle), ''),  
                   ' | Fecha: ', DATE_FORMAT(NOW(), '%d/%m/%Y'))  
        );  
  
        SET v_index = 0;  
        WHILE v_index < p_cantidad DO  
  
            INSERT INTO unidad_inventario_lima (  
                id_inventario_lima_factory, id_pedido_producto_asignado, id_origen,  
                descripcion_ubicacion, guia_factura, precio_compra, estado_producto,  
                estados_general_inventario_lima, procedencia, id_usuario_reserva,  
                motivo, numero_pedido, numero_pedido_origen  
            ) VALUES (  
                v_id_inventario_factory, NULL, NULL,  
                p_descripcion_ubicacion, NULLIF(p_numero_detalle, ''), v_precio_compra_libre, p_estado_producto,  
                'disponible', 'libre', NULL, v_comentario_final, NULL, NULL  
            );  
            SET v_id_unidad_creada = LAST_INSERT_ID();  
  
            INSERT INTO historial_unidad_inventario_lima (  
                id_unidad_inventario, id_usuario_responsable, tipo_accion, descripcion  
            ) VALUES (  
                v_id_unidad_creada, p_id_usuario, 'entrada',  
                CONCAT('Registro libre. ', v_comentario_final,  
                       IF(v_precio_compra_libre IS NOT NULL,  
                          CONCAT(' | Precio compra: ', v_precio_compra_libre), ''))  
            );  
  
            SET v_index = v_index + 1;  
        END WHILE;  
  
        IF v_error_handler = 0 THEN  
            COMMIT; SET p_resultado = 1;  
            SET p_mensaje = CONCAT(p_cantidad, ' unidad(es) registrada(s) exitosamente (flujo libre)');  
        ELSE  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('libre: error SQL al registrar unidades',  
                ' | Código: ', v_error_code, ' | Estado: ', v_sql_state, ' | ', v_error_msg);  
        END IF;  
        LEAVE sp_block;  
  
    END IF;  
  
END;
DELIMITER ;
```