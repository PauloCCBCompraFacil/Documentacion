```
mehrmaid
sequenceDiagram 
	Usuario->>App: Toca botón "Iniciar con Huella" 
	App->>OS (iOS/Android): Solicita autenticación biométrica OS-->>App: Éxito (Token local) App->>Backend API: Envía token de validación Backend API-->>App: JWT Sesión Válida 
	App->>Usuario: Acceso concedido (Home Screen)
```



```mehrmaid
graph LR
T1 --> T2 & T3 --> T4 & T5 --> T6

T1("![|100x100](https://upload.wikimedia.org/wikipedia/commons/thumb/1/10/2023_Obsidian_logo.svg/1024px-2023_Obsidian_logo.svg.png)")
T2("$\nabla_\theta \mathbb{E}_{\tau\sim p_\theta}[R(\tau)]x$")
T3("| First Name | Last Name |
| ---------- | --------- |
| Doug       | Engelbart |")
T4("#plugins/mehrmaid")
T5("[[mehrmaid]]")
T6("![|80](https://upload.wikimedia.org/wikipedia/commons/thumb/6/60/Obsidian_software_logo.svg/1297px-Obsidian_software_logo.svg.png)")
```
