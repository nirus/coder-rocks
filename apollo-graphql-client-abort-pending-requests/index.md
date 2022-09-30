### Quick intro to Apollo GraphQL

> Apollo is the industry-standard GraphQL implementation, providing the data graph layer that connects modern apps to the cloud.
> <div class="quote-credit"><a target="_blank" href="https://www.apollographql.com/">Apollo GraphQL</a></div>


In this tutorial we will understand the integration of [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController), a JavaScript API with Apollo GraphQL Client using a fullstack ReactJs sample project.

### Motivation for this post

It all started with a use case to prevent client from making a repeated request for the same GraphQL mutation query.
All the code inspired in this blog post is from a [GitHub issue thread](https://github.com/apollographql/apollo-feature-requests/issues/40) (***Read more for interesting discussion …***)

With that intro, lets understand how the solution is engineered for this topic.

### Libraries used:

- [UUID](https://www.npmjs.com/package/uuid) – Used to create unique request tracker ID and prevent namespace collision for multiple request from same component.
- [ReactJS](http://reactjs.org/) – No intro needed i guess?
- [Apollo GraphQL](https://www.apollographql.com/) – Follow the link to know more..


### Source code

All the code cited in this blogpost is documented in a [Github repository](https://github.com/nirus/fullstack-tutorial) (***full POC source code***) by forking the react project from the [Apollo GraphQL fullstack](https://github.com/apollographql/fullstack-tutorial) repo.

<div align="center" class="font-semibold"><h3>Code walkthrough</h3></div>

#### Step – 1:
Create a middleware that holds the logic to track and cancel duplicate request via [ReactJS context](https://reactjs.org/docs/context.html) API – [***cancelRequest.tsx***](https://github.com/nirus/fullstack-tutorial/blob/master/final/client/src/cancelRequest.ts) (***complete source code***)

#### Step – 2:
Generate namespace UUID and pass it using [requestTrackerId](https://github.com/nirus/fullstack-tutorial/blob/master/final/client/src/pages/login.tsx#L32) via query context as below.

```js
context:{ 
  requestTrackerId: uuidNameSpace('LOGIN', RequestNameSpace) 
}
```

#### Step – 3:
Finally wiring all the middlewares and setup it up as funnel layers using from API of Apollo GraphQL client and set [queryDeduplication](https://www.apollographql.com/docs/react/v2.5/advanced/network-layer/#query-deduplication) to **false**.

```js
const client: ApolloClient<NormalizedCacheObject> = new ApolloClient({
    cache,
    link: from([ authMiddleware, cancelRequestLink, httpLink ]),
    queryDeduplication: false
});
```

### Mechanism of action

When ever more than one request originates from the same mutation query, each query is tagged to its **requestTrackerId** which remains same to that particular query and different for other queries.

Using [UUID library](https://www.npmjs.com/package/uuid) namespace is generated for each query (***Read the code***). With this ID the middleware associates each query to its namespace generated ID and stored it in a cache object.

Subsequent incoming request are looked up using the cache object. If there’s an ongoing request which is not yet completed, it will be aborted immediately using [AbortController](https://developer.mozilla.org/en-US/docs/Web/API/AbortController) javascript API and this request is replaced with new request.

### How to run the demo?
Follow the steps stated in [readme](https://github.com/nirus/fullstack-tutorial/blob/master/README.md#installation) file from the Github repo for installation.

### Code modifications done for demoing
I have intentionally introduced a response delay of 5000ms in Login API (server code) to make [this](https://github.com/nirus/fullstack-tutorial) POC work.

Click repeatedly on **login button** and you can see the previous request getting cancelled in the **Chrome – Developer console > Network**.

<h3 align="center">See the demo video below</h3>
<video controls playsinline src="https://github.com/nirus/fullstack-tutorial/blob/master/apollo-request-cancel.mp4?raw=true"></video>

> Thank you. If you have come to the end of this blog, please leave a comment below and share this blog in your social feed.