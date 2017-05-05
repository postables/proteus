# ![proteus](https://rawgit.com/src-d/proteus/master/proteus.svg)

[![GoDoc](https://godoc.org/github.com/src-d/proteus?status.svg)](https://godoc.org/github.com/src-d/proteus) [![Build Status](https://travis-ci.org/src-d/proteus.svg?branch=master)](https://travis-ci.org/src-d/proteus) [![codecov](https://codecov.io/gh/src-d/proteus/branch/master/graph/badge.svg)](https://codecov.io/gh/src-d/proteus) [![License](http://img.shields.io/:license-mit-blue.svg)](http://doge.mit-license.org) [![Go Report Card](https://goreportcard.com/badge/github.com/src-d/proteus)](https://goreportcard.com/report/github.com/src-d/proteus) [![codebeat badge](https://codebeat.co/badges/976ff535-c79b-429d-b35c-888a048a3201)](https://codebeat.co/projects/github-com-src-d-proteus)

[Proteus](https://en.wikipedia.org/wiki/Proteus) /proʊtiəs/ is a tool to generate protocol buffers version 3 compatible `.proto` files from your Go structs, types and functions.

The motivation behind this library is to use Go as a source of truth for your models instead of the other way around and then generating Go code from a `.proto` file, which does not generate idiomatic code.

Proteus scans all the code in the selected packages and generates protobuf messages for every exported struct (and all the ones that are referenced in any other struct, even though they are not exported). The types that semantically are used as enumerations in Go are transformed into proper protobuf enumerations.
All the exported functions and methods will be turned into protobuf RPC services.

We want to build proteus in a very extensible way, so every step of the generation can be hackable via plugins and everyone can adapt proteus to their needs without actually having to integrate functionality that does not play well with the core library. We are releasing the plugin feature after Go 1.8 is released, which includes the `plugin` package of the standard library.

For an overall overview of the code architecture take a look at [the architecture documentation](/ARCHITECTURE.md).

You can read more about the motivations behind building proteus in [this blog post](https://blog.sourced.tech/post/proteus/).

### Install

```
go get -v gopkg.in/src-d/proteus.v1/...
```

### Requirements

There are two requirements for the full process.

* [`protoc`](https://github.com/google/protobuf) binary installed on your path
* `go get github.com/gogo/protobuf/...`

### Usage

You can generate the proto files, the marshal/unmarshal and the rest of protobuf stuff for your Go types, the RPC client and server interface and the RPC server implementation for your packages. That is, the whole process.

```bash
proteus -f /path/to/protos/folder \
        -p my/go/package \
        -p my/other/go/package
```

You can generate proto files only using the command line tool provided with proteus.

```bash
proteus proto -f /path/to/output/folder \
        -p my/go/package \
        -p my/other/go/package
        --verbose
```

You can also only generate gRPC server implementations for your packages.

```bash
proteus rpc -p my/go/package \
        -p my/other/go/package
```

**NOTE:** Of course, if the defaults don't suit your needs, until proteus is extensible via plugins, you can hack together your own generator command using the provided components. Check out the [godoc documentation of the package](http://godoc.org/github.com/src-d/proteus).

### Generate protobuf messages

Proteus will generate protobuf messages with the structure of structs with the comment `//proteus:generate`. Obviously, the structs have to be exported in Go (first letter must be in upper case).

```go
//proteus:generate
type Exported struct {
        Field string
}

type NotExported struct {
        Field string
}
```

**Generated by requirement**

Note that, even if the struct is not explicitly exported, if another struct has a field of that type, it will be generated as well.

```go
//proteus:generate
type Preference struct {
        Name string
        Value string
        Options Options
}

type Options struct {
        Enabled bool
}
```

In that example, even if `Options` is not explicitly generated, it will be because it is required to generate `Preference`.

So far, this does not happen if the field is an enum. It is a known problem and we are working on fixing it. Until said fix lands, please, explicitly mark enums to be generated.

**Struct embedding**

You can embed structs as usual and they will be generated as if the struct had the fields of the embedded struct.

```go
//proteus:generate
type User struct {
        Model
        Username string
}

type Model struct {
        ID int
        CreatedAt time.Time
}
```

This example will generate the following protobuf message.

```
message User {
        int32 id = 1;
        google.protobuf.Timestamp created_at = 2;
        string username = 3;
}
```

**CAUTION:** if you redefine a field, it will be ignored and the one from the embedded will be taken. Same thing happens if you embed several structs and they have repeated fields. This may change in the future, for now this is the intended behaviour and a warning is printed.

**Ignore specific fields**

You can ignore specific fields using the struct tag `proteus:"-"`.

```go
//proteus:generate
type Foo struct {
        Bar int
        Baz int `proteus:"-"`
}
```

This becomes:

```
message Foo {
        int32 bar = 1;
}
```

### Generating enumerations

You can make a type declaration (not a struct type declaration) be exported as an enumeration, instead of just an alias with the comment `//proteus:generate`.

```go
//proteus:generate
type Status int

const (
        Pending Status = iota
        Active
        Closed
)
```

This will generate:

```
enum Status {
        PENDING = 0;
        ACTIVE = 1;
        CLOSED = 2;
}
```

**NOTE:** protobuf enumerations require you to start the enumeration with 0 and do not have gaps between the numbers. So keep that in mind when setting the values of your consts.

For example, if you have the following code:

```go
type PageSize int

const (
        Mobile PageSize = 320
        Tablet PageSize = 768
        Desktop PageSize = 1024
)
```

Instead of doing an enumeration, consider not exporting the type and instead it will be treated as an alias of `int32` in protobuf, which is the default behaviour for not exported types.

### Generate services

For every package, a single service is generated with all the methods or functions having `//proteus:generate`.

For example, if you have the following package:

```go
package users

//proteus:generate
func GetUser(id uint64) (*User, error) {
        // impl
}

//proteus:generate
func (s *UserStore) UpdateUser(u *User) error {
        // impl
}
```

The following protobuf service would be generated:

```proto
message GetUserRequest {
        uint64 arg1 = 1;
}

message UserStore_UpdateUserResponse {
}

service UsersService {
        rpc GetUser(users.GetUserRequest) returns (users.User);
        rpc UserStore_UpdateUser(users.User) returns (users.UserStore_UpdateUserResponse);
}
```

Note that protobuf does not support input or output types that are not messages or empty input/output, so instead of returning nothing in `UserStore_UpdateUser` it returns a message with no fields, and instead of receiving an integer in `GetUser`, receives a message with only one integer field.
The last `error` type is ignored.

### Generate RPC server implementation

`gogo/protobuf` generates the interface you need to implement based on your `.proto` file. The problem with that is that you actually have to implement that and maintain it. Instead, you can just generate it automatically with proteus.

Consider the Go code of the previous section, we could generate the implementation of that service.

Something like this would be generated:

```
type usersServiceServer struct {
}

func NewUsersServiceServer() *usersServiceServer {
        return &usersServiceServer{}
}

func (s *userServiceServer) GetUser(ctx context.Context, in *GetUserRequest) (result *User, err error) {
        result = GetUser(in.Arg1)
        return
}

func (s *userServiceServer) UserStore_UpdateUser(ctx context.Context, in *User) (result *UserStore_UpdateUser, err error) {
        s.UserStore.UpdateUser(in)
        return
}
```

There are 3 interesting things in the generated code that, of course, would not work:
- `usersServiceServer` is a generated empty struct.
- `NewUsersServiceServer` is a generated constructor for `usersServiceServer`.
- `UserStore_UpdateUser` uses the field `UserStore` of `userServiceServer` that, indeed, does not exist.

The server struct and its constructor are always generated empty **but only if they don't exist already**. That means that you can, and should, implement them yourself to make this code work.

For every method you are using, you are supposed to implement a receiver in the server type and initialize it however you want in the constructor. How would we fix this?

```go
type userServiceServer struct {
        UserStore *UserStore
}

func NewUserServiceServer() *userServiceServer {
        return &userServiceServer{
                UserStore: NewUserStore(),
        }
}
```

Now if we generate the code again, the server struct and the constructor are implemented and the defaults will not be added again. Also, `UserStore_UpdateUser` would be able to find the field `UserStore` in `userServiceServer` and the code would work.

### Not scanned types

What happens if you have a type in your struct that is not in the list of scanned packages? It is completely ignored. The only exception to this are `time.Time` and `time.Duration`, which are allowed by default even though you are not adding `time` package to the list.

In the future, this will be extensible via plugins.

### Examples

You can find an example of a *real* use case on the [example](xample) folder.
For checking how the server and client works, see
[server.go](example/server/server.go) and
[client.go](example/client/client.go). The orchestration of server and client
is done in [example_test.go](example_test.go). The example creates a server and
a new client and tests the output.


### Features to come

- Extensible mapping and options via plugins (waiting for Go 1.8 release).

### Known Limitations

The following is a list of known limitations and the current state of them.

* Only `{,u}int32` and `{,u}int64` is supported in protobuf. Therefore, all
  `{,u}int` types are upgraded to the next size and a warning is printed. Same
  thing happens with `float`s and `double`s.
* If a struct contains a field of type `time.Time`, then that struct can only
  be serialized and deserialized using the `Marshal` and `Unmarshal` methods.
  Other marshallers use reflection and need a few struct tags generated by
  protobuf that your struct won't have. This also happens with fields whose
  type is a declaration to a slice of another type (`type Alias []base`).

### Contribute

If you are interested on contributing to **proteus**, open an [issue](https://github.com/src-d/proteus/issues) explaining which missing functionality you want to work in, and we will guide you through the implementation, and tell you beforehand if that is a functionality we might consider merging in the first place.

### License

MIT, see [LICENSE](/LICENSE)

