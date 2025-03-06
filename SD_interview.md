# System design interview: Aplicación de Delivery

## Problema a Resolver

Diseñar una aplicación de delivery que permita a los usuarios:
- Realizar pedidos de comida y otros productos.
- Seguir el estado de sus pedidos en tiempo real.
- Visualizar restaurantes disponibles en su área.
- Autenticarse de manera segura y gestionar su perfil.
- Integrar pagos a través de pasarelas externas.
- Soportar geolocalización para búsquedas y seguimiento en tiempo real.

## Escala de la Aplicación

- **Usuarios Activos Diarios:** Ej. 10 millones.
- **Pedidos Diarios:** Aproximadamente 2 millones.
- **Transacciones en Tiempo Real:** Actualización continua de estados de pedidos y seguimiento geolocalizado.
- **Alto Volumen de Consultas:** Búsqueda y filtrado de restaurantes basados en la ubicación del usuario.

## Observaciones

- Se requiere alta disponibilidad, escalabilidad y resiliencia.
- La aplicación debe integrarse con servicios de terceros (API de geolocalización, pasarelas de pago).
- Se utilizará una arquitectura basada en microservicios para desacoplar funcionalidades y facilitar la evolución independiente de cada componente.

## Base de Datos

### Esquema y Entidades

- **Usuarios:**  
  - Atributos: id, nombre, email, contraseña (hash), teléfono, direcciones.
  - Relaciones: Un usuario puede realizar múltiples pedidos.

- **Restaurantes:**  
  - Atributos: id, nombre, dirección, ubicación (coordenadas), categoría, menú, horarios.
  - Relaciones: Un restaurante puede atender múltiples pedidos.

- **Pedidos:**  
  - Atributos: id, id_usuario, id_restaurante, lista de productos, estado (ej. "recibido", "en preparación", "en camino", "entregado"), timestamps, ubicación del repartidor (para seguimiento en tiempo real).
  - Relaciones: Cada pedido pertenece a un único usuario y a un restaurante.

- **Pagos:**  
  - Atributos: id, id_pedido, método de pago, estado, detalles de la transacción.
  - Relaciones: Cada pago se asocia a un pedido.

### Elección de Bases de Datos

- **Amazon RDS (SQL):** Para datos transaccionales críticos (usuarios, pagos).
- **Amazon DynamoDB (NoSQL):** Para datos de seguimiento en tiempo real y geolocalización (pedidos, estado y actualizaciones de ubicación).

## Diseño de Servicios

### Componentes y Responsabilidades

- **Microservicio de Pedidos y Seguimiento:**  
  - Funciones: Gestión de órdenes, actualización de estados y seguimiento en tiempo real.
  - Eventos: Publica cambios de estado a través de colas de mensajes (SQS/SNS) para notificar a otros servicios y clientes (por ejemplo, vía WebSockets).

- **Microservicio de Restaurantes:**  
  - Funciones: Consulta y gestión de la información de los restaurantes, integración con APIs de geolocalización para búsquedas basadas en ubicación.

- **Microservicio de Autenticación y Usuarios:**  
  - Funciones: Registro, inicio de sesión, administración de perfiles y emisión de tokens (OAuth2, JWT).
  - Seguridad: Almacenamiento seguro de contraseñas (bcrypt), validación y refresco de tokens.

- **Microservicio de Pagos:**  
  - Funciones: Procesamiento y validación de transacciones, integración con proveedores de pago externos (MercadoPago, PayPal).

### Protocolos de Comunicación

- **Cliente a Servicios:**  
  - REST para operaciones CRUD y consultas.
  - WebSockets o SSE para recibir actualizaciones en tiempo real (seguimiento de pedidos).

- **Entre Microservicios:**  
  - Comunicación síncrona vía REST/GRPC.
  - Mensajería asíncrona (Amazon SQS/SNS) para el desacoplamiento de eventos críticos, como cambios en el estado de un pedido.

## Infraestructura

### Plataforma Cloud (AWS)

- **Amazon CloudFront (CDN):** Distribución de contenido estático a nivel global.
- **Amazon API Gateway:** Punto de entrada centralizado para las APIs, gestionando autenticación, enrutamiento y seguridad.
- **Amazon ECS/EKS (Contenedores):**  
  - Despliegue de microservicios empaquetados en contenedores.
  - La elección entre EKS (Kubernetes) o ECS (especialmente con Fargate) depende del nivel de control y experiencia del equipo.
- **Amazon RDS:** Base de datos SQL para datos transaccionales.
- **Amazon DynamoDB:** Base de datos NoSQL para datos de seguimiento en tiempo real y geolocalización.
- **Amazon SQS/SNS:** Gestión de colas de mensajes para la comunicación asíncrona y distribución de eventos.
- **Amazon CloudWatch:** Monitoreo, logging y alertas proactivas.

### Escalabilidad y Resiliencia

- **Auto-escalado:** Configuración para contenedores (ECS/EKS) y bases de datos.
- **Alta Disponibilidad:** Despliegue en Multi-AZ, replicación y balanceadores de carga.
- **Tolerancia a Fallos:** Implementación de circuit breakers, estrategias de retry y monitoreo continuo.

## Autenticación

### Mecanismos y Flujo

- **Protocolo:** OAuth2 con emisión de tokens JWT.
- **Endpoints Principales:**
  - `/register`: Registro de nuevos usuarios.
  - `/login`: Validación de credenciales y emisión de token JWT.
  - `/refresh`: Renovación de tokens para mantener la sesión activa.
- **Seguridad Adicional:**  
  - Cifrado TLS en tránsito.
  - Almacenamiento seguro de contraseñas mediante algoritmos de hashing (bcrypt).
  - Validación de tokens en el API Gateway para proteger cada endpoint.

## Interpretación de Requerimientos

### Casos de Uso y Flujos de la Aplicación

- **Seguimiento de Pedidos en Tiempo Real:**
  - El usuario realiza un pedido y el microservicio de Pedidos registra la orden.
  - Cada actualización en el estado del pedido (ej. "en preparación", "en camino", "entregado") produce un evento que se publica en Amazon SNS/SQS.
  - Un servicio de notificaciones (o el mismo microservicio de Pedidos) consume estos eventos y utiliza WebSockets/SSE para actualizar la interfaz del usuario en tiempo real.

- **Consulta de Restaurantes Disponibles:**
  - El cliente envía su ubicación actual.
  - El microservicio de Restaurantes consulta la base de datos NoSQL utilizando índices geoespaciales para filtrar y ordenar los restaurantes según la proximidad.
  - Se integra una API de geolocalización (por ejemplo, Google Maps API) para obtener datos adicionales como rutas y tiempos estimados de llegada.

- **Autenticación y Gestión de Usuarios:**
  - Registro e inicio de sesión con validación de credenciales.
  - Emisión y validación de tokens JWT en cada solicitud.
  - Soporte para múltiples dispositivos mediante el manejo adecuado de sesiones y tokens.

### Manejo de Errores y Excepciones

- Implementación de políticas de retry y circuit breakers en los microservicios.
- Monitoreo continuo y alertas en Amazon CloudWatch para detectar y mitigar fallos de manera proactiva.
- Estrategias de fallback para notificar a los usuarios en caso de fallos en servicios externos (p. ej., geolocalización o pasarelas de pago).

## Solución

### Ventajas del Enfoque

- **Escalabilidad:** Arquitectura basada en microservicios desacoplados que permite escalar horizontalmente cada componente de manera independiente.
- **Resiliencia:** Uso de colas de mensajes para desacoplar eventos críticos, replicación en bases de datos y despliegue en Multi-AZ.
- **Seguridad:** Implementación robusta de autenticación y autorización con JWT, cifrado de datos y políticas de acceso estrictas.
- **Flexibilidad:** Integración con servicios externos para pagos y geolocalización, permitiendo una rápida adaptación a cambios en el entorno y demandas del mercado.

### Desventajas y Consideraciones

- **Complejidad Operacional:** La gestión de múltiples microservicios y la coordinación de comunicaciones asíncronas pueden incrementar la complejidad de la solución.
- **Costos Operativos:** Un alto uso de servicios en la nube y estrategias de replicación y escalado pueden impactar en el costo.
- **Dependencias Externas:** La fiabilidad de la aplicación dependerá en parte de la disponibilidad y desempeño de APIs de terceros (por ejemplo, servicios de geolocalización y pasarelas de pago).

