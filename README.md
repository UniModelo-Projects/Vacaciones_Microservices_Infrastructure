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

The system is designed to handle failures gracefully. The **Order Service** and **Payment Service** have a 30% chance of failure during creation.

### Step 1: Start the environment
```powershell
docker-compose up -d --build
```

### Step 2: Trigger a failure
Send multiple order creation requests via the API Gateway. Some will fail with `500 Internal Server Error`.
```bash
curl -X POST http://localhost:8080/ordenes \
-H "Content-Type: application/json" \
-d '{"usuarioId": "user123", "productoIds": ["prod1"], "total": 100.0}'
```

### Step 3: Monitor Kafka
Open **KafkaUI** at [http://localhost:8086](http://localhost:8086).
Look for messages in the `order_retry_jobs` topic. This is the first indicator that the failure was caught and sent for retry.

### Step 4: Verify Persistence
The **Broker Service** will receive the Kafka message and save it to PostgreSQL. You can check the `retry_jobs` table in the `broker_db`.

### Step 5: Wait for the Scheduler
The Broker has a scheduler that runs every **10 seconds**. Once triggered:
1. It will pick up the `PENDING` job.
2. **Step A**: It will call the `/retry` endpoint of the original service.
3. **Step B**: It will attempt to send a confirmation email.
4. **Step C**: It will log the progress.
5. **Step D**: It will save a final tracking record in MongoDB.

### Step 6: Verify Final State
Check the `broker_final_db` in MongoDB to see the audit trail of the completed retry.
```bash
docker exec -it <mongodb-container-id> mongosh
use broker_final_db
db.order_final.find().pretty()
```

The job in PostgreSQL will now have a status of `SUCCESS`.
