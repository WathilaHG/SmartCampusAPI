# Smart Campus Sensor and Room Management API

## 1. Overview of the API Design

The Smart Campus RESTful API is a RESTful web service developed via JAX-RS specifically for the administration of Rooms and Sensors at the university level. It exposes multiple endpoints for interaction between the Campus Facilities Management staff's and automated building systems, which retrieve and manipulate campus resources associated with Rooms/Sensors, as well as their respective sensor readings

### Project Structure:
SmartCampusAPI/
   src/main/java/
      com.smartcampus.config       # AppConfig - sets up the JAX-RS application
      com.smartcampus.model        # Room, Sensor, SensorReading, ErrorResponse POJOs
      com.smartcampus.resource     # All REST endpoints (rooms, sensors, readings)
      com.smartcampus.exception    # Custom exceptions and their mappers
      com.smartcampus.filter       # Request and response logging filter
      com.smartcampus.storage      # DataStore - holds all in-memory data using HashMaps
   pom.xml                          # Maven project file with Jersey and Jackson dependencies

## 2. Build and Launch Instructions

This project is configured as a standard Maven-based web application.

### Requirements:
- Java JDK 11 or higher
- Apache Maven
- Apache Tomcat
- NetBeans IDE

### Steps to Run:

** 1. Clone the repository

```bash
git clone https://github.com/WathilaHG/SmartCampusAPI.git
cd SmartCampusAPI

```

**2. Importing the project into NetBeans**

You can open NetBeans.

Select the File > Open Project from the Menu bar, and choose the folder containing your project file.

Click Open Project.

**3. Compiling the project**

Right click on the project in the Project panel.

Click Clean and Build from the context menu. The result of compiling should display "BUILD SUCCESS" in the Output pane.

**4. Starting the application server**

Right click on the project again.

Select Run from the context Menu. The project will be deployed automatically by deploying it to the embedded Tomcat server.

Wait for Deployment succeeded to display in the Output pane.

**5. The Smart Campus API is hosted locally and can be accessed at:**

http://localhost:8080/SmartCampusAPI/api/v1

---

## 3. Sample cURL Commands

### 1. Get API discovery information
```bash
curl -X GET http://localhost:8080/SmartCampusAPI/api/v1
```

### 2. Create a room
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/rooms \
-H "Content-Type: application/json" \
-d '{"id":"LIB-301","name":"Library Quiet Study","capacity":50}'
```

### 3. Create a sensor linked to a room
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors \
-H "Content-Type: application/json" \
-d '{"id":"CO2-001","type":"CO2","status":"ACTIVE","currentValue":0.0,"roomId":"LIB-301"}'
```

### 4. Get all sensors filtered by type
```bash
curl -X GET "http://localhost:8080/SmartCampusAPI/api/v1/sensors?type=CO2"
```

### 5. Post a reading to a sensor
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors/CO2-001/readings \
-H "Content-Type: application/json" \
-d '{"value":412.5}'
```

### 6. Attempt to delete a room that still has sensors (expects 409 Conflict)
```bash
curl -X DELETE http://localhost:8080/SmartCampusAPI/api/v1/rooms/LIB-301
```

### 7. Attempt to post a reading to a sensor in MAINTENANCE (expects 403 Forbidden)
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors/OCC-001/readings \
-H "Content-Type: application/json" \
-d '{"value":25.0}'
```


## 4. Smart Campus Sensor and Room Management API Report

### Part 1: Service Architecture & Setup

**Question: In your report, explain the default lifecycle of a JAX-RS Resource class. Is a new instance instantiated for every incoming request, or does the runtime treat it as a singleton? Elaborate on how this architectural decision impacts the way you manage and synchronize your in-memory data structures (maps/lists) to prevent data loss or race conditions.**

**Answer:**
JAX-RS creates a new resource class instance for each http request, so you may not use instance-level fields to store shared data. Rather than using instance-level fields to store shared in-memory data, the shared in-memory data must be stored in static fields at the class level. In this project, we will define a dedicated DataStore class that will hold the HashMap collections as static fields, which will allow the collections to exist and be shared among all requests. When using any collection type in a multi-threaded environment where multiple requests may be processed simultaneously, using ConcurrentHashMap instead of HashMap is the way to go to avoid race conditions due to simultaneous access to the same data.

**Question: Why is the provision of ”Hypermedia” (links and navigation within responses) considered a hallmark of advanced RESTful design (HATEOAS)? How does this approach benefit client developers compared to static documentation?**

**Answer:**

HATEOAS (Hypermedia as the Engine of Application State) is a concept in API design where the response includes not just data but also links to other resources and actions that can be performed on them. As such, advanced REST APIs are self-documenting, and thus clients will be able to traverse around the entire API simply by clicking through the links that are returned to them in the response. By doing so, developers will avoid hard-coding URLs into their applications that may change in the future. When a server changes an existing resource path, clients will still function correctly by following the appropriate links as provided in the API response rather than failing. Traditional documentation may become out-of-date quickly, as such, HATEOAS-compliant APIs provide their consumers with the most current information on available actions via the API's response.

### Part 2: Room Management

**Question: When returning a list of rooms, what are the implications of returning only IDs versus returning the full room objects? Consider network bandwidth and client side processing.**

**Answer:**

By returning only the IDs of the collections, the response payload will be kept small and there will be reduced usage of network bandwidth (especially when the collection is large). However, in this case the client will still have to make one GET request to get the list of rooms and N GET requests for each room (where N equals the number of rooms returned), which creates an N + 1 problem. If the full room object is returned, the total size of the response will be greater, but all of the data for the room can be returned in one round-trip time, providing a better experience to the end client. The best practice is to return a summary of information that is lightweight (with only the most relevant fields) so that the bandwidth used is reasonable while still providing sufficient information to the client in one request.

**Question: Is the DELETE operation idempotent in your implementation? Provide a detailed justification by describing what happens if a client mistakenly sends the exact same DELETE request for a room multiple times.**

**Answer:**

Yes, the DELETE operation, in this implementation, is idempotent. The initial call to DELETE /api/v1/rooms/{roomId} will delete the room from the in-memory data store, and return the status code 204 No Content. All subsequent calls to DELETE /api/v1/rooms/{roomId} will find that the room does not exist anymore and return the status code 404 Not Found. Even though the HTTP response code will be different for the first and all later calls, the room will remain absent from the in-memory data store regardless of how many times you perform the same DELETE operation with the same room id. The idempotent property only relates to the effect the operation has on the state of the resource, and not its response code, hence DELETE is an idempotent operation per the REST constraint.

### Part 3: Sensor Operations & Linking

**Question: We explicitly use the @Consumes (MediaType.APPLICATION_JSON) annotation on the POST method. Explain the technical consequences if a client attempts to send data in a different format, such as text/plain or application/xml. How does JAX-RS handle this mismatch?**

**Answer:**

The annotation @Consumes is used to indicate the Content-Type types that a method can accept. The JAX-RS runtime will check the Content-Type of any incoming request to see if it matches the @Consumes annotation on the method being invoked. If there is no match, i.e., if the Content-Type of the incoming request does not match one of those specified in the @Consumes annotation, the JAX-RS runtime will reject the request before it is passed on to the invoked method and will return a 415 Unsupported Media Type HTTP status code back to the client. No additional checking code is required by the developer only, the framework has this covered within the annotations. As a result, only valid JSON payloads will get passed to the business logic layer, which helps maximize the security of the application by guarding against malformed or unintended formats of input payloads.

**Question: You implemented this filtering using @QueryParam. Contrast this with an alternative design where the type is part of the URL path (e.g., /api/vl/sensors/type/CO2). Why is the query parameter approach generally considered superior for filtering and searching collections?**

**Answer:**

Query parameters are meant to allow for optional filtering, sorting and searching of a collection, while the path-based version (/sensors/type/CO2) suggests that type/CO2 is an independent, addressable resource in the hierarchy, when it is not and is simply a filter on the collection.
Query parameters provide flexibility in that multiple filters can be combined without having to redesign the URL structure, such as using filters like ?type=CO2&status=ACTIVE together.
If using a path-based method, each new combination of filters would require a new route.
The query parameter method allows for the base resource path to remain uncluttered (/sensors) while allowing for optional composable filtering to occur.

### Part 4: Deep Nesting with Sub-Resources

**Question: Discuss the architectural benefits of the Sub-Resource Locator pattern. How does delegating logic to separate classes help manage complexity in large APIs compared to defining every nested path (e.g., sensors/{id}/readings/{rid}) in one massive controller class?**

**Answer:**

The Sub Resource Locator pattern gives a resource class the ability to delegate a portion of the URL path to a different, independent resource class. In this example, SensorResource is responsible for processing /sensors and /sensors/{id}. It is responsible for processing /sensors/{id}/readings, but only by returning a new instance of the SensorReadingResource class without an HTTP verb annotation. This allows for focusing each class on a single responsibility; SensorResource will handle all sensor-related data and SensorReadingResource will handle all reading-history data. If you were to put all of your nested paths in one class, it would lead to a bloated and hard-to-maintain controller. The locator pattern can also represent real-world boundaries of domain-resource boundaries which allows each resource class to be easily readable, tested, and independently extensible.


### Part 5: Advanced Error Handling, Exception Mapping & Logging

**Question: Why is HTTP 422 often considered more semantically accurate than a standard 404 when the issue is a missing reference inside a valid JSON payload?**

**Answer:**

A server's 404 error means that the URL requested is not on the server. But, when an acceptable POST request is sent to an appropriate URL (e.g., /api/v1/sensors) with an invalid roomId in the request body, the issue does not lie with the URL; it lies with the request body parameters. HTTP 422 (Unprocessable Entity) applies to this situation as well. The server receives an appropriate request in an acceptable format with a valid URL but cannot be fulfilled due to semantic issues with that request's body. If the server were to return 404 in this case, it would lead the client to believe the endpoint does not exist, rather than the request being properly formatted, but the resource contained in the request's body not being able to be found.

**Question: From a cybersecurity standpoint, explain the risks associated with exposing internal Java stack traces to external API consumers. What specific information could an attacker gather from such a trace?**

**Answer:**

Revealing raw Java stack traces presents a major security risk because it exposes the internal implementation details of your applications which should never be made visible outside of your server. Attackers can determine exactly which classes and package structures exist in your application, what framework and library versions you are using that they might be able to look up using known CVE's, what lines of code are responsible for the errors being reported to you, and, in certain cases, what database schema information exists if the error was caused by a SQL operation. With all this information, an attacker could craft an exploit to attack vulnerabilities that they identify. To mitigate this risk within this project, we use the global ExceptionMapper<Throwable> to catch all unexpected errors, log the full error details on the server, and return a generic 500 Internal Server Error message back to the client.

**Question: Why is it advantageous to use JAX-RS filters for cross-cutting concerns like logging, rather than manually inserting Logger.info() statements inside every single resource method?**

**Answer:**

By inserting log statements into each resource method, you go against the DRY principle (Don't Repeat Yourself), thus creating maintenance headaches for the codebase. You would have to change every single method in each resource class if you ever wanted to change the log format. JAX-RS filters allow for logging as a cross-cutting concern by using one instance of the LoggingFilter class to log all requests and responses using only one filter without having to modify any resource methods. This keeps your resource classes focused on business logic without mixing in observability concerns. This also ensures that no request or response can ever be accidentally missed from logging, which can happen with manual logging if a developer forgets to add the appropriate log statement to a newly-created method.