---
layout: default
title: GUI API
parent: Libraries Bukkit
nav_order: 1
---

# GUI API

## Creating a Menu

Creating a Menu is pretty simply. All you have to do is extend `Menu`:
```java
public class SomeMenu extends Menu {

    @Override
    protected Inventory createEmptyInventory(Player player) {
        String invName = "Hi %s".formatted(player.getName());
        return Bukkit.createInventory(null, 3 * 9, invName);
    }

    @Override
    protected void setup() {
        // Will be called in the constructor
    }

}
```

## Opening a menu
This Menu can then be opened by simply creating an instance and calling open on it:
```java
new SomeMenu().open(player);
```

## Adding buttons

The Button class is a simple builder type composition class with two components:
```java
Button exitButton = Button.builder()
        .itemCreator(player -> {
            ItemStack exit = new ItemStack(Material.WOOL);
            exit.setData(new Wool(DyeColor.RED));
            ItemMeta meta = exit.getItemMeta();
            meta.setDisplayName(ChatColor.RED + "Exit");
            exit.setItemMeta(meta);
            return exit;
        }).eventConsumer(event -> {
            Player player = (Player) event.getWhoClicked();
            player.sendMessage("Closed the inventory.");
            player.closeInventory();
        }).build();
```

This can also be simplified by using the ItemBuild
```java
        Button exitButton = Button.builder()
                .itemCreator(player -> new ItemBuilder(Material.WOOL)
                        .name(ChatColor.RED + "Exit")
                        .woolColor(DyeColor.RED)
                        .build())
                .eventConsumer(event -> {
                    Player player = (Player) event.getWhoClicked();
                    player.sendMessage("Closed the inventory.");
                    player.closeInventory();
                }).build();
```

You have 4 different ways of adding a button:
```java
super.buttonMap.put(8, exitButton); // Places on index 9
this.setButton(8, exitButton); // Places on index 9
this.setButton(ButtonPosition.of(8, 0), exitButton); // places on x = 8 & y = 0
this.addButton(exitButton); // Adds the button on the next free position
```

## Additional functionality

We can optionally overwrite 4 methods to gain more control over the menu:
```java
public class SomeMenu extends Menu {

    @Override
    protected Inventory createEmptyInventory(Player player) {
        String invName = "Hi %s".formatted(player.getName());
        return Bukkit.createInventory(null, 3 * 9, invName);
    }

    @Override
    protected void setup() {

    }

    @Override
    protected void onFirstOpen(InventoryOpenEvent event) {
        super.onFirstOpen(event); // Super call does nothing
    }

    @Override
    protected void onClose(InventoryCloseEvent event) {
        super.onClose(event); // Super call does nothing
    }

    @Override
    protected void onBottomClick(InventoryClickEvent event) {
        super.onBottomClick(event); // Super call cancels the event. Remove if you want to allow clicks.
    }

    @Override
    protected void onTopClick(InventoryClickEvent event) {
        super.onTopClick(event); // Super call cancels the event. Remove if you want to allow clicks.
    }
}
```

Calling super for the methods `onFirstOpen` and `onClose` does nothing.
Calling super for the methods `onBottomClick` and `onTopClick` cancels the event.
You can just decide to not call super to make the bottom or top of the inventory editable.
Clicks on buttons will always be cancelled.

## Dynamic inventories

Changing the inventory
You can have conditional buttons. This one will vanish after it is clicked one time:
```java
@Override
protected void setup() {
    this.addButton(Button.builder()
        .itemCreator(player -> {
            if (EulaCheck.hasAccepted(player)) {
                return null;
            } else {
                return /*create icon button here.*/;
            }
        }).eventConsumer(event -> {
            Player player = (Player) event.getWhoClicked();
            if (EulaCheck.hasAccepted(player)) {
                return;
            }
            EulaCheck.accept(player);
            this.reOpen(player);
        }).build());
    }
```

## Notice

If the Menu is **stateless(!)** then it is safe to have one instance which can then be opened for
several players. In most cases however you should create a new instance per player.