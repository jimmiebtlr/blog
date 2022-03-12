## Go - Generics powered http handlers

## The Goal

Our resulting handler should be: 
1. Type safe
2. Easy to understand
3. Support existing middleware and adhere to `http.HandlerFunc`
4. Be easy to use

Short version: Check out the [playground](https://gotipplay.golang.org/p/sUqxE-CTf3E)

## What we'll build

At the end of this post we'll be able to do this
```golang
func widgetPutHandler(req Request, route *WidgetPutRoute, widget *WidgetPut) {
	req.Render(http.StatusOK, widget)
}

func main() {
  handle := Handler(widgetPutHandler)

  r := httptest.NewRequest(http.MethodPut, "", strings.NewReader(`{"name":"plumbus"}`))
  w := httptest.NewRecorder()
  handle(w, r)
}
```

## Generics based handler

In most handlers we'll need to do some variety:
1. Parsing the data/route provided into a type safe form
2. Validating that user provided data
3. Instantiating per request versions of some objects (such as logging)

With that in mind we'll create a generic handler that will handle those tasks.
```golang
func Handler[RouteType any, BodyType any](h func(req Request, route *RouteType, data *BodyType)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
        // Provide a logger scoped to the request 
		log := NewLogger(r)

        // Define body type to unmarshal
		var bodyData BodyType
		err := json.NewDecoder(r.Body).Decode(&bodyData)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

        // validate the user provided body data
		err = validate.Struct(bodyData)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

        // Read route param and queryParam data into the provided route struct
		var routeData RouteType
		err = UnmarshalRouteData(r, &routeData)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

        // Validate route data makes sense.
		err = validate.Struct(routeData)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

        // Create object to store some of request specific objects 
		req := Request{r.Context(), log}

        // Call the request specific, type safe handler.
		h(req, &routeData, &bodyData)
	}
}
```

## Route specific handler

We now have our generic handler wrapper to reduce the amount of boilerplate in our handlers. We'll now set up our route specific handler function and wrap it in our generic handler.

```go
func GetWidgetHandler(s Store) func(req Request, route *WidgetPutRoute, data *WidgetPut) {
	return func(req Request, route *WidgetPutRoute, data *WidgetPut) {
		req.Log(fmt.Sprintf("%v", data))
		
		// Do whatever with data and route

		req.Render(http.StatusOK, "hello world")
	}
}
```

We pass in the database in this way since it doesn't really benefit from being part of our generic handler, but there are many other ways to handle it.

We've greatly reduced the boilerplate required for each handler, while at the same time making it much easier to test our handler.

## Full example

```golang
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"net/http"
	"net/http/httptest"
	"strings"

	"github.com/go-playground/validator/v10"
)

//
// Example Use
//
type WidgetPut struct {
	ID   string `json:"id"`
	Name string `json:"name"`
}

type WidgetPutRoute struct {
	RouteParam int    `routeParam:"someRP"`
	SomeQP     string `queryParam:"someQP"`
}

func GetWidgetHandler(s Store) func(req Request, route *WidgetPutRoute, data *WidgetPut) {
	return func(req Request, route *WidgetPutRoute, data *WidgetPut) {
		req.Log(fmt.Sprintf("%v", data))
		
		// Do whatever with data and route

		req.Render(http.StatusOK, "hello world")
	}
}


func main() {
	h := Handler(GetWidgetHandler(Store{}))

	r := httptest.NewRequest(http.MethodGet, "http://example.com", strings.NewReader(`{"id":"123"}`))
	w := httptest.NewRecorder()
	h(w, r)
}

//
// Generics based Handler implementation
//
func Handler[RouteType any, BodyType any](h func(req Request, route *RouteType, data *BodyType)) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		log := NewLogger(r)
		responder := Responder{Request: r, Writer: w}

		var bodyData BodyType
		err := json.NewDecoder(r.Body).Decode(&bodyData)
		if err != nil {
			http.Error(w, err.Error(), http.StatusBadRequest)
			return
		}

		err = validate.Struct(bodyData)
		if err != nil {
			// Respond bad request
			return
		}

		var routeData RouteType
		err = UnmarshalRouteData(r, &routeData)
		if err != nil {
			// Respond bad request
			return
		}

		err = validate.Struct(routeData)
		if err != nil {
			// Respond bad request
			return
		}

		req := Request{
			r.Context(),
			log,
			responder,
		}

		h(req, &routeData, &bodyData)

	}
}

//
// Below is mostly placeholders for common stuff such as logger with requestId scoping, and
// response format helper
//
var validate = validator.New()

// Logger would generally read trace data and attach that to logs written though this.
type Logger struct{}

// NewLogger would attach any request id or similar request level data to the logger.
func NewLogger(r *http.Request) Logger {
	return Logger{}
}
func (l *Logger) Log(s string) {
	fmt.Println(s)
}

// UnmarshalRoute parses out route params and query params and puts them in d.
func UnmarshalRouteData(r *http.Request, d interface{}) error {
	return nil
}

type Request struct {
	context.Context
	Logger

	Responder
}

type Responder struct {
	Request *http.Request
	Writer  http.ResponseWriter
}

func (r Responder) Render(statusCode int, obj interface{}) {
	// Can handle any logic around choosing content types from request here
	// r.Writer.Header().Set("Content-Type", "application/json")
	json.NewEncoder(r.Writer).Encode(obj)
	r.Writer.WriteHeader(statusCode)
}

func (r Responder) RenderError(statusCode int, err error) {
	type Error struct {
		Error string `json:"error"`
	}
	r.Render(statusCode, Error{err.Error()})
}

type Store struct{}

func (s Store) GetWidget(ctx context.Context, id string) (string, error) {
        if id == "123" {
        	return "Plumbus", nil
        }

	return "", fmt.Errorf("Error widget not found")
}

```

[Playground](https://gotipplay.golang.org/p/sUqxE-CTf3E)

Bonus: This also makes testing your handler much easier.