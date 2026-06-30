
# Modificaciones tablas


```sql
ALTER TABLE documentos_pedidos  
ADD COLUMN  tipo_documento ENUM ('DNI', 'VITAMINAS', 'FACTURA');
```



```sql
CREATE TABLE pedidos_vitaminas_notificaciones ( 
id INT AUTO_INCREMENT PRIMARY KEY, 
numero_pedido VARCHAR(50) NOT NULL, 
numero_notificacion INT NOT NULL, -- 1, 2 o 3 
created_at DATETIME DEFAULT CURRENT_TIMESTAMP, 
INDEX idx_pedido_notificacion (numero_pedido, numero_notificacion) 
);
```

