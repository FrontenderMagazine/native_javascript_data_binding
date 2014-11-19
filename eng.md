<article class="markdown-body">[Home][1] | [Blog][2] 
Two-way data-binding is such an important feature - align your JS models with
your HTML view at all times, to reduce boilerplate coding and enhance UX. We 
will observe two ways of doing this using native JavaScript, with no frameworks
- one with revolutionary technology (Object.observe), and one with an original 
concept (overriding get/set). Spoiler alert - the second one is better. See TL;
DR at bottom.

### 1: Object.observe && DOM.onChange {#objectobserve--domonchange}

*Object.observe()* is the [new kid on the block][3]. This native JS ability -
well, actually it’s a*future* ability since it’s only proposed for ES7, but
it’s already[!] available in the[current stable Chrome][4] - allows for
reactive updates to changes to a JS object. Or in simple English - a callback 
run whenever an object(‘s properties) change(s
). 

An idiomatic usage could be:

    log = console.log
    user = {}
    Object.observe(user, function(changes){    
        changes.forEach(function(change) {
            user.fullName = user.firstName + "" + user.lastName;         
        });
    });
    
    user.firstName = 'Bill';
    user.lastName = 'Clinton';
    user.fullName // 'Bill Clinton'
    

This is already pretty cool and allows reactive programming within JS - keeping
everything up-to-date by*push*. 

But let’s take it to the next level:

    //<input id="foo">
    user = {};
    div = $("#foo");
    Object.observe(user, function(changes){    
        changes.forEach(function(change) {
            var fullName = (user.firstName || "") + "" + (user.lastName || "");         
            div.text(fullName);
        });
    });
    
    user.firstName = 'Bill';
    user.lastName = 'Clinton';
    
    div.text() //Bill Clinton
    

*[JSFiddle][5]*

Cool! We just got model-to-view databinding! Let’s DRY ourselves with a
helper function.

    //<input id="foo">
    function bindObjPropToDomElem(obj, property, domElem) { 
      Object.observe(obj, function(changes){    
        changes.forEach(function(change) {
          $(domElem).text(obj[property]);        
        });
      });  
    }
    
    user = {};
    bindObjPropToDomElem(user,'name',$("#foo"));
    user.name = 'William'
    $("#foo").text() //'William'
    

*[JSFiddle][6]*

Sweet! 

Now for the other way around - binding a DOM elem to a JS value. A pretty good
solution could be a simple use of jQuery’s*.change* (http://api.jquery.com/
change
/):

    //<input id="foo">
    $("#foo").val("");
    function bindDomElemToObjProp(domElem, obj, propertyName) {  
      $(domElem).change(function() {
        obj[propertyName] = $(domElem).val();
        alert("user.name is now "+user.name);
      });
    }
    
    user = {}
    bindDomElemToObjProp($("#foo"), user, 'name');
    //enter 'obama' into input
    user.name //Obama. 
    

*[JSFiddle][7]*

That was pretty awesome. To wrap up, in practice you could combine the two into
a single function to create a two-way data-binding:

    function bindObjPropToDomElem(obj, property, domElem) { 
      Object.observe(obj, function(changes){    
        changes.forEach(function(change) {
          $(domElem).text(obj[property]);        
        });
      });  
    }
    
    function bindDomElemToObjProp(obj, propertyName, domElem) {  
      $(domElem).change(function() {
        obj[propertyName] = $(domElem).val();
        console.log("obj is", obj);
      });
    }
    
    function bindModelView(obj, property, domElem) {  
      bindObjPropToDomElem(obj, property, domElem)
      bindDomElemToObjProp(obj, propertyName, domElem)
    }
    

Take note to use the correct DOM manipulation in case of a two-way binding,
since different DOM elements (input, div, textarea, select) answer to different 
semantics (text, val). Also take note that two-way data-binding is not always 
necessary – “output” elements rarely need view-to-model binding and “input” 
elements rarely need model-to-view binding. But wait – there’s more:

### 2: Go deeper: Changing ‘get’ and ‘set’ {#go-deeper-changing-get-and-set}

We can do even better than the above. Some issues with our above implementation
is that using .change breaks on modifications that don’t trigger jQuery’s “
change” event - for example, DOM changes via the code, e.g. on the above code 
the following wouldn’t work:

    $("#foo").val('Putin')
    user.name //still Obama. Oops. 
    

We will discuss a more radical way - to override the definition of getters and
setters. This feels less ‘safe’ since we are not merely observing, we will be 
overriding the most basic of language functionality, get/setting a variable. 
However, this bit of metaprogramming will allow us great powers, as we will 
quickly see.

So, what if we *could* override getting and setting values of objects? After
all, that’s exactly what data-binding is. Turns out that using
`Object.defineProperty()` [we can in fact do exactly that][8]. 

We used to have the [old, non-standard, deprecated way][9] but now we have the
new cool (and most importantly, standard) way, using`Object.defineProperty`, as
so:

    user = {}
    nameValue = 'Joe';
    Object.defineProperty(user, 'name', {
      get: function() { return nameValue }, 
      set: function(newValue) { nameValue = newValue; },
      configurable: true //to enable redefining the property later
    });
    
    user.name //Joe 
    user.name = 'Bob'
    user.name //Bob
    nameValue //Bob
    

OK, so now user.name is an alias for nameValue. But we can do more than just
redirect the variable to be used - we can use it to create an alignment between 
the model and the view. Observe:

    //<input id="foo">
    Object.defineProperty(user, 'name', {
      get: function() { return document.getElementById("foo").value }, 
      set: function(newValue) { document.getElementById("foo").value = newValue; },
      configurable: true //to enable redefining the property later
    });
    

`user.name` is now binded to the input `#foo`. This is a very concise
expression of ‘binding’ at a native level - by defining (or extending) the 
native get/set. Since the implementation is so concise, one can easily extend/
modify this code for custom situation - binding only get/set or extending either
one of them, for example to enable binding of other data types.

As usual we make sure to DRY ourselves with something like:

    function bindModelInput(obj, property, domElem) {
      Object.defineProperty(obj, property, {
        get: function() { return domElem.value; }, 
        set: function(newValue) { domElem.value = newValue; },
        configurable: true
      });
    }
    

usage:

    user = {};
    inputElem = document.getElementById("foo");
    bindModelInput(user,'name',inputElem);
    
    user.name = "Joe";
    alert("input value is now "+inputElem.value) //input is now 'Joe';
    
    inputElem.value = 'Bob';
    alert("user.name is now "+user.name) //model is now 'Bob';
    

*[JSFiddle][10]*

Note the above still uses ‘domElem.value’ and so will still work only on 
`<input>` elements. (This can be extended and abstracted away within the
bindModelInput, to identify the appropriate DOM type and use the correct method 
to set its ‘value
’). 

Discussion:

*   DefineProperty is available in [pretty much every browser][11].
*   It is worth mentioning that in the above implementation, the *view* is now
    the ‘single point of truth’ (at least, to a certain perspective). This is 
    generally unremarkable (since the point of two-way data-binding means 
    equivalency. However on a principle level this may make some uncomfortable, and 
    in some cases may have actual effect - for example in case of a removal of the 
    DOM element, would our model would essentially be rendered useless? The answer 
    is no, it would not. Our
   `bindModelInput` creates a closure over `domElem`, keeping it in memory -
    and perserving the behavior a la binding with the model - even if the DOM 
    element is removed. Thus the model lives on, even if the view is removed. 
    Naturally the reverse is also true - if the model is removed, the view still 
    functions just fine. Understanding these internals could prove important in 
    extreme cases of refreshing both the data and the view.
   

Using such a bare-hands approach presents many benefits over using a framework
such as Knockout or Angular for data-binding, such as:

*   Understanding: Once the source code of the data-binding is in your own
    hands, you can better understand it and modify it to your own use-cases.
   
*   Performance: Don’t bind everything and the kitchen sink, only what you
    need, thus avoiding performance hits at large numbers of observables.
   
*   Avoiding lock-in: Being able to perform data-binding yourself is of course
    immensely powerful, if you’re not in a framework that supports that.
   

One weakness is that since this is not a ‘true’ binding (there is no ‘
dirty checking’ going on), some cases will fail - updating the view will not ‘
trigger’ anything in the model, so for example trying to ‘sync’ two dom elements
*via the view* will fail. That is, binding two elements to the same model will
only refresh both elements correctly when the model is ‘touched’. This can be 
amended by adding a custom ‘toucher
’:

    //<input id='input1'>
    //<input id='input2'>
    input1 = document.getElementById('input1')
    input2 = document.getElementById('input2')
    user = {}
    Object.defineProperty(user, 'name', {
      get: function() { return input1.value; }, 
      set: function(newValue) { input1.value = newValue; input2.value = newValue; },
      configurable: true
    });
    input1.onchange = function() { user.name = user.name } //sync both inputs.
    

### TL;DR: {#tldr}

Create a two way data-binding between model and view with native JavaScript as
such:

    function bindModelInput(obj, property, domElem) {
      Object.defineProperty(obj, property, {
        get: function() { return domElem.value; }, 
        set: function(newValue) { domElem.value = newValue; },
        configurable: true
      });
    }
    
    //<input id="foo">
    user = {}
    bindModelInput(user,'name',document.getElementById('foo')); //hey presto, we now have two-way data binding.
    

Thanks for reading. Comments at [discussion on reddit][12] or at sella.rafaeli@
gmail.com.</article>

 [1]: http://www.sellarafaeli.com/
 [2]: http://www.sellarafaeli.com/blog
 [3]: http://www.html5rocks.com/en/tutorials/es7/observe/
 [4]: http://kangax.github.io/compat-table/es7/#Object.observe
 [5]: http://jsfiddle.net/v2bw6658/
 [6]: http://jsfiddle.net/v2bw6658/2/
 [7]: http://jsfiddle.net/v2bw6658/4/

 [8]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/defineProperty

 [9]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/__defineGetter__
 [10]: http://jsfiddle.net/v2bw6658/6/
 [11]: http://kangax.github.io/compat-table/es5/#Object.defineProperty
 [12]: http://redd.it/2manfb