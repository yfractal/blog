# Slack’s Incident on 2-22-22 Reading Note

## Why?

We can learn many things from an incident report, such as how to react to the incident, peek at the architecture, and analyze problems in the real case.

Have to say, it is much more enjoyable to learn from others' incidents. 

## Affect

On February 22, 2022 from 6:00 AM PST to 9:14 AM PST, some customers can't access Slack[2].

## Roughly Timeline

1. user tickets
2. alarms received
3. detected database suffers higher load than usual
4. query timeout, DB overloaded
5. decrease the requests acceptance rate 
6. increase the rate => DB overloaded again
7. decrease the rate => back to normal => increase by smaller increments

## The Architecture

Let's go through how the cach works briefly.

### Read

For reading, should be something like:

``` 
v = get(k)

if v is nil {
  v = fetch from DB
  set(k, v)
}
```

### The cache layer

![Screen Shot 2022-04-29 at 12 26 31 AM](https://user-images.githubusercontent.com/3775525/165799822-50bc0477-60ce-49ee-98b2-4efd6ae6c91d.png)

Client(app server) requests will be send to mcrouter, then the mcrouter will proxy the requests to memcached servers.


### The Consul + memcached
The team wanted to upgrade Consul agent and then need to restart a memcached node.

> When the agent restart occurs on a memcached node, the node that leaves the service catalog gets replaced by Mcrib [1]

From the above sentence, we can imply the Consul agent and memcached service run in the same node and when they want to upgrade Consul agent, they need to restart memcached. 

So the architecture should be:

<img width="410" alt="Screen Shot 2022-04-29 at 12 29 44 AM" src="https://user-images.githubusercontent.com/3775525/165800411-e31f8336-5324-481c-92b8-357225dfadc8.png">

## What happened

### Cache failure causes hit rate decreased

It's the trigger of the incident.

upgrade Consul agent => need restart memcached node => cache hit decreased

### DB heavy load + read amplification => DB overload

Cache reduced a lot of reading throughput for DB. When cache failure happens, DB will suffer heavy load than usual.

For Slack, one function/API needs to query many DB shards as it is not sharded well.

=> DB overload

### Cascading failure

![Screen Shot 2022-04-29 at 9 35 05 AM](https://user-images.githubusercontent.com/3775525/165872687-9f9966e7-d2bb-41bc-a095-bae4e874c179.png)

## Thinking in general

After fond out what happens, we can think about this incident in general and then learn from others' experiences.

### Cache failure

After we add a cache layer it will have two effects: speed up requests and reduce DB read load[7].

Most applications are read-heavy, for example in Facebook, they have two orders of magnitude more reads than writes[3].

Another fact is cache is much faster say 10 times than DB usually.

So if the cache crashed, DB will suffer a really heavy load most time. So we need to make sure the cache works well in any situation.

For caching, Facebook published a good paper[3] about how they design their cache system.

### High availability

For achieving high availability, we need to ask "what happens if it fails?"[8].

Any failures can stuck(livelock[6]) or crash(cascading failure) the whole system.

We have two options, fix all potential issues or make the system recover fastly.

Options one is infeasible because there are too many possibilities.

But we can achieve fast recovery, that coverts uncertainty problems into certainty problems.

For GFS paper[7], they think "component failures are the norm rather than the exception."

And they handle availability problems through fast recovery and replication.

### Overload control

Most problems are caused by overload and no one can guarantee the system is always under-loaded.

To avoid such problems, we can design some mechanisms to protect our systems.

The most obvious way is the rate limiter, it sets a hard value and when throughput is over the value, the over-part requests will be rejected.

One inconvenience of the rate limiter is we need to dynamic change it, we can see this in the incident.

Microservices have complexity dependency and each part is dynamic changed(software upgrade, hardware config update, or something else), so it's impossible to find a good value for all situations.

Netflix created [concurrency-limits](https://github.com/Netflix/concurrency-limits) based on such observations.

Another thing we need consider is how to provide best user experience we can when we drop users' requests.

Another thing we need to consider is how to provide the best user experience when we drop users' requests.

"Overload Control for Scaling WeChat Microservices"[5] has a good answer for this problem

In the paper, they drop requests based on features and users. And they use queue time to detect overload instead request delay.

## References
1. Slack’s Incident on 2-22-22
   https://slack.engineering/slacks-incident-on-2-22-22/
2. Slack status history
   https://status.slack.com/2022-02-22
3. Scaling Memcache at Facebook at NSDI '13
   https://www.usenix.org/conference/nsdi13/technical-sessions/presentation/nishtala
4. MIT 6.824 Spring 2022 LEC 16
   https://pdos.csail.mit.edu/6.824/schedule.html
5. Hao Zhou. Overload Control for Scaling WeChat Microservices, 2018
6. J. C. Mogul and K. Ramakrishnan. Eliminating receive livelock in an interrupt-driven kernel. 1997.
7. Sanjay Ghemawat. The Google File System 2003
8. MIT 6.824 Spring 2020 Lec 16
   http://nil.csail.mit.edu/6.824/2020/schedule.html
9. J Armstrong. A history of Erlang. 2007
