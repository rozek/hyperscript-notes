# hyperscript-notes #

...just a few notes on _hyperscript

[\_hyperscript](https://github.com/bigskysoftware/_hyperscript) is a relatively new programming language inspired inspired by [HyperTalk](https://en.wikipedia.org/wiki/HyperTalk).

This repository is a (growing) collection of notes on \_hyperscript with code examples that go beyond of what a "normal" programmer would probably need.

> Just a small note: if you like this repository and seem to benefit from its contents, consider "starring" it (you will find the "Star" button on the top right of this page), so that I know which of my repositories to take most care of.

### Evaluate some Code at Runtime ###

If you want to implement a \_hyperscript REPL or a "message box" like in HyperCard, LiveCode or similar, you will need a mechanism to evaluate \_hyperscript code at runtime. One solution (perhaps not the best one) is to insert the following script element before the \_hyperscript runtime itself:

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

Does anybody have a better idea?

### Define a Behavior at Runtime ###

If you want to dynamically load behaviors or create behaviors at runtime (e.g., as part of a \_hyperscript REPL) you will need a mechanism to define behaviors at runtime. One solution (perhaps not the best one) is to insert the following script element before the \_hyperscript runtime itself:

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

### Update an existing Behavior at Runtime ###

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

### Change (or just reload) some Element Scripts at Runtime ###

if you want to change the `_` attribute (containg the element's \_hyperscript script) at runtime (e.g., as part of a \_hyperscript REPL) you can not just set that attribute to a new value as \_hyperscript will not automatically re-evaluate the new attribute contents. One solution (perhaps not the best one) is to insert the following script element before the \_hyperscript runtime itself:

```html
 <script type="text/hyperscript">
  def setScriptOf (Element, Script)
    cloneNode() from Element then set clonedElement to it
      set @_ of clonedElement to Script

      js (Element, clonedElement)
        while (Element.hasChildNodes()) {
          clonedElement.appendChild(Element.firstChild)
        }
      end
    put clonedElement after Element
    remove Element
  end
 </script>
```

You may then update the script of a given HTML element using

```
 setScriptOf(<element>,<new-script>)
```

where `<element>` refers to an existing HTML element and `<new-script>` represents the (new or initial) script for `<element>`.

`setScriptOf` is itself idempotent (i.e., may safely be called multiple times with the same arguments) provided that the given `<script>` is idempotent itself (i.e., does not produce side-effects like in an `init` block)

> Caveats: `setScriptOf` has to clone the affected HTML element in order to set the given script. While it moves any existing contents of the old HTML element to the new clone (before the new script is evaluated), some element contents could probably produce unwanted side-effects.
> 
> Additionally, any references to the original DOM element will now point to an orphaned HTML fragment, as the original element will no longer be part of the DOM and all its contents will have been moved into the newly created clone

### \_hyperscript "methods" for (scripted) HTML Elements ###

If you want \_hyperscript scripted HTML elements to act like "components" offering more complex functionality you will probably want these elements to provide "methods". Fortunately, such "methods" can be implemented with ease:

* within your element define your \_hyperscript functions as usual
* in order to make them "publically" available (i.e., to "export" them) set these functions as element properties:<br>&nbsp;`init set my <method> to <method> end`<br>where `<method>` is the \_hyperscript function name (warning: this may collide with already existing internal methods of the DOM element)

```
 <div _="
   def publicMethod
   -- insert your implementation here
   end
   
   init
     set my publicMethod to publicMethod
   end
 ">...</div>
```

From elsewhere in your \_hyperscript code

* the method may now be invoked using a "pseudo command": `<method>(<argument-list>) on <element>` (or similar)
* any values returned from the method will then become available as `the result` or simply `it`

## License ##

While \_hyperscript itself is under a [BSD 2-clause license](https://github.com/bigskysoftware/_hyperscript/blob/master/LICENSE), these notes are just MIT-licensed

[MIT License](LICENSE.md)
