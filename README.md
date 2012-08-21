# JavaScript Bindings for C and Objective-C

## Introduction

JavaScript Bindings for C / Objective-C (JSB) is the "glue" code (or wrapper code) that sits between native code (C or Objective-C) and JavaScript (JS) code.
JSB allows calling native code from JS and vice-versa.

That means that you can interact with your favorite native library from JS. As an example, you could create a [cocos2d](http://www.cocos2d-iphone.org) particle system in JS, while its logic and rendering will be executed natively. Or you could create a [Chipmunk Physics](http://www.chipmunk-physics.net) world in JS, while the whole physics simulation, including the collisions detections, would run natively.

![JSB layer ](https://raw.github.com/zynga/jsbindings/master/docs/jsb_intro.png)

The JS code is interpreted by [SpiderMonkey](https://developer.mozilla.org/en-US/docs/SpiderMonkey), Mozilla's JS virtual machine (VM).
It uses the latest stable version of SpiderMonkey (as of this writing it is v14.0.1). The JS VM is extended by JSB to support custom types, custom structures and Objective-C objects.

JSB has a flexible set of rules that could be used to select the classes, methods, functions and structs to parse or ignore; which methods are callbacks; and renaming rules among some of its features. To ease the creation of these rules, it supports regular expressions.

## Major features

Highlights of JSB:

- Supports any Objective-C / C library
- Automatically generates the JS bindings ("glue" code)
- No need to modify the source code of your libraries, or the autogenerated code
- Powerful set of rules to customize the generated JS API
- Automatically converts JS objects/types into Objective-C objects/structs/ types and vice-versa
- Supports "subclassing" native objects in JS
- Supports callbacks

## Creating your own bindings

### Quick steps

1. Download [JSB](http://github.com/zynga/jsbindings)

		$ git clone git://github.com/zynga/jsbindings.git

2. Generate the BridgeSupport files for your project. Let's assume that your project is CocosDenshion

		$ cd ~/src/CocosDenshion/CocosDenshion
		$ gen_bridge_metadata -F complete --no-64-bit -c '-DNDEBUG -I.' *.h -o ~/some/path/CocosDenshion.bridgesupport

3. Generate complement files for your project

		$ cd ~/src/CocosDenshion/CocosDenshion
		$ ~/jsb/generate_complement.py -o ~/some/path/CocosDenshion-complement.txt *.h


4. Create a JSB config file for your project

		$ vim ~/some/path/CocosDenshion_jsb.ini

5. Run the JSB generator script

		$ ~/jsb/generate_js_bindings.py -c ~/som/path/CocosDenshion_jsb.ini

6. Include the recently autogenerated files and JSB source files in your Xcode project

7. Include [SpiderMonkey](https://github.com/zynga/Spidermonkey/) and [JRSwizzle](https://github.com/rentzsch/jrswizzle/) in your Xcode project


_The CocosDenshion config files could be found here: [configs/CocosDenshion](https://github.com/zynga/jsbindings/tree/master/configs/CocosDenshion)_


### Steps in detail

JSB comes with a python script called `generate_js_bindings.py` that generates the glue code. It needs a configuration file that contains the parsing rules and the [BridgeSupport](http://developer.apple.com/library/mac/#documentation/Darwin/Reference/ManPages/man5/BridgeSupport.5.html) files.

BridgeSupport files are generated by a script called [`gen_bridge_metadata`](http://developer.apple.com/library/mac/#documentation/Darwin/Reference/ManPages/man1/gen_bridge_metadata.1.html#//apple_ref/doc/man/1/gen_bridge_metadata) that is part of OS X, and generates xml files with information like class names, method names, arguments, return values, internals of the structs, constants, etc. 

`gen_bridge_metadata`, internally, uses [`clang`](http://clang.llvm.org/) to parse the native code. The output is very reliable, but unfortunately, it is not complete: class hierarchy, protocols and properties data are missing. That's why JSB comes with another python script, called `generate_js_complement.py`, that generates the missing information.

Once we have the configuration file setup, we can run the `generate_js_bindings.py` script to generate the glue code.

To summarize, the structure of a JSB configuration file is:

- Parsing rules (optional): renaming rules, classes to ignore / parse, etc...
- BridgeSupport files (required): Class, methods, functions, structs information
- Complement files (required for Objective-C projects): Hierarchy, protocol and properties information

!["glue" code generation ](https://raw.github.com/zynga/jsbindings/master/docs/jsb_files.png)


### The configuration file

The configuration has a set of powerful rules that could transform the native API into a customized JS API.
Let's take a look at some of them.

#### Renaming rule

The renaming rule allows us to rename a method names or class names or function names or struct names.
As an example, the default JS API for:

	// CCAnimation (from cocos2d-iphone v2.0)
	+(id) animationWithAnimationFrames:(NSArray*)arrayOfAnimationFrames delayPerUnit:(float)delayPerUnit loops:(NSUInteger)loops;

would be:

	// ugly
	cc.CCAnimation.animationWithAnimationFrames_delayPerUnit_loops_( frames, delay, loops );

So, with a simple set of rules, we can rename that JS API into this one:

	// more JS friendly
	cc.Animation.create( frames, delay, loops );

In order to do that, we need to remove the `CC` prefix from the class name, since it is already using the `cc` JS namespace:

`obj_class_prefix_to_remove = CC`

And finally we add a rename rule for that method:

`method_properties =  CCAnimation # animationWithAnimationFrames:delayPerUnit:loops: =  name:"create",`


#### Merge rule

But what happens with the other constructors of `CCAnimation` ?

	// CCAnimation supports 4 different constructors
	+(id) animation; //
	+(id) animationWithSpriteFrames:(NSArray*)arrayOfSpriteFrameNames;
	+(id) animationWithSpriteFrames:(NSArray*)arrayOfSpriteFrameNames delay:(float)delay;
	+(id) animationWithAnimationFrames:(NSArray*)arrayOfAnimationFrames delayPerUnit:(float)delayPerUnit loops:(NSUInteger)loops;


What we should do, is to create a rule that merges the 4 constructors into one. JSB will call the correct one depending on the number of arguments. This is how the rule should look:

`method_properties =  CCAnimation # animationWithAnimationFrames:delayPerUnit:loops: =  name:"create"; merge: "animation" | "animationWithSpriteFrames:" | "animationWithSpriteFrames:delay:",`

And the code in JS will look like:

	// calls [CCAnimation animation]
	var anim = cc.Animation.create(); 

	// calls [CCAnimation animnationWithSpriteFrames:]
	var anim = cc.Animation.create(array);

	// calls [CCAnimation animnationWithSpriteFrames:delay:]
	var anim = cc.Animation.create(array, delay);

	// calls [CCAnimation animationWithAnimationFrames:delayPerUnit:loops:]
	var anim = cc.Animation.create(array, delay, loops);

#### Callback rule

JSB supports callback methods. In order to register a method as callback you should add a callback rule in the configuration file. eg:

`method_properties = CCNode#onEnter = callback,`

You could also rename a callback by doing:

`method_properties = CCLayer # ccTouchesBegan:withEvent: = callback; name:"onTouchesBegan",`


#### Configuration examples

In order to learn more about the configuration file, use the following working examples as a guideline:

- [cocos2d_jsb.ini](https://github.com/zynga/jsbindings/blob/master/configs/cocos2d/cocos2d_jsb.ini)
- [CocosDenshion_jsb.ini](https://github.com/zynga/jsbindings/blob/master/configs/CocosDenshion/CocosDenshion_jsb.ini)
- [chipmunk_jsb.ini](https://github.com/zynga/jsbindings/blob/master/configs/chipmunk/chipmunk_jsb.ini)
- [CocosBuilderReader_jsb.ini](https://github.com/zynga/jsbindings/blob/master/configs/CocosBuilderReader/CocosBuilderReader_jsb.ini)

## Internals of the JS bindings ("glue" code)

The JS bindings code allows to call JS code from native and vice-versa. It forwards native callbacks to JS, and JS calls to native. Let's see them in detail:

### Calling native functions from JS

The following code will call the the native C function `ccpAdd()`:

	var p1 = cc.p(0,0);
	var p2 = cc.p(1,1);
	// cc.pAdd is a "wrapped" function, and it will call the cocos2d ccpAdd() C function
	var ret = cc.pAdd(p1, p2); 


Let's take a look at the native declaration of `ccpAdd`:

	CGPoint ccpAdd(const CGPoint v1, const CGPoint v2);

So when `cc.pAdd` is executed, it will call the "glue" function code `JSB_ccpAdd`. And `JSB_ccpAdd` does:

- converts the arguments from JS to native
- calls the native `ccpAdd()` function
- converts the return value from native to JS
- it fails if there are errors converting either the arguments or the return value.

![function call flow ](https://raw.github.com/zynga/jsbindings/master/docs/jsb_calls.png)

### Calling native instance / class methods from JS

It is also possible to call instance or class methods from JS. The internal logic is similar to calling native functions. Let's have look:

	// Creates a sprite and sets its position to 200,200
	var sprite = cc.Sprite.create('image.png');
	sprite.setPosition( cc.p(200,200) );

`cc.Sprite.create("image.png")` will call the "glue" function `JSB_CCSprite_spriteWithFile_rect__static`, which does:

- converts the JS String into a native string
- creates a native instance of `CCSprite` by calling `[CCSprite spriteWithFile:@"image.png"]`
- converts the native instance into a JS Object
- Adds the newly created instance into a dictionary using the JS object as `key`
- returns the converted instance to JS.

And `sprite.setPosition(cc.p(200,200))` will call the "glue" function `JSB_CCNode_setPosition_`, which does:

- Obtains the native instance from the dictionary. The JS Object is used as `key`.
- Converts the point JS object to `CGPoint`
- calls `[instance setPosition:p]`
- Since `setPosition:` has no return value, it returns a "void" object to JS

![class instantiation flow](https://raw.github.com/zynga/jsbindings/master/docs/jsb_new_class.png)

![instance method flow](https://raw.github.com/zynga/jsbindings/master/docs/jsb_instance_call.png)

### Calling JS code from native

#### Callbacks

There are 2 types of calls that can be generated:

- Calls that are originated in JS and executes native code (the ones presented earlier)
- Calls that are originated in native and executes JS code. These ones are the "callbacks".

As an example, cocos2d-iphone's  `onEnter`, `onExit`, `udpate` are callback functions. `onEnter` is originated on native, and it should also call possible "overrides" in JS. On the following example, `onEnter` will be called from native, but only if we declare `onEnter` as "callback":


	var MyLayer = cc.Layer.extend({
	    ctor:function () {
	        cc.associateWithNative( this, cc.Layer );
	        this.init();
	    },
	    onEnter:function() {
	    	cc.log("onEnter called");
	   	},
	});

JSB supports callbacks without the need to modify the source code of the parsed library. What JSB does, is to  swap (AKA "swizzle") the original callback method with one provided by JSB. This is the full flow:

- At registration time, JSB swizzles all methods registered as callbacks
- When a swizzle method is called:
	- It calls the native callback function
	- Then it calls the JS callback function (if available)

![Callbacks flow](https://raw.github.com/zynga/jsbindings/master/docs/jsb_callbacks.png)

#### Executing scripts

In order to run a JS script from native, you should do the following

	// JSBCore is responsible registering the native objects in JS, among other things
	if( ! [[JSBCore sharedInstance] runScript:@"my_js_script.js"] )
		NSLog(@"Error running script");


## Who is using JSB?

JSB has been used to generate bindings for the following projects:

- [cocos2d-iphone v2.1] (https://github.com/cocos2d/cocos2d-iphone/tree/develop-v2) (develop-v2 branch)
- CocosDenshion
- CocosBuilder Reader
- [Chipmunk 6.1.1] (https://github.com/slembcke/Chipmunk-Physics)

cocos2d-iphone v2.1 comes with 3 demo projects that uses it: 

- [JS Tests](https://github.com/cocos2d/cocos2d-iphone/tree/develop-v2/tests/JSTests): cocos2d JS bindings tests
- [JS Watermelon With Me](https://github.com/cocos2d/cocos2d-iphone/tree/develop-v2/tests/WatermelonWithMe): a simple physics game that uses JS bindings for cocos2d, Chipmunk, CocosDenshion and CocosBuilder Reader.
- [JS MoonWarriors](https://github.com/ricardoquesada/MoonWarriors): A top-down shooter that uses JS bindings for cocos2d

It is noteworthy that thanks to the powerful set of rules, the JS API generated for cocos2d-iphone and the cocos2d-html5 API **ARE THE SAME**.

_Moon Warriors can run on top of cocos2d-iphone + JSB or on top of cocos2d-html5 without changing a single line of code. Try the Web version from here: [Moon Warriors Web] (http://www.cocos2d-iphone.org/downloads/MoonWarriors/)_


## Bugs / Limitations

As of this writing, these are the current bugs and/or limitations. For an updated list of bugs/limitations, please visit [JSB homepage](https://github.com/zynga/jsbindings)

- No JS debugger. Remote debugging capabilities will be added once SpiderMonkey 15 is released.
- No JS profiler.
- Native objects control the life of JS objects
	- It means that native objects might get released while their JS counterpart is still live
	- This logic is flawed since a JS object might point to an already released native object under certain not-so-common situations
	- The proper solution is to control the life of native objects from JS objects. Fix in progress
	- The workaround is to send the `retain` message to the object. eg: `sprite.retain();`
- Callbacks don't support return values. Limited support for arguments
- BridgeSupport constants are not being parsed by the script.
- The `gen_bridge_metadata` file that is bundled with OS X 10.6 (or older) should be avoided. It is recommended to generate the BridgeSupport files in OS X 10.8 (Mountain Lion).
- It is not easy to start a new project from scratch. Xcode templates coming soon. In the meantime, the easier way to do it is by duplicating any of the JS "targets" bundled with cocos2d-iphone v2.1



	