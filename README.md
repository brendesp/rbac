# RBAC - Simple, concurrent Role Based Access Control(GO)

[![Build Status](https://travis-ci.org/euroteltr/rbac.svg?branch=master)](https://travis-ci.org/euroteltr/rbac)
[![Go Report Card](https://goreportcard.com/badge/github.com/euroteltr/rbac)](https://goreportcard.com/report/github.com/euroteltr/rbac)
[![Codecov](https://codecov.io/gh/euroteltr/rbac/branch/master/graph/badge.svg)](https://codecov.io/gh/euroteltr/rbac/branch/master/graph/)
[![GoDoc](https://godoc.org/github.com/euroteltr/rbac?status.svg)](https://godoc.org/github.com/euroteltr/rbac)
[![GitHub release](https://img.shields.io/github/release/euroteltr/rbac.svg)](https://github.com/euroteltr/rbac/releases/latest)
[![License](https://img.shields.io/badge/license-mit-blue.svg)](https://github.com/euroteltr/rbac/blob/master/LICENCE.txt)

RBAC is role based access control library for GOlang. At core uses `sync.Map` so, it can be used from multiple goroutines concurrently. "Keep it simple" is also in core.

It supports role inheritance.

It can be used in middleware(example for echo framework is [given](https://github.com/euroteltr/rbac#usage-as-middleware) )

Go 1.9+ is required.

It is built on;

| Type       | Description                                             | Example                     |
| ---------- | ------------------------------------------------------- | --------------------------- |
| Action     | Defines what can possible for a permission              | `Create`,`Read`,`Update`... |
| Permission | Defines permission related to a resource to be accessed | `users`                     |
| Role       | Defines group of permissions with defined actions       | `admin`                     |

## Usage

Library usage has 2 phases:

- Development
- Runtime

### Development Phase

First get an instance for `RBAC`

```go
import "github.com/euroteltr/rbac"

R := rbac.New(nil)
// you can pass a logger to constructor also:
// R := rbac.New(rbac.ConsoleLogger)
```

During development you will register your permissions for your each resource with valid actions for that permission:

```go
// You can also use rbac.CRUD for those crud actions
usersPerm, err := R.RegisterPermission("users", "User resource", rbac.Create, rbac.Read, rbac.Update, rbac.Delete)
if err != nil {
    panic(err)
}
```

`userPerm` is defined with CRUD actions, which means that any action not in that list will be invalid. You can define your own actions( like `ApproveAction := rbac.Action("approve")`) and add them also.

### Runtime Phase

At run time we define our roles and permit permissions to them.

```go
adminRole, err := R.RegisterRole("admin", "Admin role")
if err != nil {
    fmt.Printf("can not add admin role, err: %v\n", err)
}
if err = R.Permit(adminRole.ID, usersPerm, rbac.CRUD, ApproveAction); err != nil {
    fmt.Errorf("can not permit crud and ApproveAction actions to role %s\n", adminRole.ID)
}
```

Now we can check if a role is granted some permission:

```go

if !R.IsGranted(adminRole.ID, usersPerm, rbac.Write) {
    fmt.Printf("admin role does not have write grant on users\n")
}else{
    fmt.Printf("admin role does have write grant on users\n")
}

// You can also check by perm.ID also
if !R.IsGrantedStr("admin", "users", rbac.CRUD) {
    fmt.Printf("admin role does not have CRUD grants on users\n")
}else{
    fmt.Printf("admin role does have CRUD grants on users\n")
}

```

## Persisting and Loading

`rbac.RBAC` is `json` compatible. You can dump all data in `RBAC` instance to JSON:

```go
b, err := json.Marshal(R)
if err != nil {
    fmt.Printf("rback marshall failed with %v\n", err)
}else{
    fmt.Printf("RBAC: %s", b)
}
```

Also you can use builtin `SaveJSON` function to save to a `io.Writer`:

```go
filename := "/tmp/rbac.json"
f, err := os.OpenFile(filename, os.O_CREATE|os.O_WRONLY, 0644)
if err != nil {
    fmt.Printf("can not create json file %s, err: %v\n", filename, err)
    return err
}
defer f.Close()
if err = R.SaveJSON(f); err != nil {
    fmt.Printf("unable to save to json file, err:%v\n", err)
}
```

And load it from file:

```go
filename := "/tmp/rbac.json"
f, err := os.Open(filename)
if err != nil {
    return err
}
defer f.Close()
if err = R.LoadJSON(f); err != nil {
    fmt.Errorf("unable to load from json file, err:%v\n", err)
}
```

In dumped JSON root `permissions` part is just for reference. Root `roles` is the part you can modify in file and reload it to define `Role`s with `Permission`s.

```json
{
  "permissions": [
    {
      "id": "users",
      "description": "User resource",
      "actions": [
        "create",
        "read",
        "update",
        "delete"
      ]
    }
  ],
  "roles": [
    {
      "id": "admin",
      "description": "Admin role",
      "grants": {
        "users": [
          "create",
          "read",
          "update",
          "delete"
        ]
      },
      "parents": []
    }
  ]
}
```

## Role inheritance

A `Role` can have parent `Role`s. You can add a parent `Role` like this:

```go
// Add a new role
sysAdmRole, err := R.RegisterRole("sysadm", "System admin role")
if err != nil {
    fmt.Printf("can not add agent role, err: %v\n", err)
}

// Now add adminRole as parent
if err = sysAdmRole.AddParent(adminRole); err != nil {
    fmt.Printf("adding parent role failed with: %v\n", err)
}

// Now all permissions in adminRole will be also valid for sysAdmRole
if R.IsGranted(sysAdmRole.ID, usersPerm, rbac.CRUD) {
    fmt.Printf("sysadmin role has all crud actions granted\n")
}
```

If circular parent reference is found, you'll get error while running `AddParent`.

## Usage as middleware

You can check example middleware function for [echo](github.com/labstack/echo) framework [here](https://github.com/euroteltr/rbac/tree/master/middlewares/echorbac/example)

## Middleware usage with granular permissions

If you want `user` role may modify his own user resource, but not others, you can build a wrapper for `RBAC.IsGranted` function like(example for `echo` framework:

```go
// idParam is the parameter name where user_id for data will be gotten
func isGrantedResource(perm *rbac.Permission, action rbac.Action, idParam string) echo.MiddlewareFunc {
    return func(next echo.HandlerFunc) echo.HandlerFunc {
        return func(c echo.Context) error {
            rolesI := c.Get("roles")
            if rolesI == nil {
                log.Errorf("No rbac roles key %s", "roles")
            } else if config.RBAC.AnyGranted(rolesI.([]string), perm, actions...) {
                // Get id parameter from route,form... example from: "user/:id"
                userID, err := strconv.ParseInt(c.Param(idParam), 10, 64)
                if err != nil {
                    return http.StatusBadRequest
                }
                // Get current user_id and check if data belongs to current user
                userIDI := c.Get("user_id")
                if userIDI==nil || userIDI.(int64) != userID {
                    return http.StatusForbidden
                }
                return next(c)
            }
            return echo.ErrUnauthorized
        }
    }
}
```
