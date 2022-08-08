# Design an map webapp
Examples: Google Map, Apple Map, Waze

Disclaimer: this is based on my knowledge only, I do not have information about how the above apps are built. 

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/map-service/map-webapp.jpg)

-----------------------

## Requirements
### Functional
- Identify roads, places & routes
- Accurate distance & ETA
    - very hard to quantify a lot of factors - road conditions, random events(unpredictable), traffic and weather
- Plugable as API, easy integration with partners like Uber or vehicle manufactures or data provider


### Non-Functional
1. Low latency
2. High availability
3. Scale - 3B MAU & 10M partners- very large volume
-----------------------

## Design Basics
* The number of roads and points of interest are huge in any map services and normally there is no single place to fetch all these data at once. We will use government provided municipality maps like [USGS](https://www.usgs.gov/programs/national-geospatial-program/national-map) and satellite imagery providers like [Planet](https://www.planet.com/?gclid=CjwKCAjw6MKXBhA5EiwANWLODKkrbtGokMc8GNQ1XuwYd8Gu02nAIQHBFByJ6_Iqp9coZOP-fKYASBoCJe4QAvD_BwE) and [Maxar](https://www.maxar.com/?utm_source=google&utm_medium=sem&utm_campaign=maxar-brand&gclid=CjwKCAjw6MKXBhA5EiwANWLODLT15QAFgMXlzM8z0VXpmN0WxpW25kAsWbiemAIDs7rvutR4Ux2LUBoCYCgQAvD_BwE) as the major source to start with. We will need to rely on geological surveys, street cars and even user (people, business and customers) inputs&feedbacks for frequent updates since the world is a vast and constantly changing place.

* It is intuitive and convenient to think the map as a graph which consists of nodes and edges. But still, we will need very powerful database to store such a **huge** amount of data. Google utilizes its in-house single-keyed database BigTable, the commercially available alternatives will be AWS DynamoDB or Cassandra. 

* Navigating from point A to Point B is the major use case to a map service. Before we jump into that, we need to understand how to define and represent these points. The most common method used in Geographic coordinate system is Latitude and Longitude, similar to x-axis and y-axis representation in any 2D plane. Calculating the distance between any two points is a primary school Math problem. You may argue that this data might not be useful in a map app simply because there might not be a straight route between them. But still, it is a good starting point. 

    In a x-y axis system, we are dividing the world into four [quadrants](https://www.splashlearn.com/math-vocabulary/geometry/quadrant) and we can give an ID to each one. Dividing the entire earth into 4 sections probably will not help much but what we can do is to taking the original ID as the most significant bit (MSB) and recursively divide each section into smaller sections. Guess what, now we have a system to know if two areas (cluster of points) are close to each other or not just based on their IDs. There is one major flaw in this design that two points can be spatially close to each other but they are in different sub-sections and this happens a lot to the points that are close to the edge. If you think about this from the probability perspective, the further you divide, less likely this will happen. Also, you can still look at all the points in all the 8 nearby sections if you want to be very sure. 

    <p align="center">
    <img src="/case-study/map-service/quadtree1.jpg" width="400">
    </p>
    
    Now, how do we store all these sections. There is a data structure for that called [Quadtree](https://en.wikipedia.org/wiki/Quadtree), each level you go down a tree, it divides the section into 4 smaller pieces. The issue with Quadtree is that range query will be a slow operation, we know there is a tree structure called [Segment Tree](https://en.wikipedia.org/wiki/Segment_tree) but it only works on a single axis(1D). So is there a way to represent 2D data using one axis? The answer is [Geohash](https://en.wikipedia.org/wiki/Geohash) which encodes a geographic location into a short string of letters and digits. 

    Back to the question of navigation, finding the path is a complex algorithm question. Breadth First Search is a good once but it's based on the assumption that all edges have the same weight. In reality weights can be affect by several factors such as distance, road condition, traffic condition, toll and etc. So we will need some algorithms like Dijkstra or Floyd-Warshall to find the best routes. Again, this process will be recursive, we will go from section to section first, them sub-section to sub-section..... (you can think this as a **DP** problem if you like to think from small to big). Dijkstra has a time complexity of *O((V+E)logV)* and it not something we want to calculate on the fly. We are going to trade time with space by precomputing (not all nodes of course, we can sample a few each section) and store the information for quick look up. As mentioned, there will be dynamic incidents that can affect the best routes. We need to update ETA recursively (bubble up), If ETA changes by a certain percentage. 


-----------------------

## Components Deep Dive
### Map Data Ingestion
As mentioned earlier, we can build our map out based on resources provided by the government and satellite imagery initially. But the world is constantly changing, so we need consistent input of regarding any changes in the map:
* User will submit feedbacks
* Business will submit their business information
* We will have street cars driving around to capture street views
* Reliable third parties can provide any geographical information
* Any changes in the resources provided by the government and satellite imagery
* (I read that Google is using [sheep with cameras](https://www.washingtonpost.com/news/animalia/wp/2017/11/07/how-sheep-with-cameras-got-these-tiny-islands-onto-google-street-view/) to collect data, I hope they get paid)

All these will have us to detect the changes we need to make to our **Map** and they are all ingested into Kafka. We then capture the Change Data Capture (CDC) and use those events to update our map. This is continuos process that happens every second. 

Other than map data, we can also allow the submission of traffic data. For example, Waze allows the users to submit if they saw a car accident so that we can use that to update our Traffic Service as well. 


### User Sharing Location
User location data will be a valuable input to the Map app. For example, we can infer that there is congestion if all users are moving slowly on a road. If the users agree to share their location, they will send a packet periodically (maybe every 5s). Given that the users are authenticated and authorized, we ingestion their location information into our system. The amount of packet we are getting are huge and the QPS varies a lot, this is a very good use case to add a buffer (message queue) in between, so we have producer/consumer pair to decouple the inflow and outflow of data. The packet will just contain user_id, location_data and a timestamp. Message queue can achieve At-Most-Once, At-Least-Once or Exactly Once delivery based on the setting. At-most-once delivery is the default option for producer/consumer architectures because it requires no additional effort on the part of the developers. Any messages that are lost due to errors or disruptions are simply disregarded. This is not a big problem for our use case because losing one packet is not that critical and we can always get another packet in another 5 seconds.

How long should that data live is another design choice. If we have limited storage resources we can set a low time-to-live. If we want to provide additional feature like Google's timeline feature we can persist the data for years. 

### User Searching & Browsing
For the searching part, again we would like to support fuzzy search with type ahead feature and therefore we will rely on Elastic Search cluster. We talked about similar use cases in the [video-streaming](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-homepage-flow) example and the [e-commerce](https://github.com/douyouzhe/system-design/tree/main/case-study/ecommerce-platform) example. The input to the ES cluster is basically all the name of places and the Map CDC events. 

Sometimes, the users just want to browser a area without searching. This is essentially query for all the nodes in a segment we talked about earlier. We do not want to display all nodes at once for sure, we will assign a priority to each node and display them according to how much the user zoomed in. 

After locating a place, the users can start the trip and this will call the Routing service to plan the detailed route. As mentioned before, we do not calculate the route the fly unless we do not have the date precomputed (we should not). It is possible that we do not have the route for the last few meters (depends on the size of our smallest section) because it is not close to any sampled nodes. In this rare cases we will need to compute it on the fly. 

How do we compute the routes offline? we make use of several component to calculate the weight of a edge and also refer to historical trip data we gathered after each trip. 

### User Navigating
This is the backbone of a Map App. 

First, we cannot think this as a one time request, the client need to be in constant connection with our App because:
* in case the user has deviated from the planned route, we need to provide a new best route.
* in case there is any incident happened along the way, we want to change the route
* in case there is a new better route just became available, we want to let the user know

To achieve that, we need a 2-way connection as our server will send request to the client as well. Therefore we use a websocket instead. In order to know which websocket the user is connected to, we have a Websocket Manager and a k-v (key is the user and value is the reference to the websocket instance) datastore associated with it. Navigation Tracking Service will check the user's current location and compare with the planned route constantly, if any discrepancy found, it will be responsible to send a new route to the user. Also, as mentioned above, any changes or accidents that delayed the ETA significantly will be notified to the user via Navigation Tracking Service as well. 

### Analysis Service
Analysis Service consists of several components and their goal is to make functional and non-functional requirement better. They make use of both ML models and cluster computation because we need intensive amount of computation power. 
* User profiling can help us understand the user's driving habit and average speed so that we can provide more accurate ETA. We can also predict where is the user's office (his location during the day) and home (his location during the night) to predict his trip earlier.
* The accuracy of ETA from point A to point B at certain time can be improved based on a large amount of data. It can be a range with percentile. 
* Sometime we can predict where the user will be, for example, if we know the user has an appointment or a plane to catch, we can plan his trip offline and send a reminder when it's time to leave.
* We have a large amount of information provider, including users and third party agencies. We can have a scoring system based on the accuracy of the data they have provided and if they are reliable, the information they provided in the future will have a higher weight.
* If certain amount of users have traveled on some unknown road to been to an unknown place, we know this is something missing and this can help us identify new roads and new places.
* If we have all the navigation data, we can predict where the hot spot is right now and where it is going to be next. Using that information, we can purposely give the users a different routes to elevated the hotspot.

-----------------------
## Reference

https://cloud.google.com/blog/products/maps-platform/9-things-know-about-googles-maps-data-beyond-map

https://www.adservio.fr/post/apache-kafkas-delivery-guarantees

https://www.washingtonpost.com/news/animalia/wp/2017/11/07/how-sheep-with-cameras-got-these-tiny-islands-onto-google-street-view/

https://en.wikipedia.org/wiki/Geohash

https://en.wikipedia.org/wiki/Quadtree

