# Go Testing Reference

In-process integration tests for PocketBase Go package mode using `github.com/pocketbase/pocketbase/tests`.

---

## Go Tests vs Python E2E — When to Use Which

| | Python E2E | Go Testing |
|---|---|---|
| Purpose | Verify API rules (access control) | Verify custom Go code (routes, hooks, middleware) |
| Runtime | HTTP requests against a running PocketBase | In-process (`go test`) — no running server needed |
| Applicable modes | Standalone + Go package | Go package mode only |
| When to use | Collection has non-`null` API rules | Custom routes or hooks written in Go |

**Use both when applicable.** Python E2E tests API rules; Go tests custom code behavior.

---

## What to Test

### 1. Custom HTTP Routes

- Route returns correct status code and response body
- Non-existent path returns 404
- Wrong HTTP method returns 405

### 2. Auth Middleware

- `apis.RequireSuperuserAuth()` → guest 401, regular user 401, superuser 200
- `apis.RequireAuth()` → guest 401, authenticated user 200
- `apis.RequireAuth("staff")` → wrong collection 401, staff member 200

### 3. Record Lifecycle Hooks

- `OnRecordCreate` sets defaults (e.g., `status` → `"draft"`)
- `OnRecordUpdate` validates changes (e.g., block status change after publish)
- `OnRecordDelete` conditionally rejects (e.g., 400 when child records exist)
- `ExpectedEvents` verifies correct hooks fire the right number of times

### 4. Hook Side Effects

- `OnRecordAfterCreateSuccess` creates related records → verify with `AfterTestFunc`
- Hook writes to DB → assert the written content

### 5. Request Hooks

- `OnRecordCreateRequest` transforms the request body correctly
- `OnRecordUpdateRequest` blocks certain field changes

---

## Step-by-Step Setup

### Step 1: Extract Custom Code into a Bindable Function

Separate hook/route registration from `main()` so the same function can be called on a `TestApp`.

```go
// hooks.go
package main

import (
    "net/http"

    "github.com/pocketbase/pocketbase/apis"
    "github.com/pocketbase/pocketbase/core"
)

func bindAppHooks(app core.App) {
    app.OnServe().BindFunc(func(se *core.ServeEvent) error {
        se.Router.GET("/api/hello/{name}", func(e *core.RequestEvent) error {
            name := e.Request.PathValue("name")
            return e.JSON(http.StatusOK, map[string]string{"message": "Hello " + name})
        }).Bind(apis.RequireAuth())
        return se.Next()
    })

    app.OnRecordCreate("posts").BindFunc(func(e *core.RecordEvent) error {
        e.Record.Set("status", "draft")
        return e.Next()
    })
}
```

```go
// main.go
package main

import (
    "log"
    "os"
    "strings"

    "github.com/pocketbase/pocketbase"
    "github.com/pocketbase/pocketbase/plugins/migratecmd"

    _ "yourmodule/migrations"
)

func main() {
    app := pocketbase.New()

    isGoRun := strings.HasPrefix(os.Args[0], os.TempDir())
    migratecmd.MustRegister(app, app.RootCmd, migratecmd.Config{
        Automigrate: isGoRun,
    })

    bindAppHooks(app)

    if err := app.Start(); err != nil {
        log.Fatal(err)
    }
}
```

### Step 2: Prepare Test Data

```bash
# Start PocketBase with a dedicated test data directory
./pocketbase serve --dir="./test_pb_data" --automigrate=0

# Open Dashboard (http://127.0.0.1:8090/_/) and create:
#   - Test collections with fields matching your app
#   - Test user (e.g., test@example.com / testpass123)
#   - Superuser (e.g., admin@example.com / adminpass123)
#   - Sample records as needed
# Then Ctrl+C to stop
```

- **Commit `test_pb_data/` to version control** — it's your test fixture
- Each test run clones the data automatically — no cross-test interference
- If using Go migrations, run them first against `test_pb_data/` to create the schema

### Step 3: Token Generation Helper

```go
// main_test.go
package main

import (
    "net/http"
    "strings"
    "testing"

    "github.com/pocketbase/pocketbase/core"
    "github.com/pocketbase/pocketbase/tests"
)

const testDataDir = "./test_pb_data"

func generateToken(collectionNameOrId string, email string) (string, error) {
    app, err := tests.NewTestApp(testDataDir)
    if err != nil {
        return "", err
    }
    defer app.Cleanup()

    record, err := app.FindAuthRecordByEmail(collectionNameOrId, email)
    if err != nil {
        return "", err
    }
    return record.NewAuthToken()
}
```

### Step 4: Test App Factory

```go
setupTestApp := func(t testing.TB) *tests.TestApp {
    testApp, err := tests.NewTestApp(testDataDir)
    if err != nil {
        t.Fatal(err)
    }
    bindAppHooks(testApp) // apply your custom hooks/routes
    return testApp
}
```

### Step 5: Write ApiScenario Tests

```go
func TestHelloRoute(t *testing.T) {
    userToken, err := generateToken("users", "test@example.com")
    if err != nil {
        t.Fatal(err)
    }
    superuserToken, err := generateToken(core.CollectionNameSuperusers, "admin@example.com")
    if err != nil {
        t.Fatal(err)
    }

    setupTestApp := func(t testing.TB) *tests.TestApp {
        testApp, err := tests.NewTestApp(testDataDir)
        if err != nil {
            t.Fatal(err)
        }
        bindAppHooks(testApp)
        return testApp
    }

    scenarios := []tests.ApiScenario{
        {
            Name:            "guest is denied",
            Method:          http.MethodGet,
            URL:             "/api/hello/world",
            ExpectedStatus:  401,
            ExpectedContent: []string{`"data":{}`},
            TestAppFactory:  setupTestApp,
        },
        {
            Name:            "authenticated user succeeds",
            Method:          http.MethodGet,
            URL:             "/api/hello/world",
            Headers:         map[string]string{"Authorization": userToken},
            ExpectedStatus:  200,
            ExpectedContent: []string{`"message":"Hello world"`},
            TestAppFactory:  setupTestApp,
        },
        {
            Name:            "superuser succeeds",
            Method:          http.MethodGet,
            URL:             "/api/hello/world",
            Headers:         map[string]string{"Authorization": superuserToken},
            ExpectedStatus:  200,
            ExpectedContent: []string{`"message":"Hello world"`},
            TestAppFactory:  setupTestApp,
        },
        {
            Name:            "wrong method returns 405",
            Method:          http.MethodPost,
            URL:             "/api/hello/world",
            Headers:         map[string]string{"Authorization": userToken},
            ExpectedStatus:  405,
            TestAppFactory:  setupTestApp,
        },
    }

    for _, s := range scenarios {
        s.Test(t)
    }
}
```

### Step 6: Run Tests

```bash
go test ./...          # all tests
go test -run TestHello # specific test
go test -v ./...       # verbose output
```

---

## ApiScenario Field Reference

| Field | Type | Description |
|-------|------|-------------|
| `Name` | `string` | Test name (displayed on failure) |
| `Method` | `string` | HTTP method (`http.MethodGet`, etc.) |
| `URL` | `string` | Request path (e.g., `/api/hello/world`) |
| `Body` | `io.Reader` | Request body (`strings.NewReader(\`{"key":"val"}\`)`) |
| `Headers` | `map[string]string` | Request headers (e.g., `"Authorization": token`) |
| `Delay` | `time.Duration` | Wait before checking expectations (for async side effects) |
| `Timeout` | `time.Duration` | Request context timeout (0 = no timeout) |
| `ExpectedStatus` | `int` | Expected HTTP status code |
| `ExpectedContent` | `[]string` | Strings that MUST appear in the response body |
| `NotExpectedContent` | `[]string` | Strings that MUST NOT appear in the response body |
| `ExpectedEvents` | `map[string]int` | Hook events and their expected fire count |
| `TestAppFactory` | `func(t testing.TB) *TestApp` | Factory that creates the test app |
| `BeforeTestFunc` | `func(t testing.TB, app *TestApp, e *core.ServeEvent)` | Runs before the request is sent |
| `AfterTestFunc` | `func(t testing.TB, app *TestApp, res *http.Response)` | Runs after the response is received |

---

## Common Test Patterns

### Auth Middleware (RequireSuperuserAuth)

```go
{
    Name:           "guest denied by superuser middleware",
    Method:         http.MethodGet,
    URL:            "/api/admin/stats",
    ExpectedStatus: 401,
    ExpectedContent: []string{`"data":{}`},
    TestAppFactory: setupTestApp,
},
{
    Name:           "regular user denied by superuser middleware",
    Method:         http.MethodGet,
    URL:            "/api/admin/stats",
    Headers:        map[string]string{"Authorization": userToken},
    ExpectedStatus: 401,
    ExpectedContent: []string{`"data":{}`},
    TestAppFactory: setupTestApp,
},
{
    Name:           "superuser allowed",
    Method:         http.MethodGet,
    URL:            "/api/admin/stats",
    Headers:        map[string]string{"Authorization": superuserToken},
    ExpectedStatus: 200,
    TestAppFactory: setupTestApp,
},
```

### Auth Middleware (RequireAuth with Collection Filter)

```go
// Route: se.Router.GET("/api/staff/dashboard", handler).Bind(apis.RequireAuth("staff"))

{
    Name:           "user from wrong collection denied",
    Method:         http.MethodGet,
    URL:            "/api/staff/dashboard",
    Headers:        map[string]string{"Authorization": regularUserToken},
    ExpectedStatus: 401,
    TestAppFactory: setupTestApp,
},
{
    Name:           "staff member allowed",
    Method:         http.MethodGet,
    URL:            "/api/staff/dashboard",
    Headers:        map[string]string{"Authorization": staffToken},
    ExpectedStatus: 200,
    TestAppFactory: setupTestApp,
},
```

### Hook Setting Default Values (ExpectedEvents)

```go
{
    Name:   "create post sets status to draft",
    Method: http.MethodPost,
    URL:    "/api/collections/posts/records",
    Body:   strings.NewReader(`{"title":"Test Post"}`),
    Headers: map[string]string{"Authorization": userToken},
    ExpectedStatus:  200,
    ExpectedContent: []string{`"status":"draft"`},
    ExpectedEvents: map[string]int{
        "OnRecordCreate":             1,
        "OnRecordAfterCreateSuccess": 1,
    },
    TestAppFactory: setupTestApp,
},
```

### Hook Side Effects (AfterTestFunc)

```go
{
    Name:   "creating order triggers audit log",
    Method: http.MethodPost,
    URL:    "/api/collections/orders/records",
    Body:   strings.NewReader(`{"product":"widget","qty":2}`),
    Headers: map[string]string{"Authorization": userToken},
    ExpectedStatus: 200,
    TestAppFactory: setupTestApp,
    AfterTestFunc: func(t testing.TB, app *tests.TestApp, res *http.Response) {
        // verify the hook created an audit_log record
        logs, err := app.FindRecordsByFilter("audit_logs", "action='order_created'", "", 0, 0)
        if err != nil {
            t.Fatal(err)
        }
        if len(logs) != 1 {
            t.Fatalf("expected 1 audit log, got %d", len(logs))
        }
    },
},
```

### Hook Rejecting Delete (Child Records Exist)

```go
{
    Name:           "cannot delete category with posts",
    Method:         http.MethodDelete,
    URL:            "/api/collections/categories/records/CATEGORY_ID",
    Headers:        map[string]string{"Authorization": superuserToken},
    ExpectedStatus: 400,
    ExpectedContent: []string{"Cannot delete category with existing posts"},
    TestAppFactory: setupTestApp,
},
```

### POST with JSON Body

```go
{
    Name:   "create record with body",
    Method: http.MethodPost,
    URL:    "/api/collections/posts/records",
    Body:   strings.NewReader(`{"title":"Hello","content":"World"}`),
    Headers: map[string]string{
        "Authorization": userToken,
        "Content-Type":  "application/json",
    },
    ExpectedStatus:  200,
    ExpectedContent: []string{`"title":"Hello"`},
    TestAppFactory:  setupTestApp,
},
```

### File Upload (MockMultipartData)

Use `tests.MockMultipartData` to simulate file uploads in tests:

```go
body, contentType, err := tests.MockMultipartData(
    map[string]string{
        "title":       "My Document",
        "description": "A test file",
    },
    "document", // field name for the file
)
if err != nil {
    t.Fatal(err)
}

// ... then in the scenario:
{
    Name:   "upload file with metadata",
    Method: http.MethodPost,
    URL:    "/api/collections/documents/records",
    Body:   body,
    Headers: map[string]string{
        "Authorization": userToken,
        "Content-Type":  contentType,
    },
    ExpectedStatus: 200,
    ExpectedContent: []string{`"title":"My Document"`},
    TestAppFactory:  setupTestApp,
},
```

`MockMultipartData(data map[string]string, fileFields ...string)` generates a multipart body with:
- `data` — form field key-value pairs
- `fileFields` — names of fields that will get a dummy file attachment

### BeforeTestFunc (Pre-Request Setup)

```go
{
    Name:   "update route with pre-existing data",
    Method: http.MethodPatch,
    URL:    "/api/collections/posts/records/RECORD_ID",
    Body:   strings.NewReader(`{"title":"Updated"}`),
    Headers: map[string]string{"Authorization": userToken},
    ExpectedStatus: 200,
    TestAppFactory: setupTestApp,
    BeforeTestFunc: func(t testing.TB, app *tests.TestApp, e *core.ServeEvent) {
        // insert a record that the test will update
        collection, _ := app.FindCollectionByNameOrId("posts")
        record := core.NewRecord(collection)
        record.Set("title", "Original")
        if err := app.Save(record); err != nil {
            t.Fatal(err)
        }
    },
},
```

---

## Gotchas

### `test_pb_data` Must Exist Before Running Tests

`tests.NewTestApp(testDataDir)` clones the directory. If `test_pb_data/` doesn't exist or is empty, tests fail immediately. Set it up per Step 2 before running `go test`.

### Token Is Generated at Test Init, Not Per-Scenario

`generateToken()` creates a short-lived JWT. If your test suite takes a long time, tokens may expire. For typical test suites this is not an issue.

### Each Scenario Gets a Fresh App Clone

`TestAppFactory` is called per scenario. The cloned SQLite DB is independent — changes in one scenario do not affect others. This means:
- You cannot rely on a record created in scenario A being present in scenario B
- Use `BeforeTestFunc` to set up per-scenario data

### `bindAppHooks` Must Be Called in TestAppFactory

If you forget `bindAppHooks(testApp)` in your factory, the test app won't have your custom routes/hooks. Tests will fail with unexpected 404s or missing hook behavior.

### ExpectedContent Matches Substrings

`ExpectedContent: []string{`"status":"draft"`}` checks that the string appears anywhere in the response body. It does not parse JSON — it's a simple `strings.Contains` check. Watch out for:
- Whitespace differences (PocketBase responses are typically compact JSON without spaces)
- Partial matches (e.g., `"stat"` would match `"status"`)

### CGO and SQLite

PocketBase uses SQLite. By default, `go test` uses the CGO-free `modernc.org/sqlite` driver (slower but portable). For faster tests with CGO-enabled SQLite:

```bash
CGO_ENABLED=1 go test ./...
```

### Running Tests in CI

Ensure `test_pb_data/` is committed to version control and available in the CI environment. No running PocketBase instance is needed — tests are fully self-contained.
