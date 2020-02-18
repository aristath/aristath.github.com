---
layout: post
title: "Migrating from jQuery to vanilla JS"
categories:
  - JavaScript
comments_id: 1
---

jQuery has been an easy way to get things done in JavaScript for years. It cuts down development time and allows us to prototype websites a lot faster. It's been around for years, and as a WordPress developer I've been using it for a long, long time. 

I stopped using it a while back and I am now trying to build things using "Vanilla" JS (which by the way is a strange name if you ask me... Vanilla is not "flavorless", it's not "plain". Vanilla is a great and intense flavor).  

Recently I wanted to migrate some scripts from jQuery to Vanilla JS. It can sometimes be a daunting task and if the script you want to migrate is big, it can be a time-consuming process. So I started investigating ways to make the process easier and cut down the time it took me to get things done.  

I ended up writing a jQuery drop-in replacement and I wanted to release it as a new script that people can use if they want to move away from jQuery and to something lighter without all the technical debt. I created a new repository on Github, pushed my code, created a new site for it, and then... I deleted everything.  
The world doesn't need yet another jQuery alternative. What we really need is developers who don't rely on heavy scripts to do simple things.  

Instead of creating a repository with a new library I'm going to show you how you can do it yourself. I think this will be more educational and you can use the method described here to add your own methods and learn to do things the "right" way.

So here's what this article is going to be about: Creating your own jQuery drop-in replacement, and hopefully in the process learn something.

## What we want to do:

What we want to achieve is to be able to take this jQuery code and convert to vanilla JS, simply by replacing `jQuery` with something else. I'll be using `lemon` in this article, but feel free to use whatever you want.
```js
jQuery( document ).ready( function() {

	let menu = jQuery( '#menu' );

	menu.find( 'li.item' )
		.addClass( 'foo' )
		.removeClass( 'bar' )
		.css( 'background-color', '#222' )
		.css( 'color', '#fff' );

	menu.find( '.hidden' ).css( 'display', 'none' );
});
```

Looks pretty simple, right? Well, not so fast... there's a lot going on there! 
1. We add an event listener to `document`, and when everything has finished loading we execute our function.
2. We search the dom for `#menu`.
3. Inside the results we found from the above query for `#menu` we find all elements `li.item`.
4. For each of these elements we are adding the class `foo`.
5. For each of these elements we are removing the class `bar`.
6. For each of these elements we are changing the background-color to `#222`.
7. For each of these elements we are changing the color to `#fff`.
8. We are looking for any elements inside `#menu` that have a class `.hidden`.
9. We are hiding elements with the `.hidden` class.

That's a lot, right? And we don't even think about these things most of the time. That is the convenience of jQuery... We don't have to think about the details. And that's what we're going to achieve too.

### Bespoke Conversion.

First of all, there is no native replacement for `jQuery( document ).ready()`. Thankfully people smarter than me have already figured that out, so here's a function you can use:
```js
/**
 * Equivalent to jQuery's ready() function.
 *
 * @see https://gomakethings.com/a-native-javascript-equivalent-of-jquerys-ready-method/
 * @param {function} fn - The callback function.
 * @return {void}
 */
lemonReady = function( fn ) {
	if ( 'function' !== typeof fn ) { // Sanity check
		return;
	}
	if ( document.readyState === 'complete'  ) {
		return fn();
	}
	document.addEventListener( 'DOMContentLoaded', fn, false );
};
```
For more details on that function you can read [the post I took it from](https://gomakethings.com/a-native-javascript-equivalent-of-jquerys-ready-method/).

To convert the above script to vanilla JS, I'd normally write something like this:
```js
lemonReady( function() {

	// Find the menu.
	let menu = document.getElementById( '#menu' );

	// Find li.item inside the menu.
	let items = menu.querySelectorAll( 'li.item' );

	// Loop items.
	for ( var i = 0; i < items.length; i++ ) {
		let item = items[ i ];

		// Add a class.
		item.classList.add( 'foo' );

		// Remove a class.
		item.classList.remove( 'bar' );

		// Add styles.
		item.style.backgroundColor = '#222';
		item.style.color = '#fff';
	}

	// Find .hidden inside the menu.
	let hiddenElements = menu.querySelectorAll( '.menu' );

	// Loop items.
	for ( var i = 0; i < hiddenElements.length; i++ ) {
		let item = hiddenElements.length[ i ];

		// Add styles.
		item.style.display = 'none';
	}
});
```

That's a lot of code for something that we did in jQuery with less than 10 lines... and that's why we've been using jQuery for years. It's just easy. If the script I want to migrate is small I'll just do the above. But when faced with 1000 lines of jQuery code, an alternative solution has to be used.

Simple things like this:
```js
jQuery( '#menu' ).hide()
```
can easily be done in vanilla JS with something like this:
```js
document.getElementById( 'menu' ).style.display = 'none';
```
but then again not everything is that easy.

## Creating our script

### Wrapper

First, we'll need to create a wrapper. This is what I came up with:

```js
/**
 * Find an element.
 *
 * @param {string|Object} selector - The css selector of the element.
 *                                   If object then it can either be document or an element.
 * @param {Object} parent - The parent element, or undefined for document.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
var _lemon = function( selector, parent ) {
	parent = parent || document;
	if ( 'object' === typeof selector ) {
		this.el = ( selector[0] && selector[0] instanceof Element ) ? selector : [ selector ];
	} else if ( 'string' === typeof selector ) {
		this.el = parent.querySelectorAll( selector );
	}
	
	// Convert ElementList to Array.
	this.el = [].slice.call( this.el )
	return this;
};
```

With the above function we can do this:
```js
var menus = new _lemon( '.menus' );
```
The result will be an Object and in its `el` key we'll have an array of elements. At the end of the function we're returning `this` to allow chaining functions. If you want to learn more about chaining (which is what jQuery does), you can read [this article](https://schier.co/blog/2013/11/14/method-chaining-in-javascript.html).

But we don't want to have to do `new _lemon( 'foo' )`, what we want is to build a drop-in replacement for jQuery. So we'll introduce a `lemon()` function:

```js
var lemon = function( selector, parent ) {
	return new _lemon( selector, parent );
};
```
now instead of `var menus = new _lemon( '.menus' );` we can write this:
```js
var menus = lemon( '.menus' );
```

Next, we need to add the methods we were using in the 1st jQuery example - which is the code we want to migrate: `find`, `addClass`, `removeClass` & `css`.

### `.each()`

Though `.each()` was not used in the code we want to convert, in all our methods we'll be looping the array of elements our wrapper found, and we'll be doing all sorts of things to them. It just makes more sense to add a new method to more efficiently running these loops.

```js
/**
 * Run a callback against located elements.
 *
 * @param {function} callback - The callback we want to run on each element.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.each = function( callback ) {
	var self = this;
	for ( var i = 0; i < this.el.length; i++ ) {
		callback( self.el[ i ] );
	}
	return this;
};
```
In the above code we're extending the `_lemon` function and adding the `.each()` method inside it. Inside our `.each()` method we're looping elements from `this.el` - which is where we're storing our elements array in the `_lemon` object. Then for each element inside `this.el` we're executing a callback.

If you're wondering why I chose to use a loop like `for ( i = 0; i < element.length; i++ )` instead of the shorter `element.forEach()`, the reason is performance. the `for` loop is slightly faster than `forEach`, and the purpose of this whole thing is to improve performance.

### `.find()`

```js
/**
 * Find a selector inside elements.
 *
 * @param {string} selector - The css selector of the element.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.find = function( selector ) {
	let found = [];
	this.each( function( el ) {
		let elFound = el.querySelectorAll( selector );
		if ( elFound ) {
			found = found.concat( elFound );
		}
	});
	return new _lemon( found );
};
```
In the above code we're extending the `_lemon` function and adding the `.find()` method. Using the `.each()` method we added before we're looping the elements and building a new array of elements. When the loop finishes, we're returning a new instance of `_lemon()`, this time using the elements we found. That's why in the `_lemon` object we initially added the `if ( 'object' === typeof selector )` condition.

### `.addClass()`

This one is simple, we want to add a new class to all elements.

```js
/**
 * Add a class to elements.
 *
 * @param {string} className - The class-name we want to add to element(s).
 * @return {Object} - Returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.addClass = function( className ) {
    this.each( function( el ) {
        el.classList.add( className );
    });
	return this;
};
```

### `.removeClass()`

Also simple, just a tweaked version of `addClass()`:

```js
/**
 * Remove a class from elements.
 *
 * @param {string} className - The class-name we want to add to element(s).
 * @return {Object} - Returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.removeClass = function( className ) {
    this.each( function( el ) {
        el.classList.remove( className );
    });
	return this;
};
```

### `.css()`

Now this one will be a little more complicated. The problem here is that css-properties don't correspond with their `style` elements in JS. `background-color` becomes `backgroundColor`, `border-radius` becomes `borderRadius` and so on. So we'll need an extra method to convert these.  
In addition to that, jQuery's `.css()` method can either retrieve the CSS value, or set it. If we define a 2nd argument we want to set the value, otherwise get it.

Here's the code I used:

```js
/**
 * Convert a string to camelCase.
 *
 * @param {string} string The string we want to convert.
 * @return {string} - Returns the string formatted in camelCase.
 */
_lemon.prototype.camelCase = function( string ) {
	return string.replace( /-([a-z])/g, function( _all, letter ) {
		return letter.toUpperCase();
	} );
};

/**
 * Get the styles for a property, or change the value if one is defined.
 *
 * @param {string} property - The CSS property we're referencing.
 * @param {string|undefined} value - The value we want to assign, or undefined if we want to get the value.
 * @return {Object|string} - returns the _lemon object to allow chaining methods, OR the value if 2nd arg is undefined.
 */
_lemon.prototype.css = function( property, value ) {
	if ( 'undefined' === typeof value ) {
		return getComputedStyle( this.el[ 0 ] )[ property ];
    }
    this.each( function( el ) {
		el.style[ _lemon.camelCase( property ) ] = value;
    });
	return 'undefined' === typeof value ? '' : this;
};
```

The `.camelCase()` function we added there is simply converting the CSS property names to a format that JS will understand.

## Putting it all together

```js
/**
 * Equivalent to jQuery's ready() function.
 *
 * @see https://gomakethings.com/a-native-javascript-equivalent-of-jquerys-ready-method/
 * @param {function} fn - The callback function.
 * @return {void}
 */
lemonReady = function( fn ) {
	if ( 'function' !== typeof fn ) { // Sanity check
		return;
	}
	if ( document.readyState === 'complete'  ) {
		return fn();
	}
	document.addEventListener( 'DOMContentLoaded', fn, false );
};

/**
 * Equivalent to jQuery's ready() function.
 *
 * @see https://gomakethings.com/a-native-javascript-equivalent-of-jquerys-ready-method/
 * @param {function} fn - The callback function.
 * @return {void}
 */
_lemon.prototype.ready = function( fn ) {
	if ( 'function' !== typeof fn ) {
		return;
	}
	if ( document.readyState === 'complete'  ) {
		return fn();
	}
	document.addEventListener( 'DOMContentLoaded', fn, false );
};

/**
 * Run a callback against located elements.
 *
 * @param {function} callback - The callback we want to run on each element.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.each = function( callback ) {
	var self = this;
	for ( var i = 0; i < this.el.length; i++ ) {
		callback( self.el[ i ] );
	}
	return this;
};

/**
 * Find a selector inside elements.
 *
 * @param {string} selector - The css selector of the element.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.find = function( selector ) {
	let found = [];
	this.each( function( el ) {
		let elFound = el.querySelectorAll( selector );
		if ( elFound ) {
			found = found.concat( elFound );
		}
	});
	return new _lemon( found );
};

/**
 * Add a class to elements.
 *
 * @param {string} className - The class-name we want to add to element(s).
 * @return {Object} - Returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.addClass = function( className ) {
    this.each( function( el ) {
        el.classList.add( className );
    });
	return this;
};

/**
 * Remove a class from elements.
 *
 * @param {string} className - The class-name we want to add to element(s).
 * @return {Object} - Returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.removeClass = function( className ) {
    this.each( function( el ) {
        el.classList.remove( className );
    });
	return this;
};

/**
 * Convert a string to camelCase.
 *
 * @param {string} string The string we want to convert.
 * @return {string} - Returns the string formatted in camelCase.
 */
_lemon.prototype.camelCase = function( string ) {
	return string.replace( /-([a-z])/g, function( _all, letter ) {
		return letter.toUpperCase();
	} );
};

/**
 * Get the styles for a property, or change the value if one is defined.
 *
 * @param {string} property - The CSS property we're referencing.
 * @param {string|undefined} value - The value we want to assign, or undefined if we want to get the value.
 * @return {Object|string} - returns the _lemon object to allow chaining methods, OR the value if 2nd arg is undefined.
 */
_lemon.prototype.css = function( property, value ) {
	if ( 'undefined' === typeof value ) {
		return getComputedStyle( this.el[ 0 ] )[ property ];
    }
    this.each( function( el ) {
		el.style[ _lemon.camelCase( property ) ] = value;
    });
	return 'undefined' === typeof value ? '' : this;
};

var lemon = function( selector, parent ) {
	return new _lemon( selector, parent );
};
```
Our script is 412 bytes when minified & gzipped. Less than 0.5kb, and we only included the functions we needed. Lighter, faster and way more efficient than enqueueing a 30kb (minified & gzipped) like jQuery!

Now we have what we need to convert our original code to vanilla JS using our own drop-in replacement. All we need to do is replace `jQuery` with `lemon`:

```js
lemon( document ).ready( function() {

	let menu = jQuery( '#menu' );

	menu.find( 'li.item' )
		.addClass( 'foo' )
		.removeClass( 'bar' )
		.css( 'background-color', '#222' )
		.css( 'color', '#fff' );

	menu.find( '.hidden' ).css( 'display', 'none' );
});
```

## More Functions

I'll include some more examples on how to add more functions to our custom wrapper below. These are some of the most-used ones and will hopefully give you a head-start.

### `.hasClass()`

```js
/**
 * Figure out if the element has a class or not.
 *
 * @param {string} className - The class-name we're looking for.
 * @return {boolean} - Whether the element has the class we need or not.
 */
_lemon.prototype.hasClass = function( className ) {
    var found = false;
    this.each( function( el ) {
        if ( el.classList.contains( className ) ) {
            found = true;
        }
    } );
	return found;
};
```

### `.hide()`
```js
/**
 * Hide the element(s).
 *
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.hide = function() {
    this.each( function( el ) {
        el.style.display = 'none';
    });
	return this;
};
```

### `.show()`
```js
/**
 * Show the element(s)
 *
 * @param {string} display - The css-display mode. Defaults to "block".
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.show = function( display ) {
	display = display || 'block';
    this.each( function( el ) {
        el.style.display = display;
    });
	return this;
};
```

### `.parent()`
```js
/**
 * Get the direct parents of elements.
 *
 * @return {Object} - Returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.parent = function( selector ) {
    let found = [];
    this.each( function( el ) {
        found = found.concat( el.parentNode );
    });
	return new _lemon( found );
};
```

### `.parents()`
```js
/**
 * Get the parents based on a selector.
 *
 * @param {string} selector - The CSS selector.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.parents = function( selector ) {
	let found = [],
		self  = this;
    this.each( function( el ) {
        let elFound = self.parents( selector, el );
        if ( elFound ) {
            found = found.concat( elFound );
        }
    });
	return new _lemon( found );
};
```

## Events

If you want to handle events like for example `$('.foo').on('click', function(){});` then you can use the code below:

```js
/**
 * Handle events.
 *
 * @param {Function} method - for example addEventListener.
 * @param {string} events - The JS event we want to add a listener for.
 * @param {Function} listener - The function to add to the listener.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.eventsHandler = function( method, events, listener ) {
    var self = this;
    events.split( ' ' ).forEach( function( event ) {
        self.each( function( el ) {
            method.call( el, event, listener, false );
		});
    });
	return this;
};

/**
 * Similar to jQuery.off().
 *
 * @param {string} event - The JS event we want to add a listener for.
 * @param {Function} listener - The function to add to the listener.
 * @return {void}
 */
_lemon.prototype.off = function( event, listener ) {
	this.eventsHandler( removeEventListener, event, listener );
};

/**
 * Similar to jQuery.on().
 *
 * @param {string} event - The JS event we want to add a listener for.
 * @param {Function} listener - The function to add to the listener.
 * @return {Object} - returns the _lemon object to allow chaining methods.
 */
_lemon.prototype.on = function( event, listener ) {
	this.eventsHandler( addEventListener, event, listener );
	return this;
};
```

If you decide to put all of the above functions in your script its size is approximately 810 bytes (minified & gzipped), which is still pretty good if you ask me.
