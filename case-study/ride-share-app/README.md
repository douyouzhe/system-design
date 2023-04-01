# Design a Ride-share APP
Examples: Uber, Lyft and DiDi

Disclaimer: this is based on my knowledge only, I do not have information about how the above apps are built. 

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/ride-share-app/ride-share.jpg)
-----------------------

## Requirements
### Functional
1. Display all available drivers' location
2. ETA & Pricing
3. Book/View trips
4. Location tracking
5. Drivers' earning report

### Non-Functional
1. Low latency
2. High availability
3. Scale(global) - 150M MAUs, 15M trips/day
-----------------------

## Design Basics

* Obviously, a ride-share App needs to rely on a lot of functionalities of a map service. We have covered the details of how to design a [map service](https://github.com/douyouzhe/system-design/tree/main/case-study/map-service) so I highly recommend going over it first. We should keep in mind when designing any system that we do not need to build everything from scratch on our own and it might make more sense to partner with external service providers if we have limited resources. For example, Uber is bouncing between Google Map API and MapBox and Lyft is using OpenStreetMap. 

* Matching riders and drivers is a core component to make any ride-share App successful.
    * The naive solution is we can match a rider to the closest driver based on physical proximity. Riders and drivers value short pick-up times, but this doesn’t mean that the closest match will meet this need. For example, whenever there is heavy traffic, one-way streets, or road detours due to construction, drivers can spend a long time reaching the closest rider. Thus, the closest doesn’t mean the quickest.
    * We can match a rider to the closest driver based on the shortest pick-up time. It is easy to think about it if we only have one rider at a time and a simple greedy solution will work for us. However, if the goal is to make the average pick-up time shorter for all the riders. For example, in the below scenario, we would match Driver A to Rider B since they are only 6 mins apart. Then we can only match Driver B to Rider A and on average it takes (6+50)/2 = 28 mins to match both riders. However, if we look at this scenario from a global perspective we can see it's better to match Driver A to Rider A and Driver B to Rider B because on average it takes (16+18)/2 = 17 mins to match both riders. 

    <p align="center">
    <img src="/case-study/ride-share-app/pickuptime.jpg" width="500">
    </p>

    * Therefore, we cannot only look at the matching process from a single rider's perspective. We need to perform something like a **batched matching** system where riders and drivers are matched in a way that lowers waiting times for everyone in the area. The basic idea is to explore as many options as possible and pick the one with shorter average pick-up time. This problem can be presented through a bipartite matching graph in this [article](https://blogs.cornell.edu/info2040/2019/10/23/uber-ride-sharing-a-matching-market/).

* It matters A LOT to display cars on roads with direction because as we mentioned, they affect the pick-up time. It is actually not easy to achieve since GPS sensor accuracy from your phone is about 3-5 meters and GPS does not have the concept called *roads*. So it is very possible that the car is not on the road or not moving along a road - we can even see that inside Uber App today. 

* Car locations need to be stored and indexed in a way that we can quickly pull all the nearby drivers. This will be useful in both matching the riders with drivers and also display all the nearby drivers in the UI when riders open the App. Car locations need to be stored with time series as well for rendering purpose.

    * Points vs. Timeseries
    
        We will be able render the path based on the points with timestamp data. 

    <p align="center">
    <img src="/case-study/ride-share-app/timeseries.jpg" width="700">
    </p>



    * Location Indexing/Sharding/Storage
        
        We expect a system to handle very high volume of read and write with low latency. We do not need long term durable storage as well, at least not in the same data store. So we can go with a in-memory search index. If we have a use case to store user's trip information permanently, we can use a distributed NoSQL database like Cassandra. The partition key will be DriverUUID and a time bucket(a time range). Within a partition, we can have the date order by location and timestamp. 
        
        Drivers' location are shard by cities, segments, and Products (UberX, UberPool).....Since locations are constantly changing, we need a mechanism to keep auto-repartition. This can be an offline job scheduled periodically using Airflow. 
        
        Uber has developed something very similar to [Consistent Hashing](https://en.wikipedia.org/wiki/Consistent_hashing) called [Ringpop](https://github.com/uber/ringpop-go) to distribute data (including replicas) among worker nodes. 

  
-----------------------

## Components Deep Dive

### Rider & Driver Registration
Similar to all other Apps we designed so far. Users can register and login via User Service. As one of the most heavily used components, we cache the data using Redis. For a ride share App, driver registration requires additional steps to ensure our service quality and rider's security. Such check is done via Driver Background Checker. Both systematic and manual checks such as driver's accident history, driver license validation, SSN and vehicle status. 

### Location Ingestion

The location ingestion process is similar to the [map app](https://github.com/douyouzhe/system-design/tree/main/case-study/map-service#user-sharing-location), where packets including location data are sent to our system periodically (maybe every 5s). The amount of packet we are getting are huge and the QPS varies a lot, this is a very good use case to add a buffer (message queue) in between, so we have producer/consumer pair to decouple the inflow and outflow of data. The packet will just contain user_id, location_data and a timestamp. Message queue can achieve At-Most-Once, At-Least-Once or Exactly Once delivery based on the setting. At-most-once delivery is the default option for producer/consumer architectures because it requires no additional effort on the part of the developers. Any messages that are lost due to errors or disruptions are simply disregarded. This is not a big problem for our use case because losing one packet is not that critical and we can always get another packet in another 5 seconds. Due to the limitation of any GPS sensor, we will get outliers or points not on a road frequently, we will be relying on a rendering or filtering service to remove those points.  

In this case, we do not need to get riders' location unless they are using our service. As mentioned above, these location data is shard by segments so that we can pull all nearby drivers quickly when needed. 

### Requesting a ride (WIP)


### After a ride
In some Apps, the fees are deducted right away after the ride and in other Apps we only require riders to complete the payment before the next ride. Either way, the payment process will be a async process. We have several choices:

* We can build our own payment system to avoid paying all the fees to payment processors and acquirers.
* We can integrate with a third party payment provider like PayPal or Block. As a partner (not SMB), we might be able to negotiate a better rate.  
* The third option is a bit rare but this is what Uber decided to do initially. They work with a payment provider (Braintree) to build a unbranded payment gateway. In this option, they have the freedom to pick different payment processors and acquirer banks in each region and more importantly they can integrated different services like UberEats and UberFreight together. 

If you think a ride share App as a service provider, there is three parties involved and we need a payout process to pay the drivers. This is another advantage if we have our own payment gateway, we can have reconcilable reports that keep our book balanced in our account. 

-----------------------
## Reference

https://www.mapbox.com/

https://eng.lyft.com/how-lyft-discovered-openstreetmap-is-the-freshest-map-for-rideshare-a7a41bf92ec

https://blogs.cornell.edu/info2040/2019/10/23/uber-ride-sharing-a-matching-market/

https://www.uber.com/us/en/marketplace/matching/

https://www.youtube.com/watch?v=GyPq2joHZv4

https://youtu.be/AzptiVdUJXg
