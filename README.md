# Tourist Microservices: Eureka Discovery + API Gateway Load Balancing (Java 25)

A complete Maven multi-module Spring Boot project with three applications:

- `eureka-server` — Eureka registry on port **8761**
- `tourist-service` — REST service registered as **TOURIST-SERVICE**, using an OS-assigned dynamic port (`server.port: 0`)
- `api-gateway` — Spring Cloud Gateway on port **8080**, routing `/tourist` to `lb://TOURIST-SERVICE` using Spring Cloud LoadBalancer

## Requirements

- JDK 25 (make sure `java -version` reports 25)
- Maven 3.9.9+ (or a compatible recent Maven)
- Internet access on first build so Maven can download dependencies

The project uses Spring Boot 3.5.6 and Spring Cloud 2025.0.0, with Java compiler release set to 25.

## Build

From this folder:

```bash
mvn clean package
```

## Run in Spring Tool Suite (STS)

1. Extract the ZIP.
2. In STS, choose **File → Import → Maven → Existing Maven Projects**.
3. Select the extracted `tourist-microservices-java25` folder and finish. If STS does not import the parent automatically, import each of the three module folders too.
4. Confirm STS is using JDK 25: **Window → Preferences → Java → Installed JREs**.
5. Run `EurekaServerApplication` as **Spring Boot App** first.
6. Run `TouristServiceApplication` as **Spring Boot App** twice. In the second run configuration, add program argument `--server.port=8082`. The default run uses a dynamic port; the second uses 8082. (Both register under `TOURIST-SERVICE`.)
7. Run `ApiGatewayApplication` as **Spring Boot App**.
8. Open `http://localhost:8761` and confirm that `API-GATEWAY` and two `TOURIST-SERVICE` instances appear in the registry.

## Run from terminal

Build first with `mvn clean package`, then open separate terminals in the project root:

**Terminal 1 — Eureka:**
```bash
java -jar eureka-server/target/eureka-server-1.0.0.jar
```

**Terminal 2 — Tourist instance 1 (dynamic port):**
```bash
java -jar tourist-service/target/tourist-service-1.0.0.jar
```

**Terminal 3 — Tourist instance 2 (fixed port 8082):**
```bash
java -jar tourist-service/target/tourist-service-1.0.0.jar --server.port=8082
```

**Terminal 4 — API Gateway:**
```bash
java -jar api-gateway/target/api-gateway-1.0.0.jar
```

Wait around 20–30 seconds after starting the services for Eureka registration and registry refresh.

## Test load balancing

Call the gateway endpoint repeatedly (not the tourist service ports directly):

```bash
curl -i http://localhost:8080/tourist
curl -i http://localhost:8080/tourist
curl -i http://localhost:8080/tourist
curl -i http://localhost:8080/tourist
```

Each successful request should return **HTTP 200** and JSON including `instancePort`. With both instances registered, the returned instance should alternate across instances over repeated requests in a round-robin pattern. Exact order can vary while instances start, stop, or registry state refreshes.

PowerShell loop:
```powershell
1..10 | ForEach-Object { (Invoke-WebRequest http://localhost:8080/tourist).Content; Start-Sleep -Milliseconds 300 }
```

Health checks:
- Eureka dashboard: `http://localhost:8761`
- Gateway health: `http://localhost:8080/actuator/health`
- Tourist service health: use the actual dynamic port shown in its startup log, e.g. `http://localhost:<port>/actuator/health`

## Failure behavior

If no healthy `TOURIST-SERVICE` instance is discoverable, Spring Cloud Gateway with the standard load-balancer configuration normally returns **HTTP 503 Service Unavailable**. The exact error may depend on the failure (for example, a connection failure to an instance may produce another 5xx). This project does not force all discovery failures to HTTP 500, because that is not the framework's default behavior.

## Troubleshooting

- **Gateway returns 503:** check Eureka at `http://localhost:8761`; confirm the tourist instances are registered and wait for registry refresh.
- **Port 8080 or 8761 is busy:** stop the process using it or change the corresponding `server.port`.
- **Only one tourist instance appears:** make sure you launched the service twice with different ports/instance IDs.
- **Java release error:** check `java -version`, `mvn -version`, and STS's installed JDK. All should point to JDK 25.
- **Build fails downloading dependencies:** check internet access and retry `mvn -U clean package`.

## Architecture

Client → `API Gateway :8080` → Eureka registry lookup → Spring Cloud LoadBalancer (round robin) → one registered `TOURIST-SERVICE` instance.
