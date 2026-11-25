# Technical Documentation

Frostpunk 2 is an Unreal Engine game. Instead of hunting for each value with Cheat Engine's normal scanner and saving pointer paths, the table uses Unreal's own bookkeeping to find the game objects that hold resources. All the logic is Lua inside Cheat Engine Auto Assembler scripts.

## Overview

```
Frostpunk2-Win64-Shipping.exe
  │
  ├─ AOB signatures ──► GNames (name pool)      used to read object names
  │                 ──► GUObjectArray           used to check that an object is alive
  │                 ──► Stockpile vtable        used to find every stockpile
  │                 ──► EconomyHolder vtable    used to find every economy component
  │
  └─ City actor (one per settlement)
        ├─ PXStockpileComponent
        │     +0xB8   TMap<UPXResource*, int64>  current stock
        │     +0x108  TMap<UPXResource*, int64>  capacity
        └─ PXEconomyHolderComponent
              +0xD8   TMap<UPXResource*, int>    production totals (militia → guard squads)
```

## 1. Finding engine globals with signatures

Addresses change with every game update and every launch, so the script searches the game's executable code for byte patterns (AOB signatures) instead. Each signature matches an instruction like `lea rax, [rip+offset]` that loads the address of a global. The script reads the 32-bit offset and resolves it:

```lua
local function riprel(a) return a + 7 + s32(readInteger(a + 3)) end
```

Four signatures are used:

| Signature | Resolves to | Purpose |
|---|---|---|
| `SIG_NAMES` | `FNamePool` (GNames) | Turns name indexes into strings |
| `SIG_OBJECTS` | `FUObjectArray` (GUObjectArray) | Checks whether an object is valid |
| `SIG_STOCKVT` | `PXStockpileComponent` vtable | Finds stockpile instances |
| `SIG_ECONVT` | `PXEconomyHolderComponent` vtable | Finds economy instances |

`AOBScanModuleUnique` is used when available so only the game module is searched. Otherwise the script falls back to a full executable-memory scan. If a signature isn't found, the script stops with an error, which usually means the game was updated.

## 2. Reading object names (`fp2.name`)

Every `UObject` stores an `FName` at `+0x18` (the name index) and `+0x1C` (the number suffix). Names live in the engine's name pool, which is split into blocks:

- block = `index >> 16`, looked up at `GNames + 0x10 + 8 * block`
- entry = `block + 2 * (index & 0xFFFF)`
- the 2-byte entry header holds the length (`header >> 6`) and a wide-string flag (bit 0)

Name pool strings are not null-terminated, so exactly `len` bytes are read. Wide names are skipped because none of the names the table needs use them.

The script reads the names of the resource objects (`DA_Core_FoodResource`, `DA_Core_Heatnotes`, and so on) and matches them against a lookup table. That's why it doesn't matter that the order of resources differs between saves.

## 3. Checking that an object is alive (`fp2.valid`)

Memory can still contain stale copies of an object's vtable pointer after the object is freed. To filter these out, the script reads the object's internal index at `+0xC` and looks it up in GUObjectArray:

- the number of elements is at `GUObjectArray + 0x24`
- the chunk table is at `GUObjectArray + 0x10`, with 65536 items per chunk
- each item is 24 bytes, and its first field is the object pointer

An object only counts as valid if the array slot at its index points back to it.

## 4. Finding every settlement (`fp2.scan`, `fp2.build`)

Every instance of a C++ class starts with a pointer to its class's vtable. So the script scans writable memory for 8-byte-aligned values equal to the stockpile vtable address. Each hit is a candidate `PXStockpileComponent`. Candidates are kept if they:

- pass the GUObjectArray check
- aren't the class default object (names that start with `Default__`)

The owner (the component's Outer at `+0x20`) gives the city actor's name. Settlements are sorted by object index so the list order stays stable.

**Main city detection:** Heatstamps, Prefabs and Cores are shared, but each settlement holds its own share. The settlement with the largest total of those three is treated as the main city.

**Linking economy components:** The same vtable scan runs for `PXEconomyHolderComponent`. Each one is matched to its stockpile by sharing the same owner pointer.

## 5. Reading Unreal's TMap

Resources are stored in `TMap<UPXResource*, int64>`. The script reads the map's element array directly:

| Offset | Field |
|---|---|
| `+0x00` | pointer to element data |
| `+0x08` | element count |

Each element is 0x18 bytes:

| Offset | Field |
|---|---|
| `+0x00` | key (`UPXResource*`) |
| `+0x08` | value (amount) |
| `+0x10` | `HashNextId` (next element in the same hash bucket) |
| `+0x14` | `HashIndex` (the bucket this element is in) |

`fp2.readMap` walks the elements, resolves each key's name and returns a table of resource name → value address. Name lookups are cached by key pointer.

**Guard squads:** The visible guard squad count is recalculated every tick from the militia entry in the economy component's production map (`+0xD8`). Editing the count directly gets overwritten, so the table edits the militia production entry instead.

## 6. Keeping values up to date (`fp2.tick`)

A Cheat Engine timer runs every 500 ms:

- It checks that every stockpile and economy component is still valid. If any isn't (for example, after loading another save), it rebuilds the whole list. Rebuilds happen at most every 5 seconds.
- Otherwise it re-reads each map and updates each table entry's address. Map elements can move when the game reallocates them, so the address is never cached for long.
- Shared resource entries show the total across all settlements in their name.
- If a resource doesn't exist in a stockpile, the entry shows "(none in this stockpile yet)".

## 7. Creating missing entries (`fp2.insert`)

A colony that has never had Food, Materials or Goods has no map entry, so there's nothing to edit. The script can add the entry by writing directly into Unreal's TMap structures, which means reproducing what the engine does on insert:

| Offset | Field |
|---|---|
| `+0x00` | element data |
| `+0x08` | `ArrayNum` |
| `+0x0C` | `ArrayMax` |
| `+0x10` | inline allocation bits |
| `+0x20` | heap allocation bits |
| `+0x28` | `NumBits` |
| `+0x30` | `FirstFreeIndex` |
| `+0x34` | `NumFreeIndices` |
| `+0x38` | inline hash buckets |
| `+0x40` | heap hash buckets |
| `+0x48` | `HashSize` |

The key hash is the MurmurHash3 finalizer applied to `pointer >> 4`, the same as Unreal's pointer hash:

```lua
local function keyHash(p)
  local x = (p // 16) % 0x100000000
  x = x ~ (x // 65536)
  x = (x * 0x85EBCA6B) % 0x100000000
  x = x ~ (x // 8192)
  x = (x * 0xC2B2AE35) % 0x100000000
  x = x ~ (x // 65536)
  return x
end
```

### Safety checks

A wrong write here could crash the game or corrupt a save, so before writing anything the script checks that its model of the map matches reality. It refuses if:

- the map has free slots (holes) from removed elements
- the allocation bit count doesn't match the element count
- the map is full (adding an element would need a reallocation, which the script doesn't do)
- the hash size is 0 or an unexpected odd number
- any existing element isn't marked allocated, or its stored `HashIndex` doesn't match the recomputed hash
- walking every hash bucket chain doesn't reach exactly every element once

### The write

1. Write the new element at index `ArrayNum`: the key, a value of 0, the current bucket head as `HashNextId`, and the bucket index.
2. Set its allocation bit.
3. Increase `NumBits` and `ArrayNum`.
4. Point the bucket head at the new element.

The resource key pointer is taken from another settlement that already has that resource, or from the capacity maps.

## 8. Fallback: "Find resources"

The older script doesn't depend on any signatures. You enter your current Heatstamps, Prefabs and Cores. It scans memory for the Heatstamps value, then looks for the other two values within 16 map entries (`0x18` apart) on either side. The first match is registered as the symbols `fp2res`, `fp2pf` and `fp2cores`.

## Limitations

- The signatures and offsets match the game version the table was made for. A game update that changes the code or the class layout will break the main script. The error message names the signature that failed.
- Inserting into a full map isn't supported, because growing the allocation safely would mean calling the game's own allocator.
- Only the resources listed in the `RES` table are shown. Adding another resource only needs its object name added to the list.
