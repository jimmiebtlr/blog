## Generics based http handlers in go - Part 1:  Why

Handlers in go have suffered from a large amount of boilerplate. Often something like this is written for each handler.
```go
func(w http.ResponseWriter, r *http.Request) {
  userID := middleware.UserIDFromContext(r.Context())

  data := Widget{}
  err := DecodeAndValidate(r, &data)
  if err != nil {
      http.Error(w, err.Error(), http.StatusBadRequest)
      return
  }

  // Do something with the data
}
```

DecodeAndValidate must come after we declare `data := Widget{}` to get us a type safe instance of widget.  This makes it difficult to put this logic that is shared across most handlers in a common middleware or similar component.

Generics can help here.  We can write our handlers like this, with our common unmarshal/validate code outside our main handler.  This gives us a much cleaner, and in my opinion potentially safer handler since it's less code to introduce bugs and maintain.
```go
func(w http.ResponseWriter, data *Widget) {
    // Do our widget work, confident things have already been validated and unmarshaled
}
```