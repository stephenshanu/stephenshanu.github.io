---
layout: post
title:  "Rate Limiter Basic (no race condition or scaling)"
date:   2024-08-28 13:36:58 -600
categories: jekyll update
---

## Rate Limiter

This is an attempt at solving the commonly asked interview question of implementing a rate limiter.

### Problem Statement
In the problem statement, it is mentioned that you need to implement a rate limiter for a service. 
The rate limiter has a method called IsAllowed which takes in clientId as input.
We need to design / implement a rate limiter that allows upto N requests from each client in K time interval.

This is designed around the problem of DDoS. By rate limiting a client, we can ensure that the client does 
not take up dispropotionaly server resource. 
A DDoS (distributed denial of service) attack can be done by multiple clients requesting a resource until the server gets flooded and is unable to service genuine requests.

The rate limiter is a technique to deny resource to clients if they make N requests in K time interval.

### Let us think this through

We will get requests from multiple clients
Each client is identified by a unique identifier.
We need to figure out if this client requested more than N requests in the last K time interval.
So each time the client requests, we need to store the request time stamp.
We need to store last N requests from each client
On recieving a new request, we need to check the oldest time stamp. 
If its more than K milliseconds older than the current request, then drop this record.
repeat for all the saved time stamps.
Take the count of all the current stored timestamps.
If the length is equal to N, then deny the request.
Otherwise, add the current time stamp to the store and accept the request.

Per client storage:
For each client, there can be at most N requests. If we compare the time stamp of each, then average case will be O(N).
In practical terms this storage can be a Queue. Because we can assume the incoming timestamps will be time ordered.

Rate limiter storage:
Each time a request comes in, we need to find the client and return the queue of last requests.
So we can get O(1) speed complexity if we use a Hashmap.

### Implementation

{% highlight python %}
access_cache = {} # python hashmap key = client id, value = list of timestamps sorted in increasing order
def isAllowed(clientId: int):
    timestamp = get_current_timestamp() # gets the current timestamp from epoch in milliseconds)
    if clientId not in access_cache:
        access_cache[clientId] = timestamp
    else:
        # existing client
        last_access_list = access_cache[clientId]
        for i in range(len(last_access_list)):
            if timestamp - last_access_list < K:
                break
        last_access_list = last_access_list[i:]

    if len(access_cache[clientId]) < N:
        access_cache[clientId].append(timestamp)
        return True
    return False   
{% endhighlight %}

### Complexity

#### Time complexity
1. Finding the client - O(1)
2. determining the permission - O(N)
3. Insertion of an accepted request - O(1)

`So total time complexity = O(N)`

#### Space complexity:
1. Storing all clients - O(M) --> when M is the number of clients
2. Storing all timestamps - O(M * K)

`So total space complexity is O(M) ie increases linearly with the number of clients`


### Test cases
This is a fairly straightforward testing scneario.

1. No client is present
  Base case where client id is not present and we accept the request and service it.
2. Client id is present and result is allowed
   Base case where client id is present and we successfully find it and service it.
3. Client id is present and request is N request in K time interval
   Now we try to find if the timestamp search is successful is within the K time interval and the boundary condition
4. Client id is present and request is N+1 request in K+1 time interval
   This case is to check if the queue is full and we are poping elements correctly from it.
5. Client id is present and request is N+1 request in K time interval
    This case is to check the fullness of queue

### Conclusion
In this example, we are not considering race condition (which will happen depending on the service being used)