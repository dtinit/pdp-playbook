
# Personal Data Portability Playbook - PDP Playbook

This project provides a playbook for implementing personal data portability.  It's intended for services that host personal data of some kind, and are unsure 
what approach to take to meet regulatory requirements, to provide data access that works for users and their 3rd-party tools, and to keep their effort and
maintenance costs low.

## Basic Approach

Our basic approach here is to help you build a HTTP+OAuth+JSON API quickly and efficiently.  A REST style is used due to its overwhelming familiarity and existence of mature and scalable tools.  The 
result should also have good data security, ops, defensibility and scaling characteristics.

```
┌─────────────────────────────┐
│        Custom Format        │
├──────────────┬──────────────┤
│     JSON     │ JSON Schema  │
├──────────────┼──────────────┤
│     HTTP     │    OAuth     │
└──────────────┴──────────────┘
```


## Who this playbook is for

Most online services and platforms that host personal data do so almost as a side-effect of 
providing their main value.  One of my long-term favourite sites is Ravelry, where the goal of 
collecting data about knitting patterns and yarn led to collecting personal data about 
individuals' knitting projects and ratings of patterns and yarn.  Then when Ravelry's 
community turned out to have engagement power, knitters' post history became another
important part of personal data. 

Other examples:

| Service Type | Personal Data Types |
| --- | --- |
| Music streaming services | Playlists and listen history |
| Video streaming services | Playlists and watch history |
| Note-taking services | Notes and folders |
| AI chat services | Chat history |
| Map services | Route history, favourites |

Some kinds of services are missing from this list.  Email and calendar servers should look to standards rather than follow this playbook.  

## How to use this playbook

There are two major ways to use this playbook

1. Read it and see what ideas and pointers may be helpful. Maybe use the playbook to fill in your project plan.
2. Point your AI coding agent at this playbook and ask it to follow the playbook until you're ready to deploy 

## Jobs to be done

1. Identifying what counts as portable data
2. Suggest some libraries and frameworks for the project
3. Explain how to use JSON Schema to define data formats
4. Explain how to hook your service's data storage into API
5. Make sure that data access is keyed to the user
6. Handle when there are references to OTHER users in personal data
7. Allow efficient async requests for chunks of bulk data with pagination or cursors
8. Hook in API keys for defensibility and rate-limiting 
9. Hook in OAuth for authorization of data access, with limited access scopes
10. Log grants and data access requests
11. View current access grants
12. Cloud deployment/ops plans
13. API discovery (including data schemas and OAuth scopes)
14. Plan for the future with a schema evolution strategy


## Meeting regulatory requirements

This approach is designed to satisfy the data portability requirements in:

- **GDPR Article 20** — the right to receive personal data in a structured, commonly used, machine-readable format, and to have it transmitted to another controller.
- **EU Digital Markets Act (DMA)** — Article 6(9), which requires gatekeepers to give end users and the third parties they authorize **continuous and real-time** access to data generated through use of the service.
- **UK Data Protection Act 2018** — which carries the UK GDPR's equivalent portability right post-Brexit.

Two words in the DMA text, and our interpretation thereof drive specific design choices above:

**Continuous** doesn't require a streaming or push protocol.  It means the user, or a 3rd party they've authorized, can come back and pull newly created data whenever they want, not just receive a one-time export. Using OAuth and pagination or cursors is powerful and flexible.

**Real-time** also doesn't require a streaming or push protocol. As long as the API can access data as soon as it's saved in your service, the API can give access to real-time data with latency that satisfies most use cases.

It's also possible to augment this API with WebSockets or another streaming or push-based mechanism,
but this RESTful approach is still where to start.
