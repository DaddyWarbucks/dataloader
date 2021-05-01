# feathers-ecosystem/batch-loader

<!-- [![Dependency Status](https://img.shields.io/david/feathers-plus/batch-loader.svg?style=flat-square)](https://david-dm.org/feathers-plus/batch-loader)
[![Download Status](https://img.shields.io/npm/dm/@feathers-plus/batch-loader.svg?style=flat-square)](https://www.npmjs.com/package/@feathers-plus/batch-loader) -->

> Reduce requests to backend services by batching calls and caching records.

## Installation

```
npm install @feathers-plus/batch-loader --save
```

## Documentation

Please refer to the [batch-loader documentation](./docs/index.md) for more details.

## Basic Example

Use the `loaderFactory` static method to create a basic batch-loader. This is simply syntatic sugar for manually creating a batch-loader. This "Basic Example" and "Complete Example" create the same batch-loader.

```js
const BatchLoader = require("@feathers-plus/batch-loader");

const usersBatchLoader = BatchLoader.loaderFactory(
  app.service("users"),
  "id",
  false
);

app
  .service("comments")
  .find()
  .then((comments) =>
    Promise.all(
      comments.map((comment) => {
        // Attach user record
        return usersBatchLoader
          .load(comment.userId)
          .then((user) => (comment.userRecord = user));
      })
    )
  );
```

## Complete Example

Use the `BatchLoader` class to create more complex loaders. These loaders can call other services, call DB's directly, or even call third party services. This example manually implements the same loader created with the `loaderFactory` above.

```js
const BatchLoader = require("@feathers-plus/batch-loader");
const { getResultsByKey, getUniqueKeys } = BatchLoader;

const usersBatchLoader = new BatchLoader((keys) =>
  app
    .service("users")
    .find({ query: { id: { $in: getUniqueKeys(keys) } } })
    .then((result) => getResultsByKey(keys, result, (user) => user.id, "!"))
);

app
  .service("comments")
  .find()
  .then((comments) =>
    Promise.all(
      comments.map((comment) => {
        // Attach user record
        return usersBatchLoader
          .load(comment.userId)
          .then((user) => (comment.userRecord = user));
      })
    )
  );
```

## License

Copyright (c) 2017 John J. Szwaronek

Licensed under the [MIT license](LICENSE).
