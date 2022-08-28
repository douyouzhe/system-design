# Design an Chat-App
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

* Obviously, a ride-share App needs to rely on a lot of functionalities of a map service. We have covered the details of how to design a [map service](https://github.com/douyouzhe/system-design/tree/main/case-study/map-service) so I highly recommend to go over it first. We should keep in mind when designing any system that we do not need to build everything from scratch on our own and it might make more sense to partner with external service provider if we have limited resources. For example, Uber is bouncing between Google Map API and MapBox and Lyft is using OpenStreetMap. 

* Matching riders and drivers is a core component to make any ride-share App successful.
    * The naive solution is we can match a rider to the closest driver based on physical proximity. Riders and drivers value short pick-up times, but this doesn’t mean that the closest match will meet this need. For example, whenever there is heavy traffic, one-way streets, or road detours due to construction, drivers can spend a longer time reaching the closest rider. Thus, the closest doesn’t mean the quickest.
    * We can match a rider to the closest driver based on shortest pick-up time. It is easy to think about it if we only have one rider at a time and a simple greedy solution will work for us. However, if the goal is make the average pick-up time shorter for all the riders. For example, in the below scenario, we would match Driver A to Rider B since they are only 6 mins apart. Then we can only match Driver B to Rider A and on average it takes (6+50)/2 = 28 mins to match both riders. However, if we look at this scenario from a global perspective we can see it's better to match Driver A to Rider A and Driver B to Rider B because on average it takes (16+18)/2 = 17 mins to match both riders. 

     <p align="center">
    <img src="/case-study/ride-share-app/pickuptime.jpg" width="400">
    </p>

    * Therefore, we cannot only look at the matching process from a single rider's perspective. We need to perform something like a **batch matching** system where riders and drivers are matched in a way that lowers waiting times for everyone in the area. 
 
-----------------------


## Components Deep Dive



-----------------------
## Reference

https://www.mapbox.com/

https://eng.lyft.com/how-lyft-discovered-openstreetmap-is-the-freshest-map-for-rideshare-a7a41bf92ec

https://blogs.cornell.edu/info2040/2019/10/23/uber-ride-sharing-a-matching-market/

https://www.uber.com/us/en/marketplace/matching/

https://www.youtube.com/watch?v=GyPq2joHZv4








