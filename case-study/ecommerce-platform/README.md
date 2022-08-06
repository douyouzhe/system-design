# Design a e-commerce platform
Examples: Amazon, TaoBao and Walmart

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Components Deep Dive](#components-deep-dive)


## Architecture Diagram
![architecture](/case-study/ecommerce-platform/e-commerce-platform.jpg)

## Requirements
### Functional
1. User Register/Login
2. Merchant creates Shops/Products
3. User Search
4. User Checkout
5. User Home Page
6. User View Orders

### Non-Functional
1. Low latency
2. High availability
3. High Consistency
4. More Purchases

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


## Components Deep Dive
### User Login Flow
User service will be a normal RESTful API provides CRUD operation for Users. Read more in the streaming platform [example](https://github.com/douyouzhe/system-design/tree/main/case-study/streaming-platform#user-login-flow).

### Merchant APP Flow
Merchants have their own APP and Services that provide different functionalities 


### User Search Flow


### User Purchase Flow


### User APP Flow






## Reference


