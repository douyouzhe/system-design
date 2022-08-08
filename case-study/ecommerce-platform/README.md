# Design an e-commerce platform
Examples: Amazon, TaoBao and Walmart

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)
-----------------------

## Architecture Diagram
![architecture](/case-study/ecommerce-platform/e-commerce-platform.jpg)
-----------------------

## Requirements
### Functional
1. User Register/Login
2. Merchant creates Shops/Products
3. User Search, Cart, Wishlist
4. User Checkout
5. User Home Page
6. User View Orders

### Non-Functional
1. Low latency
2. High availability
3. High Consistency
4. Maximizing Revenue
-----------------------

## Design Basics
* Parties
    * Merchant: Client that sells products on the platform. They have will have a client app to CRUD shops and catalogs.
    * User: Buyer can use the app to make purchases.
    * Admin: Internal administrators can access internal data and APIs.
    * *These 3 parties can be abstracted into one class and use a userType field to distinguish* 
* Transactions that involve money movement require ACID property so SQL database should be adopted for most cases.
* Instead of building everything from scratch, a lot of external third-party services can be integrated. For example, Shippo is a company that help merchants to manage shipping and inventory. Payment companies like PayPal and Stripe provide payment SDKs. 
* Making processing orders a synchronous system is only feasible for small scale systems when there is a limited number of users. Mostly likely we will have a asynchronous system for better user experience. This is especially important during **peak hours** like prime days for Amazon and 11/11 for TaoBao. 
* Should we deduct from inventory at the moment when adding to cart or when complete the purchase? We will need an algorithm to avoid overselling while maximizing sales. We will cover this in the deep dive section. 
* Ideally we want to store Products/Catalogs using SQL database due to the ACID requirement. However, the SQL scheme in this case is actually a huge burden because of the variety of Products. For example, food, clothing and electronics will not share the same set of fields so we will end up with lots of *null* fields if we want to maintain the fixed schema. Therefore, using a NoSQL database makes more sense. 
-----------------------


## Components Deep Dive

### User Login Flow

User service will be a normal RESTful API provides CRUD operation for Users. Read more in the streaming platform [example](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-login-flow).

-----------------------

### Merchant APP Flow
Merchants have their own APP and Services that provide different functionalities such as creating a new shop or adding a new product. We have Shop Service and Product Service to handle the CRUD respectively. The Shop Service is very straight forward, we gather some information from the merchant using a form then create/update based on it. Note that the shop is not immediately available after creation, there will be a review process to check the eligibility based on several factors like the merchant's background, the legal status of the shop_category and etc. We make use of a field called shop_status(ENUM) to track the lifecycle of a store. This is also used to soft-delete a shop, as we all know, an internet company will NEVER truly erase any data. ;) 

<p align="center">
<img src="/case-study/ecommerce-platform/store-class.jpg" width="400">
</p>

Another common practice is to add two timestamp columns to every table we designed even if you cannot think of any use case at this point of time: time_created and time_updated. They are super useful when debugging and generating dashboards. 

Merchants can be further divided into large enterprises(LE) and small-and-medium enterprises(SME). LEs may have special requirement sometimes, such as inserting listings by batch or generating report with required format for accounting purpose. 

As we all know, not all shop/catalog information is text-based, there will be plenty of media content such as images and even videos. We talked about storing media content in a blob storage as well as CDN in [the streaming platform design case](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-upload-flow) so we will not cover the details here. 

Any changes to the shop/catalog on our platform will be fed into an Elastic Search cluster to provide fuzzy search and type ahead capability for users. Of course this two components are decoupled using a message queue in between. 

-----------------------
### User Search Flow
User will be able to search for Products/Stores via a search box. The Search box is powered by an Elastic Search cluster which consume any [CDC event](https://hevodata.com/learn/kafka-cdc/) we talked about earlier. 

Sometimes the users will NOT be able make certain purchases because:
- legal reasons, like age restriction or local laws
- geographical reasons, like there is no way to ship the inventory to the end users
- the product is only for some members or targeted promotion
In this cases, it will be frustrating if the users can see the product but they cannot buy it and this is bad user experience. We want to have a service in place to filter out these products so that users can only see eligible products in the search results. This service is called Serviceability Service.

After searching, users will either add the product to the cart for purchase or to the wishlist. We will a service/DB for each case and they are actually extremely similar from the design point of view. In terms of data structure and API specifications, they are both operations to some product references to the user's ID. Any operations (mostly additions to cart and wishlist, as well as the search history) provide us with valuable information about the user's profile, interests and shopping pattern, so we want to collect them by feeding them into Kafka and later consume by a spark streaming APP. 

-----------------------
### User Purchase Flow
Users can complete their orders for the products in their cart anytime. By clicking the checkout button, a few steps will be carried out. There could be some unbound external calls and it is not feasible to let the users wait. It makes more sense to return a success message after collecting all the input required then process the orders asynchronously. Of course we can send notification to the users for any order updates along the process if the users choose to.

Inventory Service is responsible for checking the availability for products. The simplest way is using a non-negative constraints in the SQL database to make sure we do not oversell when there is not enough stocks. Of course there are more advance [algorithms and strategies](https://www.sortly.com/blog/what-are-the-3-major-inventory-management-techniques/) to find a balance between no-overselling and maximizing sales that we will not cover in details here.

Pricing Service will return the final sale price for products after a series of adjustments. These adjustments are normally based on promotion codes, users' reward points deduction, bundle discounts and etc. 

We will then pull the users' address and payment information from our database if they already exist. Otherwise, users are always welcomed to provide them. 

At this point of time, we have all the required information and we can create Order records via Order Service and redirect the users to a "success" page. What happened next are all considered as *offline* process. 

We can adopt both *push* and *pull* model for order processing depends on the merchants type(LE or SME) and their SLA requirements. The push model works in a way that every time a new orders is created, a order-created event will be pushed to all the stakeholders for further processing. For LEs, they normally prefer a pull model where a scheduled job will pull all the order created during a period of time and process them in a batch. The scheduler can be set using [Apache Airflow](https://airflow.apache.org/). 

Now, for the actual payment processing and shipping details, normally they should not be covered in any interview since it deviated from the topic too much and they alone can be another design question. In general, we can integrated with external service provider for some components, like [Shippo](https://goshippo.com/integrations) for shipping and [PayPal](https://www.paypal.com/c2/webapps/mpp/merchant) for payments by using their SDKs or APIs.  

Shippo SDK Example:

```
Shippo.setApiKey('<API_TOKEN>');

// To Address
HashMap<String, Object> addressToMap = new HashMap<String, Object>();
addressToMap.put("name", "Mr Hippo");
addressToMap.put("company", "Shippo");
addressToMap.put("street1", "215 Clayton St.");
addressToMap.put("city", "San Francisco");
addressToMap.put("state", "CA");
addressToMap.put("zip", "94117");
addressToMap.put("country", "US");

// From Address
HashMap<String, Object> addressFromMap = new HashMap<String, Object>();
addressFromMap.put("name", "Ms Hippo");
addressFromMap.put("company", "San Diego Zoo");
addressFromMap.put("street1", "2920 Zoo Drive");
addressFromMap.put("city", "San Diego");
addressFromMap.put("state", "CA");
addressFromMap.put("zip", "92101");
addressFromMap.put("country", "US");

// Parcel
HashMap<String, Object> parcelMap = new HashMap<String, Object>();
parcelMap.put("length", "5");
parcelMap.put("width", "5");
parcelMap.put("height", "5");
parcelMap.put("distance_unit", "in");
parcelMap.put("weight", "2");
parcelMap.put("mass_unit", "lb");

// Shipment
HashMap<String, Object> shipmentMap = new HashMap<String, Object>();
shipmentMap.put("address_to", addressToMap);
shipmentMap.put("address_from", addressFromMap);
shipmentMap.put("parcels", parcelMap);
shipmentMap.put("async", false);

Shipment shipment = Shipment.create(shipmentMap);
```
PayPal Payments API example:
```
curl -v -X GET https://api-m.sandbox.paypal.com/v2/payments/authorizations/0VF52814937998046 \
-H "Content-Type: application/json" \
-H "Authorization: Bearer <Access-Token>"
```
 
These integrated services should send update to our system so we can update the order status accordingly based on the lifecycle. How should we store order objects? There are a few things we should consider:
* obviously ACID properties are necessary
* Order data is very structured
* Orders will be frequently updated until reach the end of the lifecycle
* We might need flexible query patterns: query by id, query by users, query by timestamps, query by merchants and etc. Optimization can be made using indexes on various columns

We will use a SQL database based on the above reasons. Of course we can add a cache besides the SQL database for better performance. Refer to this [section](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-login-flow) for caching strategy. However, it is very less likely that users want to query the purchases they made long time ago and there will be no more updates. Therefore we might want to dump all the history order data to a different place. Cassandra is a highly-scalable distributed database that is good for handling large amounts of data across multiple data centers and the cloud. Plus, it is able to fast write huge amounts of data without affecting the read efficiency. Again, we can use a scheduler or a Order Archival Service to "move" data from the SQL database to the Cassandra cluster with a set threshold, like a few months or a year. 

-----------------------
### User APP Flow

There is nothing special in this section, users can view their orders(from Order Service and Historical Order Service) and recommendations (from the results generated by Recommendation Service). 

-----------------------
### Maximizing Revenue

How do we maximize our revenue? The first thing you should think about in 2020s is obviously Machine Learning. Remember that as long as we increase the number of sales, potentially we will gain more revenue, so we can think about this from both sides (merchants and buyers). To elaborate, there are a few sectors we can implement ML:
* group/segment similar users together using Regression or K-nearest neighbors (KNN). Unsupervised learning will help us identify new segments and understanding these new segments opens up important new marketing opportunities for E-Commerce stores.
* recommend the most suitable products for customers. This helps improve both user-experience by cutting down the amount of time a customer searches and, consequently, improves sales. We can take a step further by sending out recommendations to users actively via Notification Service.
* price discrimination is a *old* economical concept that charges customers different prices for the same product or service based on what the seller thinks they can get the customer to agree to. Many eCommerce sites rely on dynamic pricing algorithms to help them achieve the optimum price for their products, ensuring they remain both competitive and profitable.
* supply/inventory management. Today, many procurement teams are making use of demand estimation algorithms that use a wide range of data to more accurately predict the stock levels their company will need.

Another aspect to maximizing revenue is to minimize the *loss*. When users add a product to their cart, ideally we should decrease some count from the inventory to secure the order otherwise the it is very bad user experience if the product is sold out when they checkout. However, holding this inventory count forever will block sales for other users and result in loss for the merchants. One simple solution is to use a time-to-live (TTL) feature in some databases like Redis. We can represent this "hold" in Redis with a TTL about 10 mins and if the order is not completed by then, we add the count back to the inventory. 

-----------------------
## Reference
https://hevodata.com/learn/kafka-cdc/


https://www.sortly.com/blog/what-are-the-3-major-inventory-management-techniques/

https://airflow.apache.org/

https://www.paypal.com/c2/webapps/mpp/merchant

https://goshippo.com/integrations

https://www.bmc.com/blogs/apache-cassandra-introduction/

https://www.eukhost.com/blog/webhosting/6-ways-ecommerce-stores-benefit-from-algorithms/

https://www.investopedia.com/terms/p/price_discrimination.asp






