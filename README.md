# ![Cardinal Components API](banner.svg)
A components API for Quilt and Fabric that is easy, modular, and extremely fast.

*Cardinal Components API is a library for Minecraft mods to create data
components that can be attached to various providers. Those components
provide a standardized interface for mods to interact with otherwise opaque
objects and behaviours, thereby helping both mod creation and compatibility.*


**TL;DR: It allows you to attach data to things**


Detailed information is available in this repository's [**wiki**](https://github.com/OnyxStudios/Cardinal-Components-API/wiki).
The information below is a condensed form of the latter.

## Features\*
- 🔗 Attach your components to a variety of vanilla classes
- 🧩 Implement once, plug anywhere - your data will be saved automatically
- 📤 Synchronize data with a single helper interface
- 👥 Choose how your components are copied when a player respawns
- ⏲️ Tick your components alongside their target
- 🛠️ Fine-tune everything so that it fits your needs
- ☄️ And enjoy the blazing speed of ASM-generated extensions

*\*Non exhaustive, refer to the wiki and javadoc for the full list.*

## Adding the API to your buildscript:

**Upgrade information: versions 4.1.0 onwards of Cardinal Components API use the `dev.onyxstudios.cardinal-components-api` (lowercase) maven group instead of `io.github.onyxstudios.Cardinal-Components-API`**

Latest versions of Cardinal Components API are available on Artifactory:
```gradle
repositories {
    maven {
        name = 'Ladysnake Mods'
        url = 'https://ladysnake.jfrog.io/artifactory/mods'
    }
}

dependencies {
    // Adds a dependency on the base cardinal components module (required by every other module)
    // Replace modImplementation with modApi if you expose components in your own API
    modImplementation "dev.onyxstudios.cardinal-components-api:cardinal-components-base:<VERSION>"
    // Adds a dependency on a specific module
    modImplementation "dev.onyxstudios.cardinal-components-api:<MODULE>:<VERSION>"
    // Includes Cardinal Components API as a Jar-in-Jar dependency (optional)
    include "dev.onyxstudios.cardinal-components-api:cardinal-components-base:<VERSION>"
    include "dev.onyxstudios.cardinal-components-api:<MODULE>:<VERSION>"
}
```

You can find the current version of the API in the [**releases**](https://github.com/OnyxStudios/Cardinal-Components-API/releases) tab of the repository on Github.

Cardinal Components API is split into several modules. To depend on the all-encompassing master jar, use the dependency string
`dev.onyxstudios.cardinal-components-api:cardinal-components-api:<VERSION>`.
That artifact brings every module to your dev env, but you often do not need all of them for a project.
Also note that the maven version of the fat jar is actually empty, so you will have to require users to install it from curseforge or modrinth if you do not bundle all required modules.

**[[List of individual module names and descriptions]](https://github.com/OnyxStudios/Cardinal-Components-API/wiki#modules)**

Example:
```gradle
// Adds an API dependency on the base cardinal components module (required by every other module)
modApi "dev.onyxstudios.cardinal-components-api:cardinal-components-base:<VERSION>"
// Adds an implementation dependency on the entity module
modImplementation "dev.onyxstudios.cardinal-components-api:cardinal-components-entity:<VERSION>"
```

## Basic Usage

To get started, you only need a class implementing `Component`.
It is recommended to have it split into an interface and an implementation,
so that internals get properly encapsulated and so that the component itself can be used
as an API by other mods.

**Minimal code example:**
```java
public interface IntComponent extends ComponentV3 {
    int getValue();
}

class RandomIntComponent implements IntComponent {
    private int value = (int) (Math.random() * 20);
    @Override public int getValue() { return this.value; }
    @Override public void readFromNbt(CompoundTag tag) { this.value = tag.getInt("value"); }
    @Override public void writeToNbt(CompoundTag tag) { tag.putInt("value", this.value); }
}
```
*Note: a component class can be reused for several component types*

If you want your component to be **automatically synchronized with watching clients**,
you can also add the [`AutoSyncedComponent`](https://github.com/OnyxStudios/Cardinal-Components-API/blob/master/cardinal-components-base/src/main/java/dev/onyxstudios/cca/api/v3/component/AutoSyncedComponent.java)
interface to your implementation:

```java
class SyncedIntComponent implements IntComponent, AutoSyncedComponent {
    private int value;
    private final Entity provider;  // or World, or whatever you are attaching to

    public SyncedIntComponent(Entity provider) { this.provider = provider; }

    public void setValue(int value) {
        this.value = value;
        MyComponents.MAGIK.sync(this.provider); // assuming MAGIK is the right key for this component
    }
    // implement everything else
}
```

**[[More information on component synchronization]](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Synchronizing-components)**

If you want your component to **tick alongside its provider**, you can add the [`ServerTickingComponent`](https://github.com/OnyxStudios/Cardinal-Components-API/blob/master/cardinal-components-base/src/main/java/dev/onyxstudios/cca/api/v3/component/ServerTickingComponent.java) or [`ClientTickingComponent`](https://github.com/OnyxStudios/Cardinal-Components-API/blob/master/cardinal-components-base/src/main/java/dev/onyxstudios/cca/api/v3/component/ClientTickingComponent.java)
(or both) to your *component interface* (here, `IntComponent`). If you'd rather add the ticking interface to a single
component subclass, you can use one of the specific methods provided in the individual modules.

```java
class IncrementingIntComponent implements IntComponent, ServerTickingComponent {
    private int value;
    @Override public void serverTick() { this.value++; }
    // implement everything else
}
```

*Serverside ticking is implemented for all providers except item stacks.
 Clientside ticking is only implemented for entities, block entities, and worlds.*

The next step is to choose an identifier for your component, and to declare it as a custom property in your mod's metadata:

**quilt.mod.json** (if you use [Quilt](https://quiltmc.org))
```json
{
    "schema_version": 1,
    "quilt_loader": {
        "id": "mymod"
    },
    "cardinal-components": [
        "mymod:magik"
    ]
}
```

**fabric.mod.json** (if you use [Fabric](https://fabricmc.net))
```json
{
    "schemaVersion": 1,
    "id": "mymod",

    "custom": {
        "cardinal-components": [
            "mymod:magik"
        ]
    }
}
```

Components can be provided by objects of various classes, depending on which modules you installed.
The most common providers are [entities](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Cardinal-Components-Entity),
[item stacks](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Cardinal-Components-Item),
[worlds](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Cardinal-Components-World)
and [chunks](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Cardinal-Components-Chunk),
but more are available.
To interact with them, you need to **register a component key**, using `ComponentRegistryV3#getOrCreate`;
the resulting `ComponentKey` instance has the query methods you need. You will also need to **attach your
component** to some providers (here, to players and worlds):

```java
public final class MyComponents implements EntityComponentInitializer, WorldComponentInitializer {
    public static final ComponentKey<IntComponent> MAGIK = 
        ComponentRegistryV3.INSTANCE.getOrCreate(new Identifier("mymod:magik"), IntComponent.class);
        
    @Override
    public void registerEntityComponentFactories(EntityComponentFactoryRegistry registry) {
        // Add the component to every PlayerEntity instance, and copy it on respawn with keepInventory
        registry.registerForPlayers(MAGIK, player -> new RandomIntComponent(), RespawnCopyStrategy.INVENTORY);
    }
    
    @Override
    public void registerWorldComponentFactories(WorldComponentFactoryRegistry registry) {
        // Add the component to every World instance
        registry.register(MAGIK, world -> new RandomIntComponent());
    }    
}
```

Do not forget to declare your component initializer as an entrypoint in your mod's metadata:

**quilt.mod.json** (if you use Quilt)

```json
{
    "quilt_loader": {
        "entrypoints": {
            "cardinal-components": "a.b.c.MyComponents"
        },
    }
}
```

**fabric.mod.json** (if you use Fabric)
```json
{
    "entrypoints": {
        "cardinal-components": [
            "a.b.c.MyComponents"
        ]
    },
}
```

**[[More information on component registration]](https://github.com/OnyxStudios/Cardinal-Components-API/wiki/Registering-and-using-a-component)**

Now, all that is left is to actually use that component. You can access individual instances of your component by using the dedicated getters on your `ComponentKey`:

```java
public static void useMagik(Entity provider) { // anything will work, as long as a module allows it!
    // Retrieve a provided component
    int magik = provider.getComponent(MAGIK).getValue();
    // Or, if the object is not guaranteed to provide that component:
    int magik = MAGIK.maybeGet(provider).map(IntComponent::getValue).orElse(0);
    // ...
}
```

## Test Mod
A test mod for the API is available in this repository, under `src/testmod`. It makes uses of most features from the API.
Its code is outlined in a secondary [readme](https://github.com/OnyxStudios/Cardinal-Components-API/blob/master/src/testmod/readme.md).
