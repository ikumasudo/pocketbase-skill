# Go Hooks & Custom Routes Reference

PocketBase v0.23+ Go event hooks, custom HTTP routes, middleware, cron, and email.

---

## 1. Hook Registration Pattern

All hooks follow the same pattern: `app.OnEventName().BindFunc(handler)`.

```go
app.OnRecordCreate("posts").BindFunc(func(e *core.RecordEvent) error {
    // your logic here
    return e.Next() // REQUIRED — continues the execution chain
})
```

**Key differences from JSVM:**

| | Go | JSVM |
|-|-----|------|
| Error handling | Return `error` | Throw exception |
| Chain continuation | `return e.Next()` (propagates error) | `e.next()` (void) |
| Signature | `func(e *EventType) error` | `(e) => { ... }` |
| Collection filter | `app.OnRecordCreate("posts")` | `onRecordCreate((e) => {...}, "posts")` |

> **Critical:** Forgetting `e.Next()` in Go causes the request to hang. Always `return e.Next()`.

---

## 2. Record Lifecycle Hooks

### Create

| Hook | Timing | Use Case |
|------|--------|----------|
| `OnRecordCreate` | Before validation & DB INSERT | Set defaults, transform data, validate |
| `OnRecordCreateExecute` | After validation, before DB commit | Last chance to abort (inside transaction) |
| `OnRecordAfterCreateSuccess` | After transaction commit | Send notifications, trigger side effects |
| `OnRecordAfterCreateError` | After failed create | Cleanup, logging |

### Update

| Hook | Timing | Use Case |
|------|--------|----------|
| `OnRecordUpdate` | Before validation & DB UPDATE | Validate changes, set computed fields |
| `OnRecordUpdateExecute` | After validation, before DB commit | Inside transaction |
| `OnRecordAfterUpdateSuccess` | After transaction commit | Notifications |
| `OnRecordAfterUpdateError` | After failed update | Cleanup |

### Delete

| Hook | Timing | Use Case |
|------|--------|----------|
| `OnRecordDelete` | Before delete checks | Prevent deletion, cascade logic |
| `OnRecordDeleteExecute` | Before DB DELETE statement | Inside transaction |
| `OnRecordAfterDeleteSuccess` | After transaction commit | Cleanup external resources |
| `OnRecordAfterDeleteError` | After failed delete | Logging |

### API Request Hooks

These fire only for REST API requests (not programmatic `app.Save()`):

| Hook | Event Type | Description |
|------|-----------|-------------|
| `OnRecordCreateRequest` | `*core.RecordRequestEvent` | API record create (has `e.Request`, `e.Auth`) |
| `OnRecordUpdateRequest` | `*core.RecordRequestEvent` | API record update |
| `OnRecordDeleteRequest` | `*core.RecordRequestEvent` | API record delete |

### Example: Set Default on Create

```go
app.OnRecordCreate("posts").BindFunc(func(e *core.RecordEvent) error {
    e.Record.Set("status", "draft")
    return e.Next()
})
```

### Example: Prevent Delete

```go
app.OnRecordDelete("categories").BindFunc(func(e *core.RecordEvent) error {
    // check if category has posts
    posts, err := e.App.FindRecordsByFilter("posts", "category={:id}", "", 1, 0,
        dbx.Params{"id": e.Record.Id})
    if err != nil {
        return err
    }
    if len(posts) > 0 {
        return e.BadRequestError("Cannot delete category with existing posts", nil)
    }
    return e.Next()
})
```

### Example: After Create Notification

```go
app.OnRecordAfterCreateSuccess("orders").BindFunc(func(e *core.RecordEvent) error {
    // send email, call webhook, etc.
    log.Printf("New order created: %s", e.Record.Id)
    return e.Next()
})
```

---

## 3. Custom HTTP Routes

Register routes inside `app.OnServe().BindFunc()`:

```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    // GET with path param
    se.Router.GET("/api/hello/{name}", func(e *core.RequestEvent) error {
        name := e.Request.PathValue("name")
        return e.String(http.StatusOK, "Hello "+name)
    })

    // POST with JSON response
    se.Router.POST("/api/myapp/settings", func(e *core.RequestEvent) error {
        // process request...
        return e.JSON(http.StatusOK, map[string]bool{"success": true})
    }).Bind(apis.RequireAuth())

    return se.Next()
})
```

### Request Handling

```go
// path parameter
name := e.Request.PathValue("name")

// query parameter
page := e.Request.URL.Query().Get("page")

// read JSON body
data := struct {
    Title string `json:"title"`
}{}
if err := e.BindBody(&data); err != nil {
    return e.BadRequestError("Invalid body", err)
}
```

### Response Methods

```go
e.String(http.StatusOK, "text response")
e.JSON(http.StatusOK, map[string]any{"key": "value"})
e.HTML(http.StatusOK, "<h1>Hello</h1>")
e.NoContent(http.StatusNoContent)
e.Redirect(http.StatusTemporaryRedirect, "/other")
```

### Access Auth in Routes

```go
se.Router.GET("/api/me", func(e *core.RequestEvent) error {
    authRecord := e.Auth
    if authRecord == nil {
        return e.UnauthorizedError("Not authenticated", nil)
    }
    return e.JSON(http.StatusOK, authRecord)
}).Bind(apis.RequireAuth())
```

---

## 4. Middleware

### Route-Level Middleware

```go
se.Router.GET("/api/admin/stats", handler).Bind(apis.RequireSuperuserAuth())

se.Router.POST("/api/data", handler).Bind(apis.RequireAuth())

// allow only specific auth collections
se.Router.GET("/api/staff", handler).Bind(apis.RequireAuth("staff", "admins"))
```

### Available Built-in Middleware

| Middleware | Description |
|-----------|-------------|
| `apis.RequireAuth(collections...)` | Require authenticated user (optionally from specific collections) |
| `apis.RequireSuperuserAuth()` | Require superuser |
| `apis.RequireGuestOnly()` | Require unauthenticated |
| `apis.RequireSuperuserOrOwnerAuth(param)` | Superuser or owner of the route param record |
| `apis.Gzip()` | Gzip response compression |
| `apis.BodyLimit(bytes)` | Override default 32MB body limit |
| `apis.SkipSuccessActivityLog()` | Only log failed requests |

### Global Middleware

```go
app.OnServe().BindFunc(func(se *core.ServeEvent) error {
    se.Router.BindFunc(func(e *core.RequestEvent) error {
        // runs for every request
        log.Printf("%s %s", e.Request.Method, e.Request.URL.Path)
        return e.Next()
    })

    return se.Next()
})
```

---

## 5. Cron Jobs

```go
app.Cron().MustAdd("cleanup", "0 3 * * *", func() {
    // runs daily at 3:00 AM
    records, _ := app.FindRecordsByFilter("temp_files", "created < @todayStart", "", 0, 0)
    for _, r := range records {
        app.Delete(r)
    }
})

app.Cron().MustAdd("heartbeat", "*/5 * * * *", func() {
    // runs every 5 minutes
    log.Println("heartbeat")
})
```

Common cron expressions:

| Expression | Schedule |
|-----------|----------|
| `* * * * *` | Every minute |
| `*/5 * * * *` | Every 5 minutes |
| `0 * * * *` | Every hour |
| `0 3 * * *` | Daily at 3:00 AM |
| `0 0 * * 0` | Weekly (Sunday midnight) |
| `@hourly` | Every hour |
| `@daily` | Daily at midnight |

---

## 6. Email Sending

```go
import "github.com/pocketbase/pocketbase/tools/mailer"

message := &mailer.Message{
    From:    mail.Address{Name: "MyApp", Address: "noreply@example.com"},
    To:      []mail.Address{{Address: "user@example.com"}},
    Subject: "Welcome!",
    HTML:    "<h1>Welcome to MyApp</h1><p>Thanks for signing up.</p>",
}

err := app.NewMailClient().Send(message)
```

> Email settings (SMTP) must be configured in the Dashboard under Settings > Mail settings, or programmatically.

---

## 7. JSVM ↔ Go Comparison

| Feature | JSVM (`pb_hooks/*.pb.js`) | Go |
|---------|--------------------------|-----|
| Hook registration | `onRecordCreate((e) => {...}, "col")` | `app.OnRecordCreate("col").BindFunc(func(e *core.RecordEvent) error {...})` |
| Chain continuation | `e.next()` | `return e.Next()` |
| Find record | `$app.findRecordById("col", id)` | `app.FindRecordById("col", id)` |
| Find by filter | `$app.findFirstRecordByFilter("col", "f={:v}", {v: val})` | `app.FindFirstRecordByFilter("col", "f={:v}", dbx.Params{"v": val})` |
| Save record | `$app.save(record)` | `app.Save(record)` |
| Delete record | `$app.delete(record)` | `app.Delete(record)` |
| Get field | `record.get("field")` | `record.GetString("field")`, `record.GetInt("field")` |
| Set field | `record.set("field", value)` | `record.Set("field", value)` |
| New record | `new Record(collection)` | `core.NewRecord(collection)` |
| Custom route | `routerAdd("GET", "/path", handler)` | `se.Router.GET("/path", handler)` |
| Cron | `cronAdd("id", "expr", handler)` | `app.Cron().MustAdd("id", "expr", handler)` |
| Send email | `$app.newMailClient().send(msg)` | `app.NewMailClient().Send(msg)` |
| Raw SQL | `$app.db().newQuery(sql).execute()` | `app.DB().NewQuery(sql).Execute()` |
