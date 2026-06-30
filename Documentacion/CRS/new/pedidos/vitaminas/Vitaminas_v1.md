
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




# Nuevos SP

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