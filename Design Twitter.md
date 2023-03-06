# Design Twitter

Twitter is a microblogging application with thousands of breaking news. It's heavier on reading than writing. High-level design and detailed design can be challenging, so please spend more time on these parts.

## Functional Requirements

Before we dive into the design, let's clarify the requirements. Here are a few things I will focus on:

- A user can post a tweet.
- Users can see the tweet on their own timeline.
- Ability to follow other users' accounts and interact with them.

## Non-Functional Requirements

We also need to clarify the non-functional requirements to decide what kind of architecture it will be.

- First is **High Availability**. Twitter is used worldwide and has many users. Even a minute of downtime will affect many users.
- Next is **Scalability**. With breaking news happening, there will be many users actively using it. Ideally, the system's performance should not be affected.
- Then, **Low Latency** is crucial, especially for browsing timelines.
- Finally, **Reliability** is important. Every tweet, like, and reply should be stored in the system. However, we don't have to require strong consistency. As a tradeoff, we allow there to be a bit of lag after a user sends a tweet, and followers can see the tweet a few seconds later.

Basically, these are all the requirements we have for MVP.

## High-Level Design

Now, let's move on to high-level design.

First, a user, whether from a browser or an app, can post a tweet. The post request will first be taken by a Load Balancer. Usually, for services like Twitter that are open to the internet, a Load Balancer is needed to distribute requests evenly and choose a geographically closer server to the user to lower latency. We will have a Tweet Service to handle the post request from the Load Balancer. Finally, the tweet will be saved in the database. Considering the scalability requirements, we can use a NoSQL database here.

Next, let's see how the tweet that was just posted can be seen. Twitter is much heavier on reading than writing. Here, we have another user who is a follower of the previous user. They send a get Timeline request to the Load Balancer, which distributes it to the application service layer. I will place a Timeline Service here to handle the Timeline generation and so on. Since it is heavily on reading, we can precompute the timeline for every active user. After the Tweet Service receives the post tweet request, it can send an asynchronous request to the Timeline Service, so followers who are also active users can get the precomputed generated timeline. And the timeline will be stored in another database.

Q: For influencers who have millions of followers, is it necessary to compute timelines over and over again?

Influencers' tweets are read by millions of followers, so it will be a huge waste to recalculate the same thing. We can deal with the influencers specially in another service. Let's say we will have an Influencer Service here. All tweets from influencers will be stored in the cache. When users try to get their timeline, they will get tweets from both Timeline Service and Influencer Service. In this way, we can avoid unnecessary waste.

Q: If a PM came to you and said we have an 80/20 scenario, like 80 percent of users are interacting with 20 percent of tweets, how would you optimize your system?

Tweets that are read by many people can be stored in the cache, similar to those of influencers. To calculate likes and replies for these tweets, we can use a Sharded Counter, and then compute the Top K tweets and put them into the cache. This approach can help lower latency and reduce contention for writes.

Let's move on to the other requirements.

To allow users to interact with tweets, we can take "like" and "reply" as examples. The user's request will be distributed to the Tweet Service, and then the system will notify the relevant users.

For high availability, we can replicate the database using a master-slave cluster. If the master goes down, a new master will be elected from the slaves. Each service, such as Tweet and Timeline services, will have multiple instances, and the Load Balancer will distribute requests to healthy instances. This design can achieve high availability.

For scalability, we can partition user and tweet data based on regions. If a region has more users or tweets, we can allocate more resources to it. This approach can meet scalability requirements.

To reduce latency, we can separate the Timeline generation process from the tweet posting process using asynchronous methods. If time permits, we can also add a Pub/Sub service between the Tweet and Timeline services.

![](https://raw.githubusercontent.com/wheeler-tree/WTreePic/main/202303061047614.png)