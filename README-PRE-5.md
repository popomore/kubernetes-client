# kubernetes-client

[![Build Status][build]](https://travis-ci.org/godaddy/kubernetes-client) [![Greenkeeper badge][greenkeeper]](https://greenkeeper.io/)

[greenkeeper]: https://badges.greenkeeper.io/godaddy/kubernetes-client.svg
[build]: https://travis-ci.org/godaddy/kubernetes-client.svg?branch=master

Simplified [Kubernetes API](http://kubernetes.io/) client for Node.js.

This kubernetes-client API is deprecated. See [README.md](README.md)
for the new API that has better support for all Kubernetes API
endpoints.

## Installation

Install via npm:

```console
$ npm i kubernetes-client --save
```

## Examples

kubernetes-client provides access to all the Kubernetes objects and
includes some niceties for writing simpler code.

### Basics

kubernetes-client maps the URI paths in the Kubernetes API to
sequences of objects chained together via properties and ending in a
function. For example, to GET the `ReplicationController` named
'http-rc' in the `Namespace` 'my-project':

```js
const Api = require('kubernetes-client');
const core = new Api.Core({
  url: 'http://my-k8s-api-server.com',
  version: 'v1'  // Defaults to 'v1'
});

function print(err, result) {
  console.log(JSON.stringify(err || result, null, 2));
}

core.namespaces('my-project').replicationcontrollers('http-rc').get(print);
```

kubernetes-client supports the Extensions API group. For example, GET
the `Deployment` named `http-deployment`:

```js
const ext = new Api.Extensions({
  url: 'http://my-k8s-api-server.com',
  version: 'v1beta1'  // Defaults to 'v1beta1'
});

ext.namespaces('my-project').deployments('http-deployment').get(print);
```

kubernetes-client provides a helper to get in-cluster config and accessing the API from a Pod:

```js
const Api = require('kubernetes-client');
const core = new Api.Core(Api.config.getInCluster());
```

and a helper to get the current-context config from `~/.kube/config`:

```js
const Api = require('kubernetes-client');
const core = new Api.Core(Api.config.fromKubeconfig());
```

### **Experimental** support for promises and async/await

kubernetes-client has **experimental** support for promises. If you
omit callbacks, an HTTP method function (*e.g.*, `.get`), it will
return a promise.

```js
core.namespaces('my-project').replicationcontrollers('http-rc').get()
  .then(result => console.log(result));
```

or with `async/await`:

```js
console.log(await core.namespaces('my-project').replicationcontrollers('http-rc').get());
```

### Creating and updating

kubernetes-client objects expose `.post`, `.patch`, and `.put`
methods. Create the ReplicationController from the example above:

```js
const manifestObject = require('./rc.json');
core.namespaces('my-project').replicationcontrollers.post({ body: manifestObject }, print);
```
or update the number of replicas:

```js
const patch = { spec: { replicas: 10 } };
core.namespaces('my-project').replicationcontrollers('http-rc').patch({
  body: patch
}, print);
```

### Using the correct API group and version

kubernetes-client client includes functionality to help determine the
correct Kubernetes API group and version to use based on manifests:

```js
const Api = require('kubernetes-client').Api;
const api = new Api({ url: 'http://my-k8s-api-server.com' });

const manifest0 = {
  kind: 'Deployment',
  apiVersion: 'extensions/v1beta1'
  ...
};
const manifest1 = {
  kind: 'ReplicationController',
  apiVersion: 'v1'
  ...
};

api.group(manifest0).ns('my-project').kind(manifest0).post({ body: manifest0 }, print);
api.group(manifest1).ns('my-project').kind(manifest1).post({ body: manifest1 }, print);
```

### Object name aliases

kubernetes-client supports the same aliases as
[`kubectl`](http://kubernetes.io/docs/user-guide/kubectl-overview/)
(*e.g.*, `ns` for `namespaces`) and the singular versions of the
resource name (*e.g.*, `namespace` for `namespaces`). We can shorten
the example above:

```js
core.ns('my-project').rc('http-rc').get(print);
```

### Switching namespaces

You can call the `namespace` object to specify the namespace:

```js
core.ns('other-project').rc('http-rc').get(print);
```

### Query parameters

You can optionally specify query string object `qs` to GET
endpoints. kubernetes-client passes `qs` directly to
[`request`](https://www.npmjs.com/package/request#requestoptions-callback).
For example to filter based on [label
selector](http://kubernetes.io/docs/user-guide/labels/):

```js
core.ns('my-project').rc.get({ qs: { labelSelector: 'service=http,component=api' } }, print);
```

### Label selector filtering

kubernetes-client has a shortcut, `matchLabels`, for filtering on label
selector equality:

```js
core.ns('my-project').rc.matchLabels({ service: 'http' }).get(print);
```

and a more general `match` method based on Kubernetes Match Expressions:

```js
core.ns('my-project').rc.match([{
  key: 'service',
  operator: 'In',
  values: ['http']
}, {
  key: 'deploy',
  operator: 'NotIn',
  values: ['production', 'staging']
}]).get(print);
```

**Note:** The Kubernetes API supports label selector filtering [for GET
methods
only](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/). The
Kubernetes API does not support label selector filtering for POST,
PUT, PATCH, or DELETE.

### ThirdPartyResources

You can extend the Kubernetes API using a
[ThirdPartyResource](https://kubernetes.io/docs/user-guide/thirdpartyresources/)
and kubernetes-client:

```js
const newResource = {
  apiVersion: 'extensions/v1beta1',
  kind: 'ThirdPartyResource',
  metadata: {
    name: 'new-resource.kubernetes-client.io'
  },
  description: 'Example resource',
  versions: [{
    name: 'v1'
  }]
};

ext.thirdpartyresources.post({ body: newResource }, print);
```

and then extend an `ThirdPartyResource` API client with your new resources:

```js
const thirdPartyResources = new Api.ThirdPartyResources({
  url: 'http://my-k8s-api-server.com',
  group: 'kubernetes-client.io',
  resources: ['customresources']  // Notice pluralization!
});

// Access `customresources` as if they were a regular Kubernetes object
thirdPartyResources.ns('my-project').customresources.get(print);
thirdPartyResources.addResource('newresources');  // Notice pluralization!
// Now access `newresources`
thirdPartyResources.ns('my-project').newresources.get(print);
```

### Watching and streaming

You can call `.getStream` to stream results. This is useful for watching:

```js
const JSONStream = require('json-stream');
const jsonStream = new JSONStream();

const stream = core.ns('my-project').po.getStream({ qs: { watch: true } });
stream.pipe(jsonStream);
jsonStream.on('data', object => {
  console.log('Pod:', JSON.stringify(object, null, 2));
});
```

You can access logs in a similar fashion:
```js
const stream = core.ns('my-project').po('http-123').log.getStream({ qs: { follow: true } });
stream.on('data', chunk => {
  process.stdout.write(chunk.toString());
});
```

**Note:** the kube-apiserver eventually close watch connections
according to the
[`--min-request-timeout`](http://kubernetes.io/docs/admin/kube-apiserver/
command line argument. kubernetes-client does not attempt to reconnect
when the kube-apiserver closes a connection.

### Authentication

kubernetes-client supports Kubernetes [apiserver
authentication](http://kubernetes.io/docs/admin/authentication/).

Basic authentication (with optional certificate authority):

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  ca: fs.readFileSync('cluster-ca.pem'),
  auth: {
    user: 'user',
    pass: 'pass'
  }
});
```

or without a certificate authority:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  insecureSkipTlsVerify: true,
  auth: {
    user: 'user',
    pass: 'pass'
  }
});
```

token authentication:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  auth: {
    bearer: 'token'
  }
});
```

and client certificate authentication:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  ca: fs.readFileSync('cluster-ca.pem'),
  cert: fs.readFileSync('my-user-cert.pem'),
  key: fs.readFileSync('my-user-key.pem')
});
```

### Passing options to `request`

kubernetes-client uses
[`request`](https://github.com/request/request). You can specify
[`request`
options](https://github.com/request/request#requestoptions-callback)
for kubernetes-client to pass to `request`:

```js
const core = new Api.Core({
  url: 'https://my-k8s-api-server.com',
  request: {
    timeout: 3000
  }
});
```

### **Experimental** support for Swagger-based API clients

**This API is going to change.**

kubernetes-client has support for generating API clients from
Kubernetes Swagger documentation.

```js
const kubernetes = require('kubernetes-client');
const spec = require('./swagger-1.8.json');
const http = new kubernetes.Request(kubernetes.config.fromKubeconfig());
const client = new kubernetes.SwaggerClient({ spec, http });

console.log(await client.api.v1.namespaces.get());
```

## Testing

kubernetes-client includes unit tests and integration tests.
[Minikube](https://github.com/kubernetes/minikube) is a tool that
makes it easy to run integration tests locally.

Run the unit tests:

```console
$ npm test
```

The integration tests use a running Kubernetes server. You specify the
Kubernetes server context via the `CONTEXT` environment variable. For
example, run the integration tests with the `minikube` context:

```console
$ CONTEXT=minikube npm run test-integration
```

## More Documentation

* [Kubernetes Reference Documentation](http://kubernetes.io/docs/reference/)
* [Kubernetes Swagger Spec](http://kubernetes.io/kubernetes/third_party/swagger-ui/)

## License

[MIT](LICENSE)
