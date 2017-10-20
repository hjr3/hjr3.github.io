---
layout: post
title: "Connecting a webservice to a database in Rust"
tags:
- rustlang
status: publish
type: post
published: true
---

**Note: This blog post does will not work with rustc 1.19.0 or later due to a [regression](https://github.com/rust-lang/rust/issues/42460) in rust 1.19.0. Use the following to set rust 1.18.0 up:**

```
$ cd /path/to/project
$ rustup install 1.18.0
$ rustup override 1.18.0
```

In this post we are going to hook our [basic webservice](/2016/05/16/creating-a-basic-webservice-in-rust.html) up to a database. The webservice will accept a request for `/orders`, query the database for orders and return a json response. I will be using PostgreSQL in this example. There is a pure Rust [PostresSQL driver](https://crates.io/crates/postgres) written by Steven Fackler (sfackler) that I think is well done. That being said, the [mysql crate](https://crates.io/crates/mysql) looks well done too.

This post goes into a fair amount of detail. You can skip right to the [TL;DR](#tldr) for the final solution.

## Preparation

We have to get Postgres setup before we start writing Rust code. I am using [https://github.com/jackdb/pg-app-dev-vm](https://github.com/jackdb/pg-app-dev-vm) in combination with [Vagrant](https://www.vagrantup.com/) to automatically provision a working Postgres instance. Simply clone the git repository and then `vagrant up`. I am using the default values of `myapp` for username, `dbpass` for the password and `myapp` for the database name. I also have a script called [db-migrate.sh](https://github.com/hjr3/webservice-demo-rs/blob/blog-post-2/db-migrate.sh) that will create the orders schema necessary to get this example working.

## Crate Dependencies

At this point we have a working database instance with an orders tables containing two rows. The first thing we need to do is update our `Cargo.toml` file with our postgres dependency. We also need to add the [rustc-serialize crate](https://crates.io/crates/rustc-serialize) so we can serialize a native Rust data structure into json format. Next time we run `cargo build` both crates will automatically be downloaded and made available to our webservice.

```toml
[package]
name = "orders"
version = "0.1.0"
authors = ["Your Name <your.name@example.com>"]

[dependencies]
nickel = "0.8.1"
postgres = "0.11.7"
rustc-serialize = "0.3.19"
```

_Note: The rustc-serialize crate works, but it is not being actively developed. The future of json serialization is the [serde_json crate](https://crates.io/crates/serde_json). Unfortunately, serde's ability to automatically serialize data structures is only available on Rust nightly (the version of Rust in active development). Due to this restriction, I have chosen to use rustc-serialize instead._

We now need to open up `src/main.rs` and start adding our dependencies. We need to import the `postgres` and `rustc_serialize` crates. These two crates are not exporting macros, so we can leave off the `#[macro_use]` attribute. Also, notice that the crate name `rustc-serialize` (hyphen) is imported as `rustc_serialize` (underbar). The rustc-serialize crate is from early Rust days and the rules around crate names has changed.

Now we will alias which parts of the crates we want to use. We will be using the postgres `Connection` struct and the `SslMode` enum. We also will be using the rustc_serialize `json` module.

```rust
#[macro_use] extern crate nickel;
extern crate postgres;
extern crate rustc_serialize;

use nickel::{Nickel, MediaType};
use postgres::{Connection, SslMode};
use rustc_serialize::json;
```

## Order Struct

We will be querying the database for orders and then mapping the resulting rows into one more objects. Our database schema contains an orders table with an order id, an order total, the type of currency that was used and the status of the order. We need to create an `Order` struct to map each row to. The postgres crate provides [type correspondence](https://github.com/sfackler/rust-postgres#type-correspondence) documentation that maps each Postgres type to a Rust type. Using that information, we can create the `Order` struct with the correct types.

Once the query result has been mapped into an `Order` struct, we want to serialize that into json. We could [manually implement](https://doc.rust-lang.org/rustc-serialize/rustc_serialize/json/index.html#verbose-example-of-tojson-usage) the `ToJson` trait that tells rustc_serialize how to convert an `Order` struct into json, but I do not want to write code unless I have to. Instead, we can use the `#[derive()]` attribute and automatically generate the trait implementation for `RustcEncodable`. The `RustcEncodable` trait will allow us to call `json::encode()` on our `Order` struct.

```rust
#[derive(RustcEncodable)]
struct Order {
    id: i32,
    total: f64,
    currency: String,
    status: String,
}
```

Using `#[derive()]` can feel a bit like magic. The `Order` struct is just a shell around some primitive Rust types. The rustc_serialize crate has [already implemeted](https://doc.rust-lang.org/rustc-serialize/rustc_serialize/trait.Encodable.html) `Encodable` for pretty much all the primitive types. As such, the compiler has enough information to automatically implement the `RustcEncodable` trait for the `Order` struct. If we had used a type that did not already implement `Encodable`, then the compiler would have thrown an error.

_Note: If you are wondering why we derive `RustcEncodable` to automatically implement the `Encoding` trait, know that the rustc_serialize crate used to be part of the std library, was deprecated and migrated out to [crates.io](https://crates.io). In order for the rustc_serialize crate not to clash with the code still in the stdlib, the name we derive was modified. You can look to this [commit](https://github.com/rust-lang/rust/commit/a76a80276852f05f30adaa4d2a8a2729b5fc0bfa) for more details. This is a unique case. In the vast majority of cases, the name of the trait and the name of the trait we are deriving are the same._

## Database Connection

We are now ready to setup our database connection. Based on the Postgres connection information provided during the [Preparation](#preparation) section, we can create a database url. We then create a `Connection` object that represents our connection to the Postgres database. Using the `SslMode` enum, we opt to make create the connection over plain-text.

```rust
fn main() {

    let db_url = "postgresql://myapp:dbpass@localhost:15432/myapp";
    let db = Connection::connect(db_url, SslMode::None)
        .expect("Unable to connect to database");

   // ...
}
```

Connecting to the database can fail, so `Connection::connect()` is returning a [Result](https://doc.rust-lang.org/std/result/enum.Result.html) type. Most examples you will see choose to `.unwrap()` the `Result` type, which would yield the connection on `Ok` or panic on `Err`. I will be using `.expect()` instead of `.unwrap()`. Using `.expect()` is just like using `.unwrap()` except that it allows for a more user-friendly error message if something goes wrong. This will help us debug any issues we may encounter, especially if you are modifying these exmaples.

## Querying the Database

Let us now jump down to our `/orders` route and replace the static json response with an actual database result. We create our SQL string to fetch rows from the orders table. We also need to create a mutable `orders` vector (array) to store the `Order` objects we are mapping. We then fire off the query by passing in our SQL string and any paramters we wanted to bind. In this case, we have no parameters to bind so we pass a reference to an empty [slice](https://doc.rust-lang.org/std/primitive.slice.html) (`&[]`). We loop over each row in the result, manually convert the result into an `Order` struct and store it in the `orders` vector. After all the rows have been converted, we call `json::encode()` on the orders vector and return that result. Remember, we derived `RustcEncodable` on the `Order` struct. The rustc_serialize crate already implemented `Encodable` on the `Vec` too. The combination of all these `Encodable` trait implementations allows for the automatic serialization of `orders` using `json::encode()`.

```rust
   get "/orders" => |_request, mut response| {
       let query = "SELECT id, total, currency, status FROM orders";
       let mut orders = Vec::new();
       for row in &db.query(query, &[]).expect("Failed to select orders") {
           let order = Order {
               id: row.get(0),
               total: row.get(1),
               currency: row.get(2),
               status: row.get(3),
           };

           orders.push(order);
       }

       response.set(MediaType::Json);
       json::encode(&orders).expect("Failed to serialize orders")
   }
```

### Sync Error

Below are all the changes we have made so far:

```rust
#[macro_use] extern crate nickel;
extern crate postgres;
extern crate rustc_serialize;

use nickel::{Nickel, MediaType};
use postgres::{Connection, SslMode};
use rustc_serialize::json;

#[derive(RustcEncodable)]
struct Order {
    id: i32,
    total: f64,
    currency: String,
    status: String,
}

fn main() {

    let db_url = "postgresql://myapp:dbpass@localhost:15432/myapp";
    let db = Connection::connect(db_url, SslMode::None)
        .expect("Unable to connect to database");
    let mut server = Nickel::new();

    server.utilize(router! {
        get "/orders" => |_request, mut response| {
            let query = "SELECT id, total, currency, status FROM orders";
            let mut orders = Vec::new();
            for row in &db.query(query, &[]).expect("Failed to select orders") {
                let order = Order {
                    id: row.get(0),
                    total: row.get(1),
                    currency: row.get(2),
                    status: row.get(3),
                };

                orders.push(order);
            }

            response.set(MediaType::Json);
            json::encode(&orders).expect("Failed to serialize orders")
        }
    });

    server.listen("127.0.0.1:6767");
}
```

In our `main` function, we setup a connection to the database, create a nickel webserver and define our `/orders` route. Our `/orders` route calls a closure that uses the above database connection to fetch orders from the database and then serializes them into json. This looks pretty straight-forward, but if we try to compile this code we will get a rather initimidating error message. If we parse through multi-line error message, we can pull out two peices of information:

   1. error: the trait `core::marker::Sync` is not implemented for the type `core::cell::UnsafeCell<postgres::InnerConnection>`
   1. `core::cell::UnsafeCell<postgres::InnerConnection>` cannot be shared between threads safely

Here in lies the beauty of Rust. The `Connection` object is not thread safe and, while it may not have been apparent, nickel serves requests in different threads. Rust only allows types that implement the [Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html) trait to be shared between threads. For a moment, let us be pragmatic about this. Rather than try and figure out how to make `Connection` thread safe we will just work around it by establishing the postgres connection as part of the `/orders` request.

```rust
fn main() {

    let db_url = "postgresql://myapp:dbpass@localhost:15432/myapp";
    let mut server = Nickel::new();

    server.utilize(router! {
        get "/orders" => |_request, mut response| {
            let db = Connection::connect(db_url, SslMode::None)
                .expect("Unable to connect to database");
            let query = "SELECT id, total, currency, status FROM orders";
            let mut orders = Vec::new();
            for row in &db.query(query, &[]).expect("Failed to select orders") {
                let order = Order {
                    id: row.get(0),
                    total: row.get(1),
                    currency: row.get(2),
                    status: row.get(3),
                };

                orders.push(order);
            }

            response.set(MediaType::Json);
            json::encode(&orders).expect("Failed to serialize orders")
        }
    });

    server.listen("127.0.0.1:6767");
}
```

We can now do `cargo run` and make a curl request in another window see our json response:

```
$ cargo run
     Running `target/debug/orders`
Listening on http://127.0.0.1:6767
Ctrl-C to shutdown server
```

```
$ curl --silent localhost:6767/orders | python -mjson.tool
[
    {
        "currency": "USD",
        "id": 123,
        "status": "shipped",
        "total": 30.0
    },
    {
        "currency": "USD",
        "id": 124,
        "status": "processing",
        "total": 20.0
    }
]
```

### Fixing the Sync Error

Now that we have a functioning webservice that connects to a Postgres database, let us stop and consider our approach. Making a connection per request may be fine for a database like MySQL, where connections are [stateful and cheap to create](http://stackoverflow.com/a/99565/775246), but not recommended for Postgres. We need to create a pool of connections that can be shared across the many different requests. Luckily for us, the creator of the postgres create also created a connection pool called [r2d2](https://github.com/sfackler/r2d2) with a Postgres specific [adapter](https://github.com/sfackler/r2d2-postgres). The connection pool internally uses a [Mutex](https://doc.rust-lang.org/std/sync/struct.Mutex.html), which implements [Sync](https://doc.rust-lang.org/std/marker/trait.Sync.html), allowing the connections to be shared across threads.

We also need to consider how we are passing our connection pool to the request. The `/orders` route is implemented using a [move closure](https://doc.rust-lang.org/book/closures.html#move-closures), which will take ownership of the connection pool once we try to use it. If we create another route and try to use the connection pool, the compiler will throw an error because we now have two closures trying to take ownership of the same value. We need to take advantage of nickel middlware in order to properly share the connection pool. The nickel framework already provides [nickel-postgres](https://github.com/nickel-org/nickel-postgres) middleware for this very use-case.

## Using Connection Pool Middleware

We need to add three more crates to Cargo.toml. The `nickel_postgres` crate requires a patch that has not been merged yet, so we are specifying a git revision. If/when the PR is accepted, I will update this section.

```
r2d2 = "0.7.0"
r2d2_postgres = "0.10.1"
nickel_postgres = { git = "https://github.com/hjr3/nickel-postgres", rev = "9c1e21f" }
```

Once that is done, we need to import those three crates and then start specifying what parts of those crates we are going to use. The `r2d2_postgres` crate has a `PostgresConnectionmanager` that wraps the standard `Connection` struct provided by the `postgres` crate. The `r2d2_postgres` crate also provides a different `SslMode` enum (I am not sure why?), so we need to use that instead. This means we can get rid of the explicit postgres crate dependency and we can remove `postgres = "0.11.7"` from our Cargo.toml file.

```rust
#[macro_use] extern crate nickel;
extern crate rustc_serialize;
extern crate r2d2;
extern crate r2d2_postgres;
extern crate nickel_postgres;

use nickel::{Nickel, MediaType};
use rustc_serialize::json;
use r2d2::{Config, Pool};
use r2d2_postgres::{PostgresConnectionManager, SslMode};
use nickel_postgres::{PostgresMiddleware, PostgresRequestExtensions};
```

We will be passing the `PostgresConnectionManager` into a `Pool` provided by the `r2d2` crate. The `Pool` manages all of the complexity around sharing a fixed number of database connections across different threads. The `PostgresConnectionManager` implements the correct trait so the `Pool` can interact with Postgres connections. The `Pool` also accepts a `Config` struct that configures how the `Pool` will work. I chose to use the default settings, but you can customize it if you want a different number of connections.

Now that we have our connection pool setup, we need to create the middleware. The `PostgresMiddleware` struct abstracts away all the details of how the middleware works. We only need to create the middleware and pass on our connection pool. You will also notice that we use `PostgresRequestExtensions` from `nickel_postgres`. This is a trait that makes it easier for us to get a connection from the pool when inside of our request.

```rust
fn main() {
    let db_url = "postgresql://myapp:dbpass@localhost:15432/myapp";
    let db_mgr = PostgresConnectionManager::new(db_url, SslMode::None)
        .expect("Unable to connect to database");

    let db_pool = Pool::new(Config::default(), db_mgr)
        .expect("Unable to initialize connection pool");

    let mut server = Nickel::new();
    server.utilize(PostgresMiddleware::new(db_pool));

    // ...
}
```

When each request comes in, the middleware will put a reference to the connection pool on the request object. We can use `request.db_conn()`, made possible by the `PostgresRequestExtensions` trait, to get a database connection from the pool. Now we can use that connection just like we were before. Once our request goes out of scope, the connection will automatically be returned to the pool.

<a id="tldr"></a>

Here is our finished product:

```rust
#[macro_use] extern crate nickel;
extern crate rustc_serialize;
extern crate r2d2;
extern crate r2d2_postgres;
extern crate nickel_postgres;

use nickel::{Nickel, MediaType};
use rustc_serialize::json;
use r2d2::{Config, Pool};
use r2d2_postgres::{PostgresConnectionManager, SslMode};
use nickel_postgres::{PostgresMiddleware, PostgresRequestExtensions};

#[derive(RustcEncodable)]
struct Order {
    id: i32,
    total: f64,
    currency: String,
    status: String,
}

fn main() {

    let db_url = "postgresql://myapp:dbpass@localhost:15432/myapp";
    let db_mgr = PostgresConnectionManager::new(db_url, SslMode::None)
        .expect("Unable to connect to database");

    let db_pool = Pool::new(Config::default(), db_mgr)
        .expect("Unable to initialize connection pool");

    let mut server = Nickel::new();
    server.utilize(PostgresMiddleware::new(db_pool));

    server.utilize(router! {
        get "/orders" => |request, mut response| {
            let query = "SELECT id, total, currency, status FROM orders";
            let mut orders = Vec::new();
            let db = request.db_conn().expect("Failed to get a connection from pool");
            for row in &db.query(query, &[]).expect("Failed to select orders") {
                let order = Order {
                    id: row.get(0),
                    total: row.get(1),
                    currency: row.get(2),
                    status: row.get(3),
                };

                orders.push(order);
            }

            response.set(MediaType::Json);
            json::encode(&orders).expect("Failed to serialize orders")
        }
    });

    server.listen("127.0.0.1:6767");
}
```

It was a bit of a journey, but we now have a webservice that can properly make requests to a Postgres database and return the result as a json response. Our first attempt ran into a compiler issue when `Connection` did not implement `Sync`. We had to modify our orginal approach to fit within the rules that the Rust compiler enforces. That, briefly, meant creating a database connection per request. Realizing this approach was not recommended, we refactored our webservice to use a connection pool that provided thread safety. We also decided to use nickel middlware to expose the connection pool to each request. It added a bit more complexity to our code, but the tradeoff is that we are now guaranteed to be free of data races when serving requests on different threads. You can find the complete working example on github at [https://github.com/hjr3/webservice-demo-rs/tree/blog-post-2](https://github.com/hjr3/webservice-demo-rs/tree/blog-post-2).
