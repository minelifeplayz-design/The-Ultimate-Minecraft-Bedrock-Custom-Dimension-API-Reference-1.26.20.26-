# Minecraft Bedrock 1.26.20.26 Custom Dimension API Reference

## Executive Summary

Minecraft Bedrock Edition 1.26.20.26 introduces a *Custom Dimension* API that allows creators to register and script new world dimensions beyond the vanilla Overworld, Nether, and End. These dimensions start as void spaces, and all terrain, structures, lighting, and biomes must be defined via scripts. The API is experimental (Beta) and requires enabling the appropriate game options. Key features include the `DimensionRegistry.registerCustomDimension()` method for defining dimensions at startup, methods on the `Dimension` object (spawn experience or entities, modify blocks, query biomes/blocks/entities), and use of ticking areas to keep chunks loaded during procedural generation or teleportation. This documentation provides a full reference to the relevant classes, interfaces, events, manifest schemas, code examples, and best practices for using the Custom Dimension API in MCPE 1.26.20.26. All official APIs are cited; undocumented behaviors are noted from community sources.

## 1. Introduction and Background

Prior to 1.26.20.26, Bedrock addons could not create new dimensions via scripts. With this update, Mojang added the `DimensionRegistry` class and related APIs. Custom dimensions are defined by a unique dimension identifier (namespace:name) and behave initially as empty (void) worlds. Users must build all terrain and features via scripting. Official documentation notes that new dimensions can only be registered during the `system.beforeEvents.startup` event; attempting to add dimensions later (e.g. via `/reload`) throws a `CustomDimensionReloadNewDimensionError`. 

Custom dimensions require a behavior pack with a script module. The manifest must include the `@minecraft/server` (and any other needed modules like `@minecraft/server-ui`) as dependencies with `"version": "beta"`. Example manifest structure (format version 2) is: 

```json
{
  "format_version": 2,
  "header": {
    "name": "My Custom Dimensions Pack",
    "description": "Adds new dimensions",
    "min_engine_version": [1,26,20]
  },
  "modules": [
    {
      "type": "script",
      "language": "javascript",
      "entry": "scripts/main.js",
      "version": [1,0,0],
      "uuid": "00000000-0000-0000-0000-000000000000"
    }
  ],
  "dependencies": [
    {"module_name": "@minecraft/server", "version": "beta"},
    {"module_name": "@minecraft/server-ui", "version": "beta"}
  ]
}
```

The user must enable **Beta APIs** (and the **Custom Dimensions** toggle if separate) in the world settings. Failure to do so (or missing the script entry) will prevent the custom dimensions from registering.

## 2. Custom Dimension Classes and Types

### 2.1 DimensionRegistry (Class)

- **Package:** `@minecraft/server`  
- **Description:** Manages registration of new dimensions during startup.  
- **Key Method:** `registerCustomDimension(typeId: string): void`. The `typeId` is a namespaced string (e.g. `"myaddon:my_dimension"`). Calling this registers the dimension with a void generator. If the dimension ID is already used, it throws `CustomDimensionAlreadyRegisteredError`; invalid names or calls outside startup throw `CustomDimensionNameError` or `CustomDimensionInvalidRegistryError` respectively.

#### DimensionRegistry.registerCustomDimension
```ts
registerCustomDimension(typeId: string): void
```
Registers a new dimension identified by `typeId`. Must be called in a handler for `system.beforeEvents.startup`. Throws:
- `CustomDimensionAlreadyRegisteredError` if the ID is already in use.
- `CustomDimensionInvalidRegistryError` if called outside startup.
- `CustomDimensionNameError` if the `typeId` is malformed.

### 2.2 Dimension (Class)

- **Obtained via:** `world.getDimension(dimensionId: string)` or passed in events.  
- **Description:** Represents a loaded dimension. Provides methods to get or set blocks, spawn entities, query biomes, etc. All dimensions (overworld, Nether, End, and custom dims) use this same class.  

#### Notable Properties (Methods/Accessors):
- `id: string` – the dimension’s identifier (e.g. `"myaddon:my_dimension"`).  
- `spawnSettings: ...` – read-only default spawn settings.  
- `world: World` – the world object containing this dimension (rarely needed).  

#### Notable Methods:
- **Block and Biome Queries:**  
  - `getBlock(loc: Vector3): Block | undefined` – returns the block at `loc`, or `undefined` if chunk is unloaded. Throws `LocationInUnloadedChunkError` if the chunk isn’t loaded.  
  - `getBlockAbove(loc: Vector3, options?): Block | undefined` – finds first solid block above `loc` based on raycast options.  
  - `getBlockBelow(loc: Vector3, options?): Block | undefined` – finds first solid block below `loc`.  
  - `setBlock(loc: Vector3, blockPerm: BlockPermutation)` – sets the block at `loc` to the given permutation. Throws if chunk is unloaded.  
  - `getBiome(loc: Vector3): BiomeType` – returns the biome at `loc`.  
  - `containsBiomes(locs: Vector3[]): boolean` – returns true if all given locations have valid biomes.  

- **Entity and Item Methods:**  
  - `spawnEntity(typeId: string, loc: DimensionLocation): Entity` – spawns an entity of the given type at `loc` (fails if spawning unsupported entity).  
  - `spawnItem(itemStack: ItemStack | ItemStack[], loc: DimensionLocation): Entity[]` – spawns one or more items at `loc`.  
  - `spawnXp(location: Vector3, amount: number): void` – spawns an experience orb at `location` worth `amount` XP. (Added in 1.26.20.26; still in beta).  
  - `tick(delta: number): void` – advances dimension’s time by `delta` ticks.  

- **Explosion and Weather:**  
  - `triggerExplosion(loc: Vector3, power: number): boolean` – triggers an explosion at `loc`.  
  - `setWeather(weather: WeatherType): void` – sets the weather in the dimension (e.g. clear, rain, thunder).  
  - `getSkyLightLevel(loc: Vector3): number` – returns sky light level at `loc`.

- **World Interactions:**  
  - `spawnParticle(particleName: string, loc: Vector3): void` – spawns a particle effect.  
  - `playSound(soundId: string, loc: Vector3, options?): void` – plays a sound at `loc` (hearing is global to players).  
  - `stopAllSounds(): void`, `stopSound(soundId: string): void` – stop sounds globally for this dimension.

Most dimension functions must be called in **non-restricted execution mode**. The API docs note that functions like `spawnXp` and others are in **preview** and can throw errors (e.g. `LocationOutOfWorldBoundariesError`, `LocationInUnloadedChunkError`). Always handle or avoid operations on unloaded chunks (use ticking areas to load them first).

### 2.3 Dimension Types

- **DimensionType (Enum/Class):** Static class listing vanilla dimension types (`overworld`, `nether`, `the_end`). These are *read-only* and cannot be used for custom dimensions. Custom dimension IDs are arbitrary namespaces and do not use a `DimensionType` entry.

- **DimensionLocation (Interface):** Identifies a location in a dimension (`{ dimension: Dimension, x: number, y: number, z: number }`). Used in some UI or callback APIs (e.g. `spawnEntity(..., {dimension, x,y,z})`).

- **Enums:**  
  - `BiomeType` – identifiers like `minecraft:plains`, `minecraft:desert`, etc.  
  - `TimeOfDay` – for setting day/night.  
  - `CommandPermissionLevel` – used when registering commands (Any, Operator, GameDirector) as in examples.  

## 3. Scripting Workflow & Events

- **system.beforeEvents.startup:** Primary event for registering dimensions. Example:  
  ```js
  system.beforeEvents.startup.subscribe((event) => {
    event.dimensionRegistry.registerCustomDimension("myaddon:my_dimension");
    // ... register commands if needed
  });
  ```  
  This must run once before the world loads. Attempting to register outside this event or during a `/reload` will throw errors.

- **world.afterEvents.worldLoad:** After the world (and dimensions) is loaded, you can safely build initial terrain or spawn entities. For example, the tutorial registers platforms once the world loads. This is also a good time to initialize random seeds or persistent data.

- **world.getDimension(id):** Use this to retrieve your dimension once registered. E.g. `const dim = world.getDimension("myaddon:my_dimension");`.

- **world.tickingAreaManager:** Critical for loading chunks. Use `createTickingArea(name, {dimension, from, to})` before building or teleporting, and `removeTickingArea(name)` when done. See Tutorial sections below.

- **Player and Entity Events:** Custom dimensions are transparent to most entity spawn and chat events. However, when teleporting players, ensure you set the correct `dimension` option. E.g. `player.teleport({ x, y, z }, { dimension: dim })`. There is no built-in "portal creation" API for custom dims – portals can be implemented by detecting when a player steps on a special block and then using `player.teleport` with dimension.

## 4. Tutorials and Usage Patterns

### 4.1 Creating and Registering a Dimension

1. **Setup Manifest & Script Module:** As noted, ensure `@minecraft/server` is in manifest dependencies with `"version": "beta"`. Include `"scripts/main.js"` (or .ts) entry.

2. **Register on Startup:** In `main.js`, do:
   ```js
   import { system } from "@minecraft/server";
   const MY_DIM = "myaddon:my_dimension";
   system.beforeEvents.startup.subscribe((event) => {
     event.dimensionRegistry.registerCustomDimension(MY_DIM);
   });
   ```
   This call will create a new dimension `myaddon:my_dimension` as a void world. If the ID is invalid or duplicates, an error will be thrown.

3. **Verify Registration:** On world load (or immediately after), you can call:
   ```js
   const dim = world.getDimension(MY_DIM);
   if (!dim) {
     console.error("Dimension not registered!");
   }
   ```
   (In practice, if registration fails, scripts may not even run.)

### 4.2 Building Terrain (Using Ticking Areas)

Since new dimensions are empty voids, you must populate them via script. Example pattern (from tutorials):

```js
import { world, system } from "@minecraft/server";

const MY_DIM = "myaddon:my_dimension";
system.beforeEvents.startup.subscribe((event) => {
  event.dimensionRegistry.registerCustomDimension(MY_DIM);
});

world.afterEvents.worldLoad.subscribe(() => {
  const dim = world.getDimension(MY_DIM);
  // Define a region to populate, e.g. a 32x32 area around spawn
  const spawn = { x: 0, y: 4, z: 0 };
  const areaId = "build_my_dim";
  // Create ticking area to load chunks
  await world.tickingAreaManager.createTickingArea(areaId, {
    dimension: dim,
    from: { x: -16, y: 0, z: -16 },
    to:   { x: 16,  y: 8, z: 16 }
  });
  // Now set blocks inside that area
  for (let x = -16; x <= 16; x++) {
    for (let z = -16; z <= 16; z++) {
      dim.getBlock({ x: x, y: 4, z: z })?.setPermutation(
        BlockPermutation.resolve("minecraft:stone")
      );
    }
  }
  // Remove ticking area when done to save resources
  world.tickingAreaManager.removeTickingArea(areaId);
});
```

Creating a ticking area **guarantees** the specified chunks are loaded on the server. The example above floors the area with stone. All block changes (setPermutation) must occur within loaded chunks. The `await` on `createTickingArea` is important; it pauses script until chunks are ready.

### 4.3 Teleportation Between Dimensions

To move players into a custom dimension, use `player.teleport()` with a `{ dimension }` option. A best practice is to load the arrival area first:

```js
function teleportPlayerToDim(player, dim, spawnPos) {
  const tickingAreaId = `${dim.id}_tele`;
  // 1. Load destination chunks around spawn
  await world.tickingAreaManager.createTickingArea(tickingAreaId, {
    dimension: dim,
    from: { x: spawnPos.x - 4, y: spawnPos.y - 4, z: spawnPos.z - 4 },
    to:   { x: spawnPos.x + 4, y: spawnPos.y + 4, z: spawnPos.z + 4 }
  });
  // (Optional) build or verify platform after load
  // 2. Teleport player
  player.teleport(spawnPos, { dimension: dim });
  // 3. Remove ticking area
  world.tickingAreaManager.removeTickingArea(tickingAreaId);
}
```

This pattern (from official tutorial) addresses two problems: it keeps arrival chunks loaded until teleport completes, and allows final environment checks. For example, before step 2 you might generate a spawn platform if not already done.

**Portal mechanics:** There is no built-in portal block that transports to custom dims. You must script any portal behavior. Common approaches include:
- Detecting player standing on a custom block (via an event) then calling `teleportPlayerToDim(...)`.
- Using commands (custom command triggers UI like `/custom_dim:travel` in examples).
Vanilla Nether portals do *not* automatically connect to scripted dimensions.

### 4.4 Chunk Loading and Ticking Areas

Custom dimensions behave like any dimension: chunks unload when no players are present, unless pinned by ticking areas. Use `world.tickingAreaManager` liberally:
- **Keep areas active:** For large or persistent generation (e.g. procedural islands), continuously load chunks around each player.
- **Named areas:** Give unique IDs to each area (e.g. `areaName = \`dim${x}_${z}\``) and `await createTickingArea(...)` to load a 16×16 region as shown in examples.
- **Remove when done:** Always call `removeTickingArea` when a generation task completes or when teleport finished.

**Note:** There is a maximum number of simultaneous ticking areas per world. Avoid creating too many or leaving them indefinitely. Unused areas should be removed as soon as possible.

### 4.5 Procedural Terrain Generation

Since custom dims are void, any terrain must be script-generated. Examples of approaches:

- **Static Builds:** For a fixed map (like “Umbra Depths” in community example), pre-generate entire structure at load. Use nested loops or generators to place blocks (see Umbra build code). Typically done once in `worldLoad`.

- **Dynamic/Procedural Builds:** For infinite or large dimensions, generate chunks on demand around players. Use `system.runInterval` to periodically queue chunks near players (as in the “Aether” example). Process the queue with an async job (`system.runJob`) and fill blocks via a generator function to avoid script timeouts. 

  Example (simplified):
  ```js
  const genRadius = 2, floorY = 10, topY = 50;
  function startGenLoop() {
    system.runInterval(() => {
      for (const player of world.getAllPlayers()) {
        if (player.dimension.id !== MY_DIM) continue;
        const pcx = Math.floor(player.location.x / 16);
        const pcz = Math.floor(player.location.z / 16);
        for (let dx = -genRadius; dx <= genRadius; dx++) {
          for (let dz = -genRadius; dz <= genRadius; dz++) {
            const chunkX = pcx + dx, chunkZ = pcz + dz;
            if (!generatedChunks.has(`${chunkX},${chunkZ}`)) {
              generatedChunks.add(`${chunkX},${chunkZ}`);
              queue.push({cx:chunkX, cz:chunkZ});
            }
          }
        }
      }
      if (!jobRunning && queue.length) {
        jobRunning = true;
        void processQueue();
      }
    }, 40); // tick every 40ms
  }
  async function processQueue() {
    while (queue.length) {
      const {cx, cz} = queue.shift();
      await runChunk(cx, cz);
    }
    jobRunning = false;
  }
  async function runChunk(cx, cz) {
    const originX = cx * 16, originZ = cz * 16;
    const areaId = `gen_${cx}_${cz}`;
    const dim = world.getDimension(MY_DIM);
    await world.tickingAreaManager.createTickingArea(areaId, {
      dimension: dim,
      from: {x: originX, y: floorY-1, z: originZ},
      to:   {x: originX+15, y: topY+4, z: originZ+15}
    });
    await new Promise(r => {
      system.runJob(generateChunkBlocks(dim, cx, cz, areaId, r));
    });
  }
  function* generateChunkBlocks(dim, cx, cz, areaId, onDone) {
    // e.g., fill bedrock layer
    const bedrock = BlockPermutation.resolve("minecraft:bedrock");
    for (let x = 0; x < 16; x++) {
      for (let z = 0; z < 16; z++) {
        dim.getBlock({x: originX+x, y: floorY, z: originZ+z})?.setPermutation(bedrock);
      }
      yield;
    }
    // ... generate terrain up to topY ...
    world.tickingAreaManager.removeTickingArea(areaId);
    onDone();
  }
  ```

  This pattern (used in community “Aether Expanse”) generates islands based on hash functions. The use of a generator (`function*`) with `yield` prevents blocking the main script thread.  

**Important:** Custom terrain generation does *not* use Minecraft’s built-in world generation. There are no built-in noise or seed-based functions exposed; any consistency must be manually coded (e.g. using a hash or `Math.random()` with seeded input). (Community members have noted the limitation: there’s no official “dimension features” API yet.)

### 4.6 Spawning Entities and Effects

Once chunks are loaded, you can spawn entities or effects in the dimension just as in the Overworld. For example:

```js
const dim = world.getDimension(MY_DIM);
const loc = {dimension: dim, x: 0, y: 5, z: 0};
dim.spawnEntity("minecraft:zombie", loc);
dim.spawnXp({x:0, y:5, z:0}, 100); // spawn XP orb, new in 1.26.20
```

Dimension methods like `getEntities()`, `spawnEntity()`, `spawnItem()`, and `spawnXp()` work the same in custom dims. Use `targetLocation.dimension` in scripts to get the `Dimension` from a `DimensionLocation` (as shown in example code).

### 4.7 Lighting and Biomes

Custom dimensions have no default biomes or lighting. By default, the entire dimension is as if it had no sky (similar to the Nether), so the sky light is minimal. You may need to set light blocks manually or simulate day/night in script. The new dimension can have biomes based on block replacement; however, the API provides `getBiome(loc)` but no direct biome-setting call. To “apply” a biome to an area, you usually fill it with blocks corresponding to that biome and use spawn settings. Biome queries (`getBiome`) can be used to check what biome the game considers at a location.

Lighting (time of day) can be set via `world.setTimeOfDay(TimeOfDay.Day)` to affect global time, but custom dims follow world time. Custom “light levels” must be implemented by placing light-emitting blocks or using the `/light` command. There is no per-dimension time control yet.

## 5. Manifest and Behavior Pack Details

- **Behavior Pack Structure:** The dimension scripts live in a Behavior Pack (BP). The BP needs a `manifest.json` at its root (format version 2). Under `modules` include a script module with `language: "javascript"` and point to your entry file (`scripts/main.js`). Ensure `"min_engine_version": [1,26,20]` or higher.

- **Dependencies:** Add `"@minecraft/server"` (and any other Minecraft modules like `"@minecraft/server-ui"`) under `dependencies` with version `"beta"`. This flags that you are using beta/preview scripting APIs. Without this, your scripts will be disabled or fail to load.

- **Experimental Flags:** The world in which this BP is used must have **Experimental Gameplay** (Beta APIs) toggled on, and specifically **Custom Dimensions** if it appears separately. The GitHub example notes this requirement.

- **Manifest Fields:** In addition to standard fields (`header`, `modules`, `dependencies`), there is no extra JSON for dimensions because they are purely script-managed. (In Java, custom dimension JSON exists, but in Bedrock scripting model, everything is runtime.)

- **Resource Pack (if used):** If you want custom textures or block models in the new dimension, include a Resource Pack with appropriate assets and an entry in the add-on **Pack** UI. The custom dimension itself does not require a separate JSON (like `dimension.json`) because the environment is defined by code.

## 6. TypeScript Typings and Declarations

The Bedrock script API is available as type declarations in `@minecraft/server` (along with `@minecraft/server-ui`, etc.). Key interfaces/classes relevant to dimensions:

- `DimensionRegistry` – interface with `registerCustom