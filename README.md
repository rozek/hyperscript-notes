# hyperscript-notes #

some notes on _hyperscript

[\_hyperscript](https://github.com/bigskysoftware/_hyperscript) is a relatively new programming language inspired inspired by [HyperTalk](https://en.wikipedia.org/wiki/HyperTalk).

This repository is a (growing) collection of notes on \_hyperscript with code examples that go beyond of what a "normal" programmer would probably need.

> Just a small note: if you like this repository and seem to benefit from its contents, consider "starring" it (you will find the "Star" button on the top right of this page), so that I know which of my repositories to take most care of.

### Evaluate Code at Runtime ###

If you want to implement a \_hyperscript REPL or a "message box" like in HyperCard, LiveCode or similar, you will need a mechanism to evaluate \_hyperscript code at runtime. One solution (perhaps not the best one) is to prepend the following script element before the \_hyperscript runtime itself:

```html
 <script type="text/hyperscript">
  def evaluate (Script)
    createElement('div') on document then set Incubator to it
      js (Script)
        return Script.replace(/&/g,'&amp;').replace(/\x22/g,'&quot;')
      end
      put it into Script

      put `
        <div style="display:none" _="
          init
            ${Script}
          end
        "></div>
      ` into the innerHTML of Incubator
    get the first <div/> in Incubator then set auxDiv to it
      put auxDiv after document.body      -- actually evaluates the given script
    remove auxDiv
  end
 </script>
```

You may then evaluate some \_hyperscript code given in text form using

```
 evaluate('call alert("Hello from evaluated _hyperscript!")')
```

provided that the given code fits into the `init` section of the `_` attribute for a (temporary) HTML element.

> Caveats:
> * because of the way `evaluate` is implemented, **the given code is evaluated after a small delay** (i.e., will not finish before `evaluate` has ended)
> * as a consequence, **the given code can not return any value** to the calling \_hyperscript

### Define Behavior at Runtime ###

If you want to dynamically load behaviors or create behaviors at runtime (e.g., as part of a \_hyperscript REPL) you will need a mechanism to define behaviors at runtime. One solution (perhaps not the best one) is to prepend the following script element before the \_hyperscript runtime itself:

```html
 <script type="text/hyperscript">
  def defineBehavior (Script)
    createElement('div') on document then set Incubator to it
      js (Script)
        return Script.replace(/&/g,'&amp;').replace(/\x22/g,'&quot;')
      end
      put it into Script

      put `
        <div style="display:none" _="
          ${Script}
        "></div>
      ` into the innerHTML of Incubator
    get the first <div/> in Incubator then set auxDiv to it
      put auxDiv after document.body      -- actually evaluates the given script
    remove auxDiv
  end
 </script>
```

You may then define a behavior given in text form using

```
 defineBehavior(`
  behavior newBehavior
    ...
  end
 `)
```

and `install` it into new HTML elements (created _after_ defining their behavior) as usual:

```
 put `
  <div _="install newBehavior">...</div>
 ` after ...
```

### Update existing Behavior at Runtime ###

If you want to update the implementation of an already existing behavior at runtime (e.g., as part of a \_hyperscript REPL) you may do so by using the `defineBehavior` described above and a script which defines a behavior with the same name as the one you want to update.

As a result,

* any **already existing HTML elements** based on the affected behavior **will still use the old implementation**
* while **any HTML elements created after updating the behavior will use the new implementation**

If you want all HTML elements to use the updated behavior you will have to reload their scripts as shown below (provided that existing element scripts remain compatible with the updated behavior - otherwise you will have to update the element scripts anyway)

### List all curently known Behavior Names ###

If you want to know which behaviors have already been defined, you may use the following code:

```html
 <script type="text/hyperscript">
  def knownBehaviorNames
    set BehaviourKey to 'Behavior_' + Date.now() + '_' + Math.round(Math.random()*1000000)

    defineBehavior(`behavior ${BehaviourKey} end`)

    js (BehaviourKey)
      let global = (new Function('return this'))()

      let BehaviourPattern = global[BehaviourKey].toString()
      delete global[BehaviourKey]

      let NameList = []
        Object.getOwnPropertyNames(global).forEach(
          (Key) => {
            let Descriptor = Object.getOwnPropertyDescriptor(global,Key)
            if (
              (typeof Descriptor.value === 'function') &&
              (Descriptor.value.toString() === BehaviourPattern)
            ) { NameList.push(Key) }
          }
        )
      return NameList
    end
    return it
  end
 </script>
```

> Nota bene: function `knownBehaviorNames` depends on `defineBehavior` which has been mentioned above and should also be inserted into the HTML document.

The following examples shows how to use `knownBehaviorNames`:

```html
 <script type="text/hyperscript">
  behavior Test1 end
  behavior Test2 end
 
  init
    log knownBehaviorNames()
  end
 </script>
```



## License ##

While \_hyperscript itself is under a [BSD 2-clause license](https://github.com/bigskysoftware/_hyperscript/blob/master/LICENSE), these notes are just MIT-licensed

[MIT License](LICENSE.md)
