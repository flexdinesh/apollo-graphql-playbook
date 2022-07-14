# apollo-graphql-playbook
Notes and guides on using apollo client's APIs

[GraphQL](https://graphql.org/) allows us to build APIs based on domain modal once and consume it in different interfaces within one or more platforms without having to write (for the most part) feature specific APIs that involve the same domain modal again and again. Eg. in an address book, Write address API once and it can be used in different pages with different entity nesting and across platforms with the different nesting if we want to stretch our experience to native mobile experience.

Apollo is an open source tool that allows writing both GraphQL APIs using [apollo server](https://www.apollographql.com/docs/apollo-server/) and GraphQL clients using [apollo client](https://www.apollographql.com/docs/react/). They don't necessary have to coexist in applications and one can function independent of the other. This guide currently only focuses on apollo client.

## Apollo Client

Apollo client is primarily a GraphQL API consumption tool that doubles down as a state management solution for React applications. So when we're using apollo client in our application, we take away the need to manage state using state management tools since apollo does it internally and very well as we can expect a state management solution to work. If you want a peek into how external state management works, please take a look at this [contrived example in codesandbox](https://codesandbox.io/s/external-store-contrived-example-0pxevk).

### Queries, Mutations and Subscriptions

In GraphQL world queries are how you request data from the API. Mutations are how you send a payload to API requesting changes to data. Subscription are real time updates pushed from the API without having to long-poll requests.

### Cache policies

Apollo has a handful of [cache policies](https://www.apollographql.com/docs/react/data/queries/#supported-fetch-policies) to work with with the default being `cache-first`.

**cache-first:** This is the default. `cache-first` means that apollo looks for data in the cache and only when cache doesn't have data apollo loads the data from network. This loosely translates to - apollo loads data from network only once when query hook is mounted for the first time. After that all re-renders and unmount/mount-again of the hooks use cached data. The data is not reset until the page is refreshed. This is problematic sometimes —

- with filters since when filters are changed we want to refetch and not use existing cached data
- for long standing cache, if we don't refetch and the entity changes often without subscriptions then we show stale data to our users until the page is refreshed

**cache-and-network:** This policy fetches data from network on first load and on every mount of the hook. Data is fetched from cache only during re-renders.

**cache-only:** The data is only fetched from cache and never resorts to network. This is rarely useful and only ever useful in nextFetchPolicy rarely and often problematic because when we insert incomplete payload into cache, this policy prevents apollo from automatically refetching queries when cache entry is missing fields. Don't ever use this unless you have a very good reason to.

## Handling query response

Query responses automatically get updated in cache and query hooks work as you expect. Only thing to be aware of is to [learn how apollo's cache config works](https://www.apollographql.com/docs/react/caching/cache-configuration) so you'll have a better understanding on merging entities and fields.

Apollo usually always auto-merges object fields by replacing keys. When it comes to array fields apollo does the same it does to key fields as default behaviour - apollo replaces the array. But during pagination, we might want to preserve existing data and merge new data into the array in a non-destructive way and this is where merge functions come in. Merge functions let you handle how you want to merge new network data with existing cache data.

## Handling mutation response

Mutation responses can be handled in a quite a handful of ways —

### Refetch queries

Once the mutation succeeds, we could request apollo to send a refetch request to queries that need to be updated. Eg. in an address book, when we create a new address entry, we could request apollo to refetch relevant and related queries that might also have new data as a result of the create mutation, like a separate query to show a list of unique postcodes in the address book.

- This is not practical since it creates a poorer experience for users since the feedback will be shown only after two round trips - one is mutation and the other is the refetch query.

- We however sometimes will have no other option but to refetch and it's okay as a last resort

### Optimistic update

Optimistic update allows us to write predictable data into cache if we know ahead what it will be. This allows us to provide quicker experience feedback for our users. Eg. in an address book when the user marks an address as favourite, we know the mutation will respond with true but the mutation might take a few hundred ms to respond. So we use optimistic response to mark the address as favourite and show quicker UI feedback to our users. When the optimistic response and the actual mutation response match, apollo will ignore the response. If the actual response is different from optimistic response apollo will go through usual failure flow.

- use optimistic response only for edits where you can predict the response well
- don't use optimistic response for creates since you will need an id to insert into cache that you only usually get in mutation response

### Cache update

Cache updates are how we insert data into apollo cache after successful mutation response.

There are usually two ways to go about this —

**Complete payload:** When your mutation response has the entire payload that's needed in the cache, we can insert the entire payload into the cache and things will work fine.

Eg. In a social network, when the user creates a comment, we request the id, comment and all the relevant info necessary for the comment and insert it into the post to which it belongs to.

- This approach only works for entities that don't have nested entities and not needed in multiple interfaces using different queries.

- This approach is usually not practical for top level entities that has deep nested entities. Eg. Social network application has categorised topics, each topic will have posts, each post will have comments, etc. Now while creating a topic, it's not practical to fetch the entire nested entities in create mutation response.

**Partial payload:** This is the most practical approach and suits for most cases. Apollo is smart enough to refetch when there are entities with missing fields in cache for an active query. Eg. in an address book, if an address query needs 5 fields and if we insert a new address into cache with only 1 field, apollo identifies these missing fields and sends a refetch request for the address query to get latest data. We could make use of this mechanism for most create experiences. We only need to request id (or a few more relevant fields) in create mutations and insert that into where ever it needs to be inserted. We then let apollo take care of fetching whichever query is active and if that query is missing a required field. This way we will be able to build cache and mutation solutions independent of feature specific requirements and reuse these solutions across the app in different interfaces.

- Very practical. Always resort to this approach or at least try to.

## APIs for cache update

There are two ways to insert data into apollo's cache after mutation response or at any time using apollo's cache API.

`cache.updateFragment`: This is by far the easiest way to insert data into apollo cache. This cache API is very similar to how you write actual graphql queries/mutation. The only difference is that the updateFragment is a shorthand for `readFragment` + `writeFragment` together and instead of making a network request to read/write, apollo reads/writes from the cache. So `cache.updateFragment` is more of an apollo's way to reading and writing data to cache using the familiar graphql query/mutation schema.

- be warned and only use this when you want to completely replace data in cache or only when you want to write object shaped data into cache

- `cache.updateFragment` invokes the merge functions when you try to merge lists and this becomes problematic for pagination and leads unintended bugs

`cache.modify`: This is not an easy cache API to work with but by far the most stable and error-free way to work with cache. What you write with this API is exactly what goes into cache since this bypasses any defined merge functions. It's like brute forcing data into cache.

- use this for inserting data into lists that are paginated
