---
title: Ollama, LangchainJS and Custom Headers
author: Varad Chavan
date: 2024-11-17 14:10:00 +0800
render_with_liquid: false
---

This post specifies some temporary fixes that can be made to LangchainJS in order to use an `Authorization` header with a Bearer token.

## Preface

### Requirement of the afformentioned header
Google Cloud Platform (GCP) and its service Cloud Run, has two methods for defining access rights for any running service.

- Unauthenticated Invocations
- Authenticated Invocations

The latter requires an `Authorization` header to be set with an authentcation token. This has to be the case for all packets sent to the service, and if not provided, the service always returns a `404`. Such packets never reach the service, and are discarded at the network level.

### Langchain and Ollama
When I started working on a feature for LLM integration at my current organization, Langchain JS had matured quite a bit. Thing is, the newer `import` statements that are required changed their locations from `@langchain/community/models/ollama` to `@langchain/ollama`. The entire module was rebuilt to be more independant by the maintainers, rather than a disjoint sub-module contributed by the community.

Though, this meant that there were many features available in the commmunity edition, which still aren't directly present in the current one. Specifically, simple `<string>:<string>` custom headers for HTTP requests made by the Ollama API endpoint.
## Problem at hand

Basically, when using `@langchain/ollama`, you specify your chain object like this:

```typescript
  const model = new ChatOllama({
    baseUrl,
    model: 'llama3.2:3b',
    temperature: 0.7,
    format: 'json',
  });
```

If you want a custom header, maintainers have recently added a way for you to do this via the `Headers()` object:
```typescript
  const model = new ChatOllama({
    baseUrl,
    model: 'llama3.2:3b',
    temperature: 0.7,
    format: 'json',
    headers,
  });
```
Which, in my case, meant I had to first correctly set the `headers` object with my authorization header, like this:
```typescript
  const bearerToken = await getIAMToken(baseUrl);
  const headers = new Headers();
  headers.set("Authorization", `Bearer ${bearerToken}`);
```
Now, it may seem like this should work. Everything here is, at least to my current knowledge, done exactly the way it needs to be. This is, this doesnt work.
If you log the `headers` object in console, it shows up as:
```typescript
HeadersList {
  [Symbol(headers map)]: Map(1) {
    'authorization' => {
      name: 'Authorization',
      value: 'Bearer xxxx'
    }
  },
  [Symbol(headers map sorted)]: null
}
```
This, when used, will not allow you to communicate with a Cloud Run service. Google's network does not see this object's `Authorization` header correctly, and assumes that it isn't set.

Hence, it should ideally look like:

```typescript
HeadersList {
    'Authorization':'Bearer xxxx'
}
```

As far as I've researched, this just isn't possible with the current rendition of `@langchain/ollama`. For my organization, I had to instead revert back to the now deprecated `@langchain/community` package instead, and set the header like this:

```typescript
    const model = new ChatOllama({
      baseUrl,
      model: 'llama3.2:3b',
      temperature: 0.7,
      headers: {
        'Authorization': `Bearer xxxx`
      }
```
This works correctly, and I eventually used this in producation, even knowing that this package may not work with future versions of Ollama.


## Conlusion
Honestly, the fact that there is a potential future issue in my organization's production code gives my jitters. But I honestly couldn't find a way to make it work. Days spent on this tiny issue possibly kept me from working on other much needed features, which isn't a good trade-off for right now. Writing this article to possibly guide any other developers facing the same issue hopefully helps.