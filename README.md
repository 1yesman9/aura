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
3. effect.Reduce is run on each currently active effectInstance belonging to the effect in order to determine a final calculated value.
4. effect.Apply(object, calculatedValue)

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

## Auras
An Aura is a package of one or more effects. Although we've talked about applying effects, there is no way for a user of the library to apply one directly. Users apply auras, then effect instances are generated and applied for each effect contained in the aura. Auras also serve as an interface between the caller and the underlying effect.

In actuality, an aura is of the type: Function(settings: Table) -> auraInstance

### Aura Instance
Like an effect instance, an aura instance are tables representing an active auras applied to an object.

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

Since effects can only be applied through auras, to use our speed boost effect, we must create at least 1 aura which contains it.
```lua
local speedBoost = function(settings)
  return {
    EffectInstances = {
      speedBoost = {Value=settings.Value or 10}
    }
  }
end
```

Given this definition, when a user applies the speedBoost aura, a settings table can be optionally passed in. In that case, an aura instance containing a speed boost effect instance will be generated. The custom value field of the speed boost effect instance will depend on settings, and default to 10 if no Value setting was specified.

##Aura Instance I

# Library
aura.applyAura(object: Instance, auraName: String, settings: [Table]) -> auraInstanceId

Given an auraName and settings, generates an auraInstance and applies it 


