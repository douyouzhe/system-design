# Design a hotel booking App
Examples: Booking.com, Hotel.com, Airbnb

Disclaimer: This is based on my knowledge only, I do not have information about how the above apps are built. 

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
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
* Users:
    * Search for hotels/rooms
    * Make reservations
    

### Non-Functional
* high consistency
* low latency
* high availability
* scalable

-----------------------

## Design Basics

*  

-----------------------

## Components Deep Dive

### Hotel & Room Listing Service
* The number of hotels in the world is limited, and the number which will be listed using our service will be even less. Also, the growth is foreseeable and it is very less likely we will have a scalability problem even if we use an SQL database to store hotel and room listings.
* Each listing will come with a lot of media resources like images and videos, we will be storing these resources using file storage like AWS S3. The reference (URI) to the resources will be stored in the SQL database together with the listing records. Most likely our App will be used by users across different locations so adding CDN to distribute these resources will improve user experience significantly.  
* Similar to the E-commerce platform design, for any newly added listing, if we want them to appear in our search engine, we need to let the search engine "consume" this new record first. Therefore, we send a message containing the listing's metadata to a message queue (Kafka) which will then feed it to the search engine. *Note that when we use Kafka, we normally create an abstraction service (for example, ListingProducer and ListingConsumer) to produce and consume the message, so that we do not need to interact with Kafka directly in our Listing Service and also we can maintain a specification for the messages.*

### Search Service

### Booking Service

-----------------------
## Reference
