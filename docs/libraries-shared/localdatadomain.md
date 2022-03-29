---
layout: default
title: LocalDataDomain
parent: Libraries Shared
nav_order: 1
---

# LocalDataDomain

## Preparation

First you should make sure that `Libraries` was initialized at some point and that you have an instance
of it ready
```java
Libraries libraries = new Libraries();
libraries.enable();
```
This will make sure that you have both a connection and client for Redis and MongoDB.

## Data class

Data classes dont have to fulfill any special conditions besides having a **unique** identifier in
their namespace.
```java
@Data
@RequiredArgsConstructor
public class MyData {

    private final String id;
    private int value;

}
```
The type of the identifier is used as a key type by the data domain.

## Database

You should have prepared a database for all your data to store in. The name of the database is
irrelevant but using the same name for all your grouped data is strongly recommended. This means that
all player related data might be stored in a database using the name `"playerdata"`.

```java
MongoDatabase database = libraries.getMongoClient().getDatabase("test");
```

## Creating a DomainContext

```java
DomainContext<String, MyData> context = DomainContext.<String, MyData>builder()
        .namespace("mydata")
        .redissonClient(libraries.getRedissonClient())
        .keyClass(String.class)
        .valueClass(MyData.class)
        .creator(MyData::new)
        .keyFunction(MyData::getId)
        .mongoDatabase(database)
        .build();
```