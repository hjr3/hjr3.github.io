+++
title = "Landing Page Router Using Fastly Compute@Edge and WASM"
date = 2022-04-24

[taxonomies]
tags=["rustlang"]
+++

A company often has a landing page for first time visitors that is optimized for describing and educating that person on what product or service the company is offering. This page is usually not useful for people already familiar with the company. Ideally, a new user would see the marketing landing page and the returning user would see a more functional page. There are two common approaches to solving this problem that both have pitfalls. I want to explore a third option using Fastly's Compute@Edge offering.

<!-- more -->

Here are the two common approaches to serving different content when a user visits the company website:

1. A user browses to the index page of our website. That HTTP request is sent to our server and some server side language (Node.JS, PHP, Python, Ruby, etc) checks if some cookie exists and serves the appropriate page. The problem is that we can now no longer cache this page. Also, landing pages are usually static. It would be nice not use a static landing page or at least one that can be cached for a very long time.
1. Another option is to serve always the same landing page user browses to the index page of our website. We can cache this page at the edge for a very long time. Once the page loads, we can use JavaScript to check if some cookie exists and redirect the user to the more functional page. The problem is that a returning user will often see the marketing landing page flicker before they are redirected to the more functional page. Even if we put the JavaScript high up in the `<head>` element and try to prevent the flicker, we have a more subtle problem. We responded with a bunch of content that we immediately through away. This is wasteful and if we have a lot of people visiting our website, the bandwidth adds up.

A third option I want to explore is to perform this logic in the CDN itself. A user browses to the index page of our website and that HTTP request first goes to our CDN. We have some compute that checks for some cookie and serves the appropriate page. Since we are still within the CDN, those pages are served from the CDN cache. We are going to use Fastly as our CDN. Now, we could do this check in Fastly's VCL itself. However, VCL is hard to dev and test. Let us explore what we can do with Rust and Compute@Edge.

This is going to require some set up. We will need a Fastly account and a website to serve as a backend. Fastly has a limited free plan, but it should be good enough. I will use my personal website as the backend. First time visitors (no cookie) will see the index page. Returning visitors will see the [Tag: #rustlang](https://hermanradtke.com/tags/rustlang/) page.

Go to 
Start here: https://developer.fastly.com/learning/compute/
Go to https://manage.fastly.com/compute/
Add my domain lpr.hermanradtke.com
   This is the domain fastly will use
   We need to create the CNAME record. Let us skip TLS for right now.
   Verify it using `dig lpr.hermanradtke.com +short`
I made sure to name my service so it was easy to identify later
Now create a token
   Go to https://manage.fastly.com/account/personal/tokens
   Set global scope
   I set to never expire because this is a simple demo
   Save and store  it securely
Now install $ brew install fastly/tap/fastly
I opted to set up a fastly profile so I would not have to use -t or an env var

Finally, we can start coding.

cd /path/to/Code
mkdir landing-page-router
cd !$
fastly compute init
choose option

```
fastly compute init

Creating a new Compute@Edge project.

Press ^C at any time to quit.

Name: [landing-page-router]
Description:
Author: [herman@hermanradtke.com]
Language:
[1] Rust
[2] JavaScript
[3] AssemblyScript (beta)
[4] Other ('bring your own' Wasm binary)
Choose option: [1] 1
Starter kit:
[1] Default starter for Rust
    A basic starter kit that demonstrates routing, simple synthetic responses and
    overriding caching rules.
    https://github.com/fastly/compute-starter-kit-rust-default
[2] Authenticate at edge with OAuth
    Connect to an identity provider such as Auth0 using OAuth 2.0 and validate
    authentication status at the Edge, to authorize access to your edge or origin hosted
    applications.
    https://github.com/fastly/compute-rust-auth
[3] Beacon termination
    Capture beacon data from the browser, divert beacon request payloads to a log
    endpoint, and avoid putting load on your own infrastructure.
    https://github.com/fastly/compute-starter-kit-rust-beacon-termination
[4] Empty starter for Rust
    An empty starter kit project template.
    https://github.com/fastly/compute-starter-kit-rust-empty
[5] Static content
    Apply performance, security and usability upgrades to static bucket services such as
    Google Cloud Storage or AWS S3.
    https://github.com/fastly/compute-starter-kit-rust-static-content
Choose option or paste git URL: [1] 4

✓ Initializing...
✓ Fetching package template...
✓ Updating package manifest...
✓ Initializing package...

Initialized package landing-page-router to:
	/Users/herman/Code/landing-page-router

To publish the package (build and deploy), run:
	fastly compute publish

To learn about deploying Compute@Edge projects using third-party orchestration tools, visit:
	https://developer.fastly.com/learning/integrations/orchestration/


SUCCESS: Initialized package landing-page-router
```

Now we are supposed to run `$ fastly compute build` to verify. Great that works. But wait, I want to use familiar tools. Let us see if `cargo check` still works. It does.

Let us now deploy to make sure this simple example works.

```
fastly compute deploy

There is no Fastly service associated with this package. To connect to an existing service
add the Service ID to the fastly.toml file, otherwise follow the prompts to create a
service now.

Press ^C at any time to quit.

Create new service: [y/N] N
```

I stop because we already made a service. Let us find the service id. `fastly service list` and we can now add the id into the `fastly.toml` file.

But now we need to tell Fastly to send requests to our backend

`fastly backend create --version=2 --name="Blog" --address="hermanradtke.com" --use-ssl`
`fastly backend describe --version=latest --name="Blog"`
`fastly service-version activate --version=latest`
`fastly domain validate --name=lpr.hermanradtke.com --version=active`
