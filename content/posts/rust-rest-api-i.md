+++
title = "A modern REST web service, written in Rust, Part I"
date = 2022-03-06 23:51:21
updated = 2023-06-10 00:26:10
authors = ["d34db4b3"]
description =  "We will dive into web back-end design with a simple REST API, using the reliable and efficient programming language Rust"
[taxonomies]
tags = ["rust", "web", "backend", "REST"]
+++

# TLDR

> IN PROGRESS

# Introduction

The idea of this project is to implement a simple REST API using a number of common features. It will be split in several parts in order to focus on each aspect.

Among them, we will first see the setting up of the web server, the routes registration and the management of requests and responses. We will then see the various challenges of proper logging, access restriction, internationalization and database management.

We will use the [Rust](https://www.rust-lang.org/) language, the [axum](https://github.com/tokio-rs/axum) framework to manage most of the HTTP server functionalities of our API and various other libraries (crates) detailed later. The particularities and the syntax of this language will not be discussed here.

Provided commands will assume a linux host.

Rust version `1.70.0` will be used and it is assumed that the build tools are installed beforehand ([Rust installation](https://www.rust-lang.org/tools/install)).

[Visual Studio Code](https://code.visualstudio.com/) will be used as an IDE throughout the project.

In order to debug and test our API, the [HOPPSCOTCH](https://hoppscotch.io/) software can be used. The collection files will be provided for easier testing.

For beginners in Rust, a tutorial will be available soon on this blog. In the meantime, it is recommended to read the excellent [Rust book](https://doc.rust-lang.org/book/).

The particularities of serialization/deserialization via [serde](https://serde.rs/) will not be studied in detail in this article.

During the course of the article, snapshots of the example project will be available and will consist of branches from the [d34db4b3/rest-api-tutorial](https://github.com/d34db4b3/rest-api-tutorial) repository.

# Project initialization

## Project creation

As for any Rust project, we will use the [cargo](https://doc.rust-lang.org/cargo/index.html) utility to initialize the sources. Once placed in the folder where the project will be created, the following command will generate the `rest-api-tutorial` project.

```sh
cargo new rest-api-tutorial
```

The newly created folder can then be opened in Visual Studio Code.

The two files we are interested in are `Cargo.toml` and `src/main.rs` which are respectively the [application manifest](https://doc.rust-lang.org/cargo/reference/manifest.html) and the entry point of the program.


```toml
# Cargo.toml
[package]
name = "rest-api-tutorial"
version = "0.1.0"
edition = "2021"

[dependencies]
```

```rs
// src/main.rs
fn main() {
    println!("Hello, world!");
}
```

It is now possible to compile and run the program with the following command from the project directory.

```sh
cargo run
```

A beautiful "Hello, world!" is presented to us, but it is not really the subject of this article!

## Adding crates

We will need a few dependencies, notably [`axum`](https://docs.rs/axum/0.6/axum) and [`tokio`](https://docs.rs/tokio/1/tokio). Axum will provide most of the features required for an HTTP server (routing, HTTP protocol implementation, middlewares, etc.). As it takes advantages of the [`async`](https://rust-lang.github.io/async-book/) Rust features, we will also need `tokio`.

Let's add those in the `Cargo.toml` manifest file.
To do this, insert the lines `axum = "0.6"` and `tokio = { version = "1", features = ["full"] }` to the category `[dependencies]`. The file then looks like

```toml
# Cargo.toml
[package]
name = "rest-api-tutorial"
version = "0.1.0"
edition = "2021"

[dependencies]
axum = "0.6"
tokio = { version = "1", features = ["full"] }
```

The "0.6" represents the version of `axum` to use. For more details on dependencies and the manifest file, see [this page](https://doc.rust-lang.org/cargo/reference/manifest.html).

Dependencies can also (and probably should for simplicity sake) be added using

```sh
cargo add axum@0.6
cargo add tokio@1 -F full
```

## Test program

In order to check the correct configuration of the manifest and to start our adventure with `axum`, the following program can be injected into the `src/main.rs` file instead of the starting "helloworld". Our file looks like this 

```rs
use axum::{routing::get, Router};
use std::net::SocketAddr;

#[tokio::main]
async fn main() {
    // register a "helloworld" handler to the `/` route
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("listening on http://{}", addr);
    // start the HTTP server and serve our newly created service
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

This program resembles the structure of the previous program with its `main` function, but there are some new keywords such as `async` and `await`. These describe the asynchronous operation of `axum` and we will come back to these details later. For now we will focus on the behavior of the application.

### Application

In order for the server to do its job, we need to describe how it should work. This is the role of the `Router`, on which we will define routes, services and what it will need to do its job. Here, we ask it to register a `/` route (`.route("/", ...)`) on which a function will be grafted that will respond to the HTTP `GET` method (`get(...)`). This function described by a lambda (`|| async { "Hello, World!" }`) will simply return the string `"Hello, world!"`. A lambda is an unnamed function, used here to limit the size of the code and the amount of information present for the sake of clarity.

### Http server

Then, we will need to serve our app with an HTTP server. An `axum::Server` is bound to the network interface and will listen on the TCP port `3000` (`::bind(&addr)`). We provide the `app` for the server to serve. The `.await` will finally let `tokio` "manage" the lifecycle of our app.

## Running the program

To facilitate development, the `cargo-watch` utility will be used throughout this article. This utility allows an automatic recompilation at each change of the source code. To install it, just do

```sh
cargo install cargo-watch
```

Then the compilation process starts via

```sh
cargo watch -x run
```

This will re-run the `cargo run` command every time the source code changes.

### Checking the behavior

The expected operation of the program is an HTTP server listening on port `3000` and responding on route `/` with the message `"Hello, world!"`. Using a browser such as Firefox, a request to the <http://127.0.0.1:3000/> page should display the message

```
Hello, world!
```

[Source code snapshot](https://github.com/d34db4b3/rest-api-tutorial/tree/snapshot/init)

# Analysis of a route

## Route management function

Each route is associated with a function to process the request and respond appropriately. These functions called [handlers](https://docs.rs/axum/0.6/axum/handler/index.html) are simply asynchronous functions that are given extractors as arguments (see [`axum::extract`](https://docs.rs/axum/0.6/axum/extract/index.html)) and return an object that must be convertible into a response (see [`axum::response`](https://docs.rs/axum/0.6/axum/response/index.html)).

### Request analysis

Most of the request data is obtained via "extractors" (a type implementing the `FromRequest` or `FromRequestParts` trait). Several extractors are provided by `axum` in order to ease the implementation, but it is possible to add more as needed (which we will see with access restrictions in a next part). Among them we can mention [`Path`](https://docs.rs/axum/0.6/axum/extract/struct.Path.html) allowing the extraction of parameters in the path of the resource, [`Query`](https://docs.rs/axum/0.6/axum/extract/struct.Query.html) which in the same way allows the extraction of the parameters of the request ([more information on URLs](https://developer.mozilla.org/fr/docs/Learn/Common_questions/What_is_a_URL)), [`Json`](https://docs.rs/axum/0.6/axum/struct.Json.html) which allows the deserialization of the request body from the [JSON](https://www.json.org/json-fr.html) format and finally [`Form`](https://docs.rs/axum/0.6/axum/struct.Form.html) which has a similar behavior from the classic URL-encoded form format.

These extractors are given as parameters to our handlers and allow us to obtain data extraction with static and safe typing. The deserialization errors can thus be properly managed and it will be possible to have some guarantee over the presence and correctness of parameters before the processing of the request. This allows the code to be more robust, clear and concise.

### Response analysis

In order to respond to requests, functions associated with routes must return an object that implements [`IntoResponse`](https://docs.rs/axum/0.6/axum/response/trait.IntoResponse.html).

`axum` provides for example the [`Json`](https://docs.rs/axum/0.6/axum/struct.Json.html) type which allows to transform any serializable object into a response.

## Example

### Implementation

In order to illustrate the different elements mentioned earlier, we will imagine a `/greet` route which via the `GET` method accepts as request parameters a name `name` and will return a JSON object containing a `greeting` message which will contain the formatted string `"Hello, {}!"` with the `name` present as parameter.

We will therefore first define our parameters and response types, implementing respectively deserialization and serialization.

We need te bring the [`serde`](https://serde.rs/) framework.

```sh
cargo add serde@1.0 -F derive
```

The code looks like this

```rs
#[derive(Deserialize)]
struct GreetQuery {
    name: String,
}

#[derive(Serialize)]
struct GreetResponse {
    greeting: String,
}
```

The `GreetQuery` type describes an object with a `name` field of type `String`. The `GreetResponse` type describes an object with a `greeting` field of type `String`.

We use the `Query` extractor on the `GreetQuery` type and the `Json` wrapper on the `GreetResponse` type. The handler thus looks like 

```rs
async fn greet(greet_query: Query<GreetQuery>) -> Json<GreetResponse> {
    Json(GreetResponse {
        greeting: format!("Hello, {}!", greet_query.name),
    })
}
```

Then we just need to register the service on our application and our `main` function now looks like

```rs
#[tokio::main]
async fn main() {
    // register our "greeting" handler to the `/greet` route
    let app = Router::new().route("/greet", get(greet));

    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    println!("listening on http://{}", addr);
    // start the HTTP server and serve our newly created service
    axum::Server::bind(&addr)
        .serve(app.into_make_service())
        .await
        .unwrap();
}
```

### Service test

The `/greet` route should therefore respond with a nice message in the form of a JSON document.


#### Valid test

A query on the <http://127.0.0.1:3000/greet?name=John%20Doe> resource should show us the following response:

```json
{
  "greeting": "Hello, John Doe!"
}
```

#### Invalid test

If we try a query without the `name` parameter <http://127.0.0.1:3000/greet>, we instead see the following error message:

```
Failed to deserialize query string: missing field `name`
```

We see that the request is not process because the required parameters are not present and get a default error. We will later see how we can customize this behavior (to get a JSON error far example)

#### Optional field

It would also have been possible to make the name optional by using the :

```rs
#[derive(Deserialize)]
struct GreetQuery {
    name: Option<String>,
}
```

In this way, it would have been feasible to display a default name, for example :


```rs
#[get("/greet")]
async fn greet(greet_query: web::Query<GreetQuery>) -> web::Json<GreetResponse> {
    web::Json(GreetResponse {
        greeting: format!(
            "Hello, {}!",
            greet_query
                .name
                .as_ref()
                .map(String::as_str)
                .unwrap_or("Anonymous")
        ),
    })
}
```

Instead of the error when requesting <http://localhost:8080/greet> we would then have had the following response:

```json
{
  "greeting": "Hello, Anonymous!"
}
```

[Source code snapshot](https://github.com/d34db4b3/rest-api-tutorial/tree/snapshots/greetings)

> IN PROGRESS