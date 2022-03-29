---
layout: default
title: LocalDataDomain
parent: Libraries Shared
nav_order: 1
---

# LocalDataDomain

## Preparation

First you should make sure that `Libraries` was initialized at some point and that you have an instance
of it ready:
```java
Libraries libraries = new Libraries();
libraries.enable();
```
This will make sure that you have both a connection and client for Redis and MongoDB.

## Data class

Data classes dont have to fulfill any special conditions besides having a **unique** identifier in
their namespace and being serializable by your GsonProvider.
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

This context is used as a backbone for a LocalDataDomain or GlobalDataDomain. You should create only
one DomainContext for each namespace individually. Namespaces for global and local domains should never be mixed.

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

Breakdown

The namespace that is used for the MongoCollection and Redis map space.
```java
        .namespace("mydata")
```

Just the RedissonClient instance you get from Libraries.
```java
        .redissonClient(libraries.getRedissonClient())
```

The classes of both key and value. Its type safe so you don't have to worry about messing up too much.
```java
        .keyClass(String.class)
        .valueClass(MyData.class)
```

The creator which is used if no instance is present in the system.
``Function<K, V>``
```java
        .creator(MyData::new)
```
Its short for:
```java
        Function<String, MyData> creator = (str) -> new MyData(str);
        ...
        .creator(creator)
```

The function that is used to get the id for a specific value. It is used for internal mappings.
In almost all cases you want to make sure that the ID is a property of the serialized class.
``Function<V, K>``
```java
        .keyFunction(MyData::getId)
```
Its short for:
```java
        Function<MyData, String> keyFunction = (data) -> data.getId();
        ...
        .keyFunction(keyFunction)
```

The backing database. Refer to the creation further up to make sure you use the proper DB.
```java
        .mongoDatabase(database)
```

## Creating the LocalDataDomain

For every namespace only one LocalDataDomain should be created. It is now the access point for
all of your data that should be persistent.
```java
LocalDataDomain<String, MyData> domain = DataManager.getInstance().getOrCreateLocalDomain(context);
```

## Using the LocalDataDomain

The LocalDataDomain is completely thread safe.
Before getting data from the domain you need to make sure that it was loaded at least once before or else
it will throw an `IllegalStateException`.
```java
domain.loadDataSync("BobBerger");
```
Or async:
```java
CompletableFuture<Void> future = domain.loadDataAsync("BobBerger");
```


To unload the data you can simply call:
```java
domain.unloadDataSync("BobBerger");
```
Or async:
```java
CompletableFuture<Void> future = domain.unloadDataAsync("BobBerger");
```

This should usually be tightly coupled to some sort of session. 
For example: Player data should be loaded when the player connects and unloaded when the player disconnects.

During the session you can always get the data using:
```java
MyData data = domain.getData("BobBerger");
```
This is all done on the host and no requests are being made. Only when loading and unloading the data.

To completely remove data from the local storage, the redis cache and the database you can call:
```java
domain.deleteDataGloballySync("BobBerger");
```
Or async:
```java
CompletableFuture<Void> future = domain.deleteDataGloballyAsync("BobBerger");
```
Now next time the data is requested, the creator will create a new instance.

If you want to temporarily edit data locally then you can chain calls like this:
```java
domain.loadAndGetAsync("BobBerger")
        .thenAccept(data -> data.setValue(10))
        .thenRun(() -> domain.unloadDataSync("BobBerger"));
```

However, working with local data domains async is strongly discouraged as it can lead to IllegalStateExcptions being
thrown really quickly if not careful.

Now to demonstrate how powerful this is, im going to provide a single class solution that persistently
tracks all the food and drinks a player consumes without having to worry about IO ever.
```java
public class FoodTracker implements Listener {

    private final LocalDataDomain<UUID, FoodData> domain;

    public FoodTracker(Libraries libraries) {
        MongoDatabase database = libraries.getMongoClient().getDatabase("playerdata");
        DomainContext<UUID, FoodData> context = DomainContext.<UUID, FoodData>builder()
                .namespace("food-data")
                .redissonClient(libraries.getRedissonClient())
                .keyClass(UUID.class)
                .valueClass(FoodData.class)
                .creator(FoodData::new)
                .keyFunction(FoodData::getOwnerID)
                .mongoDatabase(database)
                .build();
        this.domain = DataManager.getInstance().getOrCreateLocalDomain(context);
    }

    @EventHandler(priority = EventPriority.HIGH)
    public void onJoin(AsyncPlayerPreLoginEvent event) {
        if (event.getLoginResult() != AsyncPlayerPreLoginEvent.Result.ALLOWED) {
            return;
        }
        this.domain.loadDataSync(event.getUniqueId());
    }

    @EventHandler
    public void onQuit(PlayerQuitEvent event) {
        this.domain.unloadDataAsync(event.getPlayer().getUniqueId());
    }

    @EventHandler
    public void onConsume(PlayerItemConsumeEvent event) {
        FoodData food = this.domain.getData(event.getPlayer().getUniqueId());
        Material material = event.getItem().getType();
        food.data.compute(material, (key, value) -> value == null ? 1 : value + 1);
    }

    public int getAmountsConsumed(UUID playerID, Material type) {
        if (this.domain.isDataLoaded(playerID)) {
            return this.domain.getData(playerID).data.get(type);
        } else {
            int amount = this.domain.loadAndGetSync(playerID).data.get(type);
            this.domain.unloadDataAsync(playerID);
            return amount;
        }
    }

    @Data
    @RequiredArgsConstructor
    public static class FoodData {
        private final UUID ownerID;
        private final Map<Material, Integer> data = new HashMap<>();
    }

}
```

This is a super compact solution. To create new persistent data you don't need to register anything
or even sync old data. You simply add new data to existing classes or create new data classes with their own domains.