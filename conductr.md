# ConductR: Typesafe's move in the decade of lightweight containers and age of distributed clusters...

During my career as an administrator I've tried multiple well-known and developed a few home-grown ways of dealing with applications created in micro-services architecture.
Every time I fell into pitfalls of these approaches: the problem of maintaining them in reactive way.
ConductR is a Typesafe's response to this problem, addressing full life cycle of an application in a way convenient for both the Developers and Operators.

## As a Scala developer...
...one gets a nice set of tools: sbt plugins ```sbt-typesafe-conductr``` (which provides control commands for loading, unloading, starting, etc. of application bundles onto the ConductR cluster) and ```sbt-bundle``` (for creating application bundles) and also Scala libraries ```LocationService``` (for service discovery) and ```StatusService``` (for reporting application state).

Wherever you run an application, either in the production environment or during development on local machine your code can behave correctly due to easy-to-use resolving of other bundle services:
```
val accounts = LocationService.getLookupUrl("/accounts", "http://127.0.0.1:9000/accounts")
```
On ConductR cluster this method call returns proper load balancer URL pointing to requested service, falling back to the provided default value otherwise. If you cannot use this library for any reason, just ask ConductR for it using REST API. Quite simple. 

"Is this application running or starting" problem has been addressed in similar fashion.
For each node, each application can be in one of following states:
* stored - bundle has been replicated onto this node,
* starting - it has been started,
* running - application started correctly *and* reported its status to ConductR.

Again, calling
```
StatusService.signalStartedOrExit()
```
as the last part of initialization will do the job perfectly.

## As an operator...
What got me mesmerized is the simplicity, just two new services:
* The first one (```conductr```) does most of the job: given a greedy developer with a dozen of his apps it will patiently receive the bundles from him, replicate them over the cluster and start desired number of instances ensuring proper resource usage distribution at the same time.
* The second one (```conductr-haproxy```) is a little bridge which works its magic on haproxy, reconfiguring the load-balancer after each change to the cluster. This handy service guarantees that your list of upstream nodes for each endpoint is always up to date. Almost instantly - I was never able to achieve this behavior in my pull-oriented implementations; long live message passing!


## As a hacker...
I'm a man who disassembles his newly bought phone even before turning it on. Given the Debian package of a application server intended to run Scala apps I've installed it on Fedora and deployed simple service written in Go.
Thanks to ```shazar``` tool provided with ```conduct``` CLI utils, after adding proper manifest to your application, you can bundle it in ConductR-compatible way. The only thing you'll need to do is reporting your own status back to status service. Isn't it a job for CURL?
```
root@debian:~/goxample# cat bundle.conf 
version    = "1.0.0"
name       = "goxample"
system     = "goxample-1.0.0"
nrOfCpus   = 1
memory     = 67108864
diskSpace  = 35000000
roles      = ["web-server"]
components = {
  "goxample" = {
    description      = "goxample"
    file-system-type = "universal"
    start-command    = ["./goxample $WEB_BIND_IP $WEB_BIND_PORT"]
    endpoints        = {
      "web" = {
        protocol  = "http"
        bind-port = 0
        services  = ["http://:8081"]
      }
    }
  }
  "goxample-status" = {
    description      = "fake goxample status"
    file-system-type = "universal"
    start-command    = ["curl", "-X PUT http://${CONDUCTR_STATUS_IP}:${CONDUCTR_STATUS_PORT}/bundles/${BUNDLE_ID}?isStarted=true"]
    endpoints        = {}
  }
}
```
Yep, we've deployed multiple versions of this bundle to test zero-downtime application updates.
After some REST calls, our haproxy directed the traffic to old, both and then only new one - it works perfect.

