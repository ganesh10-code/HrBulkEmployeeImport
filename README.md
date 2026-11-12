# Member 3 – Duplicate Management & Manager Resolution

## HR Bulk Employee Import – MuleSoft Hackathon

**Project:** Cognizant 2027 Nurture Partner Network (NPN) MuleSoft Hackathon  
**Module Owner:** Member 3 – Duplicate Management & Manager Resolution  
**Repository:** [HR Bulk Employee Import](https://github.com/ganesh10-code/HrBulkEmployeeImport)  
**Technology:** MuleSoft 4, Anypoint Studio, DataWeave 2.0, MUnit

---

# 1. Module Overview

This module implements the business logic responsible for detecting duplicate employee records, validating employee email uniqueness, resolving manager references, detecting circular reporting relationships, and controlling whether an employee import should proceed based on the configured rejection threshold.

It is designed as an independent MuleSoft module. It accepts canonical employee JSON records and existing employee records, processes them through reusable Mule flows and subflows, and returns structured validation results.

The module is divided into five functional components:

1. Employee ID Duplicate Detection
2. Email Duplicate Detection
3. Manager Resolution
4. Manager Hierarchy Validation
5. Rejection Threshold Validation

Each component has its own Mule flow and MUnit test suite.

## 1.1 Objectives

- Prevent duplicate employee IDs within an incoming file.
- Detect employee IDs that already exist in the database.
- Support configurable INSERT, UPDATE and REJECT strategies.
- Prevent duplicate official email addresses.
- Resolve manager references independently of employee record ordering.
- Detect unresolved managers and invalid reporting relationships.
- Detect self-manager assignments and circular reporting chains.
- Calculate the overall rejection percentage.
- Prevent database persistence when the rejection threshold is exceeded.
- Return standardized validation results for integration with other modules.

---

# 2. Module Architecture

```text
                  Incoming Employee Records
                            |
                            |
                 Existing Employee Records
                            |
                            v
              +---------------------------+
              | Employee ID Duplicate     |
              | Detection                 |
              +---------------------------+
                            |
                            v
              +---------------------------+
              | Email Duplicate           |
              | Detection                 |
              +---------------------------+
                            |
                            v
              +---------------------------+
              | Manager Resolution        |
              | (Two-Pass Validation)     |
              +---------------------------+
                            |
                            v
              +---------------------------+
              | Manager Hierarchy         |
              | Validation                |
              +---------------------------+
                            |
                            v
              +---------------------------+
              | Rejection Aggregation     |
              | (Integration Stage)       |
              +---------------------------+
                            |
                            v
              +---------------------------+
              | Rejection Threshold       |
              | Validation                |
              +---------------------------+
                            |
                   +--------+--------+
                   |                 |
                CONTINUE           ABORT
                   |                 |
                   v                 v
             Database Batch     Skip Database
              Persistence         Persistence
```

**Important:** The five components have been implemented and independently tested. The complete orchestration and database persistence integration must be performed with the other team members.

---

# 3. Project File Structure

The following files belong to Member 3's implementation.

```text
src/
├── main/
│   ├── mule/
│   │   ├── duplicateDetectionFlow.xml
│   │   ├── emailDuplicateFlow.xml
│   │   ├── managerResolutionFlow.xml
│   │   ├── managerHierarchyFlow.xml
│   │   └── rejectionThresholdFlow.xml
│   │
│   └── resources/
│       └── config.properties
│
└── test/
    └── munit/
        ├── duplicateDetectionTest-suite.xml
        ├── emailDuplicateTest-suite.xml
        ├── managerResolutionTest-suite.xml
        ├── managerHierarchyTest-suite.xml
        └── rejectionThresholdTest-suite.xml
```

The actual project may contain additional configuration and resource files.

---

# 4. Employee ID Duplicate Detection

**File:** `duplicateDetectionFlow.xml`  
**Main Subflow:** `duplicateDetectionLogic`  
**Standalone Endpoint:** `POST /duplicates`

## 4.1 Purpose

Detect duplicate employee identifiers within the incoming file and against existing employee records.

The module supports three configurable strategies:

- INSERT
- UPDATE
- REJECT

It also distinguishes between identical duplicate records and conflicting records with the same employee ID.

## 4.2 Input Contract

```json
{
  "strategy": "INSERT",
  "employees": [
    {
      "employeeId": "EMP1001",
      "firstName": "John",
      "departmentCode": "IT"
    },
    {
      "employeeId": "EMP1002",
      "firstName": "David",
      "departmentCode": "HR"
    }
  ],
  "existingEmployees": [
    {
      "employeeId": "EMP2001",
      "firstName": "Robert"
    }
  ]
}
```

### Input fields

| Field | Description |
|---|---|
| strategy | Configured duplicate strategy: INSERT, UPDATE or REJECT |
| employees | Incoming employee records |
| existingEmployees | Existing employee records retrieved from the database |

For independent testing, `existingEmployees` can be provided as mock input. During integration, Member 4 should supply the actual database lookup results.

## 4.3 Business Rules

| Scenario | Decision |
|---|---|
| Employee ID does not exist in the database | INSERT |
| Employee ID exists and strategy is INSERT | REJECT |
| Employee ID exists and strategy is UPDATE | UPDATE |
| Employee ID exists and strategy is REJECT | REJECT |
| Same employee ID with identical data appears multiple times in the file | Keep first occurrence; reject subsequent occurrences |
| Same employee ID with conflicting data appears in the file | Reject all conflicting occurrences |

### Rejection codes

| Reject Code | Description |
|---|---|
| `DUPLICATE_IDENTICAL_RECORD` | Subsequent occurrence of an identical employee record |
| `CONFLICTING_EMPLOYEE_RECORDS` | Same employee ID appears with conflicting employee information |
| `EMPLOYEE_ID_ALREADY_EXISTS` | Employee ID already exists in the database and cannot be accepted under the selected strategy |

## 4.4 Implementation Logic

The DataWeave implementation uses:

- `groupBy employeeId` to identify repeated employee IDs.
- Database employee IDs to identify existing records.
- `distinctBy` to distinguish identical records from conflicting records.
- Previous occurrence counting to identify subsequent identical records.
- Strategy-based conditions to determine INSERT, UPDATE or REJECT.

Each incoming employee receives an individual decision.

## 4.5 Output Contract

The employee ID duplicate flow returns a record-level decision structure.

Conceptual example:

```json
{
  "totalRecords": 2,
  "records": [
    {
      "recordIndex": 0,
      "employeeId": "EMP1001",
      "decision": "INSERT",
      "existsInDatabase": false
    },
    {
      "recordIndex": 1,
      "employeeId": "EMP2001",
      "decision": "UPDATE",
      "existsInDatabase": true
    }
  ]
}
```

The exact output fields should be taken from the current `duplicateDetectionLogic` implementation when building the integration adapter.

**Integration note:** Records marked REJECT must not be passed to the persistence layer. Records marked INSERT or UPDATE remain eligible for subsequent validation and persistence.

---

# 5. Email Duplicate Detection

**File:** `emailDuplicateFlow.xml`  
**Main Subflow:** `emailDuplicateLogic`  
**Standalone Endpoint:** `POST /email-duplicates`

## 5.1 Purpose

Ensure that every accepted employee has a unique official email address.

The module checks email uniqueness within the incoming file and against existing database employees.

## 5.2 Input Contract

```json
{
  "employees": [
    {
      "employeeId": "EMP1001",
      "officialEmail": "ganesh@company.com"
    },
    {
      "employeeId": "EMP1002",
      "officialEmail": "karthik@company.com"
    }
  ],
  "existingEmployees": [
    {
      "employeeId": "EMP2001",
      "officialEmail": "existing@company.com"
    }
  ]
}
```

## 5.3 Business Rules

1. An email appearing more than once in the incoming file causes all records sharing that email to be rejected.
2. An email already assigned to another database employee is rejected.
3. An employee retaining their own existing email is allowed.
4. Email comparisons are case-insensitive.
5. Leading and trailing whitespace is removed before comparison.
6. Missing email validation is handled by the employee field-validation module. It is not treated as a duplicate email by this module.

### Rejection codes

| Reject Code | Description |
|---|---|
| `DUPLICATE_EMAIL_IN_FILE` | Email is shared by multiple incoming employee records |
| `EMAIL_ALREADY_ASSIGNED` | Email belongs to another existing database employee |

## 5.4 Implementation Logic

Email normalization:

```dataweave
lower(trim(email))
```

The implementation compares normalized email values rather than the raw input values.

For database comparison, the employee ID is also checked so that an employee can retain their own email during an update.

Email addresses are not included in the rejection output. This avoids unnecessarily exposing sensitive contact information.

## 5.5 Output Contract

```json
{
  "totalRecords": 2,
  "validRecords": [
    {
      "employeeId": "EMP1001",
      "officialEmail": "ganesh@company.com"
    }
  ],
  "rejectedRecords": [
    {
      "employeeId": "EMP1002",
      "rejectCode": "DUPLICATE_EMAIL_IN_FILE",
      "rejectReason": "Official email is already used by another incoming employee"
    }
  ],
  "summary": {
    "totalValid": 1,
    "totalRejected": 1
  }
}
```

This is a representative example. The actual rejection reason and output fields should follow the current DataWeave implementation.

---

# 6. Manager Resolution

**File:** `managerResolutionFlow.xml`  
**Main Subflow:** `managerResolutionLogic`  
**Standalone Endpoint:** `POST /managers`

## 6.1 Purpose

Resolve employee manager references using both incoming employee records and existing database employees.

The main challenge is supporting forward references, where an employee's manager appears later in the same input file.

## 6.2 Forward Reference Problem

Consider the following records:

```json
[
  {
    "employeeId": "EMP1001",
    "managerId": "EMP1003"
  },
  {
    "employeeId": "EMP1002",
    "managerId": "EMP1003"
  },
  {
    "employeeId": "EMP1003",
    "managerId": null
  }
]
```

EMP1001 and EMP1002 refer to EMP1003, which appears later in the file.

A sequential lookup could incorrectly reject these employees if it only considers previously processed records.

## 6.3 Two-Pass Resolution

The implementation uses a two-pass approach.

**Pass 1: Collect employee identifiers**

- Extract employee IDs from all incoming employee records.
- Extract employee IDs from existing database employees.
- Combine both collections into a list of known employee identifiers.

Conceptually:

```dataweave
allKnownEmployeeIds =
    incomingEmployeeIds ++ existingEmployeeIds
```

**Pass 2: Validate manager references**

For every incoming employee:

- If managerId is null, accept it in this module.
- If managerId exists in the combined employee ID list, accept the manager reference.
- Otherwise, reject the employee.

This approach supports forward manager references without depending on file ordering.

## 6.4 Input Contract

```json
{
  "employees": [
    {
      "employeeId": "EMP1001",
      "managerId": "EMP1003"
    },
    {
      "employeeId": "EMP1003",
      "managerId": null
    }
  ],
  "existingEmployees": [
    {
      "employeeId": "EMP9001"
    }
  ]
}
```

## 6.5 Rejection Codes

| Reject Code | Description |
|---|---|
| `MANAGER_NOT_FOUND` | Referenced manager does not exist in the incoming file or database |

## 6.6 Output Contract

```json
{
  "totalRecords": 2,
  "validRecords": [
    {
      "employeeId": "EMP1001",
      "managerId": "EMP1003"
    },
    {
      "employeeId": "EMP1003",
      "managerId": null
    }
  ],
  "rejectedRecords": [],
  "summary": {
    "totalValid": 2,
    "totalRejected": 0
  }
}
```

**Important:** This module validates manager existence. It does not independently detect circular reporting relationships. That responsibility belongs to `managerHierarchyFlow.xml`.

---

# 7. Manager Hierarchy Validation

**File:** `managerHierarchyFlow.xml`  
**Main Subflow:** `managerHierarchyLogic`  
**Standalone Endpoint:** `POST /manager-hierarchy`

## 7.1 Purpose

Validate the organizational reporting hierarchy and prevent invalid manager relationships.

The module detects:

- Self-manager assignments.
- Direct circular reporting.
- Indirect circular reporting.
- Circular relationships involving existing database employees.

## 7.2 Example: Self-Manager

```json
{
  "employeeId": "EMP1001",
  "managerId": "EMP1001"
}
```

Rejection:

```json
{
  "employeeId": "EMP1001",
  "rejectCode": "SELF_MANAGER",
  "rejectReason": "Employee cannot be their own manager"
}
```

## 7.3 Example: Direct Circular Relationship

```json
[
  {
    "employeeId": "EMP1001",
    "managerId": "EMP1002"
  },
  {
    "employeeId": "EMP1002",
    "managerId": "EMP1001"
  }
]
```

Neither employee can be accepted as part of this circular relationship.

## 7.4 Example: Indirect Circular Relationship

```json
[
  {
    "employeeId": "EMP1001",
    "managerId": "EMP1002"
  },
  {
    "employeeId": "EMP1002",
    "managerId": "EMP1003"
  },
  {
    "employeeId": "EMP1003",
    "managerId": "EMP1001"
  }
]
```

The reporting chain forms a cycle:

```text
EMP1001
   |
   v
EMP1002
   |
   v
EMP1003
   |
   +------> EMP1001
```

## 7.5 Implementation Logic

The implementation builds a manager mapping:

```text
employeeId -> managerId
```

It combines incoming employees with existing database employees:

```dataweave
allEmployees = existingEmployees ++ employees
```

The incoming employee record takes precedence when building the manager mapping.

A recursive function, conceptually:

```text
isInCycle(startId, currentId, visited, relationships)
```

follows the manager chain to determine whether the chain returns to its starting employee.

The implementation uses a visited collection to detect repeated nodes.

### Rejection codes

| Reject Code | Description |
|---|---|
| `SELF_MANAGER` | Employee is assigned as their own manager |
| `MANAGER_NOT_FOUND` | Referenced manager does not exist |
| `CIRCULAR_MANAGER_REFERENCE` | Reporting chain creates a circular relationship |

### Current cycle-handling policy

The current implementation rejects incoming employees who are themselves part of a detected cycle.

It does not automatically reject every downstream employee who reports to an employee involved in a cycle.

This is an explicit implementation policy and should be reviewed with the team during integration.

---

# 8. Rejection Threshold Validation

**File:** `rejectionThresholdFlow.xml`  
**Main Subflow:** `rejectionThresholdLogic`  
**Standalone Endpoint:** `POST /rejection-threshold`

## 8.1 Purpose

Calculate the percentage of rejected employee records and determine whether the import can proceed to database persistence.

The challenge requires the import to be aborted before persistence if the rejection percentage exceeds the configured threshold.

## 8.2 Configuration

File: `config.properties`

```properties
employee.import.rejection.threshold=20
```

The value is configurable and should not be hardcoded in the DataWeave logic.

## 8.3 Calculation

```text
Total Records = Valid Records + Rejected Records

Rejection Percentage =
    (Rejected Records / Total Records) * 100
```

The implementation rounds the calculated rejection percentage to two decimal places.

For zero total records, the rejection percentage is treated as zero.

## 8.4 Decision Rule

```text
If rejectionPercentage > threshold:
    processingDecision = ABORT
    persistenceAllowed = false

Otherwise:
    processingDecision = CONTINUE
    persistenceAllowed = true
```

The comparison is strictly greater than (`>`), not greater than or equal to (`>=`).

## 8.5 Examples

Assuming a configured threshold of 20%:

| Total | Valid | Rejected | Rejection % | Decision |
|---:|---:|---:|---:|---|
| 10 | 9 | 1 | 10% | CONTINUE |
| 10 | 8 | 2 | 20% | CONTINUE |
| 10 | 7 | 3 | 30% | ABORT |
| 3 | 3 | 0 | 0% | CONTINUE |
| 3 | 0 | 3 | 100% | ABORT |

## 8.6 Input Contract

```json
{
  "validRecords": [
    {
      "employeeId": "EMP1001"
    },
    {
      "employeeId": "EMP1002"
    }
  ],
  "rejectedRecords": [
    {
      "employeeId": "EMP1003",
      "rejectCode": "INVALID_EMAIL"
    }
  ]
}
```

## 8.7 Output Contract

```json
{
  "processingDecision": "CONTINUE",
  "persistenceAllowed": true,
  "rejectionPercentage": 33.33,
  "threshold": 20,
  "totalRecords": 3,
  "totalValid": 2,
  "totalRejected": 1,
  "validRecords": [],
  "rejectedRecords": []
}
```

The above values are illustrative; with a 20% threshold and 33.33% rejection rate, the actual decision should be ABORT.

The threshold flow returns the original valid and rejected collections along with its decision.

## 8.8 Critical Integration Requirement

The threshold decision must control database persistence.

```text
                 Threshold Validation
                          |
                 +--------+--------+
                 |                 |
              CONTINUE           ABORT
                 |                 |
                 v                 v
           Batch Database      Skip Database
             Persistence        Persistence
```

Member 4 must never execute employee insert/update operations when:

```json
{
  "persistenceAllowed": false
}
```

The standalone threshold flow calculates the decision. The main orchestration flow must enforce it.

---

# 9. Common Integration Contracts

The following contracts should be followed by all team members to avoid integration problems.

## 9.1 Canonical Employee Structure

Member 1 should convert both CSV and XML inputs into the same canonical employee structure.

Member 2 and Member 3 should use the same employee field names.

```json
{
  "recordIndex": 1,
  "employeeId": "EMP10041",
  "firstName": "Karthik",
  "lastName": "Subramanian",
  "dateOfBirth": "1994-03-18",
  "dateOfJoining": "2026-09-15",
  "officialEmail": "karthik.s@company.com",
  "mobile": "+91-9884012345",
  "departmentCode": "DEPT-ENG",
  "designation": "Senior Engineer",
  "locationCode": "LOC-CHN-01",
  "employmentType": "PERMANENT",
  "managerId": "EMP10002"
}
```

### Important field rules

| Field | Expected usage |
|---|---|
| recordIndex | Original record position in the source file |
| employeeId | Employee's unique identifier |
| officialEmail | Official email address |
| managerId | Employee identifier of the reporting manager |
| departmentCode | Department reference code |
| locationCode | Location reference code |
| designation | Designation used for grade resolution |

Do not rename fields between modules without updating the integration mapping.

## 9.2 Existing Employee Data Contract

Member 4 should provide existing employee records using the agreed field names.

```json
{
  "existingEmployees": [
    {
      "employeeId": "EMP1001",
      "officialEmail": "existing@company.com",
      "managerId": "EMP9001"
    }
  ]
}
```

The current modules accept mock existing employee data. During integration, the database lookup should supply the actual records.

## 9.3 Standard Rejection Format

All modules should use the same rejection structure.

```json
{
  "recordIndex": 5,
  "employeeId": "EMP10045",
  "rejectCode": "INVALID_DEPARTMENT",
  "rejectReason": "Unknown department code: DEPT-XXX"
}
```

### Required fields

| Field | Purpose |
|---|---|
| recordIndex | Original file row/record number |
| employeeId | Employee identifier, if available |
| rejectCode | Machine-readable rejection category |
| rejectReason | Human-readable explanation |

Use specific rejection codes rather than generic values such as `VALIDATION_FAILED`.

## 9.4 Record Preservation Rules

All team members should follow these rules:

1. Preserve `recordIndex` throughout the entire pipeline.
2. Preserve the complete employee record, including enrichment fields.
3. Do not remove original employee fields when adding validation results.
4. Do not pass already rejected records into subsequent valid-record processing.
5. Keep the final valid and rejected collections separate.
6. Do not count one employee more than once in the final rejection percentage.
7. Do not expose personal data in logs or notification messages.

## 9.5 Rejection Aggregation

The final orchestration flow must combine rejections from Member 2 and Member 3.

Example:

```text
Member 2 rejected records
          +
Member 3 rejected records
          |
          v
Final rejectedRecords
```

The final valid records must contain only employees that passed all applicable validation checks.

**Important:** The same employee may fail more than one validation rule. Agree whether to retain the first rejection or collect multiple rejection reasons. Regardless of the policy, count that employee only once in the rejection percentage.

---

# 10. Integration Instructions

The following integration responsibilities are important for the other members.

## Member 1 – File Intake

Provide:
- Canonical employee JSON.
- Consistent employee field names.
- Original record index.
- File format and run metadata.

Member 3 does not depend on CSV/XML parsing internally. All duplicate and manager logic operates on canonical employee records.

## Member 2 – Employee Validation and Reference Enrichment

Provide:
- Field-validated employees.
- Rejected employee records.
- Preserved original employee fields.
- Enriched reference information where available.

Member 3 should process only employees that are eligible for further business validation.

## Member 4 – Database and Persistence

Provide database lookup functionality for:
- Existing employee IDs.
- Existing official email addresses.
- Existing manager employee IDs and manager relationships.

The current Member 3 modules use `existingEmployees` as input, so Member 4 can supply the result of a database lookup without requiring the duplicate and manager algorithms to be rewritten.

Member 4 should also:
- Respect each employee's INSERT or UPDATE decision.
- Skip REJECT records.
- Execute persistence only when the threshold allows it.
- Return per-record and per-batch outcomes.

## Member 5 – Reject Files and Notifications

Consume the final:
- `validRecords`
- `rejectedRecords`
- `summary`
- `processingDecision`
- `rejectionPercentage`

Use the standardized rejection structure to generate CSV/XML reject files and summary emails.

## 10.1 Suggested Orchestration Order

```text
1. Receive canonical employee records
2. Retrieve existing employee information from DB
3. Run employee ID duplicate detection
4. Run email duplicate detection
5. Run manager resolution
6. Run manager hierarchy validation
7. Aggregate valid and rejected records
8. Calculate final rejection percentage
9. Check threshold decision
10. If CONTINUE, invoke database batch persistence
11. If ABORT, skip employee persistence
12. Generate reject file and summary notification
```

This is the recommended integration order. The main orchestration flow should use `flow-ref` to invoke reusable subflows rather than relying on standalone HTTP endpoints.

---

# 11. Standalone API Endpoints

The current modules expose endpoints for independent testing.

| Endpoint | HTTP Method | Purpose |
|---|---|---|
| `/duplicates` | POST | Employee ID duplicate detection |
| `/email-duplicates` | POST | Email duplicate detection |
| `/managers` | POST | Manager existence and resolution |
| `/manager-hierarchy` | POST | Circular hierarchy validation |
| `/rejection-threshold` | POST | Rejection threshold decision |

Base URL:

```text
http://localhost:8081
```

Example:

```text
POST http://localhost:8081/duplicates
```

These endpoints are intended for standalone module testing. They are not a replacement for the final end-to-end orchestration or the API-led architecture required by the hackathon.

---

# 12. Configuration

The rejection threshold is externalized in `config.properties`.

```properties
employee.import.rejection.threshold=20
```

The Mule flow reads this property using the Mule property function.

The global `configuration-properties` and HTTP Listener configuration are currently defined in `duplicateDetectionFlow.xml`.

**Integration note:** Do not define duplicate global configurations when merging the XML files. The integrated application should use shared global configurations, ideally in a dedicated global configuration XML file.

Any sensitive configuration values introduced during integration should use secure properties rather than being committed as plain text.

---

# 13. MUnit Test Coverage

Each module has a separate MUnit suite.

| MUnit Suite | Number of Tests | Scenarios |
|---|---:|---|
| `duplicateDetectionTest-suite.xml` | 7 | Unique IDs, identical duplicates, conflicting records, INSERT, UPDATE and REJECT strategies |
| `emailDuplicateTest-suite.xml` | 5 | Unique email, file duplicate, DB duplicate, own email retention and normalization |
| `managerResolutionTest-suite.xml` | 5 | Manager in file, forward reference, DB manager, missing manager and null manager |
| `managerHierarchyTest-suite.xml` | 7 | Self-manager, direct/indirect cycles, valid forward references and existing DB relationships |
| `rejectionThresholdTest-suite.xml` | 5 | Below threshold, exact threshold, above threshold, zero rejected and all rejected |

### Test execution

Run the individual MUnit suites from Anypoint Studio.

Alternatively, run the Maven test command from the Mule project root:

```bash
mvn clean test
```

If you need to execute a specific suite, use the MUnit Maven plugin's suite-selection configuration supported by the project.

### Reported test results

| Module | Passed | Failed | Errors |
|---|---:|---:|---:|
| Employee ID duplicate detection | 7 | 0 | 0 |
| Email duplicate detection | 5 | 0 | 0 |
| Manager resolution | 5 | 0 | 0 |
| Manager hierarchy | 7 | 0 | 0 |
| Rejection threshold | 5 | 0 | 0 |
| **Total** | **29** | **0** | **0** |

These are the independently reported test results. They do not represent end-to-end integration test coverage.

---

# 14. Important Implementation Decisions

The following decisions should be considered during integration.

| Area | Current implementation decision |
|---|---|
| Employee ID duplicates | First identical occurrence retained; subsequent identical occurrences rejected |
| Conflicting employee records | All conflicting occurrences rejected |
| DB employee ID duplicate | Decision depends on INSERT/UPDATE/REJECT strategy |
| Email comparison | Case-insensitive and whitespace-normalized |
| Same employee retaining email | Allowed |
| Forward manager reference | Supported using all incoming employee IDs |
| Manager lookup | Incoming records and existing DB records |
| Null manager | Allowed by current manager modules |
| Circular reporting | Recursive manager-chain traversal |
| Cycle rejection | Incoming employees that are part of a cycle are rejected |
| Threshold comparison | Strict greater-than comparison |
| Empty record set | Rejection percentage is zero |
| Threshold configuration | Externalized property |

### Policies requiring team confirmation

1. Whether duplicate employee IDs within the same input file should follow all three configured strategies or retain the current first-identical-record policy.
2. Whether missing `managerId` is allowed for all employment types and employee categories.
3. Whether a downstream employee reporting to someone in a circular chain should also be rejected.
4. Whether multiple validation failures should produce multiple rejection reasons or only one primary reason.

These decisions should be consistent with the final challenge interpretation and documented in the HLD.

---

# 15. Current Limitations and Pending Integration

The following items are outside the completed independent module logic or still need end-to-end integration.

| Item | Status |
|---|---|
| Employee ID duplicate algorithms | Implemented |
| Email duplicate algorithms | Implemented |
| Manager resolution | Implemented |
| Circular reporting validation | Implemented |
| Threshold calculation | Implemented |
| Real database lookup integration | Pending |
| Canonical model integration with Member 1 | Pending |
| Employee validation integration with Member 2 | Pending |
| Final rejection aggregation | Pending |
| Threshold-controlled batch persistence | Pending |
| End-to-end MUnit tests | Pending |
| Full CSV/XML import verification | Pending |

The current modules are independently testable and use mock database inputs. Real database connectivity and final orchestration are integration responsibilities.

---

# 16. Git Collaboration

The module should be shared through a dedicated feature branch.

Suggested branch:

```bash
feature/member3-duplicate-manager
```

### Pull latest changes

```bash
git checkout main
git pull origin main
```

### Create the feature branch

```bash
git checkout -b feature/member3-duplicate-manager
```

### Commit the implementation

```bash
git add src/
git commit -m "Implement duplicate management and manager resolution"
```

### Push the branch

```bash
git push -u origin feature/member3-duplicate-manager
```

Create a pull request into the team's agreed integration branch.

Avoid committing local credentials, passwords, secure property keys or sensitive employee information.

---

# 17. Summary

Member 3's module provides the business logic required to prevent duplicate employee records, maintain unique official email addresses, validate manager references and reporting relationships, and enforce the rejection threshold before persistence.

The implementation is divided into reusable Mule subflows and independently tested using MUnit.

**Current independent testing status: 29 MUnit tests passed across five suites.**

The next integration steps are to connect the modules with:

- Member 1's canonical employee records.
- Member 2's validated and enriched employees.
- Member 4's actual database lookup and batch persistence.
- Member 5's rejection file generation and notifications.

The most important integration rule is that the final rejection threshold must be calculated using a unique, complete partition of the original input records, and an ABORT decision must prevent all employee database writes.

---

**End of Member 3 Implementation Documentation**