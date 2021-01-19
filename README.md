# Description
Aura is a simple library for managing status effects.

Specifically, there's an issue where the removal of one status effect interferes with other existing status effects. For example, suppose that stuns A & B are applied simultaneously, then removed after 1 and 2 seconds respectively. Depending on the implementation, the removal of stun A might allow the player to move again, even though stun B is still active for another second. In short, Aura solves this problem by checking if the player has any remaining active stuns before unstunning them.

# Documentation
Since you define effects and auras on your own, it's necessary to have a basic understanding of the libraries' datatypes and execution model.

## Effects
An effect is a scheme for a specific way to manipulate an object. For example, a speed boost effect would describe the default speed of the object, how to figure out the object's speed based on all the speed effects currently applied to it, and how to give the object that speed.

You create effects by constructing tables with 3 fields:

```lua
{
  Default: Variant, --describes default speed
  reduce: Function(sum: Variant, effectInstance: EffectInstance), --figures out the target speed based on all currently applied speed objects
  apply: Function(object: Instance, sum: Variant) -- applies the target speed
}
```

Here's an example of a speed boost effect

```lua
local speedBoost = {
  Default = 16,
  reduce = function(sum, effectInstance) return sum + effectInstance.Value end,
  apply = function(character, sum)
    character.Humanoid.WalkSpeed = sum
  end
}
```

The meaning of these fields will become more clear once we consider what aura does when applying and removing effects.

### Effect Instances
An effect instance is a table that represents an actual effect applied to an object. For every effect you actually apply, one effect instance will be generated. Effect Instances are empty by default, but you can specify custom fields to highlight the differences between instances of the same effect. For example, one stun effect may have a duration of 1, while another might have a duration of 2.

### Calculating Effects
Behind the scenes, every time an effect instance is applied/removed to or from an object the follow steps are executed:
1. The effect the effect instance belongs to is determined.
2. A value is initially set to effect.Default.
3. for each currently active effectInstance belonging to the effect, effect.Reduce is run on it and the previous value to determine a new value
4. effect.apply(object, value)

```lua
--pseudo code expressing what aura does behind the scenes
local effect -- whatever effect is associated with the effect instance
local value = effectInstance.Default

for _, effectInstance in pairs(object.EffectInstances[effect]) do
   value = effect.reduce(value, effectInstance)
end

effect.apply(object, value)
```

Suppose that there's one speed boost effect instance A with a custom field Value=10 ( meaning the speed boost will increase your walkspeed by 10 ) currently applied to an object. Consider what will happen if we apply another speed boost effect instance B with a Value of 20.

1. value is set to 16
2. value is set to effect.reduce(value, effectInstanceA), or 26.
3. value is set to effect.reduce(value, effectInstanceB), or 46.
4. effect.apply(object, value)

Recall that the speedBoost.reduce returns the first argument + the secondArgument.Value, and that effect.apply sets object.Humanoid.WalkSpeed equal to the second argument. With this, the character will have the correct walkspeed based on the effect instances currently applied.

Note that this specific case is academic. Addition is associative, so there's no issues with just adding and subtracting to walkspeed to implement speed boost, rather than using this library.

## Auras
An Aura is a package of one or more effects. Although we've talked about applying effects, there is no way for a user of the library to apply one directly. Users apply auras, then effect instances are generated and applied for each effect contained in the aura. Auras also serve as an interface between the caller and the underlying effect.

In actuality, an aura is of the type: Function(settings: Table) -> auraInstance

### Aura Instances
Like an effect instance, an aura instance is a tables representing an active aura applied to an object.

You create aura instances by constructing a table with the following fields:
```lua
{
  ...
  EffectInstances = {
    [effectId] = Table
    ...
  }
}
```

When an aura instance is applied, for each element in it's EffectInstances field, an effectInstance will be constructed and applied to the object. The effect is specified by effectId, and the effectInstance itself will be a copy of the table in the element's value.

Since effects can only be applied through auras, to use our speed boost effect, we must create at least 1 aura which contains it.
```lua
local speedBoost = function(settings)
  return {
    EffectInstances = {
      speedBoost = {Value=settings.Value or 10} --speedBoost is the effect, {Value=settings.Value or 10} is the effectInstance
    }
  }
end
```

Given this definition, when a user applies the speedBoost aura, a settings table can be optionally passed in. In that case, an aura instance containing a speed boost effect instance will be generated. The custom field Value of the speed boost effect instance will depend on settings, and default to 10 if no Value setting was specified.

### Special Fields
Although you can generally construct effect instances with any custom fields, some fields are reserved and have special behavior.

`Duration: Number`
This effect will be removed after Duration ( in seconds ) has elapsed.

`Tick: Number`
So long as the effect is active, it will be recalculated every Tick ( in seconds ).

`Cleanup: Bool`
This effect will be recalculated when the object is removed. Useful if the effect has side effects beyond the object.

### Aura Instance Field Replication
All fields of an aura instance except EffectInstances are applied to each effectInstance. This is commonly used for fields like duration, since it's typical that every effect in an aura will last for the same duration.

Not taking advantage of field replication:
```lua
local stun = function(settings)
  return {
    EffectInstances = {
      snare = {Duration=settings.Duration},
      silence = {Duration=settings.Duration},
    }
  }
end
```

Taking advantage of field replication:
```lua
local stun = function(settings)
  return {
    Duration = settings.Duration,
    EffectInstances = {
      snare = {},
      silence = {},
    }
  }
end
```

## Aura Instance Id
A GUID identifying a specific aura instance that's been applied to an object. Used to remove aura instances from objects.

# Library
#### `aura.applyAura(object: Instance, auraName: String, settings: [Table]) -> auraInstanceId`

Given an auraName and settings, generates an auraInstance and applies it to the given object. Returns the id of the aura instance, which can be used to remove it later on. When the aura is called, settings will be passed in as the first argument. If settings is left empty, an empty table is passed in instead.

#### `aura.removeAuraInstance(object: Instance, id: auraInstanceId)`

Removes an aura instance from an object.

#### `aura.removeAura(object: Instance, auraName: String)`

Removes all aura instances of the given aura applied from an object.

#### `aura.hasAura(object: Instance, auraName: String) -> bool`

Returns wether or not an object currently has 1 or more instances of a given aura applied.

#### `aura.getAuras(object: Instance)`

Returns array of names of all auras currently applied to the object

#### `aura.hasEffect(object: Instance, effectName: String) -> bool`

Returns wether or not an object currently has 1 or more instances of a given effect applied.

#### `aura.getEffectValue(object: Instance, effectName: String) -> bool`

Returns the calculated value of all effect instances of a given effect currently applied to the object. If no instances are applied, will return the effects default value.

# Usage

## Common Reduce Functions

Reduce functions are used to figure out the current "value" of an effect, given currently applied effect instances. Naturally, since the default is the first argument of the first call of reduce during effect calculation, we must specify it when specifying reduce functions.

### One or More
```lua
local effect = {
  Default = false,
  reduce = function() return true end
}
```

This reduce function will return true, if one or more effect instances exist. This should be used for something like a stun, where you want to apply the stun if 1 or more stuns are applied, and remove it when 0 are applied.

### Sum
```lua
local effect = {
  Default = 0, --additive identity
  reduce = function(sum, effectInstance) return sum + effectInstance.Value end
}
```

By the end of calculation, this reduce function will return the sum of all effectInstance value fields. This isn't particularly useful if your apply is just gonna set something to the sum, but may be if ur apply does something more interesting, like switching based on the range your sum is in.

### Max
```lua
local effect = {
  Default = -math.huge, --whatever ur default value is
  reduce = function(sum, effectInstance) return sum > effectInstance.Value and sum or effectInstance.Value end
}
```

By the end of calculation, this reduce function will return the maximum value, or negative infinity ( if no effect instances are active ). We can replace -math.huge with a more usable value like 0, so long as it's less than the minimum value you can pass in ( you can implement a minimum by clamping a setting in the aura ). Alternatively we can deinit when sum < some interesting value in our apply function.

Min can be implemented by replacing -math.huge with math.huge and > with <.

## Common Effects & Auras

You can implement these apply functions however, the important thing the library offers is that value is recalculated via the reduce function on addition and removal of the effect.

### Stun
```lua
--effects
local cooldown = {
  Default = false,
  reduce = function() return true end,
  apply = function() end
}

local speedBoost = {
  Default = 16,
  reduce = function(sum, effectInstance) return sum + effectInstance.Value end
  apply = function(character, sum)
    character.Humanoid.WalkSpeed = sum
  end
}

--aura
local Stun = function(settings)
  return {
    Duration = settings.Duration,
    EffectInstances = {
      cooldown = {},
      speedBoost = {Value=-10000}
    }
end
```

Supposing that you run aura.hasEffect(character, 'cooldown') to determine wether or not the character can cast abilities, and that the sum of other speed boosts don't exceed 10000, this works as expected.


### Percent or Flat Speed Modifiers
```lua
--effects
local speedBoost = {
  Default = {Flat=16,Percent=1},
  reduce = function(sum, effectInstance) 
    local newSum = {}
    newSum[effectInstance.Type] = newSum[effectInstance.Type] + effectInstance.Value
    return newSum
  end,
  apply = function(character, speedModifier)
    character.Humanoid.WalkSpeed = speedModifier.Flat * speedModifier.Percent / 100
  end
}

--aura
local SpeedBoost = function(settings)
  return {
    Duration = settings.Duration,
    EffectInstances = {
      speedBoost = {Type=settings.Type, Value=settings.Value}
    }
end

--usage
aura.applyAura(character, "SpeedBoost", {Duration=10, Type="Flat", Value=20}) -- apply 20 walkspeed buff
wait(1)
aura.applyAura(character, "SpeedBoost", {Duration=5, Type="Percent", Value=100}) -- apply 100% walkspeed buff

```

Here we demonstrate how we can handle a more complicated scenarios, like allowing the user to mutate a single property in multiple ways. In these cases, it comes down to passing extra information into the effect instance that determines how it influences the final calculation.








