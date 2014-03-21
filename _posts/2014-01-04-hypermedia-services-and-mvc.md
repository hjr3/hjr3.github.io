---
layout: post
title: "Hypermedia Services and MVC"
tags:
- mvc
- rest
- hypermedia
- api
- hateoas
status: publish
type: post
published: true
---

# Hypermedia Services and MVC

This blog post is in response to the discussion started by the [tweet](https://twitter.com/nateabele/statuses/418965626410270720) below:

<blockquote class="twitter-tweet" lang="en" align="center"><p>Incidentally, MVC is a REALLY poor fit for designing hypermedia services. So that's fun, everyone who thought you knew what you were doing.</p>&mdash; Nate Abele (@nateabele) <a href="https://twitter.com/nateabele/statuses/418965626410270720">January 3, 2014</a></blockquote>
<script async="async" src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

This post is not intended to present the best way to design and build hypermedia services. My goal is present a high level description of how we have built and designed a hypermedia API at HauteLook within the context of the MVC Architectual Pattern. That being said, I am always looking for ways to improve.

Let's start with the controller. According to the [Gang of Four](http://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612), the controller "defines the way the user interface reacts to user input". In the context of a hypermedia service, _user input_ comes in the form of a request. The request is most commonly sent using HTTP, but could use another protocol such as FTP. We will be assuming HTTP for the rest of this post. The _user interface_ is the API response the server sends back to the client. In a hypermedia API, the response is based on which resources the client (user) is requesting and what media type the client prefers.

Let us use the example below. We can assume that the client is knowledgeable of the `/users/42` URL because it made a HTTP GET request to `/` and received a list of user relations it can then request.

```
GET /users/42 HTTP\1.1
Host: example.com
Accept: application/hal+json
```

The controller will first fetch the user resource. If successful, the controller will specify a 200 response code, any necessary headers and then pass that resource to the view along with the desired client's desired media type. However, there are many reasons why the request may not be successful. The resource `/users/42` may not exist, the client may not be authorized to make the request, pre-conditions of the request may not be satisifed or any other number of problems. In any of those error cases, the controller will issue the proper response code, headers and any body necessary to represent the current state to the client.

The resources in our hypermedia API are models. We have some code samples of models for address and user resources below. I chose ruby as it makes the code very concise. How exactly these models are populated is an exercise left up to the reader. What we do __not__ want to do is simply transfer the data in our persistence layer (i.e. database) directly to the client. If we are using something like a relational database, there may be many tables required to accurately represent one resource. Take a look at the `Address` model below. We may need to execute a SQL statement that joins some hypothetical `addresses`, `states` and `countries` tables together in order to create address resources. Our goal is to encapsulate the resources you want to represent to the client. If the underlying persistence layer changes, the resource should not change.

```ruby
class Address
    attr_accessor :line1, :line2, :state, :country,
        :postal_code
end

class User
    attr_accessor :user_id, :email, :first_name, :last_name,
        :addresses

    def initialize(addresses)
        @addresses = addresses
    end

    def fullName()
        "#{first_name} #{last_name}"
    end
end
```

Once we have our resources created, we need to represent them to the client using the view. The view is only concerned with how we are presenting our resource to the client. It is not scalable to explicitly write out every resource/media-type combination. We want to _describe_ how to represent a resource in an agnostic way. We then feed those descriptions into a serializer that is aware of HAL, Atom, JSON-API, etc and can generate the output based on the desired media type of the client.

Here is an example DSL that describes how to represent a user resource to the client:

```yaml
relations:
    self:
        href:
            route: /users/:user_id
            params:
                user_id: id

    /rels/orders:
        href:
            route: /orders?user_id=:user_id
            params:
                user_id: id

    /rels/addresses:
        href:
            route: /users/:user_id/addresses
            params:
                user_id: id
        embed:
            property: addresses

properties:
    email: email
    name: fullName
```

The user class is serialized into the correct media type based on the description. The `relations` section describes how the user relates to other resources in the API. The `properties` section describes how to show the resource to the client. In this case, we do not show first and last name separate properties but instead show one `name` property. More importantly, we do not include the user_id in the response. It is probably not important to the client what the user id is, except to create URLs. We have the relations to avoid client-side URL generation though. Also, notice how the addresses are embedded in the representation of a user. The serializer would look up how to represent the address and serialize it accordingly.

The above DSL is loosely based on the [Symfony 2 HATEOAS bundle](http://hateoas-php.org/) that we use at HauteLook. Reading through the documentation you may notice that it uses funky PHP annotations. This is just a preference of the maintainers. There is also built-in support to use separate PHP or XML files to describe the view.

This architecture has made it fairly straight-forward to design and maintain a hypermedia API. The risk of changes at persistent layer leaking into the client representations is low. The models encapsulate the resources and we can make them as resistant to change as we want. The view presents the representation of the resource and how the resource relates to other resources to the client.
