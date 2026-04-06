# Mejia Examen - Microservices Infrastructure

This project contains the infrastructure configuration to orchestrate a suite of microservices with a robust **Retry Mechanism** using Kafka and a **Chain of Responsibility** (CoR) pattern.

## Architecture
- **Eureka Server**: Service Registry.
- **API Gateway**: Entry point for all services.
- **Product Service**: Product management (MongoDB).
- **Order Service**: Order processing (MongoDB).
- **Payment Service**: Payment processing (MongoDB).
- **Broker Service**: Core retry engine (PostgreSQL + Kafka).
- **Kafka**: Message broker for failed job notifications.
- **PostgreSQL**: Stores persistent retry jobs.
- **MongoDB**: Final tracking and audit logs.
- **KafkaUI**: Web interface for Kafka monitoring.

## Microservices Repositories
- [API Gateway](https://github.com/UniModelo-Projects/Vacaciones_Microservices_API_Gateway)
- [Eureka Server](https://github.com/UniModelo-Projects/Vacaciones_Microservices_Eureka_Server)
- [Broker Service](https://github.com/UniModelo-Projects/Vacaciones_Microservices_Broker)
- [Product Service](https://github.com/UniModelo-Projects/Vacaciones_Microservices_Products)
- [Order Service](https://github.com/UniModelo-Projects/Vacaciones_Microservices_Orders)
- [Payment Service](https://github.com/UniModelo-Projects/Vacaciones_Microservices_Payments)

## Port Map
| Service | Port | Description |
|---------|------|-------------|
| API Gateway | 8080 | Entry point |
| Eureka Server | 8761 | Registry Dashboard |
| KafkaUI | 8086 | Kafka Monitoring |
| Product Service | 8081 | Backend |
| Order Service | 8082 | Backend |
| Payment Service | 8083 | Backend |
| Broker Service | 8084 | Retry Engine |
| MongoDB | 27017 | Database |
| PostgreSQL | 5432 | Database |

---

## How to Test the Error Flow (Retry Mechanism)

The system is designed to handle failures gracefully across all core services (**Products, Orders, and Payments**).

### Failure Simulation:
- **Order Service**: 30% chance of failure on creation.
- **Payment Service**: 40% chance of failure on processing.
- **Product Service**: 20% chance of failure on creation.

### Step 1: Start the environment
```powershell
docker-compose up -d --build
```

### Step 2: Trigger a failure
You can trigger failures for any of the services using `curl` or Postman.

#### Example for Orders:
```bash
curl -X POST http://localhost:8080/ordenes \
-H "Content-Type: application/json" \
-d '{"usuarioId": "user123", "productoIds": ["prod1"], "total": 100.0}'
```

#### Example for Payments:
```bash
curl -X POST http://localhost:8080/pagos/procesar \
-H "Content-Type: application/json" \
-d '{"ordenId": "order123", "monto": 100.0}'
```

#### Example for Products:
```bash
curl -X POST http://localhost:8080/productos \
-H "Content-Type: application/json" \
-d '{"nombre": "Keyboard", "precio": 50.0, "stock": 20}'
```

### Step 3: Monitor Kafka
Open **KafkaUI** at [http://localhost:8086](http://localhost:8086).
Check the corresponding topics: `order_retry_jobs`, `payments_retry_jobs`, or `product_retry_jobs`.

### Step 4: Verify Persistence
The **Broker Service** saves jobs to PostgreSQL. Check the `retry_jobs` table in `broker_db`.

### Step 5: Wait for the Scheduler (10 seconds)
The Broker executes the **Chain of Responsibility**:
1. **Step A**: Calls the `/retry` endpoint of the failed service.
2. **Step B**: Sends a confirmation email.
3. **Step C**: Logs progress.
4. **Step D**: Saves final tracking to MongoDB.

### Step 6: Verify Final State
Check `broker_final_db` in MongoDB for the corresponding collections: `order_final`, `payment_final`, or `product_final`.
```bash
docker exec -it <mongodb-container-id> mongosh
use broker_final_db
db.order_final.find().pretty()   # Or payment_final / product_final
```
