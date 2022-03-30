---
layout: default
title: GlobalDataDomain
parent: Libraries Shared
nav_order: 2
---

# GlobalDataDomain

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

## Creating the GlobalDataDomain

For every namespace only one LocalDataDomain should be created. It is now the access point for
all of your data that should be persistent.
```java
GlobalDataDomain<String, MyData> domain = DataManager.getInstance().getOrCreateGlobalDomain(context);
```

## Changing remote data

Editing remote data is always done using a functional approach.
This will
1) Get a lock and lock it
2) Get the data or create new one if not present
3) Apply the Consumer<T> to the data
4) Write the data back
5) Release the lock
```java
domain.applyToData("SomeData", data -> {
    if (data.getValue() < 10) {
        data.setValue(data.getValue() + 1);
    }
});
```
Or asnyc:
```java
        CompletableFuture<Void> future = domain.applyToDataAsync("SomeData", data -> {
            if (data.getValue() < 10) {
                data.setValue(data.getValue() + 1);
            }
        });
```

It assures 100% consistency with every other application that is modifying this data as long
as everyone **only** uses the data domains to access the backing cache/database.

## Read only

Sometimes you just want to read some information
```java
// Gets a fast snapshot of the current data. No locks required.
Optional<MyData> dataOptional = domain.getDataSnapshot("SomeData");
// Gets a safer snapshot of the data. Is locked while retrieving but the
// data should still be considered outdated instantly.
Optional<MyData> dataOptional = domain.getRealTimeData("SomeData");
```

## Deleting data

Deleting data globally is as simple as calling:
```java
domain.deleteDataGloballySync("SomeData");
```
Or async
```java
CompletableFuture<Void> future = domain.deleteDataGloballyAsync("SomeData");
```

## Additional functionality

Example of transferring the value of one object to another:
```java
domain.applyToBoth("DataA", "DataB", (valueA, valueB) -> {
    valueB.setValue(valueA.getValue());
    valueA.setValue(0);
});
```

Sometimes you want to assure that some computations want to be run in sync.
This will first acquire all locks for every single key and afterwards apply an action
to them before releasing every lock again:
```java
domain.applyToAll(List.of("Bob", "Peter", "Sandy"), member -> member.setValue(0));
```

## Examples

This is simple read example of telling a player his money:
```java
public void tellPlayerMoney(Player player) {
    player.sendMessage("Loading your balance...");
    this.domain.getDataSnapshotAsync(player.getUniqueId()).thenAccept(balance -> {
        Bukkit.getScheduler().runTask(LibrariesPlugin.getInstance(), () -> {
            player.sendMessage("Your balance is %.2f".formatted(balance.orElse(0D)));
        });
    });
}
```

A bit more complex when it comes to transactions:
```java
public boolean transferMoney(UUID senderID, UUID receiverID, int amount) {
    return this.domain.biCompute(senderID, receiverID, (senderData, receiverData) -> {
        MoneyAccount receiverAccount = receiverData.getMoneyAccount();
        MoneyAccount senderAccount = senderData.getMoneyAccount();
        if (senderAccount.hasAtLeast(amount)) {
            return false;
        }
        senderAccount.remove(amount);
        receiverAccount.add(amount);
        return true;
    });
}
 
public void transferMoney(Player sender, Player receiver, int amount) {
    CompletableFuture.supplyAsync(() -> this.transferMoney(sender.getUniqueId(), receiver.getUniqueId(), amount))
        .thenAccept(result -> {
            if (result) {
                sender.sendMessage("You have sent %d$ to %s.".formatted(amount, receiver.getName()));
                receiver.sendMessage("You have received %d$ to %s.".formatted(amount, sender.getName()));
            } else {
                sender.sendMessage("You dont have enough money.");
            }
        });
}
```