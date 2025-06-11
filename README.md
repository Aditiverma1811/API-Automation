# API-Automation

This repository contains an API automation framework built in Java using Maven, RestAssured, and TestNG. It validates all the major functionalities (CRUD operations, error handling, and request chaining) of the FastAPI implementation. The framework generates detailed execution reports using Allure and integrates with GitHub Actions for CI/CD.
Table of Contents

- [Project Overview](#project-overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Configuration Management](#configuration-management)
- [Framework Implementation](#framework-implementation)
- [Test Execution](#test-execution)
- [Reporting with Allure](#reporting-with-allure)
- [Continuous Integration (CI/CD)](#continuous-integration-cicd)
- [Testing Strategy](#testing-strategy)
- [Challenges and Considerations](#challenges-and-considerations)
- [Submission Guidelines](#submission-guidelines)

  Project Overview

This project automates the testing of a FastAPI application. The goal is to cover all critical functionalities including:
- CRUD Operation Testing (Create, Read, Update, Delete)
- Validation of HTTP status codes, response payloads, and error handling
- Request chaining (using output from one API call as input for another) for workflow testing
- Both positive and negative test scenarios to ensure robustness

The framework is designed to be scalable, maintainable, and reusable.
 Prerequisites

- **Java 8 or above** installed on your machine.
- **Maven** installed for dependency management and running tests.
- An IDE such as IntelliJ IDEA or Eclipse.
- FastAPI service is set up and running on your local machine (refer to the FAST API project's README for service setup).
- [Allure Commandline](https://docs.qameta.io/allure/) installed (optional, for generating test reports).

---

## Project Structure

```
project-root/
│
├── src/
│   ├── main/java           # (Optional) For helper utilities or additional libraries
│   ├── test/java           # Contains all API test classes and utility classes
│   │   ├── tests/          # Test classes (e.g., BaseTest.java, UserApiTest.java)
│   │   └── utils/          # Utility classes (e.g., ConfigReader.java)
│   └── test/resources/     # Contains configuration files (e.g., config.properties)
│
├── .github/
│   └── workflows/          # Contains GitHub Actions CI/CD configuration file (ci.yml)
│
└── pom.xml                 # Maven project file containing all dependencies and build configs
```

---

## Configuration Management

The project uses a `config.properties` file located in `src/test/resources` to manage environment-specific parameters such as the base URL.

**config.properties**
```properties
base.url = http://localhost:8000/api
env = dev
```

A simple utility class (`ConfigReader.java`) reads these properties at runtime. This ensures that the tests can be run against different environments (dev, QA, prod) without code changes.
Framework Implementation

### Dependencies
The project uses the following main dependencies:
- **RestAssured** (for API testing)
- **TestNG** (for test execution)
- **Allure** (for generating detailed execution reports)
- **Jackson** (optional, for JSON parsing if required)

All dependencies are managed via Maven in the `pom.xml` file.

### Base Test Setup
A base test class initializes the RestAssured URI from the configuration.

**BaseTest.java**
```java
package tests;

import io.restassured.RestAssured;
import org.testng.annotations.BeforeClass;
import utils.ConfigReader;

public class BaseTest {
    @BeforeClass
    public void setup() {
        RestAssured.baseURI = ConfigReader.getProperty("base.url");
    }
}
```

### Example Test Cases
The tests cover each CRUD operation, including negative tests to ensure error handling is correctly implemented.

**UserApiTest.java**
```java
package tests;

import io.restassured.http.ContentType;
import io.restassured.response.Response;
import org.testng.Assert;
import org.testng.annotations.Test;

import static io.restassured.RestAssured.given;

public class UserApiTest extends BaseTest {

    private static String resourceId;

    @Test(priority = 1)
    public void testCreateResource() {
        String payload = "{ \"name\": \"John Doe\", \"email\": \"john@example.com\" }";
        Response response = given()
            .contentType(ContentType.JSON)
            .body(payload)
            .when()
            .post("/users")
            .then().extract().response();

        Assert.assertEquals(response.getStatusCode(), 201, "Status code mismatches in resource creation");
        resourceId = response.jsonPath().getString("id");
    }

    @Test(priority = 2, dependsOnMethods = "testCreateResource")
    public void testReadResource() {
        Response response = given()
            .when()
            .get("/users/" + resourceId)
            .then().extract().response();

        Assert.assertEquals(response.getStatusCode(), 200, "Status code mismatches in resource retrieval");
        Assert.assertEquals(response.jsonPath().getString("name"), "John Doe", "User name mismatch");
    }

    @Test(priority = 3, dependsOnMethods = "testReadResource")
    public void testDeleteResource() {
        Response response = given()
            .when()
            .delete("/users/" + resourceId)
            .then().extract().response();

        Assert.assertEquals(response.getStatusCode(), 204, "Resource deletion failed");
    }

    @Test(priority = 4)
    public void testNegativeScenario() {
        Response response = given()
            .when()
            .get("/users/invalidId")
            .then().extract().response();

        Assert.assertEquals(response.getStatusCode(), 404, "Expected a 404 for non-existent resource");
    }
}
```
*Note: Adjust the endpoints (`/users`) and the payload to match the FastAPI implementation.*

---

## Test Execution

### Running Tests Locally
1. Open a terminal in your project directory.
2. Run the tests using Maven:
   ```bash
   mvn clean test
   ```

### Generating Allure Reports
After running tests, generate the report using:
```bash
allure serve allure-results
```
This command will open a browser window displaying a detailed test report with Pass/Fail status for each test case.

---

## Continuous Integration (CI/CD)

GitHub Actions is used to automate test execution on every push. Below is a sample configuration file located at `.github/workflows/ci.yml`:

**ci.yml**
```yaml
name: API Automation CI

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout Code
      uses: actions/checkout@v2

    - name: Set up JDK 11
      uses: actions/setup-java@v3
      with:
        java-version: '11'

    - name: Cache Maven packages
      uses: actions/cache@v2
      with:
        path: ~/.m2
        key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
        restore-keys: ${{ runner.os }}-maven

    - name: Build with Maven
      run: mvn clean test

    - name: Generate Allure Report
      run: |
        allure generate allure-results --clean -o allure-report
        allure open allure-report
```
This configuration checks out the code, sets up Java, caches Maven packages, runs tests, and generates an Allure report automatically upon pushes or pull requests.

---

## Testing Strategy

### Test Flows
- **CRUD Operations:** Each test method covers one aspect:
  - **Create:** Validate resource creation with status code 201.
  - **Read:** Retrieve the created resource and verify payload details.
  - **Update:** (Add tests for PUT/PATCH as needed for modifications).
  - **Delete:** Ensure proper resource deletion and response code 204.
- **Negative Testing:** Invalid inputs (e.g., malformed payloads, invalid IDs) produce the expected error codes (e.g., 404).

### Ensuring Reliability
- **Request Chaining:** The output (ID) from the creation call is dynamically used in read, update, and delete tests.
- **Assertions:** Each API response is verified with specific assertions to confirm correct statuses and payloads.
- **Configuration:** Environment-specific details are managed using a configuration file, ensuring tests run in various environments without code changes.

### Challenges Encountered
- **Dynamic Data Handling:** Request chaining required careful storage and retrieval of dynamic data (e.g., resource IDs).
- **Error Handling:** Incorporating negative tests helped in identifying how the application handles unexpected inputs and ensured robustness.

---
