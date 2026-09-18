# Lesson 3 Ticket Breakdown: Logging in the Library Lending API

Extend the **Library Lending API you built in Lesson 2**. Add logs that let a developer see which operations completed, inspect lookup results, and understand why a member could not be deleted.

Continue using `Member`, `Loan`, their existing services and repositories, MySQL, and your **Library Lending** Postman collection. Keep your existing records and database connection settings. You do not need a new project, new endpoints, or another database setup script.

**Goal:** Add useful logging without changing the API's existing responses or database behavior.

## Files you will change

| File | Work |
| --- | --- |
| `src/main/java/com/example/library/services/MemberService.java` | Log member lookups, successful changes, and blocked deletion. |
| `src/main/java/com/example/library/services/LoanService.java` | Log loan lookups, successful changes, and missing member references. |
| `src/main/resources/application.properties` | Control which messages appear and enable file output. |

These paths use yesterday's package, `com.example.library`. If your project uses another package, keep your existing package declarations and use that same package in the logging setting below.

Your models, repositories, controllers, constructor injection, validation, and HTTP status codes should keep working as they did yesterday. Keep any optional methods you already added.

## Ticket 1: Add a logger to each service

**Purpose:** Identify the Java class that produced each message.

### Tasks

1. In both service files, import `org.slf4j.Logger` and `org.slf4j.LoggerFactory`.
2. Add a `private static final Logger` field named `log` to each class. Initialize it with `LoggerFactory.getLogger(...)`, passing that service's own class.
3. Leave the repository fields and constructors in place. The logger does not replace either of them.
4. Add or update these properties. Keep one entry per property:

```properties
logging.level.root=INFO
logging.level.com.example.library=INFO
spring.jpa.show-sql=false
```

5. Keep your existing Maven dependencies. Spring Web already brings in Spring Boot's default logging setup; no additional SLF4J, Logback, or Lombok dependency is required. [Spring Boot logging](https://docs.spring.io/spring-boot/3.5/reference/features/logging.html)

`spring.jpa.show-sql=false` hides the SQL printout so you can focus on your messages. Database queries still execute.

**Acceptance criteria:** Both services compile with SLF4J imports. Each logger uses its own service class, and the application still starts normally. The messages added in the next tickets must identify `MemberService` or `LoanService` as their source.

## Ticket 2: Record successful changes at INFO

**Purpose:** Show which records were actually created, updated, or deleted.

### Tasks

1. In both services, add an INFO message after a successful `create(...)` and `update(...)` save. If a method directly returns `repository.save(...)`, store that result in a local variable, log its ID, then return the same saved object.
2. Add an INFO message after a successful `delete(...)` repository call and before returning `true`.
3. Include the operation and record ID. For loan creation and updates, include the associated member ID as well.
4. Use SLF4J `{}` placeholders with values supplied as arguments. Log IDs rather than entire entity objects or request bodies. [SLF4J manual](https://slf4j.org/manual.html)

Use these message shapes; the numbers below are examples, not values to hard-code:

| Operation | Example message |
| --- | --- |
| Create member | `Created member id=7` |
| Update member | `Updated member id=7` |
| Delete member | `Deleted member id=7` |
| Create loan | `Created loan id=110 for member id=7` |
| Update loan | `Updated loan id=110 for member id=7` |
| Delete loan | `Deleted loan id=110` |

**Acceptance criteria:** Each successful write produces one corresponding success message from your service. A missing record, validation failure, or blocked deletion produces no success message. Return values and HTTP responses remain unchanged.

## Ticket 3: Inspect reads and missing records at DEBUG

**Purpose:** Make diagnostic details available when investigating behavior, while keeping them hidden during normal INFO logging.

### Tasks

1. In each service's `findAll()`, save the retrieved list in a local variable. Log its size at DEBUG, then return that same list. Do not run the query a second time just to obtain a count.
2. In each `findById(Long id)`, log the requested ID at DEBUG. In the existing missing-record branch, log that the record was not found before returning `null`.
3. In `LoanService.findByMemberId(Long memberId)`, log both the requested member ID and the number of matching loans after retrieving the list. A zero count is a valid result.
4. In `LoanService.create(...)`, add a DEBUG message in the existing branch where the requested member does not exist. Include that member ID and keep returning `null` so the controller still responds with 404.
5. In each service's `delete(...)`, add a DEBUG message in the existing missing-record branch before returning `false`.

### Check the threshold

First send GET requests with the application package set to INFO. Then change **that same property** to:

```properties
logging.level.com.example.library=DEBUG
```

Restart the application and repeat the requests. Use your actual package if it differs. DEBUG enables DEBUG, INFO, WARN, and ERROR messages for that package; you do not add separate INFO and WARN entries. [Spring Boot log levels](https://docs.spring.io/spring-boot/3.5/reference/features/logging.html#features.logging.log-levels)

**Acceptance criteria:** Your DEBUG messages are hidden at INFO and visible at DEBUG. Both settings produce the same HTTP responses. Counts come from the returned lists, and a missing lookup still returns 404. An unmatched member-loan query still returns 200 with `[]`.

## Ticket 4: Log a blocked member deletion at WARN

**Purpose:** Explain an operation the application rejected because the member still has a loan.

### Tasks

1. Locate the existing `catch (DataIntegrityViolationException ex)` block in `MemberService.delete(...)`.
2. Add a WARN message inside that block, before the existing 409 exception is thrown. Include the member ID and a short reason, such as `Member deletion blocked: id=7 has a database constraint`.
3. Keep the existing `ResponseStatusException` and its CONFLICT status. Keep the foreign key and the rule that loans must be removed or reassigned before deleting their member.

**Acceptance criteria:** Attempting to delete a member with a loan returns 409 and produces a WARN from `MemberService`. The member and loan remain stored, and there is no `Deleted member` success message for that attempt.

Hibernate may also write an ERROR for the rejected SQL operation. That framework message can appear alongside your service's WARN; use the logger name to distinguish them.

## Ticket 5: Verify the logs in Postman and save them to a file

### A. Enable file output

Add this property, without a leading `#`, and restart the backend:

```properties
logging.file.name=logs/library.log
```

Keep the application package at DEBUG for this verification. New log events now appear in both the console and the file. `logs/library.log` is relative to the application's working directory; if you run from `backend`, look in `backend/logs/library.log`. Earlier console messages are not copied into the file. [Spring Boot file output](https://docs.spring.io/spring-boot/3.5/reference/features/logging.html#features.logging.file-output)

### B. Create temporary records

Use **Body > raw > JSON** for POST and PUT. GET and DELETE requests have no body.

Create a member:

```http
POST http://localhost:8080/api/members
```

```json
{
  "name": "Jordan Lee",
  "email": "jordan.logging@example.test"
}
```

Expect **201**, a generated ID in the response, and an INFO message identifying that member ID.

**The remaining examples assume this POST returned member ID `7`. If yours differs, use your returned number everywhere `7` appears in a URL or JSON body. Do not assign the ID yourself.**

Create a loan for that member:

```http
POST http://localhost:8080/api/loans
```

```json
{
  "bookTitle": "Learning Spring Boot",
  "member": {"id": 7}
}
```

Expect **201** and an INFO message with both the loan ID and member ID.

**The remaining examples assume the loan's outer `id` is `110`. Use your actual loan ID wherever `110` appears below. The nested `member.id` identifies the member, not the loan.**

### C. Read and update the records

Send these requests and compare the response with your service logs:

| Request | Expected response | Required log evidence |
| --- | --- | --- |
| GET `http://localhost:8080/api/members` | 200, member list | DEBUG count matches the number of returned members. |
| GET `http://localhost:8080/api/loans` | 200, loan list | DEBUG count matches the number of returned loans. |
| GET `http://localhost:8080/api/members/7` | 200 | DEBUG lookup identifies member 7. |
| GET `http://localhost:8080/api/loans/110` | 200 | DEBUG lookup identifies loan 110. |
| GET `http://localhost:8080/api/loans/member/7` | 200, the temporary member's loan list | DEBUG includes member 7 and a count of 1. |

Update the member:

```http
PUT http://localhost:8080/api/members/7
```

```json
{
  "name": "Jordan Reed",
  "email": "jordan.reed@example.test"
}
```

Update the loan:

```http
PUT http://localhost:8080/api/loans/110
```

```json
{
  "bookTitle": "Spring Boot Practice",
  "member": {"id": 7}
}
```

Both updates must return **200** and produce the matching INFO messages. GET both records again: the values changed, but the IDs stayed the same.

### D. Verify rejected operations and delete in the correct order

First confirm member ID `999999` is absent with GET `http://localhost:8080/api/members/999999`. Expect **404** and a DEBUG not-found message. If that ID exists in your database, choose another absent positive ID for this check and the following POST.

Try to create a loan for the missing member:

```http
POST http://localhost:8080/api/loans
```

```json
{
  "bookTitle": "Logging Practice",
  "member": {"id": 999999}
}
```

Expect **404**, a DEBUG message explaining the missing member, and no `Created loan` success message. GET the full loan list and confirm no loan was added.

Now send these requests in order, using the actual IDs from your successful POST responses:

| Request | Expected response and log evidence |
| --- | --- |
| DELETE `http://localhost:8080/api/members/7` | 409; WARN identifies the blocked member deletion. No deletion success message. |
| GET `http://localhost:8080/api/members/7` | 200; member still exists. |
| GET `http://localhost:8080/api/loans/110` | 200; loan still exists. |
| DELETE `http://localhost:8080/api/loans/110` | 204; INFO identifies the deleted loan. |
| GET `http://localhost:8080/api/loans/110` | 404; DEBUG identifies the missing loan. |
| DELETE `http://localhost:8080/api/loans/110` again | 404; DEBUG identifies the skipped deletion. No second deletion success message. |
| GET `http://localhost:8080/api/loans/member/7` | 200 with `[]`; DEBUG reports zero matching loans. |
| DELETE `http://localhost:8080/api/members/7` | 204; INFO identifies the deleted member. |
| GET `http://localhost:8080/api/members/7` | 404; DEBUG identifies the missing member. |
| DELETE `http://localhost:8080/api/members/7` again | 404; DEBUG identifies the skipped deletion. No second deletion success message. |

### E. Confirm the file and restore INFO

1. Open `logs/library.log`. Find an INFO success message, a DEBUG lookup or count, and your WARN for the blocked deletion. Record the service class and ID beside each message.
2. Change the application package threshold back to INFO and restart.
3. Repeat GET `http://localhost:8080/api/members` and GET `http://localhost:8080/api/loans`. Both still return 200, but these new requests must not produce your DEBUG count messages in either destination.
4. Check timestamps when reading the file: earlier DEBUG entries remain in it after you change the threshold.

**Acceptance criteria:** Postman results match the existing API contract, the file contains your new messages, and changing the threshold controls new output without changing application data.


