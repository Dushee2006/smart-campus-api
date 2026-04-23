# Smart Campus API

## Overview

The Smart Campus API is a RESTful web service developed using JAX-RS. It simulates a smart campus environment where rooms and sensors are managed, and real-time as well as historical sensor readings are recorded and retrieved.

The system follows REST principles and uses in-memory data structures such as HashMap and ArrayList instead of a database, as required by the coursework specification.

**API Version:** v1


---

## API Design

The system consists of three main resources:

* Room Resource → manages rooms
* Sensor Resource → manages sensors
* Sensor Reading Resource → manages historical readings

### Resource Structure

```
/api/v1/rooms
/api/v1/rooms/{id}
/api/v1/sensors
/api/v1/sensors/{id}/readings
```

A **Sub-Resource Locator pattern** is used for nested resources.

---

## Project Structure

```
smart-campus-api/
│
├── src/main/java/com/smartcampus/
│   ├── resource/
│   ├── model/
│   ├── exception/
│   ├── mapper/
│   ├── filter/
│   └── store/
│
├── src/main/webapp/
├── pom.xml
└── .gitignore
```

---

## How to Run the Project

1. Clone repository

   ```
   git clone https://github.com/YOUR_USERNAME/smart-campus-api.git
   ```

2. Open in NetBeans

3. Build project

   ```
   mvn clean package
   ```

4. Deploy to Tomcat:

   * Copy `target/smart-campus-api.war` → `apache-tomcat/webapps/`
     OR
   * Right-click project → Run/Deploy in NetBeans

5. Start Tomcat

6. Access API:

   ```
   http://localhost:8080/smart-campus-api/api/v1
   ```

---

## Sample cURL Commands

### Create Room

```bash
curl -X POST http://localhost:8080/smart-campus-api/api/v1/rooms \
-H "Content-Type: application/json" \
-d '{"id":"R1","name":"Lab 1","capacity":50,"sensorIds":[]}'
```

### Create Sensor

```bash
curl -X POST http://localhost:8080/smart-campus-api/api/v1/sensors \
-H "Content-Type: application/json" \
-d '{"id":"S1","type":"CO2","status":"ACTIVE","currentValue":400,"roomId":"R1"}'
```

### Get Sensors (Filtered)

```bash
curl "http://localhost:8080/smart-campus-api/api/v1/sensors?type=CO2"
```

### Add Sensor Reading

```bash
curl -X POST http://localhost:8080/smart-campus-api/api/v1/sensors/S1/readings \
-H "Content-Type: application/json" \
-d '{"id":"SR1","timestamp":1710000000,"value":420.5}'
```

### Get Sensor Readings

```bash
curl http://localhost:8080/smart-campus-api/api/v1/sensors/S1/readings
```

### Delete Room (409 test)

```bash
curl -X DELETE http://localhost:8080/smart-campus-api/api/v1/rooms/R1
```

### Invalid Sensor Creation (422 test)

```bash
curl -X POST http://localhost:8080/smart-campus-api/api/v1/sensors \
-H "Content-Type: application/json" \
-d '{"id":"S2","type":"TEMP","status":"ACTIVE","currentValue":22,"roomId":"INVALID"}'
```

---

## HTTP Status Codes

| Code | Meaning                 |
| ---- | ----------------------- |
| 200  | Success                 |
| 201  | Created                 |
| 403  | Sensor in maintenance   |
| 404  | Resource not found      |
| 409  | Room has sensors        |
| 422  | Invalid linked resource |
| 500  | Internal server error   |

---

# PART 1.1

### EXACT QUESTION:

"In your report, explain the default lifecycle of a JAX-RS Resource class..."

### Answer:

JAX-RS resource classes follow a per-request lifecycle, meaning a new instance is created for each incoming request. This ensures thread safety because no instance is shared across multiple requests.

However, storing data in instance variables would result in data loss between requests. To address this, the implementation uses a centralized DataStore class with static HashMaps and ArrayLists. Since static fields belong to the class rather than an instance, they persist across all requests.

At the same time, shared mutable data introduces the possibility of race conditions when multiple requests access or modify the same structures concurrently. In a production environment, thread-safe structures such as ConcurrentHashMap would be used to ensure safe access and prevent data corruption.

---

# PART 1.2

### EXACT QUESTION:

"Why is the provision of Hypermedia (links and navigation within responses)..."

### Answer:

HATEOAS (Hypermedia As The Engine Of Application State) is a REST principle where API responses include links that guide clients to available actions.

In this system, the discovery endpoint (`GET /api/v1`) provides links to `/api/v1/rooms` and `/api/v1/sensors`. Clients can follow these links dynamically without needing prior knowledge of the API structure.

This reduces dependency on hardcoded URLs and ensures that if endpoints change in future versions, clients that follow links will still function correctly. Unlike static documentation, which can become outdated, hypermedia responses are always current.

For example, the discovery endpoint returns links that clients can follow instead of hardcoding paths. If the API structure changes, clients relying on hypermedia links will continue to function correctly.

This appro1ach improves flexibility, discoverability, and reduces coupling between client and server.

---

# PART 2.1

### EXACT QUESTION:

"When returning a list of rooms..."

### Answer:

Returning only room IDs minimizes payload size and reduces network bandwidth usage. However, it introduces the N+1 problem, where the client must make additional requests to retrieve full details for each room.

Returning full objects increases payload size but provides all required information in a single response. For example, returning `["R1","R2"]` is efficient, but the client must call `/rooms/R1` separately. Returning full objects eliminates this need.

For smaller datasets, returning full objects is more practical and simplifies client-side processing. For large datasets, returning IDs or using pagination would be more efficient.

---

# PART 2.2

### EXACT QUESTION:

"Is the DELETE operation idempotent..."

### Answer:

An operation is idempotent if repeating it produces the same server state. In this implementation, DELETE satisfies this property.

The first DELETE request removes the room and returns 204 No Content. A second DELETE request for the same room returns 404 Not Found because the resource no longer exists.

Although the response changes, the server state remains unchanged after the first deletion. This confirms idempotency.

This behavior is important in distributed systems where duplicate requests may occur due to network issues.

---

# PART 3.1

### EXACT QUESTION:

"We explicitly use the @Consumes..."

### Answer:

The @Consumes(MediaType.APPLICATION_JSON) annotation specifies that the method only accepts JSON input. If a client sends a request with a different Content-Type such as text/plain or application/xml, JAX-RS detects the mismatch during request processing.

Before invoking the method, the framework attempts to match the Content-Type with supported media types. Since no match exists, it fails to find a suitable MessageBodyReader to deserialize the input.

As a result, JAX-RS automatically returns HTTP 415 Unsupported Media Type, and the resource method is never executed.

This ensures strict validation of input formats and prevents processing invalid data.

---

# PART 3.2

### EXACT QUESTION:

"You implemented this filtering using @QueryParam..."

### Answer:

Query parameters are more appropriate for filtering because they represent optional conditions applied to a collection. In contrast, path-based filtering incorrectly treats filters as resources.

For example, `/sensors?type=CO2&status=ACTIVE` allows multiple filters to be combined in a single request. If path-based filtering was used (e.g., `/sensors/type/CO2/status/ACTIVE`), the API would require many different endpoints, making it harder to maintain and less flexible.

Query parameters also maintain REST semantics by clearly indicating that the request is for a filtered view of the collection.

---

# PART 4.1

### EXACT QUESTION:

"Discuss the architectural benefits of the Sub-Resource Locator pattern..."

### Answer:

The Sub-Resource Locator pattern allows nested resources to be handled by separate classes. In this implementation, SensorResource delegates requests to SensorReadingResource using a method annotated with @Path("/{sensorId}/readings").

This design follows the Single Responsibility Principle, keeping each class focused on a specific resource. It reduces complexity, improves readability, and allows each resource to be tested independently.

It also improves extensibility, as new nested resources can be added without modifying existing classes.

---

# PART 5.1

### EXACT QUESTION:

"Why is HTTP 422 often considered..."

### Answer:

HTTP 422 is more accurate because the request is valid but contains incorrect data. The endpoint exists, so 404 is not appropriate.

In this case, the roomId does not exist, making the request semantically invalid. HTTP 422 clearly communicates that the issue lies in the request payload.

---

# PART 5.2

### EXACT QUESTION:

"From a cybersecurity standpoint..."

### Answer:

Exposing stack traces leads to information leakage. It reveals internal details such as class names, line numbers, and framework versions.

For example, a stack trace may reveal:

* Exact framework version (e.g., Jersey version)
* Internal class names and file paths
* Line numbers where errors occurred
* Server type (e.g., Tomcat)

Attackers can use this information to identify vulnerabilities and exploit the system.

To prevent this, the system uses a global exception handler that returns generic error messages instead of exposing internal details.

---

# PART 5.3

### EXACT QUESTION:

"Why is it advantageous to use JAX-RS filters..."

### Answer:

Filters handle cross-cutting concerns such as logging in a centralized manner. This avoids duplicating logging code in every resource method.

Using filters improves maintainability, keeps resource classes clean, and ensures consistent logging across the application.

---

## Technologies Used

* Java
* JAX-RS (Jersey)
* Maven
* Apache Tomcat

---

## Notes

* No database used
* Fully compliant with coursework requirements
