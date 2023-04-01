# Design a video streaming platform
Examples: Youtube, Amazon Prime Video, Netflix

## Table of Contents
1. [Architecture Diagram](#architecture-diagram)
2. [Requirements](#requirements)
3. [Design Basics](#design-basics)
4. [Resources Estimation](#resources-estimation)
5. [Components Deep Dive](#components-deep-dive)

-----------------------
## Architecture Diagram
![architecture](/case-study/streaming-platform/Video%20Streaming%20Platform.jpg)

-----------------------

## Requirements
### Functional
1. User Register/Login
2. Upload Videos
3. Play Videos
4. User HomePage
5. Multi-platform
    * different devices
    * different formats
    * different video dimensions
    * different video quality
    * different regions and languages
### Non-Functional
1. Low latency and high availability - no buffering
2. more user stickiness - good & right content

-----------------------

## Design Basics
* Parties
    * Users: human beings using the platform. Can be both content creators and viewers.
    * Clients: devices that users are using, which have video-playing functionalities and with the app installed.
* Videos are sent in chunks sequentially. In theory, we can use both TCP and UDP for transmitting these chunks and traditionally we will pick UDP over TCP since UDP is faster (no handshake and ack required) and packet loss is normally acceptable for video streaming. Surprisingly, both Youtube and Netflix adopted TCP mainly because:
    * to have better video quality - no packet loss
    * TCP's security feature can be a huge plus (especially for Netflix)
    * UDP is not supported by most browsers

    Having said that, UDP still has its advantages and Google is developing its own protocol QUIC (Quick UDP Internet Connections) which is faster than TCP. 
    
    *Note: most live streaming platforms are using UDP*
* Video quality can be adjusted based on the internet bandwidth.
* For very high-quality videos (from Production Houses), an SFTP folder can be used to upload videos.

-----------------------

## Resources Estimation
* assume the platform has 10 million DAU, and users watch 10 videos per day.
* assume 1% of users are content creators and they upload 2 videos per day.
* assume the average video size is 500MB (including transcoded contents).
* total storage: 10 million * 1% * 2 * 500 MB = 100 TB per day - that's a lot!
-----------------------


## Components Deep Dive
### User Login Flow
User service will be a normal RESTful API that provides CRUD operation for Users. 

As one of the most fundamental and important domains of any system, we would like to have an ACID property and this leads to the choice of a SQL database. 

Along with it, we add a Redis cache for better performance. The choice of caching patterns will be Cache-Aside (if SLA is not strictly required). Lazy loading is preferred in this case since the number of users is huge and active Write-Through can be expensive. You can refer to this link for the choice of [caching patterns](https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html) and [6 common caching design](https://www.gomomento.com/blog/6-common-caching-design-patterns-to-execute-your-caching-strategy).

-----------------------

### User Upload Flow
The content creators will use a form to submit a video for uploading and provide the original video along with some metadata. Obviously, video uploading cannot be a synchronous request since it is a time-consuming process. 

Content Onboarding Initializer Service will take the form and the original video, and act as a producer to the Message Queue (Kafka). Kafka is used as an intermediate buffer as well as a distributor to all the services we want to trigger. Content Onboarding Initializer Service also provides a layer of abstraction to provide customized uploading procedures. For example, a large production house might need a dedicated SFTP server to upload its original high-quality content, or a trusted user can skip some checks. 


![video_upload](/case-study/streaming-platform/video_upload.jpg)

After getting the original video from the user, several operations will be performed:
* Multi-part resolver will chunk the file into smaller pieces so that it will be easier to transfer between components. It also helps to speed up later operations by processing smaller chunks in parallel using a cluster. This might not be necessary if we are using some MR tools on the market, like Spark or AWS EMR since these tools handle the partition internally. 
* Content filter will use filter out illegal and inappropriate content, some image processing tools or ML models can be used for screening.
* Content Classifier and Tagger will be used to generate tags based on sampling frames and classify the content into appropriate ratings.
* Transcoder will convert the original content into different resolutions, dimensions, formats, and bitrates.

    ![transcoder](/case-study/streaming-platform/transcoder.jpg)
* After the upload, metadata about the content will be ingested into the Elastic Search cluster for indexing. This supports the Search Service on the homepage. A variety of options are available when delivering your data into the Elastic Stack: Elastic Agent, Beats, Logstash, or a direct connection from a client application via RESTful API.
* *Other optional operations like thumbnail generator or watermark addition.*
* At this point, we have all the contents (both original data and derived data) that are ready to be uploaded. Kafka will send another message to Content Service which handles the actual CRUD of contents. Media contents will be stored in a file system (blob storage) like AWS S3 and its metadata (including s3 URI) will be stored in a distributed NoSQL DB such as Cassandra because this will be a huge amount of read and "fixed" query patterns. Another important aspect of storing media content is the usage of CDN. CDN allows us to distribute our content to different geographical locations via a group of interconnected servers. They provide cached internet content from a network location closest to a user to speed up its delivery.

    A lot of cloud providers like AWS, GCP, Azure, and Cloudflare provide CDN services. It could be potentially very costly since our video contents require a lot of space so it might be more cost-efficient to build our own CDN network if we have the scale and resources. This will be one of the important design choices that require more evaluation. One thing to note here is that we do not have to put all our content on every CDN server (it's like putting the entire database in a cache). A carefully designed CDN strategy can be really helpful, for example, we can distribute content based on the language, popularity, and trends. We can also use a lazy loading strategy to preserve bandwidth and network contention.

-----------------------

### User HomePage Flow
The user homepage is supported by two major components:
* Search Service

    In order to support Fuzzy search, typo tolerance, and type ahead, we use Elastic Search as the text search engine. The input to the ES cluster is triggered by the message queue after uploading so the search results will be available shortly after that. 
* Recommendations

    Other than the search box, we also display recommended content to the user based on the user's profile and behaviors. These predictions are not calculated on the fly. They are computed **offline** and stored in a key-value NoSQL database so a simple GET request will fetch the recommendations. The recommendation data is generated by an ML model based on a variety of inputs, like user behaviors, user profiles, and trends. 

-----------------------

### User Play Flow
The users can play any video by simply clicking on it. Since there are several copies across different CDN servers, we need to decide which one the client will be pulling from, this is handled by the Host Optimizer based on the user's location. When the user is playing, we log the status continuously to track where the user stopped watching for historical reference. Such information like, how long and which video has been watched is very valuable input to our Recommendation model. So we feed that information to the Kafka topic which will be later consumed by the Recommendation/Prediction Service. 


-----------------------
## Reference
https://docs.aws.amazon.com/whitepapers/latest/database-caching-strategies-using-redis/caching-patterns.html

https://www.geeksforgeeks.org/why-is-youtube-using-tcp-but-not-udp/

https://www.elastic.co/guide/en/cloud/current/ec-ingest-methods.html

