
# Nuevas Tablas

Catálogo de todos los estados posibles en tu operación

```sql
CREATE TABLE pedidos_productos_documentos (
    id_documento BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    id_pedido BIGINT UNSIGNED NOT NULL,
    tipo_documento ENUM, -- Ej: 'DNI', 'VITAMINAS', 'FACTURA'
    url_archivo TEXT NOT NULL,
    estado_verificacion VARCHAR(30) DEFAULT 'PENDIENTE', -- 'PENDIENTE', 'APROBADO', 'RECHAZADO'
    fecha_subida TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_docs_pedido FOREIGN KEY (id_pedido) REFERENCES pedidos(id_pedido)
);
```





