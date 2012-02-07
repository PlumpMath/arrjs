#HTTP and WebSocket application routing
===

You can expose several HTTP or WebSocket applications over a single TCP port using ARR.JS. This is useful for better 
utilization of servers, including shared hosting. 

- Host N applications on M servers, each application in K(N) instances, 1 <= K(N) <= M.
- Routing based on the Host HTTP request header.
- HTTP and HTTPS.
- WebSockets and secure WebSockets.
- Message-based process activation.
- Process monitoring and crash recovery. 
- Host applications built with arbitrary technology with its own HTTP stack, including node.js.
- Works on Windows, MacOS, and *nix. 
- Built with node.js and MongoDB. 

## Prerequisities

- Windows, MacOS, *nix (tested on Windows 7 & 2008 Server, MacOS Lion, Ubuntu 11.10)
- [node.js v0.7.0 or greater](http://nodejs.org/dist/). MacOS and *nix may use earlier versions.
- [MongoDB](http://www.mongodb.org/downloads). The database is used to store application metadata and must be 
accessible from all backends. 

## Getting started

Instructions below are for setting up a single machine deployment (e.g. development environemnt) on a MacOS. 
Other OSes are conceptually similar. 

Start unsecure MongoDB server on localhost:

```
mongod
```

Import application metadata for the three sample applications:

```
mongoimport -d arr -c apps src/apps.json
```

Configure your HOSTS file (/etc/hosts on MacOS and *nix, %systemroot%\system32\drivers\etc\hosts on Windows) 
to resolve domain names used by the sample applications to localhost by adding the following lines:

```
127.0.0.1 app1.janczuk.org
127.0.0.1 app1.tangyorange.com
127.0.0.1 app2.janczuk.org
127.0.0.1 app2.tangyorange.com
127.0.0.1 ws1.janczuk.org
127.0.0.1 ws1.tangyorange.com
```

Start the ARR.JS router to listen for unsecured traffic on port 80 and SSL traffic on port 443 
(these ports cannot not be used by other processes on the box)

```
cd src
sudo node arrdwas.js --mongo=mongodb://localhost/arr -p 80 -s 443
```

Issue a few requests to test the system:

```
curl http://app1.tangyorange.com
curl https://app1.janczuk.org -k
curl http://app2.janczuk.org
curl https://app2.janczuk.org -k
```

In your favorite modern browser navigate to ```http://ws1.janczuk.org```. You should see Dante's Divine Comedy 
streamed downed to you over a WebSocket connection, a stanza every 2 seconds. When connecting 
over ```https://ws1.janczuk.org``` you will first see a security warning because the certificate exposed by 
the application is not trusted (it is self-signed). 

## Deployment on a server farm

### Database

The MongoDB dabatase holds application metadata and must be accessible from all servers in the farm. 
You can use your own instance or get started with a free instance
provided by [MongoHQ](https://mongohq.com/home). Bottom line is you need a MongoDB connection URL to provide to 
all instances of arrdwas.js you will run
on the backends.

The application metadata must be stored in a single MongoDB collection called ```apps```. Each document in this collection 
must have the following structure:

```
{
  process: {
    executable: "node",                 // specify the executable name here, including path if needed
    args: [                             // specify whatever command line arguments must be passed to the executable
      "apps/app1/server.js"
    ]
  },
  hosts: [
    { 
      host: "app1.janczuk.org",
      ssl: "allowed",                       // you must say "allowed", "required", or "none" (for unsecure access only)
      cert: "certs/app1-janczuk-cert.pem",  // optional file with X.509 certificate associated with this host name
      key: "certs/app1-key.pem"             // optional file with the private key for the X.509 certificate
    },
    // ... you can specify multiple host names that can be used to access this application
  ],
  instances: 2,                             // maximum number of instances of this application to create
  machines: [                               // this is the array of machines the application is currently running on
  ]                                         // and it must be set to empty initially; it is managed by arr.js
}
```

For an example of application metadata, check out [apps.json](https://github.com/tjanczuk/arrjs/blob/master/src/apps.json), 
a file that can be imported into the MongoDB database using the 
[mongoimport](http://www.mongodb.org/display/DOCS/Import+Export+Tools#ImportExportTools-mongoimport) tool.

### Backends

Each server in the farm dedicated to running applications must be running an instance of arrdwas.js. Typically each of 
the instances of arrdwas.js would be configured identically, which allows any run of the mill TCP level load balancer
to be put in front of the server farm. Ardwas.js accepts the following parameters:

```
Usage: node ./arrdwas.js

Options:
  -m, --mongo    Mongo DB connecton string                      [default: "mongodb://localhost/arr"]
  -r, --range    Managed TCP port range                         [default: "8000-9000"]
  -p, --port     Unsecured listen port                          [default: 31415]
  -s, --sslport  SSL listen port                                [default: 31416]
  -c, --cert     Non-SNI (wildcard) server certificate for SSL  [default: "certs/wildcard-janczuk-cert.pem"]
  -k, --key      Private key for SSL                            [default: "certs/wildcard-janczuk-key.pem"]
```

The [MongoDB connetion URL](http://www.mongodb.org/display/DOCS/Connections) (-m) must point to the central MongoDB 
database with the ```apps``` collection that holds the application metadata. 

The managed TCP port range (-r) indicates the range of TCP ports arrdwas.js will assign to managed applications when 
they are activated. The TCP port number assigned to an instance of an application is passed to the application through
the process environment variable named ```PORT```. For example, in case of node.js applications, this value is accessible 
via the ```process.env.PORT``` property and can be used to set up an HTTP listener. 
See [server.js](https://github.com/tjanczuk/arrjs/blob/master/src/apps/app1/server.js) for an example. 

The unsecured (-p) and secured (-s) port number are the two TCP port numbers arrdwas.js will listen on. Arrdwas.js will 
accept HTTP and WS traffic over the unsecured port, and HTTPS and WSS traffic over the secured port. SSL security terminates 
at arrdwas.js. Arrdwas.js acts as an HTTP[S]/WS[S] reverse proxy, routing incoming requests to appropriate applications, 
and activating them if necessary. Communication between arrdwas.js and an instance of a application is not secured with SSL
(i.e. HTTP or WS), so the application code should always set up their listeners for HTTP and WS, regardless whether client
calls arrive over SSL or not. Appropriate set of proxy ```x-forwarded-*``` headers that arrdwas.js adds to the requests 
allow applications to learn more about the original client connection. 

Each instance of arrdwas.js is configured with an X.509 certificate (-c) and the associated private key (-k). This
certificate and private key are used to identify the server during the SSL handshake, unless application specific
configuration overrdies this setting by providing its own certificate and private key to be used  with 
[Server Name Identification (SNI)](http://en.wikipedia.org/wiki/Server_Name_Indication). Note that for SNI to take effect
the client must support it, but most modern browsers do. If you plan to support SSL for multiple applications with 
different host names and use a single server side certificate to secure them, the domain suffix of all these applications
must be the same (e.g. foo.barbaz.com and baz.barbaz.com), and the certificate must be a wildcard certificate (e.g. 
CN=*.barbaz.com). 