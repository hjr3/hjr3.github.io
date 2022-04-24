+++
title = "How To Mock Functions That Have External HTTP Requests"
date = 2022-04-23

[taxonomies]
tags=["rustlang"]
+++

When writing tests, we do not want to hit the external API each time we run our tests. If we are coming from a dynamic language, such as Node.JS, we may want to a solution like [fetch-mock][fetch-mock] which will patch the implementation of `fetch` at runtime. This is not practical in Rust. There are some attempts, like the [hotpatch](https://github.com/Shizcow/hotpatch) crate, but we will use a different strategy.

<!-- more -->

The complete code for this post can be found at: [https://github.com/hjr3/the-cat-api-http-mocks](https://github.com/hjr3/the-cat-api-http-mocks)

## Calling The Cat API

Let us start with an example. We will write a program to make search for cat breeds using [The Cat API](https://docs.thecatapi.com/). First, let us discover how this API works. Reading the docs for [GET /breeds/search](https://docs.thecatapi.com/api-reference/breeds/breeds-search) we can search for breeds using the `q` query parameter. Using curl, we can try this out:

```
$ curl https://api.thecatapi.com/v1/breeds/search?q=sib | jq
```

```json
[
  {
    "weight": {
      "imperial": "8 - 16",
      "metric": "4 - 7"
    },
    "id": "sibe",
    "name": "Siberian",
    "cfa_url": "http://cfa.org/Breeds/BreedsSthruT/Siberian.aspx",
    "vetstreet_url": "http://www.vetstreet.com/cats/siberian",
    "vcahospitals_url": "https://vcahospitals.com/know-your-pet/cat-breeds/siberian",
    "temperament": "Curious, Intelligent, Loyal, Sweet, Agile, Playful, Affectionate",
    "origin": "Russia",
    "country_codes": "RU",
    "country_code": "RU",
    "description": "The Siberians dog like temperament and affection makes the ideal lap cat and will live quite happily indoors. Very agile and powerful, the Siberian cat can easily leap and reach high places, including the tops of refrigerators and even doors. ",
    "life_span": "12 - 15",
    "indoor": 0,
    "lap": 1,
    "alt_names": "Moscow Semi-longhair, HairSiberian Forest Cat",
    "adaptability": 5,
    "affection_level": 5,
    "child_friendly": 4,
    "dog_friendly": 5,
    "energy_level": 5,
    "grooming": 2,
    "health_issues": 2,
    "intelligence": 5,
    "shedding_level": 3,
    "social_needs": 4,
    "stranger_friendly": 3,
    "vocalisation": 1,
    "experimental": 0,
    "hairless": 0,
    "natural": 1,
    "rare": 0,
    "rex": 0,
    "suppressed_tail": 0,
    "short_legs": 0,
    "wikipedia_url": "https://en.wikipedia.org/wiki/Siberian_(cat)",
    "hypoallergenic": 1,
    "reference_image_id": "3bkZAjRh1"
  }
]
```

Now that we know how to make a request and what the shape of the response looks like, we can write a program. We will use the reqwest crate to make our HTTP requests. I will opt for blocking behavior to avoid any async type juggling.

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Breed {
    id: String,
    name: String,
}

type BreedResponse = Vec<Breed>;

fn search_breeds(query: &str) -> Result<BreedResponse, Box<dyn std::error::Error>> {
    let url = format!("https://api.thecatapi.com/v1/breeds/search?q={}", query);
    let resp = reqwest::blocking::get(url)?.json::<BreedResponse>()?;

    Ok(resp)
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let resp = search_breeds("sib")?;
    println!("{:#?}", resp);
    Ok(())
}
```

Note: The serde\_json crate allows us to define a subset of the response. I only specified a few fields in `Breed` for brevity.

If we run our program, we should see something like:

```
$ cargo run
[
    Breed {
        id: "sibe",
        name: "Siberian",
    },
]
```

## Create Mocks Using Traits

Now, we want to test our program without making an actual HTTP request. We can use a [trait](https://doc.rust-lang.org/book/ch10-02-traits.html) to define the types of requests we can make. We will define a trait called `TheCatApi` and then implement a concrete `TheCatApiClient` type.

```rust
trait TheCatApi {
    fn search_breeds(&self, query: &str) -> Result<BreedResponse, Box<dyn std::error::Error>>;
}

struct TheCatApiClient {}
impl TheCatApi for TheCatApiClient {
    fn search_breeds(&self, query: &str) -> Result<BreedResponse, Box<dyn std::error::Error>> {
        let url = format!("https://api.thecatapi.com/v1/breeds/search?q={}", query);
        let resp = reqwest::blocking::get(url)?.json::<BreedResponse>()?;

        Ok(resp)
    }
}
```

Now our main function looks like this:

```rust
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let client = TheCatApiClient {};
    let resp = client.search_breeds("sib")?;
    println!("{:#?}", resp);

    Ok(())
}
```

If we run our program, we should get the same output above.

Now that we have a `TheCatApi` trait, we can also implement a mock client. We use the output from our curl request above and implement the `search_breeds` function to deserialize the JSON.

```rust
struct TheCatApiClientMock {}

impl TheCatApi for TheCatApiClientMock {
    fn search_breeds(&self, _query: &str) -> Result<BreedResponse, Box<dyn std::error::Error>> {
        let data = r#"
            [
              {
                "weight": {
                  "imperial": "8 - 16",
                  "metric": "4 - 7"
                },
                "id": "sibe",
                "name": "Siberian",
                "cfa_url": "http://cfa.org/Breeds/BreedsSthruT/Siberian.aspx",
                "vetstreet_url": "http://www.vetstreet.com/cats/siberian",
                "vcahospitals_url": "https://vcahospitals.com/know-your-pet/cat-breeds/siberian",
                "temperament": "Curious, Intelligent, Loyal, Sweet, Agile, Playful, Affectionate",
                "origin": "Russia",
                "country_codes": "RU",
                "country_code": "RU",
                "description": "The Siberians dog like temperament and affection makes the ideal lap cat and will live quite happily indoors. Very agile and powerful, the Siberian cat can easily leap and reach high places, including the tops of refrigerators and even doors. ",
                "life_span": "12 - 15",
                "indoor": 0,
                "lap": 1,
                "alt_names": "Moscow Semi-longhair, HairSiberian Forest Cat",
                "adaptability": 5,
                "affection_level": 5,
                "child_friendly": 4,
                "dog_friendly": 5,
                "energy_level": 5,
                "grooming": 2,
                "health_issues": 2,
                "intelligence": 5,
                "shedding_level": 3,
                "social_needs": 4,
                "stranger_friendly": 3,
                "vocalisation": 1,
                "experimental": 0,
                "hairless": 0,
                "natural": 1,
                "rare": 0,
                "rex": 0,
                "suppressed_tail": 0,
                "short_legs": 0,
                "wikipedia_url": "https://en.wikipedia.org/wiki/Siberian_(cat)",
                "hypoallergenic": 1,
                "reference_image_id": "3bkZAjRh1"
              }
            ]
        "#;

        let resp: BreedResponse = serde_json::from_str(data)?;

        Ok(resp)
    }
}
```

You can also shorten this up by doing `BreedResponse { id: "sibe", name: "Siberian" }` but for real world examples I find it easier to paste the JSON string.

Now we can wire up some tests. In this simple case, we are only testing that we implemented the `BreedResponse` type and `Breed` struct correctly.

```rust
#[test]
fn search_breeds() {
    struct TheCatApiClientMock {}
    impl TheCatApi for TheCatApiClientMock {
        fn search_breeds(
            &self,
            _query: &str,
        ) -> Result<BreedResponse, Box<dyn std::error::Error>> {
           // removed for brevity. use implementation above
        }
    }

    let client = TheCatApiClientMock {};
    let resp = client
        .search_breeds("does not matter what i put")
        .expect("search_breeds failed");
    assert_eq!(resp[0].id, "sibe");
    assert_eq!(resp[0].name, "Siberian");
}
```

```
$ cargo test
    Finished test [unoptimized + debuginfo] target(s) in 0.15s
     Running unittests src/main.rs (target/debug/deps/rust_mocks-aa82d6388d1da1bd)

running 1 test
test tests::search_breeds ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

And we should also test a decoding failure to make sure we are testing what we expect.

```rust
#[test]
fn search_breeds_decode_error() {
    struct TheCatApiClientMock {}
    impl TheCatApi for TheCatApiClientMock {
        fn search_breeds(
            &self,
            _query: &str,
        ) -> Result<BreedResponse, Box<dyn std::error::Error>> {
            let data = "nope";

            let resp: BreedResponse = serde_json::from_str(data)?;

            Ok(resp)
        }
    }

    let client = TheCatApiClientMock {};
    let err = client
        .search_breeds("does not matter what i put")
        .unwrap_err();
    assert_eq!(err.to_string(), "expected ident at line 2 column 18");
}
```

Notice that I put the mock client implementation inside the test function. If we define it outside the test function, then each mock needs a unique name.

## Injecting Traits As Dependencies

This is all good but we have side stepped a big part of implementing this strategy in a real program. A big reason people reach for testing libraries like [fetch-mock][fetch-mock] is because they have no other way to tell their function to use a different implementation of fetch. Indeed, we need to structure our program to inject dependencies. I mostly write web servers, so let us create a simple web server for a more real world example. Our web server will accept a request to search for breeds and then use our implementation of `TheCatApi` trait get the data. I am going to use [rocket.rs](https://rocket.rs) as it has really good docs. I will also be using the v0.4 version, which is blocking. Be aware that the v0.4 version requires the nightly compiler.

When we create our web server, we want to inject our dependencies. Specifically, we want to inject `TheCatApiClient`. We can do this in rocket.rs by using the [manage](https://api.rocket.rs/v0.4/rocket/struct.Rocket.html#method.manage) method. This will allow us to access the client from the request handler.

```rust
fn main() {
    let the_cat_api_client = TheCatApiClient {};

    rocket::ignite()
         // we inject our dependency here
        .manage(the_cat_api_client)
        .mount("/", routes![index, get_breed])
        .launch();
}

#[get("/breed?<search>")]
fn get_breed(
    // we access our dependency here
    client: State<TheCatApiClient>,
    search: &RawStr,
) -> Result<String, Box<dyn std::error::Error>> {
    let resp = client.inner().search_breeds(search)?;

    Ok(resp[0].name.clone())
}
```

Now we can run our server

```
cargo run
   Compiling rust-mocks v0.1.0 (/Users/herman/Code/rust-mocks)
    Finished dev [unoptimized + debuginfo] target(s) in 2.10s
     Running `target/debug/rust-mocks`
🔧 Configured for development.
    => address: localhost
    => port: 8000
    => log: normal
    => workers: 16
    => secret key: generated
    => limits: forms = 32KiB
    => keep-alive: 5s
    => read timeout: 5s
    => write timeout: 5s
    => tls: disabled
🛰  Mounting /:
    => GET /breed?<search> (get_breed)
🚀 Rocket has launched from http://localhost:8000
```

and make a request

```rust
$ curl 'localhost:8000/breed?search=cal'
California Spangled
```

Now that we have a working web server that makes requests to The Cat API, we want to write a test for our `get_breed` handler. The rocket.rs framework makes this fairly straight-forward as `client` is a parameter to `get_breeds`. We will need to change the type of client from the concrete implementation of `TheCatApiClient` to a type that will allow us to use any implementation of the `TheCatApi` trait. There are two ways to do this: generics and boxed traits. Unfortunately, rocket.rs does not allow us to use generic functions. If we try to write a generic function, then we will get a compiler error.

```rust
#[get("/breed?<search>")]
fn get_breed<T: TheCataApi>(
    client: State<T>, // <---- compiler error here
    search: &RawStr,
) -> Result<String, Box<dyn std::error::Error>> {}
```

So, boxed traits it is! We only create one instance of our API client when our program runs, so it creating our client on the stack or heap does not make much of a difference.

```rust
#[get("/breed?<search>")]
fn get_breed(
    client: State<Box<dyn TheCatApi>>,
    search: &RawStr,
) -> Result<String, Box<dyn std::error::Error>> {
    let resp = client.inner().search_breeds(search)?;

    Ok(resp[0].name.clone())
}
```

We need to update our main function as well.

```rust
fn main() {
    let the_cat_api_client = Box::new(TheCatApiClient {});

    rocket::ignite()
        .manage(the_cat_api_client)
        .mount("/", routes![index, get_breed])
        .launch();
}
```

Now our `get_breed` function can accept any implementation of `TheCatApi` trait. We have one more thing to do before we can write our test. Notice that the `client` type in `get_breed` is `State<Box<dyn TheCatApi>>`. We need some way of creating that `State` type. The rocket.rs docs have a [Testing with `State`](https://api.rocket.rs/v0.4/rocket/request/struct.State.html#testing-with-state) section that gives us the hint. So, to make this work we will need to extract the set up logic for our web server out of the `main` function. We create a `setup` function that allows us to pass in an implementation of `TheCatApi` trait.

```rust
fn setup(the_cat_api: Box<dyn TheCatApi>) -> Rocket {
    rocket::ignite()
        .manage(the_cat_api)
        .mount("/", routes![index, get_breed])
}

fn main() {
    let the_cat_api_client = Box::new(TheCatApiClient {});

    let rocket = setup(the_cat_api_client);
    rocket.launch();
}
```

With that in place, we can write a test that allows us to ensure `get_breed` succeeds.

```rust
#[test]
fn breed_succeeds() {
    struct TheCatApiClientMock {}
    impl TheCatApi for TheCatApiClientMock {
        fn search_breeds(
            &self,
            query: &str,
        ) -> Result<BreedResponse, Box<dyn std::error::Error>> {

            // removed for brevity. use implementation above
        }
    }

    // create our mock client
    let mock_client = Box::new(TheCatApiClientMock {});

    // inject it into our web server
    let rocket = setup(mock_client);

    // get our state
    let state: State<Box<dyn TheCatApi>> =
        State::from(&rocket).expect("managing `TheCatApiClientMock`");

    let resp = get_breed(state, RawStr::from_str("sib")).expect("get_breed failed");
    assert_eq!(resp, "Siberian");
}
```

We can also create a mock client that fails.

```rust
#[test]
fn breed_decode_error() {
    struct TheCatApiClientMock {}
    impl TheCatApi for TheCatApiClientMock {
        fn search_breeds(
            &self,
            _query: &str,
        ) -> Result<BreedResponse, Box<dyn std::error::Error>> {
            let data = "nope";

            let resp: BreedResponse = serde_json::from_str(data)?;

            Ok(resp)
        }
    }

    let mock_client = Box::new(TheCatApiClientMock {});
    let rocket = setup(mock_client);
    let state: State<Box<dyn TheCatApi>> =
        State::from(&rocket).expect("managing `TheCatApiClientMock`");
    let err = get_breed(state, RawStr::from_str("sib")).unwrap_err();
    assert_eq!(err.to_string(), "expected ident at line 1 column 2");
}
```

## Further Discussion

Structuring our application this way makes it easier to inject dependencies. I tend to inject any dependency that is accessing the network or file system. This includes common parts of a web server like databases, loggers, tracing and API clients. The benefits go beyond testing.

When serverless computing came onto the scene, many developers would write their serverless function in such a way that it could only run in the cloud. This created painfully long dev cycles where each change would take 30 seconds to upload to the cloud to verify. Some folks reached for complex solutions like [serverles framework](https://www.serverless.com/) that tried to emulate the behavior of the cloud. It may have been better to design the application to accept a `Server` trait. We could implement that trait to work for [hyper.rs](https://hyper.rs/) and to work for the cloud implementation. Our dev cycle is now much faster and has less complexity than an emulated system.

This starts to look like [Hexagonal architecture](https://en.wikipedia.org/wiki/Hexagonal_architecture_(software)). We define traits for our network and filesystem dependencies so we can change the behavior of the system. Do not go overboard on this pattern. I tend to implement a trait when the need arises. I also like the [Boundaries](https://www.destroyallsoftware.com/talks/boundaries) talk by Gary Bernhardt.

## Criticism and Alternatives

One downside to this approach is that we are not testing the actual implementation of `search_breeds`. We may have a bug in our code that does not show up in our tests. It is important that we keep our `search_breeds` function as small as possible to mitigate this downside.

If testing the actual implementation of `search_breeds` is a real concern, then we want to reach for libraries like [httptest](https://github.com/ggriffiniii/httptest). This will define a local web server that can be configured to return specific responses. If we have other types of dependencies, like a database, then we can look for an in-memory implementation of that database. There is no silver bullet solution, so pick the option that best suits your needs.

[fetch-mock]: https://www.npmjs.com/package/fetch-mock
