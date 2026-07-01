
# Modificaciones tablas


```sql
ALTER TABLE documentos_pedidos  
ADD COLUMN  tipo_documento ENUM ('DNI', 'VITAMINAS', 'FACTURA');
```



```sql
CREATE TABLE pedidos_vitaminas_notificaciones (
    id INT AUTO_INCREMENT PRIMARY KEY,
    numero_pedido VARCHAR(50) NOT NULL,
    id_cliente_facturacion INT NOT NULL, -- <--- Nuevo campo añadido
    numero_notificacion INT NOT NULL,    -- 1, 2 o 3 (Calculado por el SP)
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_pedido_cliente_notif (numero_pedido, id_cliente_facturacion, numero_notificacion)
);
```


```sql
ALTER TABLE pedidos_productos  
ADD COLUMN is_vitamin BOOLEAN default  false;
```


# Nuevos SPs

```sql
CREATE DEFINER = super_scraper@`%` PROCEDURE SP_automation_get_pedidos_vitaminas(IN p_fecha_actual DATETIME)  
BEGIN  
    -- 1. Agrupamos las notificaciones previas que tu script externo ha insertado en la tabla de control  
    WITH historial_notificaciones AS (  
        SELECT  
            numero_pedido,  
            MAX(CASE WHEN rn = 1 THEN created_at END) AS fecha_notificacion_1,  
            MAX(CASE WHEN rn = 2 THEN created_at END) AS fecha_notificacion_2,  
            COUNT(*) AS total_notificaciones  
        FROM (  
            SELECT  
                numero_pedido,  
                created_at,  
                ROW_NUMBER() OVER (PARTITION BY numero_pedido ORDER BY created_at) AS rn  
            FROM pedidos_vitaminas_notificaciones 
            WHERE numero_pedido IS NOT NULL  
        ) ranked  
        GROUP BY numero_pedido  
    ),  
      
    -- 2. Universo base: Pedidos activos de la categoría 22 que NO tienen el documento subido  
    pedidos_base AS (  
        SELECT DISTINCT  
            ped.numero_pedido,  
            ped.fecha_pedido,  
            ped.tiendas  
        FROM pedidos ped  
        INNER JOIN pedidos_productos pp ON ped.id_pedido = pp.id_pedido  
        LEFT JOIN productos prod ON pp.sku_cf = prod.sku COLLATE utf8mb4_unicode_ci AND prod.is_enable   
LEFT JOIN categories cat ON prod.category_code_id = cat.id  
        LEFT JOIN documentos_pedidos dp ON ped.id_pedido = dp.id_pedido   
AND pp.id_cliente_facturacion = dp.id_cliente_facturacion  
                                       AND dp.tipo_documento = 'VITAMINAS'  
        WHERE cat.principal_id = 22  
          AND dp.id_documento_pedido IS NULL -- Si el cliente lo sube, el pedido desaparece automáticamente aquí  
    )  
  
    -- 3. Tu script recibirá el listado "propuesto" filtrado por las reglas de tiempo e intentos  
    SELECT   
pb.numero_pedido,  
        pb.tiendas,  
        resultado.numero_notificacion  
    FROM pedidos_base pb  
    INNER JOIN (  
          
        -- NOTIFICACIÓN 1: Pasaron 10 min desde el pedido y nunca se le ha enviado nada de este flujo  
        SELECT   
pb.numero_pedido,  
            1 AS numero_notificacion  
        FROM pedidos_base pb  
        LEFT JOIN historial_notificaciones hn ON pb.numero_pedido COLLATE utf8mb4_unicode_ci = hn.numero_pedido COLLATE utf8mb4_unicode_ci  
        WHERE hn.numero_pedido IS NULL   
AND p_fecha_actual >= DATE_ADD(pb.fecha_pedido, INTERVAL 10 MINUTE)  
          
        UNION ALL  
        -- NOTIFICACIÓN 2: Ya se le envió la primera y pasaron 2 días completos  
        SELECT   
pb.numero_pedido,  
            2 AS numero_notificacion  
        FROM pedidos_base pb  
        INNER JOIN historial_notificaciones hn ON pb.numero_pedido COLLATE utf8mb4_unicode_ci = hn.numero_pedido COLLATE utf8mb4_unicode_ci  
        WHERE hn.total_notificaciones = 1  
          AND p_fecha_actual >= DATE_ADD(hn.fecha_notificacion_1, INTERVAL 2 DAY)  
            
        UNION ALL  
        -- NOTIFICACIÓN 3: Ya se le envió la segunda y pasaron otros 2 días  
        SELECT   
pb.numero_pedido,  
            3 AS numero_notificacion  
        FROM pedidos_base pb  
        INNER JOIN historial_notificaciones hn ON pb.numero_pedido COLLATE utf8mb4_unicode_ci = hn.numero_pedido COLLATE utf8mb4_unicode_ci  
        WHERE hn.total_notificaciones = 2  
          AND p_fecha_actual >= DATE_ADD(hn.fecha_notificacion_2, INTERVAL 2 DAY)  
  
    ) resultado ON pb.numero_pedido = resultado.numero_pedido  
    ORDER BY resultado.numero_notificacion, pb.numero_pedido;  
  
END;
```


```sql

CREATE DEFINER = super_scraper@`%` PROCEDURE sp_automation_registrar_notificacion_vitamina(
    IN p_numero_pedido VARCHAR(50),
    IN p_id_cliente_facturacion INT,
    OUT p_codigo_respuesta INT,       -- 0 = Éxito, 1 = Error/Límite excedido
    OUT p_mensaje_respuesta VARCHAR(100)
)
BEGIN
    DECLARE v_siguiente_notificacion INT DEFAULT 1;

    -- Manejo de errores críticos a nivel de BD
    DECLARE EXIT HANDLER FOR SQLEXCEPTION
    BEGIN
        ROLLBACK;
        SET p_codigo_respuesta = 1;
        SET p_mensaje_respuesta = 'Error crítico en el servidor al registrar la notificación.';
    END;

    START TRANSACTION;

    -- 1. Calculamos automáticamente el número de la siguiente notificación
    SELECT COALESCE(MAX(numero_notificacion), 0) + 1 
    INTO v_siguiente_notificacion
    FROM pedidos_vitaminas_notificaciones
    WHERE numero_pedido = p_numero_pedido 
      AND id_cliente_facturacion = p_id_cliente_facturacion;

    -- 2. Validación: Si ya se enviaron las 3 notificaciones, no permitimos más inserts
    IF v_siguiente_notificacion > 3 THEN
        SET p_codigo_respuesta = 1;
        SET p_mensaje_respuesta = CONCAT('Atención: Ya se alcanzó el límite máximo (3) de notificaciones para este pedido y cliente.');
        ROLLBACK;
    ELSE
        -- 3. Inserción limpia con el número calculado automáticamente
        INSERT INTO pedidos_vitaminas_notificaciones (
            numero_pedido, 
            id_cliente_facturacion,
            numero_notificacion, 
            created_at
        ) 
        VALUES (
            p_numero_pedido, 
            p_id_cliente_facturacion,
            v_siguiente_notificacion, 
            NOW()
        );

        SET p_codigo_respuesta = 0;
        SET p_mensaje_respuesta = CONCAT('Notificación ', v_siguiente_notificacion, ' registrada exitosamente.');
        COMMIT;
    END IF;

END;
```


```sql
CREATE PROCEDURE SP_automatizacion_vitaminas_detalle(  
    IN p_numero_pedido VARCHAR(100)  
)  
BEGIN  
    WITH DatosAgrupados AS (  
        -- 1. Primero agrupamos tu consulta original para obtener la cantidad por producto  
        SELECT  
            ped.numero_pedido,  
            ped.id_pedido,  
            ped.fecha_pedido,  
            ped.tiendas,  
            cf.id_cliente_facturacion,  
            cf.nombre_cliente_facturacion,  
            cf.documento_cliente_facturacion,  
            cf.direccion_cliente_facturacion,  
            cf.distrito_cliente_facturacion,  
            cf.provincia_cliente_facturacion,  
            cf.departamento_cliente_facturacion,  
            cf.telefono_cliente_facturacion,  
            COUNT(pp.id_pedido_producto) AS cantidad,        -- Calculamos la cantidad de items idénticos  
            MIN(pp.id_pedido_producto) AS id_pedido_producto, -- Tomamos el primer ID del producto para referencia  
            CONCAT('https://www.amazon.com/dp/', pp.sku_cf) AS url_producto,  
            pp.nombre_producto,  
            COALESCE(prod.image_1_webp, prod.image_2_webp, prod.image_3_webp) AS url_imagen  
  
        FROM pedidos ped  
        INNER JOIN pedidos_productos pp ON ped.id_pedido = pp.id_pedido AND pp.is_vitamin = TRUE  
        INNER JOIN pedido_cliente_facturacion pcf ON pcf.id = ped.id_pedido_cliente_facturacion  
        INNER JOIN cliente_facturacion cf ON pcf.id_cliente_facturacion = cf.id_cliente_facturacion  
        LEFT JOIN productos prod ON pp.sku_cf = prod.sku COLLATE utf8mb4_unicode_ci AND prod.is_enable  
        LEFT JOIN categories cat ON prod.category_code_id = cat.id  
        LEFT JOIN documentos_pedidos dp  
                  ON ped.id_pedido = dp.id_pedido  
                 AND pp.id_cliente_facturacion = dp.id_cliente_facturacion  
                 AND dp.tipo_documento = 'VITAMINAS'  
        WHERE cat.principal_id = 22  
          AND dp.id_documento_pedido IS NULL  
          AND ped.numero_pedido = p_numero_pedido COLLATE utf8mb4_unicode_ci -- Filtramos por el parámetro del SP  
        GROUP BY  
            ped.numero_pedido, ped.id_pedido, ped.fecha_pedido, ped.tiendas,  
            cf.id_cliente_facturacion, cf.nombre_cliente_facturacion, cf.documento_cliente_facturacion,  
            cf.direccion_cliente_facturacion, cf.distrito_cliente_facturacion, cf.provincia_cliente_facturacion,  
            cf.departamento_cliente_facturacion, cf.telefono_cliente_facturacion,  
            pp.sku_cf, pp.nombre_producto, url_producto, url_imagen  
    )  
    -- 2. Luego construimos la estructura JSON exacta que solicitaste  
    SELECT  
        JSON_OBJECT(  
            'numero_pedido', MAX(numero_pedido),  
            'id_pedido', MAX(id_pedido),  
            'fecha_pedido', MAX(fecha_pedido),  
            'tiendas', MAX(tiendas),  
            'cliente_facturacion', JSON_OBJECT(  
                'id_cliente_facturacion', MAX(id_cliente_facturacion),  
                'nombre_cliente_facturacion', MAX(nombre_cliente_facturacion),  
                'documento_cliente_facturacion', MAX(documento_cliente_facturacion),  
                'direccion_cliente_facturacion', MAX(direccion_cliente_facturacion),  
                'distrito_cliente_facturacion', MAX(distrito_cliente_facturacion),  
                'provincia_cliente_facturacion', MAX(provincia_cliente_facturacion),  
                'departamento_cliente_facturacion', MAX(departamento_cliente_facturacion),  
                'telefono_cliente_facturacion', MAX(telefono_cliente_facturacion)  
            ),  
            'productos', JSON_ARRAYAGG(  
                JSON_OBJECT(  
                    'cantidad', cantidad,  
                    'id_pedido_producto', id_pedido_producto,  
                    'nombre_producto', nombre_producto,  
                    'url_imagen', url_imagen,  
                    'url_producto', url_producto  
                )  
            )  
        ) AS resultado_json  
    FROM DatosAgrupados;  
  
END
```

