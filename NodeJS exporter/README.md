**Adding dependencies to package.json**

To start monitoring the Node.js engine we need a prometheus client for node.js that supports histogram, summaries, gauges and counters. Prometheus themselves recommend prom-client as the best option, which you can install with

`npm install --save prom-client`

Not only do we want to monitor the health of the app, we also want to see the response times for the various actions. So we need a middleware that records the response time for requests in HTTP servers.

`npm install --save response-time`

These two dependencies will make sure we can monitor the Node.js engine as well as collect the response times.
Creating a module

Node.js has a simple module loading system. In Node.js, files and modules are in one-to-one correspondence (each file is treated as a separate module). To follow the best practices of Node.js, and make our code as reusable and readable as possible, we'll create a new module and use that in our main code later on. It is a lot of JavaScript code, but I've really done my best to document it.

`

/**
 * Newly added requires
 */

var Register = require('prom-client').register;  
var Counter = require('prom-client').Counter;  
var Histogram = require('prom-client').Histogram;  
var Summary = require('prom-client').Summary;  
var ResponseTime = require('response-time');  
var Logger = require('./logger');

/**
 * A Prometheus counter that counts the invocations of the different HTTP verbs
 * e.g. a GET and a POST call will be counted as 2 different calls
 */

module.exports.numOfRequests = numOfRequests = new Counter({  
    name: 'numOfRequests',
    help: 'Number of requests made',
    labelNames: ['method']
});

/**
 * A Prometheus counter that counts the invocations with different paths
 * e.g. /foo and /bar will be counted as 2 different paths
 */

module.exports.pathsTaken = pathsTaken = new Counter({  
    name: 'pathsTaken',
    help: 'Paths taken in the app',
    labelNames: ['path']
});

/**
 * A Prometheus summary to record the HTTP method, path, response code and response time
 */

module.exports.responses = responses = new Summary({  
    name: 'responses',
    help: 'Response time in millis',
    labelNames: ['method', 'path', 'status']
});

/**
 * This funtion will start the collection of metrics and should be called from within in the main js file
 */

module.exports.startCollection = function () {  
    Logger.log(Logger.LOG_INFO, `Starting the collection of metrics, the metrics are available on /metrics`);
    require('prom-client').collectDefaultMetrics();
};

/**
 * This function increments the counters that are executed on the request side of an invocation
 * Currently it increments the counters for numOfPaths and pathsTaken
 */

module.exports.requestCounters = function (req, res, next) {  
    if (req.path != '/metrics') {
        numOfRequests.inc({ method: req.method });
        pathsTaken.inc({ path: req.path });
    }
    next();
}

/**
 * This function increments the counters that are executed on the response side of an invocation
 * Currently it updates the responses summary
 */

module.exports.responseCounters = ResponseTime(function (req, res, time) {  
    if(req.url != '/metrics') {
        responses.labels(req.method, req.url, res.statusCode).observe(time);
    }
})

/**
 * In order to have Prometheus get the data from this app a specific URL is registered
 */

module.exports.injectMetricsRoute = function (App) {  
    App.get('/metrics', (req, res) => {
        res.set('Content-Type', Register.contentType);
        res.end(Register.metrics());
    });
};

`

Adding code to server.js

Now from the server.js file you only need a few lines of JavaScript code

`

'use strict';

...
/**
 * This creates the module that we created in the step before.
 * In my case it is stored in the util folder.
 */

var Prometheus = require('./util/prometheus');  
...

/**
 * The below arguments start the counter functions
 */

App.use(Prometheus.requestCounters);  
App.use(Prometheus.responseCounters);

/**
 * Enable metrics endpoint
 */

Prometheus.injectMetricsRoute(App);

/**
 * Enable collection of default metrics
 */

Prometheus.startCollection();  
...

`
By only adding 5 lines of code in the server.js file you can effectively monitor your app!

Deploy it and modify the your Prometheus `yml` file.

eg:

`
  - job_name: 'NodeJS Server'

    scrape_interval: 50s

    static_configs:

      - targets: ['NodeJSserver:4000']
`

Referance: [https://community.tibco.com/wiki/monitoring-your-nodejs-apps-prometheus](https://community.tibco.com/wiki/monitoring-your-nodejs-apps-prometheus)