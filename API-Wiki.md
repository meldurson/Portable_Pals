# Portable Pals API

The **Portable Pals API** allows other Valheim mods to integrate with Portable Pals and preserve mod-specific creature data when a creature is captured and stored in a Palstone.

The API provides several integration points for:

* Saving ZDO values to a Palstone
* Saving custom mod data to a Palstone
* Restoring/modifying a creature before it wakes
* Modifying a creature after it wakes
* Preventing specific creatures from being captured
* Safely adding or updating custom data

## Referencing the API

Add `PortablePals.dll` as a reference to your mod project.

Then import the namespace:

```csharp
using PortablePals;
```

You can then access the API through:

```csharp
PortablePals.API
```

> **Important:** Portable Pals does not need to be a hard dependency if your mod uses reflection or another optional-mod compatibility system. If Portable Pals is required for your integration, however, it can be referenced normally.

---

# 1. Saving ZDO Values

## `AddZDOToSavedValues`

```csharp
public static void AddZDOToSavedValues(string zdoKey)
```

Use this if all you need is for Portable Pals to automatically preserve that a zdo value when the creature is captured, and write it when released.

### Example

Suppose your mod stores a value in the creature's ZDO:

```csharp
character.m_nview.GetZDO().Set("MyMod_CustomValue", 10);
```

Register that ZDO key with Portable Pals:

```csharp
PortablePals.API.AddZDOToSavedValues("MyMod_CustomValue");
```

Portable Pals will then include that ZDO value when the creature is converted into a Palstone.

### Recommended usage

Register your ZDO keys when your mod initializes:

```csharp
void Awake()
{
    PortablePals.API.AddZDOToSavedValues("MyMod_CustomValue");
    PortablePals.API.AddZDOToSavedValues("MyMod_AnotherValue");
}
```

You only need to register each key once.

Portable Pals internally checks whether the key has already been registered, so calling this multiple times with the same key is safe.



Do not register the value's current contents, only the key.

---

# 2. Patching Catch and Release

### `AddOrUpdateCustomData`
This is a convenience method I have for safely adding or updating an entry in the custom data dictionary.
```csharp
public static void AddOrUpdateCustomData(Dictionary<string, string> customData, string key, string newValue)
```
The `customData` dictionary a reference to the `ItemDrop.ItemData` that Portable Pals is preparing to save to the Palstone.

## `SaveDataToPalStone`
By patching this method you can add your own information to the custom data stored in the Palstone.
```csharp
public static void SaveDataToPalStone(Character character, Dictionary<string, string> customData)
```

### Example

For example, using Harmony:

```csharp
[HarmonyPatch(typeof(PortablePals.API), nameof(PortablePals.API.SaveDataToPalStone))]
public static class SaveDataToPalStonePatch
{
    static void Prefix(Character character,Dictionary<string, string> customData)
    {
        AddOrUpdateCustomData(customData, "MyMod_Level", "5")
        AddOrUpdateCustomData(customData, "MyMod_Type", "Fire")
    }
}
```

### What should custom data be used for?

Use custom data for information that isn't already stored in a ZDO, or when you need to construct data specifically for the Palstone.

For example:

```text
MyMod_Level = 5
MyMod_Type = Fire
```

### Naming your keys

Use a unique prefix based on your mod name to avoid conflicts with other mods.


**Recommended:**

```csharp
"MyMod_Level"
"MyMod_Type"
```

**Avoid:**

```csharp
"Level"
"Type"
```

Using a unique prefix prevents different mods from accidentally overwriting each other's data.

---

# 3. Modifying a Creature When it is Released

I have made two different places to patch to modify the creature. Before `Character.Awake` and After.

## `ReleaseCreature_PreAwake`

```csharp
public static void ReleaseCreature_PreAwake(Character character, Dictionary<string, string> customData)
```

This is a Harmony patch point intended for modifications that need to happen **before the released creature's components have run their "Awake" function**.

### Example

```csharp
[HarmonyPatch(typeof(PortablePals.API),nameof(PortablePals.API.ReleaseCreature_PreAwake))]
public static class ReleaseCreaturePreAwakePatch
{
    static void Prefix(Character character, Dictionary<string, string> customData)
    {
        if (customData.TryGetValue("MyMod_Level",out string level))
        {
            int value = int.Parse(level);

            character.GetComponent<YourModComponent>().level = value;
        }
    }
}
```

### When should I use PreAwake?

Use `PreAwake` when your data needs to be available **before the creature initializes**.

For example:

* Setting initialization parameters
* Restoring data required by another component's `Awake()`
* Setting state that other components read during initialization

This is the preferred integration point when timing is important.

---


## `ReleaseCreature_PostAwake`

```csharp
public static void ReleaseCreature_PostAwake(Character character, Dictionary<string, string> customData)
```

This is called after the released creature has awakened.

Use this when your mod needs to interact with components that require the creature to already be initialized.

### Example

```csharp
[HarmonyPatch(typeof(PortablePals.API),nameof(PortablePals.API.ReleaseCreature_PostAwake))]
public static class ReleaseCreaturePostAwakePatch
{
    static void Prefix(Character character,Dictionary<string, string> customData)
    {
        if (customData.TryGetValue("MyMod_Type",out string type))
        {
            character.GetComponent<YourModComponent>().setType(type);
        }
    }
}
```


### When should I use PostAwake?

Use `PostAwake` when:

* You need to access initialized components
* Your component requires `Awake()` to have already executed
* You need to perform an action on the fully initialized creature
* The operation cannot safely happen during `PreAwake`

---

# 5. Preventing a Creature From Being Captured

## `CanCapture`

```csharp
public static bool CanCapture(Character character, Player player)
```

This API provides a compatibility point for mods that need to prevent certain creatures from being captured into a Palstone.

If the creature can be captured it should return `true`. Best is to only change the `__result` if you are setting it to false.

__Important:__ When modifying the result, use: `ref bool __result` which allows your patch to change the value returned by the API.

### Example Harmony patch

```csharp
[HarmonyPatch(typeof(PortablePals.API),nameof(PortablePals.API.CanCapture))]
public static class CanCapturePatch
{
    static void Postfix(Character character,Player player,ref bool __result)
    {
        if(!__result){return;}//don't bother checking, as already failed
        if (character.GetComponent<MySpecialComponent>() != null)
        {
            __result = false;
        }
    }
}
```

This allows your mod to add additional capture restrictions without directly modifying Portable Pals.



---



# API Summary

| API                           |   Purpose                                                       |
| ----------------------------- | ------------------------------------------------------------- |
| `AddZDOToSavedValues()`       | Tell Portable Pals to preserve a specific ZDO value           |
| `AddOrUpdateCustomData()`     | Safely add or update Palstone custom data                     |
| `SaveDataToPalStone()`        | Patch point for adding custom data when capturing             |
| `ReleaseCreature_PreAwake()`  | Patch point for restoring data before the creature wakes      |
| `ReleaseCreature_PostAwake()` | Patch point for restoring/using data after the creature wakes |
| `CanCapture()`                | Patch point for preventing a creature from being captured     |



### Best Practices

**Use ZDO storage when:**

* The data already exists in the creature's ZDO.
* You want Portable Pals to automatically preserve it.

**Use custom data when:**

* The information isn't stored in the ZDO.
* You need to construct or transform the data during capture.

**Use `PreAwake` when:**

* The data must be available during initialization.
* Another component reads the value during `Awake()`.

**Use `PostAwake` when:**

* You need initialized components.
* Your restoration code requires the creature to be fully awake.

**Use unique custom-data keys:**

```csharp
"MyMod_SomeValue"
```

rather than generic names such as:

```csharp
"SomeValue"
```

This prevents compatibility problems between mods.
