GO GRPC STUDIES.




go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

protoc --go_out=. --go_opt=paths=source_relative
--go-grpc_out=. --go-grpc_opt=paths=source_relative
routeguide/route_guide.proto


protoc --go_out=. --go_opt=paths=source_relative
--go-grpc_out=. --go-grpc_opt=paths=source_relative
proto/blog.proto

go run server/server.go
go run client/client.go
1.A simple RPC where the client sends a request to the server using the stub and waits for a response to come back, just like a normal function call.

// Obtains the feature at a given position.
rpc GetFeature(Point) returns (Feature) {}

2.A server-side streaming RPC where the client sends a request to the server and gets a stream to read a sequence of messages back. The client reads from the returned stream until there are no more messages. As you can see in our example, you specify a server-side streaming method by placing the stream keyword before the response type.

// Obtains the Features available within the given Rectangle.  Results are
// streamed rather than returned at once (e.g. in a response message with a
// repeated field), as the rectangle may cover a large area and contain a
// huge number of features.
rpc ListFeatures(Rectangle) returns (stream Feature) {}

3.A client-side streaming RPC where the client writes a sequence of messages and sends them to the server, again using a provided stream. Once the client has finished writing the messages, it waits for the server to read them all and return its response. You specify a client-side streaming method by placing the stream keyword before the request type.

// Accepts a stream of Points on a route being traversed, returning a
// RouteSummary when traversal is completed.
rpc RecordRoute(stream Point) returns (RouteSummary) {}

4.A bidirectional streaming RPC where both sides send a sequence of messages using a read-write stream. The two streams operate independently, so clients and servers can read and write in whatever order they like: for example, the server could wait to receive all the client messages before writing its responses, or it could alternately read a message then write a message, or some other combination of reads and writes. The order of messages in each stream is preserved. You specify this type of method by placing the stream keyword before both the request and the response.

// Accepts a stream of RouteNotes sent while a route is being traversed,
// while receiving other RouteNotes (e.g. from other users).
rpc RouteChat(stream RouteNote) returns (stream RouteNote) {}

Fortunately there is a tool to help us generate these `protoc` commands for us: [https://github.com/stevvooe/protobuild/](https://github.com/stevvooe/protobuild/)

**Protobuild** walks through your repository and finds *.proto* files to generate. Instead of scripting this ourselves, we are going to create a single *Protobuild.toml* file at the root of the project and call the `protobuild`command line tool.

```
????????? api
???   ????????? api.pb.txt
???   ????????? todo
???   ???   ????????? v1
???   ???       ????????? doc.go
???   ???       ????????? todo.proto
???   ????????? version
???       ????????? v1
???           ????????? doc.go
???           ????????? version.proto
[...]
```

The [*Protobuild.toml*](https://github.com/abronan/todo-grpc/blob/master/Protobuild.toml)* *file defines the common imports necessary for protoc as well as the mappings for *gogoproto*. Whenever something changes in the structure of the protobuf files or dependencies, we could edit this single file rather than editing countless `protoc` statements in a makefile or a script. Very convenient!

## Injecting tags into protobuf generated structs

**go-pg** is smart enough to detect the fields of a Golang struct and map it to a database table for a very simple structure. Although when dealing with specific properties or table joins, we have to use *struct tags* in order to let go-pg know about our specific use case.

The issue there is that gRPC???s `protoc`command generates our data structures in a file that we have no control upon. We???d like to avoid maintaining a mapping between a **Todo** and an **InternalTodo** just to add these *go-pg* struct tags.

Fortunately, we have access to a project that achieves this task: [https://github.com/favadi/protoc-go-inject-tag](http://proton-go-inject-tag/)

With **protoc-go-inject-tag**, and as the name suggests, we???ll inject the ORM tags onto the fields of the structs generated by protobuf. This is convenient because we could now use the protobuf generated structs directly into the query statements and avoid having to maintain a mapping between two structs each time we manipulate a `Todo` item.

In the **todo.proto** file we???re going to add the following comments on top of the **google.protobuf.Timestamp** fields:

Inject pg ORM tags for protobuf generated structsThis tells **pg** to put the *completed* field to the default value *false*.

Additionally we also instruct **pg** to treat the *protobuf.Timestamp* fields as a *timestamptz* with PostgresQL (rather than the default *timestamp* type).

For the field *created_at*, we add *default:now()* to the tag in order to initialize *created_at* when a new entry is inserted into the database.

> **There is a catch though**: in order to seamlessly support *gRPC timestamps*, we have to extend `<em class="hi">go-pg</em>` (see: [https://github.com/go-pg/pg/compare/master...abronan:grpc_timestamp_support](https://github.com/go-pg/pg/compare/master...abronan:grpc_timestamp_support)).

Now, what if we want a field appearing in the protobuf file but ignored by the ORM? Quite simple:

Discard a column with the go-pg ORMIn the same vein, what if we want a field to be scanned through go-pg but never returned to the client using gRPC? We could just override the main **Todo **with additional fields:

Additional column scanned with pg ORM but ignored in protobuf structsWe could support joins, a tree-like structure and references to parent todos (using `Recursive`queries or the `ltree`module), using postgresql arrays ( `hstore`) and JSON types, etc. We can do pretty much anything supported by **go-pg**.

If your schema is too complex or performance critical for these tricks to be applied, then you could just use plain **libpq** of **go-sqlx/pgx**([https://github.com/JackC/pgx](https://github.com/JackC/pgx)). **go-pg** also allows you to write plain queries if you feel restricted somehow.

## Generate HTTP REST Reverse Proxy

Now that we have a service that could serve clients through gRPC, we also want to add **RESTful API** support with minimal effort.

A common way to do this would be the HTTP transport logic and redirect to the GRPC methods back and forth. Again there is an existing project doing just that:

[https://github.com/grpc-ecosystem/grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)

grpc-gateway create a reverse-proxy from gRPC to JSON with minimal effort, using the **google.http.api** extension in our protobuf files.

We only need to add the routes to the Service methods defined in `todo.proto`and add the protoc generation command to the makefile. grpc-gateway will automatically generate all the json unmarshalling code and build the expected protobuf message before calling the right service method. Very convenient!

> **Bonus**: It also generates swagger files for your API!

## Middleware: tracing and monitoring

Monitoring and tracing are critical aspects of modern software development. This could help you gather insights about why a given method runs poorly in terms of response time or the current status of your components in a micro-service architecture.

Again, instead of creating the middleware part ourselves for each and every method of our service, we are going to use an existing library which does exactly that:

[https://github.com/grpc-ecosystem/go-grpc-middleware](https://github.com/grpc-ecosystem/go-grpc-middleware)

go-grpc-middleware uses **GRPC interceptors** to achieve this task and eases the integration with tools such as *Jaeger* or *Prometheus. *Setting up the interceptors is as simple as a few lines of code:

Setting up interceptors for tracing, monitoring, logging, etc.The entrypoint for our program is as simple as it could get: we setup our gRPC server with the appropriate interceptors and extensions and simply start it.

> To get an overview of the entrypoint for the program, look at the [main.go](https://github.com/abronan/todo-grpc/blob/master/main.go) file.

## Generating a client/SDK

Additionally and in complement to the use of a RESTful API, we would like our application to interact with clients developed in various languages. Thus we need an **SDK** available outside of the context of our backend application.

The code of such an SDK is almost always trivial to implement, with transport, connection logic, error management and finally RPC calls/handling of responses.

We could as well generate this part and avoid error prone operations when adding a new service method.

[https://github.com/moul/protoc-gen-gotemplate](https://github.com/moul/protoc-gen-gotemplate) allows us to walk through the AST of the gRPC services, methods and fields in order to generate the SDK accordingly. Thus every time we add a new call to our service, the template generation pipeline could be ran and automatically generate the SDK to match the new changes.

This is a very nice and advanced trick (that I???m not showing in the example repository but you could look at examples [there](https://github.com/moul/protoc-gen-gotemplate/tree/master/examples)), especially if your services contain a non negligible amount of calls, all requiring to update the SDK or other consumer libraries. The SDK could then be used to conveniently implement integration tests as a complement to local unit tests.

# Conclusion

In this post, we walked through quite a few tricks for dealing with projects using Golang and gRPC. We demonstrated that we could deliver a complete and useful API in a few lines of code while still maintaining an extended feature set with the support for monitoring, tracing, database backend, RESTful API, etc.

gRPC is full of libraries and utilities that now makes it a breeze to create and manage an application at scale with Go such as:** protobuild**, **grpc-gateway**, **go-grpc-middleware **or **protoc-gen-gotemplate**.

I invite you to read the source code carefully and apply some of these principles and tools to your application to keep complexity under control.

Obviously, this is one example amongst many. Pick up the right tool for the task but always reconsider and question the current state of your application in order to simplify and remove what is unnecessary.
