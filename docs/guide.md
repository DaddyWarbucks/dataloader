## Reusing loaders

Pass the loader to any and all service calls. This maximizes performance by allowing the loader to re-use its cache and batching mechanism as much as possible. If you are using Feathers v5, you can also use Node's `AsyncLocalStorage` mechanism to automatically pass the loader from service to service.

```js
const { AppLoader } = require('feathers-dataloader');
const { AsyncLocalStorage } = require('async_hooks');

const asyncLocalStorage = new AsyncLocalStorage();

const initializeLoader = async (context, next) => {
  if (context.params.loader) {
    return next();
  }

  const store = asyncLocalStorage.getStore();

  if (store && store.loader) {
    context.params.loader = store.loader;
    return next();
  }

  const loader = new AppLoader({ app: context.app });

  asyncLocalStorage.run({ loader }, async () => {
    context.params.loader = loader;
    return next();
  });
};

app.hooks({
  around: [initializeLoader]
})

// No need to manually pass the loader because AsyncLocalStorage
// in the initializeLoader functions automatically passes it!
const postResultsResolver = resolve({
  properties: {
    user: (value, post, context) => {
      const { loader } = context.params;
      return loader.service('users').load(post.userId);
    }
  }
});

app.service('posts').hooks({
  after: {
    all: [resolveResult(postResultsResolver)]
  }
});
```

## Use loaders EVERYWHERE

The more the loader is used, the better performance can be. This is generally accomplished by passing the loader from service to service in hooks and resolvers. But, you can use loaders in other places which will also lead to performance gains.

```js
const validateUserId = async (context) => {
  const { loader } = context.params;
  const { userId } = context.data;

  // Note we use the loader to lookup this user
  const user = await loader.service('users').load(userId);

  if (!user) {
    throw new Error('Invalid userId');
  }

  return context;
};

const postResultsResolver = resolve({
  properties: {
    user: (value, post, context) => {
      const { loader } = context.params;
      // We get user for free here! The loader is already cached
      // because we used it in the validateUserId before hook.
      return loader.service('users').load(post.userId);
    }
  }
});

app.service('posts').hooks({
  before: {
    create: [validateUserId],
    update: [validateUserId],
    patch: [validateUserId],
  },
  after: {
    all: [resolveResult(postResultsResolver)]
  }
});
```

## Clear loaders after mutation

Even though loaders are generally created/destroyed with each request, its good practice to clear the cache after mutations. When using `AppLoader` and `ServiceLoader`, there is only one method `clear()` to clear the loader caches. Because of the lazy config when using these classes, its difficult for the developer to know all of the potential method/ids/params combos that may be cached. Instead, the `clear()` method dumps the whole cache. If any subsequent calls are made to the loader for this service it will return new results.

```js
const clearLoaderCache = async (context) => {
  const { loader } = context.params;

  // Clear this service's cache
  loader.service('posts').clear();

  // This update may have affected other services too.
  // If you know what services were changed you can clear them.
  loader.service('posts').clear();

  // Or you can clear all services at once
  loader.clear();

  return context;
};

app.service('posts').hooks({
  after: {
    all: [clearLoaderCache]
  }
});
```

But, you can get access to the underlying caches as well. You should use this with caution...cache invalidation is hard. If you are considering doing this, it may be best to use `DataLoader` classes directly so you can better control each loader.

```js
const clearLoaderCache = async (context) => {
  const loader = context.params.loader.service('users');
  loader.cacheMap.forEach((result, key) => {
    // Write code that will break your app...
  });
  return context;
};

app.service('posts').hooks({
  after: {
    all: [clearLoaderCache]
  }
});
```

## Use a `maxBatchSize`

The ServiceLoader's underlying DataLoader takes a `maxBatchSize` option. This option limits the number of ids in the `$in` query. Huge arrays of ids can lead to performance issues and even database lockup. By using the `maxBatchSize` you can break those queries into smaller ones. You should monitor your own application to determine the best number to use for your batch size, but setting some maximum is recommended.

```js
const { AppLoader, ServiceLoader } = require('feathers-dataloader');

const loader = new AppLoader({ app, maxBatchSize: 100 });

await Promise.all([
  // ...1,000 service calls
  loader.service('users').load(id)
])

// The query will be broken up into 10 calls with 100 ids each
const query = {
  id: { $in: [...100 ids] }
}
```

## Don't mutate loader results

Loaders returned cached results each time you call a method. This is dissimilar to standard `get()` and `find()` methods in Featers where we expect each set of results to be unique each time a method is called. Take care not to mutate loader results as you work with them.

```js
// For get/find requests, each result is unique
const user1 = await app.service('users').get(1);
const user2 = await app.service('users').get(1);
user1.name = 'FeathersUser';
user2.name === 'FeathersUser' // false

// For each method of a loader, results are cached and shared
const user1 = await loader.service('users').load(1);
const user2 = await loader.service('users').load(1);
user1.name = 'FeathersUser';
user2.name === 'FeathersUser'; // true
```


## Params Caching

Loaders use a stringified copy of the id and params used for each method call. This means that functions, classes, transactions, etc cannot be used in a cache key. By default, this library recursively traverses params and removes any functions from the params before stringifying them. But, you may need to create a custom strategy for your use case.

```js

const { AppLoader } = require('feathers-dataloader');

const cacheParamsFn = params => {
  return {
    user: params.user,
    query: params.query
  }
}

// Use the global configuration.
const loader = new AppLoader({ app, cacheParamsFn });

// Configure each service independently.
// This will override the global configuration.
const loader = new AppLoader({
  app
  services: {
    posts: {
      cacheParamsFn
    }
  }
});

// Pass a cacheParamsFn on each method invocation.
// This will override the service and global configuration.
const result = loader.service('users').load(1, params, cacheParamsFn);
```

## Extending the base class

The AppLoader constructor takes an option that allows you to pass a class that will be used to create each new ServiceLoader. You can use this class to extend the base ServiceLoader class with your own methods.

```js
const { AppLoader, ServiceLoader } = require('feathers-dataloader');

class MyLoader extends ServiceLoader {
  constructor(...args) {
    super(...args)
  }

  async findOne(params) {
    const cacheKey = this.stringifyKey({ method: 'findOne', params });

    const cachedResult = await this.cacheMap.get(cacheKey);

    if (cachedResult) {
      return cachedResult
    }

    const result = await this.options.service.find({
      ...params,
      query: {
        ...params.query,
        $limit: 1
      }
    })

    const found = result.data[0] || null;

    await this.cacheMap.set(cacheKey, found);

    return found;
  }

  async count(params) {
    const cacheKey = this.stringifyKey({ method: 'count', params });

    const cachedResult = await this.cacheMap.get(cacheKey);

    if (cachedResult) {
      return cachedResult;
    }

    const result = await this.options.service.find({
      ...params,
      query: {
        ...params.query,
        $limit: -1
      }
    })

    await this.cacheMap.set(cacheKey, result.total);

    return result.total;
  }
}

const loader = new AppLoader({ app, ServiceLoader: MyLoader });

const result = loader.service('users').findOne(params);
const result = loader.service('users').count(params);
```