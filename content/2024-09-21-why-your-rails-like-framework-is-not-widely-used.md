+++
title = "Why Your Rails-like Framework Is Not Widely Used"

[taxonomies]
tags=["frameworks"]
+++

Ruby on Rails defined the ethos of the web development community for many years. I have observed people trying to replicate "Rails" in other languages with very mixed results. I consider Laravel, ASP.NET and Phoenix successful rails-like frameworks. Python already has Django. Sadly Java, Kotlin, Scala, Node.js, Go and Rust all lack a widely adopted rails-like experience.

<!-- more -->

I believe Ruby on Rails was successful because the creator had strong opinions and  expressed those options as conventions in the framework. In 2003, Rails was in stark contrast to the configuration-heavy frameworks that tried to be everything to everyone.

When recently evaluating some frameworks that claim to be "Ruby on Rails"-like, I noticed responses from the framework creator(s) like:

> we decided to not decide

or 

> up to each individual to make a choice

Look, people giving away their code for free can do whatever they want. However, if your goal is to create a modern rails-like framework this is counter-productive. Rails means the framework author has _strong opinions expressed as conventions_ so developers using the framework do not have to make a bunch of upfront decisions before they can get started.

I think the fear is that our strong opinion will not hold up over time and we made the wrong decision. A developer is going to argue with us about it. Here is the thing: a developer that chooses to argue with you about a decision you made was never going to use your framework anyways. Block them. Maintain a learning mindset and remember that it is ok to be wrong on the internet. Also, saying "no" is a decision that is perfectly acceptable.

So, in the spirit of strong opinions and being wrong, below are my thoughts on what will make a modern rails-like framework successful.

## Auto-increment Keys Should Not Be IDs

Rails-like frameworks make heavy use of [ORMs](https://en.wikipedia.org/wiki/Object%E2%80%93relational_mapping) and the [ActiveRecord](https://www.martinfowler.com/eaaCatalog/activeRecord.html) pattern, which almost always means there is an auto-incrementing primary key for each database table. That was a fine default when Ruby on Rails was created in 2003 considering [uuids were proposed in 2005](https://datatracker.ietf.org/doc/html/rfc4122), but we know better now. Letting a user enumerate all the keys in a table is a bad idea. Even if we properly secure our API, auto-increment keys still leak information. Seeing `/orders/43534` tells me a lot. And no, starting the order id from some number greater than 1 is not a solution.

Use a uuid. I am not going to argue about performance. [Uuids are pretty fast](https://ardentperf.com/2024/02/03/uuid-benchmark-war/), your database probably fits in RAM and if you do need to scale the database then something like a uuid affords us horizontal scaling options that are very hard to do with an auto-increment primary key.

 If someone makes a strong case for the primary key being a ulid, nanoid or some other format then that works for me too. I also don't care if the decision is to create every table with an internal id that auto-increments and an external id that is a unique value. Maybe there is an even better solution I have not thought of. Great! I only care that the rails-like framework has decided, by default, the public API uses an id that is safe to share.

## API First

A rails-like framework typically uses the model–view–controller (MVC) pattern. One big impediment to establishing conventions is that the "view" has been in a constant state of flux for years as the frontend ecosystem for web development has been fragmented with no clear winner. To make matters worse, the frontend space is much more diverse than it was in 2003. We now have mobile apps, TUIs, IoT, VR and many more interfaces. These frontends often have their own development stories that are not easily integrated with. The common denominator to all of these frontend interfaces is that they need an API. It is also exceedingly common to build APIs that are only called by other APIs. Focus on APIs first and worry about the view layer later.

Being API first is only part of the decision. We have multiple protocols to choose from: REST, GraphQL, gRPC and many more. We need a protocol that can be used for both public and private APIs. We know that public GraphQL APIs have adoption issues. Even [Github failed to convince developers to use their GraphQL API](https://github.blog/changelog/2022-08-18-deprecation-notice-graphql-for-packages/). So, GraphQL is out. We also know that gRPC makes heavy use of HTTP/2 but [the majority of websites still use HTTP/1.1](https://w3techs.com/technologies/details/ce-http2). That being said, [96% of browser support HTTP/2](https://caniuse.com/?search=HTTP%2F2) so gRPC could be a viable option. I think the main reason to continue to use a RESTful API is that every frontend supports RESTful APIs but not every frontend has the tooling to support gRPC.

## RESTful API Specification

If a modern rails-like framework is API first, then it must have an API specification. This specification should be able to generate great looking documentation as well. GraphQL and gRPC are both schema-first, but the typical RESTful API is woefully under-specified.

We can fix this by having the rails-like framework automatically create an OpenAPI specification that can be shared with others and/or used to create nice looking documentation. The OpenAPI spec should be generated from the code. I think [dropshot](https://github.com/oxidecomputer/dropshot) and [fastapi](https://github.com/fastapi/fastapi?tab=readme-ov-file#interactive-api-docs) do a good job in this area. I am bearish on spec-first API development, but I am only too happy to be proven wrong.

Is OpenAPI perfect? No. Is there a viable alternative? No. Is the OpenAPI specification much better than doing nothing? Yes.

I considered [RAML](https://raml.org/) as an alternative but Salesforce owns it and we should only consider open specifications.

## Models From Upstream APIs

Scientifically poll developers and ask them what types of projects they dislike the most. I would bet a crisp $100 bill that "third-party integrations" is in the top 3. Why? A mentioned above, _the typical RESTful API is woefully under-specified._

Most frameworks models tightly couple their models to the database. Here is an idea: _consume an OpenAPI spec to codegen the models_. Now, our rails-like framework can both generate and consume OpenAPI spec. This means we can now easily compose applications built with this rails-like framework. Think of the flywheel! The more applications using this API first rails-like framework, the easier it is for developers to do integrations. Internal teams would quickly see this benefit. Imagine SaaS companies using it as a value proposition: "our API is generated by rails-like framework and works seamlessly when consumed by that same rails-like framework".

This approach also requires better tooling. Most API codegen tools produce low-quality code. My guess is that these tools try to handle every use case an OpenAPI document could have. Better tooling can be made by having strong opinions on what to support. For example, [progenitor](https://github.com/oxidecomputer/progenitor) is purpose-built to codegen OpenAPI documents from the aforementioned [dropshot](https://github.com/oxidecomputer/dropshot) framework.

## Feature First Structure

Most frameworks use a layer-first structure. Our controllers, models and views are grouped together.

```
src/
├── controllers/
├── models/
└── views/
```

As the application grows in size, developers will accidentally couple the layers together making your [monolithic application is difficult to scale](https://www.cortex.io/post/monoliths-vs-microservices-whats-the-difference) and harder to refactor. Before we start arguing about monoliths vs micro-services, I want to suggest that we can start with a better default for our project structure: feature first.

A feature-first structure might look something like:
```
src/
└── features/
    ├── feature1/
    ├── feature2/
    └── feature3/
```

The controllers, models and views are grouped together by feature. The [flutter community](https://codewithandrea.com/articles/flutter-project-structure/) is one group that is making feature-first a convention.

Now, a feature first structure is no silver bullet. It is basically domain driven design and people make design mistakes all the time. I believe the upsides outweigh the downsides though. A feature first structure makes it harder to create accidental coupling. And when coupling is inevitably introduced, feature first makes it easier to refactor the code to remove that coupling. Finally, if you go [monolith first](https://martinfowler.com/bliki/MonolithFirst.html) a feature first structure makes it easier to extract parts of the monolith into separate services should scaling become an issue.

A feature-first approach would almost certainly necessitate examples that show developers how to handle cross cutting concerns, such as authentication. These guides would focus on encouraging simple, local solutions that work for the feature, without forcing generalized abstractions that might not stand the test of time. Developers would grow comfortable with duplicating some code in order to avoid early abstractions. Avoiding premature generalization ensures that the development process stays agile and adaptable, even as the application grows. This approach also empowers teams to revisit abstractions when the patterns and needs become clearer, rather than guessing early and facing rigid, difficult-to-change systems down the road.

## Make Decisions for Modern Application Development

A rails-like framework will almost certainly not be successful by simply cloning Ruby on Rails conventions as-is. Application development in 2003 was very different from application development today. Framework authors wanting adoption must establish new conventions that allow developers to get up and running quickly without making many decisions about setup while also avoiding common maintenance pitfalls.
