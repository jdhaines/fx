# Parameter Objects

A parameter object is an objects with the sole purpose of carrying parameters
for a specific function or method.

The object is typically defined exclusively for that function,
and is not shared with other functions.
That is, a parameter object is not a general purpose object like "user"
but purpose-built, like "parameters for the `GetUser` function".

In Fx, parameter objects contain exported fields exclusively,
and are always tagged with `fx.In`.

**Related**

- [Result objects](result-objects.md) are the result analog of
  parameter objects.

## Using parameter objects

To use parameter objects in Fx, take the following steps:

1. Define a new struct type named after your constructor
   with a `Params` suffix.
   If the constructor is named `NewClient`, name the struct `ClientParams`.
   If the constructor is named `New`, name the struct `Params`.
   This naming isn't strictly necessary, but it's a good convention to follow.

     ```go
     --8<-- "parameter-objects/define.go:empty-1"
     --8<-- "parameter-objects/define.go:empty-2"
     ```

2. Embed `fx.In` into this struct.

     ```go
     --8<-- "parameter-objects/define.go:fxin"
     ```

3. Add this new type as a parameter to your constructor *by value*.

     ```go
     --8<-- "parameter-objects/define.go:takeparam"
     ```

4. Add dependencies of your constructor as **exported** fields on this struct.

     ```go
     --8<-- "parameter-objects/define.go:fields"
     ```

5. Consume these fields in your constructor.

     ```go
     --8<-- "parameter-objects/define.go:consume"
     ```

Once you have a parameter object on a function,
you can use it to access other advanced features of Fx:

- [Consuming value groups with parameter objects](value-groups/consume.md#with-parameter-objects)

<!--
TODO: cover various tags supported on a parameter object.
-->

## Adding new parameters

You can add new parameters for a constructor
by adding new fields to a parameter object.
For this to be backwards compatible,
the new fields must be **optional**.

1. Take an existing parameter object.

     ```go
     --8<-- "parameter-objects/extend.go:start-1"
     --8<-- "parameter-objects/extend.go:start-2"
     --8<-- "parameter-objects/extend.go:start-3"
     ```

2. Add a new field to it for your new dependency
   and **mark it optional** to keep this change backwards compatible.

     ```go
     --8<-- "parameter-objects/extend.go:full"
     ```

3. In your constructor, consume this field.
   Be sure to handle the case when this field is absent --
   it will take the zero value of its type in that case.

     ```go
     --8<-- "parameter-objects/extend.go:consume"
     ```

## Providing parameters in `fx.New` with `Supply`

To make use of parameter ojects, you need to `Supply` them to the `fx.New` function.  A toy example is below.  Note the `fx.Supply` constructor and how that is used as a parameter to `NewHTTPServer`.

```go
package main

// if using fx.Supply, we don't use the fx.In 
type ServerParams struct {
	// fx.In
	Port int
}

func main() {
	port := 3000
	fx.New(
		fx.Provide(NewHTTPServer),
		fx.Supply(ServerParams{Port: port}),
		fx.Invoke(func(server *http.Server) {
		}),
	).Run()
}

func NewHTTPServer(lc fx.Lifecycle, p ServerParams) *http.Server {
	NewServer := &Server{}

	// Declare Server config
	server := &http.Server{
		Addr:         fmt.Sprintf(":%d", p.Port),
	}
	lc.Append(fx.Hook{
		OnStart: func(ctx context.Context) error {
			ln, err := net.Listen("tcp", server.Addr)
			if err != nil {
				return err
			}
			fmt.Println("Starting HTTP server at", server.Addr)
			go server.Serve(ln)
			return nil
		},
		OnStop: func(ctx context.Context) error {
			return server.Shutdown(ctx)
		},
	})
	return server
}
