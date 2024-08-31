---
layout: post
title:  "Rate Limiter 2 - concurrency and scaling"
date:   2024-08-28 13:36:58 -600
categories: system design
---
## Rate Limiter - concurrency and scaling

This is part two of Rate limiter implementation. In first part we looked at an implementation with storing the client ids in a hashmap with the value being a list of K sized priority queue. The priority queue part was implicit since we assume we will get all requests in sequence.
Tackle in todays assignment
- what happens if the requests come out of order ?
- what happens if multiple requests come at the same time ?

### Concurrency
In part 1 the implementation has two data stores.
1. a hashmap of client id as key
2. a priority queue of timestamps

If we get multiple requests at the same time from multiple / same clients, then the current implementation will fail. 
The point of failure will the access to the hashmap. This will cause a race condition.

To prevent this race condition, the data needs to be protected with a mutex. 

This is more difficult than imagined because the class would be a facade to access the access_cache.
So the mutex lock should not be for the whole access cache but instead client specific.

{% highlight python %}
from threading import Lock

class ClientAccess:
    """ a wrapper class to provide RAII access to __last access times__ """
    def __init__(self, clientId):
        self.client_id = clientId
        self.access_mutex = Lock()

    def isAllowed(self, client_id, access_time):
        ''' returns true, if access allowed and store the latest time '''
        local_mutex = Mutex()
        ...
        return # the return will trigger the destruction of the mutex 

{% endhighlight %}

#### Analysis
this looks and sounds good. But does it work?

Lets look at a commmon webserver and how they handle multiple concurrent requests.
Techincally the webserver is listening on a port. So there is no way a port can recieve more than concurrent request.
Of course since its TCP, the pot would be recieving the data but I do not know if t

[[How server supports servicing multiple clients]]
We go on a small detour. The server is listening on a port, say port 80 for tcp connections. They are not http data at this point.
Once it gets the tcp connection, then the server has to call an accept api to accpet the connection. At this point 
a new connection is established. 
This connection can uniquely be identified by [client ip, client port, server ip, server port]
The server process handles / accepts the connection and proceeds to service the connection, then its not listening on the port.
If the server offloads the execution of the accept and further process to another thread, then its available to go back and do a listen on the port.
Thus the server can support concurrent requests.

Server offload:
1. Async, in languages / runtimes which support cooperative multitasking, main thread can continue to process the accept and do the server process until it encounters an io block. And IO block in this case cannot be any io block but explicit I/O wait supported by the language / runtime. 
When it encounters this I/O block and instead of waiting for the block/ wait to resolve, the runtime or the tool can hold the stack here and go back to the listening on the port.
Listening on the port is also another I/O wait and hence the main loop is idle until one of the IO waits resolved.

2. Multi threaded. When the incoming request is accepted, we can delegate the work to a thread pool or a dynamically created thread. Note thread creation / process creation has its own performance overhead. It is better to use a thread pool over dynamic thread creation. There is also a problem of determinism. the thread / lwp could be created but not initialized. And during the first writeback there would be an cpu utilization spike due to minor page faults.

3. Multi process. This is similar to multiprocess but instead of thread a process would handle the one incoming requests.

Here is an answer from ChatGpt on the various webservers and the kind of concurrency they support.
Here's a table of some popular servers and their support for multi-process, multi-threaded, and asynchronous models:

| **Server**                  | **Multi-Process** | **Multi-Threaded** | **Asynchronous** | **Notes**                                                                 |
|-----------------------------|-------------------|--------------------|------------------|---------------------------------------------------------------------------|
| **Apache HTTP Server**      | Yes               | Yes                | No               | Can be configured to use either multi-process (default) or multi-threaded.|
| **Nginx**                   | Yes               | No                 | Yes              | Primarily event-driven and asynchronous, uses worker processes.           |
| **Node.js**                 | No                | No                 | Yes              | Asynchronous, single-threaded, non-blocking I/O.                          |
| **Python (Gunicorn)**       | Yes               | Yes                | No               | Supports multi-process and multi-threaded configurations.                 |
| **Python (uvicorn)**        | No                | No                 | Yes              | Asynchronous, used with frameworks like FastAPI and Starlette.            |
| **Django (WSGI)**           | No                | Yes                | No               | Default WSGI servers are multi-threaded, can be used with multi-process servers like Gunicorn. |
| **Flask (Werkzeug)**        | No                | Yes                | No               | Development server is multi-threaded; use with Gunicorn for multi-process.|
| **Ruby on Rails (Puma)**    | Yes               | Yes                | No               | Supports multi-process (workers) and multi-threaded (threads).            |
| **Java (Tomcat)**           | No                | Yes                | No               | Multi-threaded by default; each request is handled in a separate thread.  |
| **Go (net/http)**           | No                | Yes                | No               | Built-in server is multi-threaded by default.                             |
| **ASP.NET (Kestrel)**       | No                | Yes                | Yes              | Supports asynchronous programming and is multi-threaded.                  |
| **Elixir (Phoenix)**        | No                | No                 | Yes              | Asynchronous and highly concurrent, leveraging the BEAM VM.               |
| **Spring Boot (Java)**      | No                | Yes                | Yes              | Multi-threaded, can handle asynchronous operations with proper configuration. |
| **Microsoft IIS**           | Yes               | Yes                | Yes              | Can use multi-threaded, multi-process, and asynchronous models.           |
| **Express.js (Node.js)**    | No                | No                 | Yes              | Asynchronous, single-threaded, non-blocking I/O.                          |
| **Tornado (Python)**        | No                | No                 | Yes              | Asynchronous, non-blocking I/O.                                           |
| **Twisted (Python)**        | No                | No                 | Yes              | Asynchronous, event-driven networking engine.                             |

### Summary:
- **Multi-Process:** Common in traditional servers like Apache and Nginx, where each worker process handles requests independently.
- **Multi-Threaded:** Common in application servers like Tomcat and Kestrel, where threads handle individual requests within a process.
- **Asynchronous:** Popular in event-driven servers like Node.js, Nginx, and frameworks like Tornado, where non-blocking I/O allows handling many connections simultaneously.

This table gives a general overview, and specific configurations or extensions can enable additional features in many of these servers.


With this background lets try redesigning our rate limiter.
Since we used a hashmap to store the last access information.
Instead of an in-memory hash, we could use a database service like redis or postgres.
In which case it wuold be respoinsible for handling the multiple requests and they would make it sequential.
Just because of the sheer amount of writes, it may be a good idea to do a persistant database like postgresql.
It would be more efficient with the read operation rather than write operation.
the worst case operation would alawys be K operations in T time interval. So the db would always from writing and poping at teh same time.
So imo, redis and postgresql would be an overkill for tsumc han operation.

Lets continue with an in-memory hashmap t osolve our problem.

#### Async server
If the server is async, then we only have to create a singleton version of the class and one instance of the class would be sufficient.
The oepration we are dealing with is a cpu bound process and hence there would be very little chance of reilnquishing the CPU to go and 
service another request.

Hence they would by default be serialized to process the incoming requests in sequence. 
But just to be safe we have to add a mutex here to protect the data structure.
we need not have a data structure for the whole cache but instead only for the queue of timestamps.

#### Multi-threaded server and multi-process server
conceptually both these servers will guarantee us concurrency via parallelism. The difference would be in terms of address space and resource access.
We can delve into points where they differ.
if the server supported multithreaded, it starts getting complicated.
The implementation that we did ie its instance of executing code could be called from multiple threads at any given time.
Since its threaded model, the access cache can be accessed from all the threads. 
The existing async model would work fine in this case.

#### Round robin server
In this base multi threaded example we did not mention how the client requests are distributed to the threads.
This could be round robin. 
But there is one major fproglem with this approach. 
If a client is sending too many requests then it can start bottlencking all the threads.

#### Sticky server
One way of mitigating the cliient affecting all the other clients is to have sticky threads.
The first time a request comes in then it needs to be assigned to a thread.
Every time the request comes in, it **shall** go to the same thread.
This not only stops one client from bottlenecking all the other clients, but it also eliminates the need for a lock.
this is because all requests we send to the thread will be via some IPC mechanism which resembles the queue ADT.
This will in effect serialize all the incoming requests and prevent all race conditions.

We cannot create / allocate one thread per client. this would kills all our resources.
So we have to associate multiple clients to each thread. 
so the worst case would be a bad actor could bottleneck multiple clients but not all.

#### Comparison
We now have a case where the all the clients are affected vs some of the clients are affected.
Of course in a real world scenario, we would run some prototype benchmarking and cost analysis.
And look at Commercial / open source off the shelf tooling / products that have already solved this problem.

#### Optimize
Since some clients are definitely affected, we need to optimize our design, algorithm, implementatino to protect those clients.
Instead of not just rate limiting, we are now going to start giving time outs to clients which are involved in this malicious activity.
If a request from a Client gets denied multiple times say K times in M time interval, then they must be send to a timeout list for U time interval.
Or all offending clients should be sent the same thread :D which would be their jail. 
We would have to implement more booking keeping for the clients. This could be as simple as current request time > (last_accepted_request_time + LargeT)
Another interesting would be to migrate the good clients out of a thread. Not sure if this viable in this case but it would be a good use case to investigate 
for application to another program.

```
Am I implementing a load balancer? 
```