# Description
Aura is a simple library for managing status effects.

Specifically, there's an issue where the removal of one status effect interferes with other existing status effects. For example, suppose that stuns A & B are applied simultaneously, then removed after 1 and 2 seconds respectively. Depending on the implementation, the removal of stun A might allow the player to move again, even though stun B is still active for another second. In short, Aura solves this problem by checking if the player has any remaining active stuns before unstunning them.

# Documentation
## Effect
An effect is a scheme for a specific way to manipulate an object. For example, a speed boost effect would describe the default speed of the object, how to figure out the object's speed based on all the speed effects currently applied to it, and how to actually given the object that speed.

You create effects by constructing tables with 3 fields:

```lua
{
  Default: Variant, 
  reduce: Function(sum: Variant, effectInstance: EffectInstance), 
  apply: Function(object: Instance, sum: Variant)
}
```

Here's an example of a speed boost effect

```lua
local speedBoost = {
  Default = 16,
  reduce = function(totalBoost, effectInstance) return totalBoost + effectInstance.Boost end,
  apply = function(character, totalBoost)
    character.Humanoid.WalkSpeed = totalBoost
  end
}
```

The meaning of these fields will become more clear once we consider what aura does when applying and removing effects.

## Effect Instance
An effect instance is a table that represents an actual effect applied to an object. For every effect you actually apply, one effect instance will be generated. Effect Instances are empty by default, but you can specify fields to highlight the differences between instances of the same effect. For example, one stun effect may have a duration of 1, while another might have a duration of 2.

## Calculating Effects
Everytime an effect instance is applied/removed to or from an object the follow steps are executed:
1. The effect the effect instance belongs to is determined.
2. A value is initially set to effect.Default.
3. effect.Reduce is run on each currently active effectInstance belonging to the effect in order to determine a final calculated value.
4. effect.Apply(object, calculatedValue)

```lua
--pseudo code for calculating an effect
local effect -- whatever effect is associated with the effect instance
local value = effectInstance.Default

for _, effectInstance in pairs(object.EffectInstances[effect]) do
   value = effect.reduce(value, effectInstance)
end

effect.apply(object, value)
```

# aura
aura.applyAura(object: Instance, aura: String, settings: [Table]) -> auraInstance

Creates an instance of the given aura with the given settings


