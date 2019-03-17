---
title: "How I like to Structure and Build Go Services"
description: "A nice way I've found to structure and build my Go services."
date: "2019-03-16T20:00:00Z"
---

I've worked with Go on and off for a little while now and have a handful of projects in production. I thought I'd use this as an opportunity to write down what I've found works quite well and what I've found is simple enough for any developer to pick up.

My projects tend to follow the [Standard Go Project Layout](https://github.com/golang-standards/project-layout). But with a few modifications, like removing the extra nested directories under the `internal/` directory (`internal/app/` and `internal/pkg/`). I find that just putting my packages under `internal/` works fine. But other than that my projects follow the standard layout pretty close.

```
app/
└── internal/
    ├── someservice/
    └── auth/
```

All packages under internal are grouped by commonality. I prefer keeping code that's related close together and I'm not a huge fan of splitting code by its function e.g. all handlers under a handlers package. So, for example, if my service requires a user to login or register, I will group that functionality under an auth package.

Within each package under `internal/` I usually have 3 main files:

- **handlers.go**, this contains all HTTP handlers needed for this package as well as handles anything specific to the request and response, any other file within this package doesn't have knowledge of where the data comes from or goes to.
- **store.go**, this is essentially a repository to access some sort of datastore. I use the store as a wrapper around a third-party database provider like the standard libraries `database/sql` package. This makes it easy to mock out in any package that requires database access.
- **service.go**, which contains most of the business logic for the package. For example, if we had an auth package, and part of that package was for registering a user, service.go would have a method that could validate that the user doesn't already exist in the database through the provided store. It could then hash the inputted password using another injected service, then it could pass the final user details to another store method for adding a user, and finally, it could trigger an email to notify that user.

```
app/
└── internal/
    ├── someservice/
    └── auth/
        ├── handlers.go
        ├── handlers_test.go
        ├── store.go
        ├── store_test.go
        ├── service.go
        └── service_test.go
```

I'm not strict on the file names, I group related functionality under each file and name them appropriately. The above are just the common filenames I use for common scenarios. In reality, as long as the names mean something and are consistent, it doesn't matter.

Each component under the auth package usually has an associated interface, this allows us to inject those components into where they need to go which makes testing a whole lot easier.

So **store.go** might look like:

```go
package auth

type Store interface {
  CreateUser(name, password string) User
}

type store struct {}

func NewStore() Store {

  return &store{}
}

func (s *store) CreateUser(name, password string) User {

  return User{}
}
```

Then in service.go, we can easily inject store:

```go
package auth

type Service interface {
  Register(name, password string) User
}

type service struct {
  store Store
}

func NewService(store Store) Service {

  return &service{ store }
}

func (s *service) Register(name, password string) User {

  // Here we could bcrypt the password, or do anything
  // needed before the creation of the user
  // ...
  return s.store.CreateUser(name, password)
}
```

And in handler.go we can finally use everything together:

```go
package auth

func Handler(svc Service) http.HandlerFunc {

  return func(w http.ResponseWriter, r *http.Request) {

    svc.Register(r.FormValue("name"), r.FormValue("password"))
    w.WriteHeader(http.StatusOK)
  }
}
```

When it comes to testing we can provide mock instances of each injected component where needed.

Everything is then glued together in **main.go**, which is located under `cmd/<appname>/main.go`. I tend to find that **main.go** can get quite large, but I prefer seeing everything being initialised and passed to it's required destination in one place.

With the above examples, initialising the auth package might look like:

```go
package main

import (
  "log"

  "github.com/gorilla/mux"
  ...
)

func main() {

  // Create router
  router := mux.NewRouter()

  // Create postgres client for our store
  postgres := postgres.New()

  // Auth handlers
  authStore := auth.NewStore(postgres)
  authService := auth.NewService(authStore)

  // Routes
  router.Handle("/user", auth.Handler(authService)).Methods("GET")

  if err := http.ListenAndServe(":8000", router); err != nil {
    log.Fatal("Error starting HTTP server")
  }
}
```

One problem I've noticed with this setup is circular dependency issues that crop up every so often. Which can be a pain. I've found that this is usually down to sharing domain types between packages. An easy solution to this has been to extract any shared types into their own "root package" which doesn't import any other package.
