# CLAUDE.md - AI Assistant Guide for OMOPonFHIR

## Project Overview

**OMOPonFHIR** is a healthcare data integration platform that translates OMOP Common Data Model (CDM) v5.4 data into FHIR R4 compliant REST APIs. It enables healthcare organizations to expose their OMOP-formatted clinical data through standardized FHIR interfaces.

- **Organization:** Georgia Tech Research Institute (CHAI)
- **License:** Apache License 2.0
- **Current Version:** 1.7.x
- **Java Version:** JDK 21
- **Website:** http://omoponfhir.org

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    FHIR REST API Layer                      │
│  (RestfulServlet + 20 Resource Providers + SMART on FHIR)   │
│                   omoponfhir-r4-server                      │
├─────────────────────────────────────────────────────────────┤
│              Mapping Layer (22 Omop* Classes)               │
│  (Bidirectional OMOP ↔ FHIR conversion, Parameter mapping)  │
│                omoponfhir-omopv5-r4-mapping                 │
├─────────────────────────────────────────────────────────────┤
│         Service Layer (IService + 22 Implementations)       │
│  (Generic CRUD, search operations with ParameterWrapper)    │
│                   omoponfhir-omopv5-sql                     │
├─────────────────────────────────────────────────────────────┤
│      Database Layer (JDBC + SqlRender + HikariCP)           │
│  (PostgreSQL, SQL Server, Google BigQuery support)          │
├─────────────────────────────────────────────────────────────┤
│                OMOP v5.4 Database (26 Entities)             │
└─────────────────────────────────────────────────────────────┘
```

## Repository Structure

This is a multi-module Maven project with Git submodules:

```
omoponfhir-main-v54-r4/
├── pom.xml                          # Parent POM (aggregator)
├── Dockerfile                       # Docker multi-stage build
├── server.xml                       # Tomcat config (increased header size)
├── updateSubmodules.sh              # Script to update submodules
├── README.md                        # User documentation
│
├── omoponfhir-omopv5-sql/           # SUBMODULE: Database layer
│   ├── pom.xml
│   └── src/main/java/edu/gatech/chai/omopv5/
│       ├── dba/
│       │   ├── service/             # 22 IService implementations
│       │   ├── config/              # Database configuration
│       │   └── util/                # SQL utilities
│       └── model/entity/            # 26 JPA entity classes
│
├── omoponfhir-omopv5-r4-mapping/    # SUBMODULE: OMOP ↔ FHIR mapping
│   ├── pom.xml
│   └── src/main/java/edu/gatech/chai/omoponfhir/
│       ├── omopv5/r4/
│       │   ├── mapping/             # 22 OmopXxx mapping classes
│       │   ├── utilities/           # CodeableConceptUtil, DateUtil, etc.
│       │   ├── model/               # Extended FHIR models
│       │   └── provider/            # Bundle providers
│       └── local/
│           ├── dao/                 # Local code mapping storage
│           ├── model/               # Mapping entry models
│           └── task/                # Scheduled mapping tasks
│
└── omoponfhir-r4-server/            # SUBMODULE: FHIR server
    ├── pom.xml
    └── src/main/
        ├── java/edu/gatech/chai/omoponfhir/
        │   ├── r4/provider/         # 20 FHIR Resource Providers
        │   ├── config/              # Spring configurations
        │   ├── security/            # OAuth/OIDC interceptors
        │   ├── servlet/             # RestfulServlet
        │   └── smart/               # SMART on FHIR (dao, servlet, jwt)
        ├── resources/
        │   └── application.properties
        └── webapp/
            └── WEB-INF/
                ├── web.xml          # Servlet configuration
                └── database-config.xml
```

## Key Technologies

| Component | Version | Purpose |
|-----------|---------|---------|
| Java | 21 | Runtime |
| Spring Framework | 6.1.14 | Dependency injection, web |
| HAPI FHIR | 7.6.1 | FHIR R4 server framework |
| Maven | 3.9.6+ | Build tool |
| Tomcat | JRE21 | Application server |
| PostgreSQL Driver | 42.7.2 | Primary database |
| SQL Server Driver | 11.2.3 | Alternative database |
| Google BigQuery | 2.23.0 | Cloud data warehouse |
| SqlRender | 1.16.1 | Cross-database SQL translation |
| HikariCP | 5.0.1 | Connection pooling |
| OAuth2 (Apache Oltu) | - | Security |
| JWT (jjwt) | 0.11.1 | Token management |

## Build Commands

```bash
# Clone with submodules
git clone --recurse https://github.com/omoponfhir/omoponfhir-main-v54-r4.git

# Initialize submodules (if cloned without --recurse)
git submodule update --init --recursive

# Update submodules to latest commits
./updateSubmodules.sh

# Build all modules
mvn clean install

# Build without tests
mvn clean install -DskipTests

# Build specific module
mvn clean install -pl omoponfhir-omopv5-sql

# Generate WAR file only
mvn clean package -pl omoponfhir-r4-server
```

**Build Output:**
- `omoponfhir-omopv5-sql/target/omoponfhir-omopv5-sql-1.7.1.jar`
- `omoponfhir-omopv5-r4-mapping/target/omoponfhir-omopv5-r4-mapping-1.7.3.jar`
- `omoponfhir-r4-server/target/omoponfhir-r4-server.war`

## Environment Variables

Required environment variables for runtime configuration:

```bash
# Database Connection
export JDBC_URL="jdbc:postgresql://hostname:5432/dbname"
export JDBC_USERNAME="your_username"
export JDBC_PASSWORD="your_password"
export JDBC_DATASOURCENAME="org.postgresql.ds.PGSimpleDataSource"
export JDBC_POOLSIZE=5

# Schema Configuration
export JDBC_DATA_SCHEMA="omopv54"      # Schema for clinical data
export JDBC_VOCABS_SCHEMA="vocab"       # Schema for vocabulary tables

# Database Target (for SqlRender)
export TARGETDATABASE="postgresql"      # postgresql, sqlserver, bigquery

# Server Configuration
export SERVERBASE_URL="http://localhost:8080/fhir/"
export OMOPONFHIR_NAME="OMOP v5.4 on FHIR R4"
export FHIR_READONLY="False"

# SMART on FHIR (Optional)
export SMART_INTROSPECTURL="http://localhost:8080/smart/introspect"
export SMART_AUTHSERVERURL="http://localhost:8080/smart/authorize"
export SMART_TOKENSERVERURL="http://localhost:8080/smart/token"
export AUTH_BEARER="<any_value>"
export AUTH_BASIC="username:password"

# Local Code Mapping (Optional)
export LOCAL_CODEMAPPING_FILE_PATH="/path/to/mapping/file"
export MEDICATION_TYPE="code"

# BigQuery (if using BigQuery)
export BIGQUERYDATASET="dataset_name"
export BIGQUERYPROJECT="project_name"
```

## Deployment

### Docker Deployment (Recommended)

```bash
# Build Docker image
docker build -t omoponfhir .

# Run with environment file
docker run --env-file env.list --name omoponfhir -p 8080:8080 -d omoponfhir:latest

# Or with inline environment variables
docker run -e JDBC_URL=... -e JDBC_USERNAME=... -p 8080:8080 -d omoponfhir:latest
```

### Tomcat Deployment

```bash
# Copy WAR to Tomcat
cp omoponfhir-r4-server/target/omoponfhir-r4-server.war $CATALINA_HOME/webapps/

# Set environment in setenv.sh
vi $CATALINA_HOME/bin/setenv.sh

# Start Tomcat
$CATALINA_HOME/bin/startup.sh
```

### Verify Deployment

```bash
# Check metadata endpoint
curl http://localhost:8080/fhir/metadata

# Check patient count
curl http://localhost:8080/fhir/Patient?_summary=count
```

## Key Design Patterns

### 1. Generic Service Pattern
All database services implement `IService<T extends BaseEntity>`:
```java
public interface IService<v extends BaseEntity> {
    v findById(Long id);
    List<v> searchWithParams(int fromIndex, int toIndex,
                            List<ParameterWrapper> paramList, String sort);
    v create(v entity);
    v update(v entity);
    Long getSize(List<ParameterWrapper> paramList);
}
```

### 2. Entity Mapping Pattern
All OMOP↔FHIR mappers implement `IResourceMapping<V, T>`:
```java
public interface IResourceMapping<v extends Resource, t extends BaseEntity> {
    v toFHIR(IdType id);           // OMOP → FHIR
    Long toDbase(v fhirResource);  // FHIR → OMOP
}
```

### 3. Resource Provider Pattern
FHIR resource providers use HAPI FHIR annotations:
```java
@Search
public IBundleProvider search(@RequiredParam(name="identifier") TokenParam identifier);

@Read
public Patient read(@IdParam IdType theId);

@Create
public MethodOutcome create(@ResourceParam Patient patient);
```

### 4. ParameterWrapper Pattern
Flexible query parameter handling without SQL injection:
```java
ParameterWrapper param = new ParameterWrapper(
    "String",                    // Parameter type
    Arrays.asList("givenName1"), // Column names
    Arrays.asList("LIKE"),       // Operators
    Arrays.asList("%John%"),     // Values
    "or"                         // Relationship
);
```

## Supported FHIR Resources

The server supports 19 FHIR R4 resource types:

| Resource | OMOP Mapping Class | Provider Class |
|----------|-------------------|----------------|
| Patient | OmopPatient | PatientResourceProvider |
| Practitioner | OmopPractitioner | PractitionerResourceProvider |
| Organization | OmopOrganization | OrganizationResourceProvider |
| Encounter | OmopEncounter | EncounterResourceProvider |
| Condition | OmopCondition | ConditionResourceProvider |
| Procedure | OmopProcedure | ProcedureResourceProvider |
| Observation | OmopObservation | ObservationResourceProvider |
| Medication | OmopMedication | MedicationResourceProvider |
| MedicationRequest | OmopMedicationRequest | MedicationRequestResourceProvider |
| MedicationStatement | OmopMedicationStatement | MedicationStatementResourceProvider |
| Immunization | OmopImmunization | ImmunizationResourceProvider |
| AllergyIntolerance | OmopAllergyIntolerance | AllergyIntoleranceResourceProvider |
| Device | OmopDevice | DeviceResourceProvider |
| DeviceUseStatement | OmopDeviceUseStatement | DeviceUseStatementResourceProvider |
| DocumentReference | OmopDocumentReference | DocumentReferenceResourceProvider |
| Specimen | OmopSpecimen | SpecimenResourceProvider |
| CodeSystem | OmopCodeSystem | CodeSystemResourceProvider |
| ValueSet | OmopValueSet | ValueSetResourceProvider |
| ConceptMap | OmopConceptMap | ConceptMapResourceProvider |

## OMOP Entity Classes (26 total)

**Core Entities:**
- `Person`, `FPerson` (enhanced view)
- `Provider`, `CareSite`, `Location`

**Clinical Events:**
- `ConditionOccurrence`, `DrugExposure`, `ProcedureOccurrence`
- `Measurement`, `Observation`, `VisitOccurrence`
- `DeviceExposure`, `Note`, `Specimen`, `Death`

**Vocabulary/Reference:**
- `Concept`, `ConceptRelationship`, `ConceptAncestor`
- `Vocabulary`, `SourceToConceptMap`

**Special Views:**
- `FObservationView`, `FImmunizationView`

## Code Conventions

### Package Structure
```
edu.gatech.chai.omopv5                    # SQL/database layer
edu.gatech.chai.omoponfhir.omopv5.r4      # Mapping layer
edu.gatech.chai.omoponfhir                # Server layer
```

### Naming Conventions
- **Entity Classes:** Match OMOP table names (e.g., `ConditionOccurrence`)
- **Service Classes:** `<Entity>ServiceImp` (e.g., `ConditionOccurrenceServiceImp`)
- **Mapping Classes:** `Omop<FhirResource>` (e.g., `OmopCondition`)
- **Provider Classes:** `<Resource>ResourceProvider` (e.g., `ConditionResourceProvider`)

### Annotation Usage
- Custom JPA-like annotations in `edu.gatech.chai.omopv5.model.entity.custom`:
  - `@Table`, `@Column`, `@Id`, `@GeneratedValue`, `@JoinColumn`
- Spring annotations: `@Configuration`, `@ComponentScan`, `@Autowired`
- HAPI FHIR annotations: `@Search`, `@Read`, `@Create`, `@Update`, `@Delete`

## Testing

Tests are located in:
- `omoponfhir-omopv5-sql/src/test/java/`
- `omoponfhir-omopv5-r4-mapping/src/test/java/`

```bash
# Run tests
mvn test

# Run tests for specific module
mvn test -pl omoponfhir-omopv5-sql
```

Note: Tests are skipped by default in standard builds (`maven-surefire-plugin` config).

## Git Workflow

### Submodule Management
```bash
# Update all submodules to latest
./updateSubmodules.sh

# Update specific submodule
git submodule update --remote -- omoponfhir-r4-server

# Check submodule status
git submodule status
```

### Branch Information
- **omoponfhir-omopv5-sql:** branch `5.4`
- **omoponfhir-omopv5-r4-mapping:** branch `sqlRender`
- **omoponfhir-r4-server:** branch `sqlRender`

## Common Tasks for AI Assistants

### Adding a New FHIR Resource
1. Create entity class in `omoponfhir-omopv5-sql/src/main/java/edu/gatech/chai/omopv5/model/entity/`
2. Create service in `omoponfhir-omopv5-sql/src/main/java/edu/gatech/chai/omopv5/dba/service/`
3. Create mapping class in `omoponfhir-omopv5-r4-mapping/src/main/java/edu/gatech/chai/omoponfhir/omopv5/r4/mapping/`
4. Create provider in `omoponfhir-r4-server/src/main/java/edu/gatech/chai/omoponfhir/r4/provider/`
5. Register provider in `FhirServerConfig`

### Modifying OMOP-FHIR Mapping
- Mapping logic is in `omoponfhir-omopv5-r4-mapping/src/main/java/edu/gatech/chai/omoponfhir/omopv5/r4/mapping/`
- Key methods: `toFHIR()` (OMOP→FHIR), `toDbase()` (FHIR→OMOP)

### Adding Database Support
- SqlRender handles cross-database SQL translation
- Add dialect-specific code in `omoponfhir-omopv5-sql` if needed
- Update `TARGETDATABASE` environment variable

### Debugging FHIR Requests
- Check `omoponfhir-r4-server/src/main/resources/logback.xml` for logging config
- Enable debug logging: `<logger name="edu.gatech.chai" level="DEBUG"/>`

## Important Files to Know

| File | Purpose |
|------|---------|
| `pom.xml` (root) | Parent Maven configuration |
| `Dockerfile` | Multi-stage Docker build |
| `server.xml` | Tomcat config (65536 byte header limit) |
| `omoponfhir-r4-server/.../web.xml` | Servlet mappings |
| `omoponfhir-r4-server/.../FhirServerConfig.java` | Spring configuration |
| `omoponfhir-r4-server/.../RestfulServlet.java` | Main FHIR servlet |
| `omoponfhir-omopv5-r4-mapping/.../StaticValues.java` | Supported resource types |

## Troubleshooting

### Database Connection Issues
- Verify `JDBC_URL`, `JDBC_USERNAME`, `JDBC_PASSWORD` environment variables
- Check `JDBC_DATASOURCENAME` matches your database driver
- Ensure `JDBC_DATA_SCHEMA` and `JDBC_VOCABS_SCHEMA` exist

### Build Failures
- Ensure JDK 21 is installed and `JAVA_HOME` is set
- Run `mvn clean` before rebuilding
- Check submodules are initialized: `git submodule update --init --recursive`

### Runtime Errors
- Check Tomcat logs: `$CATALINA_HOME/logs/catalina.out`
- Verify metadata endpoint: `curl http://localhost:8080/fhir/metadata`
- Enable debug logging in `logback.xml`
