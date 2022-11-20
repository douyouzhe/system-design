# Design a hotel booking App
Examples: Booking.com, Hotel.com, Airbnb

Disclaimer: This is based on my knowledge only, I do not have information about how the above apps are built. 

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/hotel-booking/hotel-booking.jpg)

-----------------------

## Requirements
### Functional
* Hotel Admin:
    * Hotels CRUD
    * Rooms CRUD
    * Bookings management
    * Business Insights
* Users:
    * Search for hotels/rooms
    * Make reservations
    

### Non-Functional
* high consistency
* low latency
* high availability
* scalable


-----------------------

## Components Deep Dive

### Hotel & Room Listing Service
* The number of hotels in the world is limited, and the number which will be listed using our service will be even less. Also, the growth is foreseeable and it is very less likely we will have a scalability problem even if we use an SQL database to store hotel and room listings. Some of the APIs we need are:
    
    ```
    POST api/hotels
    PUT  api/hotels/{hotel_id}
    GET  api/hotels/{hotel_id}

    PUT  api/hotels/{hotel_id}/rooms/{room_id}
    GET  api/hotels/{hotel_id}/rooms/{room_id}
    ```

    <p align="center">
    <img src="/case-study/hotel-booking/schema.jpg" width="480">
    </p>
* Each listing will come with a lot of media resources like images and videos, we will be storing these resources using file storage like AWS S3. The reference (URI) to the resources will be stored in the SQL database together with the listing records. Most likely our App will be used by users across different locations so adding CDN to distribute these resources will improve user experience significantly.  
* Similar to the E-commerce platform design, for any newly added listing, if we want them to appear in our search engine, we need to let the search engine "consume" this new record first. Therefore, we send a message containing the listing's metadata to a message queue (Kafka) which will then feed it to the search engine. *Note that when we use Kafka, we normally create an abstraction service (for example, ListingProducer and ListingConsumer) to produce and consume the message, so that we do not need to interact with Kafka directly in our Listing Service and also we can maintain a specification for the messages.*

### Search Service
* Fuzzy Search and Typeahead Search are essential to the user experience. Popular open-source search platforms such as Elastic Search and Solr can be used. Although they are both based on Apache Lucene and perform roughly the same. Solr has more advantages when it comes to static data, because of its caches and the ability to use an un-inverted reader for faceting and sorting – for example, e-commerce. On the other hand, Elasticsearch is better suited – and much more frequently used – for time-series data use cases, like log analysis use cases. If you have mostly static data and need full precision for data analysis and fast performance you should look at Solr. 

### Booking Service
* The Booking Service is nothing but just a CRUD service maintaining a table for all the bookings. It is very similar to the Order Service we covered in the [e-commerce platform design](https://updoggtech.com/e-commerce-platform). We would like to maintain the ACID property for all the bookings so we will go with a SQL database here. 
* In the [e-commerce platform design](https://updoggtech.com/e-commerce-platform), we introduced an Order Archival Service which is essentially an ETL job to move historical order data from the master SQL database to a NoSQL database so that we can keep the master SQL database smaller and faster since only active orders will be there. Now, do we want to do the same thing depending on the scale and size of our platform? Normally we do not expect the same number of bookings as the number of orders for an e-commerce platform so it might be ok to just keep everything in a master SQL database.
* The API of making a booking is just a POST call to the Booking Service with some information such as user_id, room_id, quantity, start_date, end_date, etc in the request body. We do need to check the availability before inserting a new booking record. One way to do that is maintaining a table just for available room count which contains room_id, date, available_count and available_count can be a row-level constraint that is always non-negative. After the verification, we can then proceed to create a new booking record. 
* How do we handle overselling or incomplete transactions? We covered this part in the [e-commerce platform design](https://updoggtech.com/e-commerce-platform). In short, we can use a cache with a time-to-live(TTL) feature such as Redis. Whenever a user started to checkout, we *"reserve"* his booking in Redis with a TTL (let's say 10 mins) and deduct the count temporarily from available_count. Whenever a cache is invalidated, we will add the count back to the available_count. 
* A valid booking record should contain the following fields: 
    <p align="center">
    <img src="/case-study/hotel-booking/bookingschema.jpg" width="220">
    </p>
    and Status should be an Enum with these possible values: Reserved, Booked, Payment Declined, User Cancelled and Completed. 

* The last step of the Booking Service will be sending a message to the Search engine and removing this room from the search results if it is fully booked. 

### Analytics and Reporting Service
* When we have the supply and demand(from search trending and history information) data, we will be able to optimize the price for the hotel business. Therefore we feed all the search actions and order transactions to an Analytic Service which is a streaming service. 
* We can also provide business insights and reports to hotel managers which can be used to drive their growth or for accounting purposes. 

-----------------------
## Reference

https://sematext.com/blog/solr-vs-elasticsearch-differences/

https://www.nexsoftsys.com/articles/how-to-design-backend-system-of-an-online-hotel-booking-app-using-java.html

