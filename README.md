#Redbird Reverse Proxy
##with built in Cluster and Docker support.


![redbird](http://cliparts.co/cliparts/6cr/o9d/6cro9dRzi.jpg)

Handling dynamic virtual hosts, load balancing, proxying web sockets and SSL encryption should be
easy and robust.

With redbird you get a complete library to build dynamic reverse proxies with the speed and robustness of http-proxy.

This light-weight package includes everything you need for easy reverse routing of your applications.
Great for routing many applications from different domains in one single host, handling SSL with ease, etc.

Developed by [manast](http://twitter.com/manast)

[![BuildStatus](https://secure.travis-ci.org/OptimalBits/redbird.png?branch=master)](http://travis-ci.org/OptimalBits/redbird)
[![NPM version](https://badge.fury.io/js/redbird.svg)](http://badge.fury.io/js/redbird)

##Features

- Flexible and easy routing.
- Websockets.
- Seamless SSL Support (HTTPS -> HTTP proxy)
- Automatic HTTP to HTTPS redirects.
- Load balancer.
- Register and unregister routes programatically without restart (allows zero downtime deployments)
- Docker support for automatic registration of running containers
- Cluster support that enables automatic multi-process.
- Based on top of rock-solid node-http-proxy and battle tested on production in many sites.
- Optional logging based on bunyan.

##Install


```sh
npm install redbird
```

##Example


You can programatically register or unregister routes dynamically even if the proxy is already running:

```js
var proxy = require('redbird')({port: 80});

// OPTIONAL: Setup your proxy but disable the X-Forwarded-For header
var proxy = require('redbird')({port: 80, xfwd: false});

// Route to any global ip
proxy.register("optimalbits.com", "http://167.23.42.67:8000");

// Route to any local ip, for example from docker containers.
proxy.register("example.com", "http://172.17.42.1:8001");

// Route from hostnames as well as paths
proxy.register("example.com/static", "http://172.17.42.1:8002");
proxy.register("example.com/media", "http://172.17.42.1:8003");

// Subdomains, paths, everything just works as expected
proxy.register("abc.example.com", "http://172.17.42.4:8080");
proxy.register("abc.example.com/media", "http://172.17.42.5:8080");

// Route to any href including a target path
proxy.register("foobar.example.com", "http://172.17.42.6:8080/foobar");

// You can also enable load balancing by registering the same hostname with different
// target hosts. The requests will be evenly balanced using a Round Robin scheme.
proxy.register("balance.me", "http://172.17.40.6:8080");
proxy.register("balance.me", "http://172.17.41.6:8080");
proxy.register("balance.me", "http://172.17.42.6:8080");
proxy.register("balance.me", "http://172.17.43.6:8080");



```


##About HTTPS

The HTTPS proxy supports virtual hosts by using SNI (which most modern browsers support: IE7 and above).
The proxying is performed by hostname, so you must use the same SSL certificates for a given hostname independently of its paths.


##HTTPS Example

Conceptually HTTPS is easy, but it is also easy to struggle getting it right. With redbird its straightforward, check this complete example:

1) Generate a localhost development SSL certificate:

```sh
/certs $ openssl genrsa -out dev-key.pem 1024
/certs $ openssl req -new -key dev-key.pem -out dev-csr.pem

// IMPORTANT: Do not forget to fill the field! Common Name (e.g. server FQDN or YOUR name) []:localhost

/certs $ openssl x509 -req -in dev-csr.pem -signkey dev-key.pem -out dev-cert.pem

```

Note: For production sites you need to buy valid SSL certificates from a trusted authority.

2) Create a simple redbird based proxy:

```js
var redbird = new require('redbird')({
	port: 8080,

	// Specify filenames to default SSL certificates (in case SNI is not supported by the 
	// user's browser)
	ssl: {
		port: 8443,
		key: "certs/dev-key.pem",
		cert: "certs/dev-cert.pem",
	}
});

// Since we will only have one https host, we dont need to specify additional certificates.
redbird.register('localhost', 'http://localhost:8082', {ssl: true});
```

3) Test it:

Point your browser to ```localhost:8000``` and you will see how it automatically redirects to your https server and proxies you to 
your target server.


You can define many virtual hosts, each with its own SSL certificate. And if you do not define any, they will use the default one
as in the example above:

```js
redbird.register('example.com', 'http://172.60.80.2:8082', {
	ssl: {
		key: "../certs/example.key",
		cert: "../certs/example.crt",	
		ca: "../certs/example.ca"
	}
});

redbird.register('foobar.com', 'http://172.60.80.3:8082', {
	ssl: {
		key: "../certs/foobar.key",
		cert: "../certs/foobar.crt",	
	}
});
```

You can also specify https hosts as targets and also specify if you want the connection to the target host to be secure (default is true).

```js
var redbird = require('redbird')({
	port: 80,
	secure: false,
	ssl: {
		port: 443,
		key: "../certs/default.key",
		cert: "../certs/default.crt",
	}
});
redbird.register('tutorial.com', 'https://172.60.80.2:8083', {
	ssl: {
		key: "../certs/tutorial.key",
		cert: "../certs/tutorial.crt",
	}
});

```

##Docker support
If you use docker, you can tell Redbird to automatically register routes based on image
names. You register your image name and then everytime a container starts from that image,
it gets registered, and unregistered if the container is stopped. If you run more than one
container from the same image, redbird will load balance following a round robin schema:

```js
var redbird = require('redbird')({
  port: 8080,
});

require('redbird')
  .docker(redbird)
  .register("example.com", 'company/myimage:latest');
```

##Cluster support
Redbird support automatic support for node cluster. Just specify in the options object
the number of processes that you want redbird to use. Redbird will automatically re-start
any thread thay may crash automatically, increasing even more its reliability.

```js
var redbird = new require('redbird')({
	port: 8080,
  cluster: 4
});
```

##NTLM support
If you need NTLM support, you can tell Redbird to add the required header handler. This 
registers a response handler which makes sure the NTLM auth header is properly split into
two entries from http-proxy.

```js
var redbird = new require('redbird')({
  port: 8080,
  ntlm: true
});
```


##Custom Resolvers

With custom resolvers, you can decide how the proxy server handles request. Custom resolvers allow you to extend redbird considerably. With custom resolvers, you can perform the following:

- Do path-based routing
- Do wildcard domain routing.
- Use variable upstream servers based on availability, for example in conjunction with Etcd or any other service discovery platform.
- And more.

Resolvers should be:

  1. Be invokable function. The `this` context of such function is the Redbird Proxy object. The resolver function takes in two parameters : `host` and `url`
  2. Have a priority, resolvers with higher priorities are called before those of lower priorities. The default resolver, has a priority of 0.
  3. A resolver should return a route object or a string when matches it matches the parameters passed in. If string is returned, then it must be a valid upstream URL, if object, then the object must conform to the following:

```
  {
     url: string or array of string [required],
     path: path prefix for route, [optional], defaults to '/',
     opts: {} //redbird target options, see Redbird.register() [optional],
  }
```

### Defining Resolvers

Resolvers can be defined when initializing the proxy object with the `resolvers` parameter. An example is below:

```javascript
 //for every URL path that starts with /api/, send request to upstream API service
 var customResolver1 = function(host, url) {
   if(/^\/api\//.test(url)){
      return 'http://127.0.0.1:8888';
   }
 };

 //assign high priority
 customResolver1.priority = 100;

 var proxy = new require('redbird')({
    port: 8080,
    resolvers: [
    customResolver1,
    //uses the same priority as default resolver, so will be called after default resolver
    function(host, url) {
      if(/\.example\.com/.test(host)){
        return 'http://127.0.0.1:9999'
      }
    }]
 })

```

### Adding and Removing Resolvers at Runtime.

You can add or remove resolvers at runtime, this is useful in situations where your upstream is tied to a service discovery service system.

```javascript
var topPriority = function(host, url) {
  return /app\.example\.com/.test(host) ? 'http://127.0.0.1:8000' : null;
};

topPriority.priority = 200;
proxy.addResolver(topPriority);


//remove top priority after 10 minutes,
setTimeout(function() {
  proxy.removeResolver(topPriority);
}, 600000);
```


##Roadmap

- Statistics (number of connections, load, response times, etc)
- CORS support.
- Rate limiter.
- Simple IP Filtering.
- Automatic routing via Redis or Etcd backend.


##Reference

<a name="redbird"/>
###Redbird(opts)

This is the Proxy constructor. Creates a new Proxy and starts listening to
the given port.

__Arguments__

```
    opts {Object} Options to pass to the proxy:
    {
    	port: {Number} // port number that the proxy will listen to.
    	ssl: { // Optional SSL proxying.
    		port: {Number} // SSL port the proxy will listen to.
    		// Default certificates
    		key: keyPath,  
    		cert: certPath,
    		ca: caPath // Optional.
            redirect: true, // Disable HTTPS autoredirect to this route.
    	}
        bunyan: {Object} Bunyan options. Check [bunyan](https://github.com/trentm/node-bunyan) for info.
        If you want to disable bunyan, just set this option to false. Keep in mind that
        having logs enabled incours in a performance penalty of about one order of magnitude per request.
        resolvers: {Function | Array}  a list of custom resolvers. Can be a single function or an array of functions. See more details about resolvers above.
	}
```

---------------------------------------

<a name="register"/>
#### Redbird##register(src, target, opts)

Register a new route. As soon as this method is called, the proxy will
start routing the sources to the given targets.
  
__Arguments__

```javascript
    src {String} {String|URL} A string or a url parsed by node url module.
    	Note that port is ignored, since the proxy just listens to one port.

    target {String|URL} A string or a url parsed by node url module.
    opts {Object} route options:
    examples:
    {ssl : true} // Will use default ssl certificates.
    {ssl: {
        redirectPort: port, // optional https port number to be redirected if entering using http.
    	key: keyPath,
    	cert: certPath,
    	ca: caPath // optional
    	}
    }
```

---------------------------------------

<a name="unregister"/>
#### Redbird##unregister(src, [target])

 Unregisters a route. After calling this method, the given route will not
 be proxied anymore.
  
__Arguments__

```javascript
    src {String|URL} A string or a url parsed by node url module.
    target {String|URL} A string or a url parsed by node url module. If not 
    specified, it will unregister all routes for the given source.
```

---------------------------------------

<a name="close"/>
#### Redbird##close()

 Close the proxy stoping all the incoming connections.
 
---------------------------------------





