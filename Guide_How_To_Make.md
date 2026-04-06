# Guía Completa: Construcción de Microservicios de Extremo a Extremo (0 a 100%)

Esta guía detalla el proceso técnico, arquitectónico y de resolución de problemas para construir el ecosistema de microservicios de este proyecto.

---

## 1. Visión General de la Arquitectura
El sistema utiliza un patrón de microservicios distribuido con los siguientes componentes:
- **Descubrimiento de Servicios**: Eureka Server.
- **Enrutamiento Centralizado**: API Gateway.
- **Persistencia**: MongoDB independiente por servicio.
- **Observabilidad**: Logs centralizados en CloudWatch (vía LocalStack).
- **Contenedorización**: Docker y Docker Compose.

### Tecnologías Clave:
- **Java 21** (LTS)
- **Spring Boot 3.2.5**
- **Spring Cloud 2023.0.1**
- **Maven** (Gestor de dependencias)

---

## 2. Paso 1: Configuración del Service Discovery (Eureka Server)
Es el primer componente en levantarse. Permite que los servicios se conozcan entre sí sin IPs estáticas.

1. **Creación**: Usar Spring Initializr con la dependencia `Eureka Server`.
2. **Código**: Añadir `@EnableEurekaServer` en la clase principal.
3. **Configuración (`application.yml`)**:
   ```yaml
   server:
     port: 8761
   eureka:
     client:
       register-with-eureka: false # No se registra a sí mismo
       fetch-registry: false
   ```

---

## 3. Paso 2: Desarrollo del API Gateway
Actúa como fachada única para el cliente (Postman/Frontend).

1. **Dependencias**: `Gateway`, `Eureka Discovery Client`.
2. **Configuración Crítica**: Definir los predicados de ruta para redirigir el tráfico.
   ```yaml
   spring:
     cloud:
       gateway:
         routes:
           - id: product-service
             uri: lb://PRODUCT-SERVICE # Balanceo de carga vía Eureka
             predicates:
               - Path=/productos/**
   ```

---

## 4. Paso 3: Microservicios de Negocio (Productos, Órdenes, Pagos)
Cada microservicio sigue un patrón MVC.

### Estructura de Capas:
- **Model**: Clases POJO con anotaciones `@Document` de MongoDB.
- **Repository**: Interfaces que extienden de `MongoRepository<T, ID>`.
- **Controller**: Exposición de los 13 endpoints CRUD y de lógica.

### Ejemplo de Implementación (Product Service):
1. **Dependencias**: `Web`, `MongoDB`, `Eureka Client`, `Lombok`, `Logback-AWSLogs`.
2. **Lógica de Logs**: Se utiliza `SLF4J` con `@Slf4j` o `LoggerFactory` para registrar cada operación técnica.

---

## 5. Paso 4: Infraestructura de Logs Centralizados (El "Gran Reto")
El sistema envía logs a **AWS CloudWatch** simulado por **LocalStack**.

### El Problema Técnico Encontrado:
La librería `ca.pjer:logback-awslogs-appender` ignora la propiedad estándar `<endpoint>`. Esto causa que los microservicios intenten conectar con AWS real, fallando por falta de credenciales.

### La Solución:
Se debe utilizar específicamente la propiedad **`<cloudWatchEndpoint>`** en el archivo `logback-spring.xml`:
```xml
<appender name="CLOUDWATCH" class="ca.pjer.logback.AwsLogsAppender">
    <logGroupName>mi-log-group</logGroupName>
    <logRegion>us-east-1</logRegion>
    <cloudWatchEndpoint>http://localstack:4566</cloudWatchEndpoint> <!-- CLAVE -->
    <accessKeyId>test</accessKeyId>
    <secretAccessKey>test</secretAccessKey>
</appender>
```

---

## 6. Paso 5: Orquestación con Docker
Para que todo funcione al unísono, se crearon Dockerfiles optimizados.

1. **Dockerfile base**:
   ```dockerfile
   FROM eclipse-temurin:21-jre-jammy
   COPY target/*.jar app.jar
   ENTRYPOINT ["java", "-jar", "/app.jar"]
   ```
2. **Docker Compose**: Define el orden de encendido (`depends_on`) para asegurar que Eureka y MongoDB estén listos antes que los microservicios.

---

## 7. Paso 6: Resiliencia con Kafka y Chain of Responsibility (Broker Service)
El sistema incluye un mecanismo de auto-recuperación para fallos en la creación de registros.

### Flujo de Trabajo:
1. **Detección**: Si un microservicio falla al guardar (simulado o real), envía el objeto envuelto a Kafka.
2. **Orquestación (Broker)**: El `broker-service` captura el evento y ejecuta una cadena de responsabilidad:
   - **Step A**: Reintento HTTP vía Feign.
   - **Step B**: Envío de email de éxito.
   - **Step C**: Actualización de control en PostgreSQL.
   - **Step D**: Persistencia final en MongoDB.

### Configuración de Reintentos:
Se implementó un **Backoff Exponencial** en el Scheduler para evitar saturar los servicios en caso de caídas prolongadas.

---

## 8. Paso 7: Pruebas con Postman (Ciclo de Vida)
Se entrega una colección con **variables de sistema** (`{{Product_ID}}`, etc.).

### Flujo de Prueba Recomendado:
1. **Crear Producto**: Genera un ID en MongoDB y un log en CloudWatch.
2. **Copiar ID**: Guardar el ID en la variable de colección `Product_ID`.
3. **Crear Orden**: Usa `{{Product_ID}}`.
4. **Pagar**: Usa `{{Order_ID}}`.
5. **Verificar Logs**: Ejecutar "Ver Logs Producto" en Postman para confirmar que LocalStack recibió la traza.

---

## 8. Conclusión y Mantenimiento
Para realizar cambios:
1. Modificar código.
2. Re-compilar: `.\mvnw.cmd clean package -DskipTests`.
3. Re-levantar: `docker-compose up -d --build`.
