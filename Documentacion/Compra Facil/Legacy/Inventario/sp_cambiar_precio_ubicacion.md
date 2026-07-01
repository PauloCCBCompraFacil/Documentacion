Actualiza la ubicación física y/o el estado del producto de una unidad de inventario Lima. (Nota: pese al nombre, no modifica precio, solo ubicación y estado).

### Parámetros

| Parámetro | Tipo | Descripción |
|---|---|---|
| `p_id_producto_fisico` | bigint unsigned | ID de la unidad física a modificar (`id_producto_fisico_inventario`). |
| `p_ubicacion` | varchar | Nueva ubicación física. NULL = no se modifica. No puede ser string vacío. |
| `p_id_usuario` | int | Usuario que realiza el cambio (queda registrado en el historial). |
| `p_estado_producto` | varchar | Nuevo estado del producto. NULL = no se modifica. Debe ser uno de: `nuevo`, `open_box`, `caja_golpeada`, `sin_empaque_original`. |
| `p_resultado` (OUT) | int | 1 = éxito, 0 = error. |
| `p_mensaje` (OUT) | varchar | Mensaje descriptivo del resultado o error. |

### Validaciones
- La unidad debe existir (`id_producto_fisico_inventario`).
- `p_ubicacion` no puede ser string vacío (sí puede ser NULL para no tocarla).
- `p_estado_producto`, si viene, debe ser un valor válido del enum.

### Lógica
1. Obtiene la ubicación y estado actuales de la unidad (para el registro de historial).
2. Actualiza `descripcion_ubicacion` y/o `estado_producto` solo si el parámetro correspondiente no es NULL (usa `IF(param IS NOT NULL, param, valor_actual)`).
3. Arma un texto descriptivo con el detalle de cada cambio (de "X" a "Y").
4. Inserta un registro en el historial de la unidad con ese detalle.
5. Si hay cualquier error SQL, hace `ROLLBACK` y retorna el mensaje de error capturado.

### Tablas utilizadas
- `unidad_inventario_lima` — unidad física a actualizar (tabla principal)
- `historial_unidad_inventario_lima` — registro del cambio realizado
```sql
create  procedure sp_cambiar_precio_ubicacion(IN p_id_producto_fisico bigint unsigned,  
                                                               IN p_ubicacion varchar(500), IN p_id_usuario int,  
                                                               IN p_estado_producto varchar(50), OUT p_resultado int,  
                                                               OUT p_mensaje varchar(500))  
BEGIN  
    DECLARE v_existe INT DEFAULT 0;  
    DECLARE v_ubicacion_anterior VARCHAR(500);  
    DECLARE v_estado_anterior VARCHAR(50);  
    DECLARE v_descripcion_historial TEXT DEFAULT '';  
    DECLARE v_error_msg VARCHAR(500);  
  
    DECLARE EXIT HANDLER FOR SQLEXCEPTION  
        BEGIN            GET DIAGNOSTICS CONDITION 1 v_error_msg = MESSAGE_TEXT;  
            ROLLBACK;  
            SET p_resultado = 0;  
            SET p_mensaje = CONCAT('SQL Error: ', v_error_msg);  
        END;  
  
    START TRANSACTION;  
  
    SELECT COUNT(*)  
    INTO v_existe  
    FROM unidad_inventario_lima  
    WHERE id_producto_fisico_inventario = p_id_producto_fisico;  
  
    SELECT descripcion_ubicacion, estado_producto  
    INTO v_ubicacion_anterior, v_estado_anterior  
    FROM unidad_inventario_lima  
    WHERE id_producto_fisico_inventario = p_id_producto_fisico;  
  
    IF v_existe = 0 THEN  
        SET p_resultado = 0;  
        SET p_mensaje = 'Producto físico no encontrado';  
        ROLLBACK;  
    ELSEIF p_ubicacion IS NOT NULL AND p_ubicacion = '' THEN  
        SET p_resultado = 0;  
        SET p_mensaje = 'La ubicación no puede ser un string vacío';  
        ROLLBACK;  
    ELSEIF p_estado_producto IS NOT NULL AND  
           p_estado_producto NOT IN ('nuevo', 'open_box', 'caja_golpeada', 'sin_empaque_original') THEN  
        SET p_resultado = 0;  
        SET p_mensaje =  
                'Estado de producto inválido. Valores permitidos: nuevo, open_box, caja_golpeada, sin_empaque_original';  
        ROLLBACK;  
    ELSE  
        UPDATE unidad_inventario_lima  
        SET descripcion_ubicacion = IF(p_ubicacion IS NOT NULL, p_ubicacion, descripcion_ubicacion),  
            estado_producto       = IF(p_estado_producto IS NOT NULL, p_estado_producto, estado_producto)  
        WHERE id_producto_fisico_inventario = p_id_producto_fisico;  
  
        IF p_ubicacion IS NOT NULL THEN  
            SET v_descripcion_historial = CONCAT(v_descripcion_historial,  
                                                 'Ubicación cambiada de "',  
                                                 COALESCE(v_ubicacion_anterior, 'sin ubicación'),  
                                                 '" a "', p_ubicacion, '". ');  
        END IF;  
  
  
        IF p_estado_producto IS NOT NULL THEN  
            SET v_descripcion_historial = CONCAT(v_descripcion_historial,  
                                                 'Estado cambiado de "', COALESCE(v_estado_anterior, 'sin estado'),  
                                                 '" a "', p_estado_producto, '".');  
        END IF;  
  
        INSERT INTO historial_unidad_inventario_lima (id_unidad_inventario,  
                                                      id_usuario_responsable,  
                                                      descripcion)  
        VALUES (p_id_producto_fisico,  
                p_id_usuario,  
                v_descripcion_historial);  
  
        COMMIT;  
        SET p_resultado = 1;  
        SET p_mensaje = 'Actualización realizada correctamente';  
    END IF;  
END;

```
