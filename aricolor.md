---
layout: page
title: ariColor PHP Class
permalink: /aricolor/
feature_image: pink-floyd-dark-side-of-moon.jpg
---

`ariColor` is a PHP library that will hopefully help WordPress theme developers do their job easier and more effectively.

It does not provide you with methods like `lighten()`, `darken()` etc. Instead, what it does is give you the ability to create these yourself with extreme ease by giving you all the properties of a color at hand, and allowing you to manipulate them however you see fit.

Example:

First, let's create our color object:
```php
$color = ariColor::newColor( '#049CBE', 'hex' );
```
If you don't like using that method you can write your own proxy function:
```php
function my_custom_color_function( $color = '#ffffff' ) {
	return ariColor::newColor( $color, 'auto' );
}
```
Notice that we used `auto` as the mode. If you use `auto` or completely omit the 2nd argument, `ariColor` will auto-detect it for you. You can use `rgb`, `rgba`, `hsl`, `hsla`, or even arrays as colors.

Then you can use it like this:
```php
$color = my_custom_color_function( '#049CBE' );
```

Say you want to get the values for red, green, blue:
```php
// Get red value:
$red = $color->red;
// Get green value:
$green = $color->green;
// Get blue value
$blue = $color->blue;
```

Or you want to get the hue, saturation, lightness or even luminance of your color:
```php
// Get hue
$hue = $color->hue;
// Get saturation
$saturation = $color->saturation;
// Get lightness
$lightness = $color->lightness;
// Get luminance
$luminance = $color->luminance;
```

### Scenario 1:

You have an option where users can define the background color for their `<body>`. In order to make sure the text is always readable, you can either give them a 2nd option to set the text color, or auto-calculate it for readability.

Example function that given a background color decides if we're going to use white/black text color:

```php
/**
 * determine the luminance of the given color
 * and then return #FFFFFF or #000000 so that our text is always readable
 * 
 * @param $background color string|array
 *
 * @return string (hex color)
 */
function custom_get_readable_color( $background_color = '#FFFFFF' ) {
	$color = Kirki_WP_Color::newColor( $background_color );
	return ( 127 > $color->luminance ) ? '#000000' : '#FFFFFF';
}
```

Usage: 
```php
$text_color = custom_get_readable_color( get_theme_mod( 'bg_color', '#ffffff' ) );
```
Easy, right? What we did above is simply check the luminance of the background color, and then if the luminance is greater than 127 we return black, otherwise we return white.

### Scenario 2:

We have a HEX color, and we want to get the same color as rgba, with an opacity of `0.7`:

```php
function my_theme_get_semitransparent_color( $color ) {
	// Create the color object
	$color_obj = Kirki_WP_Color::newColor( $color );
	// Set alpha (opacity) to 0.7
	$color_obj->alpha = 0.7;
	// return a CSS-formated rgba color
	return $color_obj->toCSS( 'rgba' );
}
```
## Properies list:
* `mode` (string: hex/rgb/rgba/hsl/hsla)
* `red` (red value, `integer`, range: 0-255)
* `green` (green value, `integer`, range: 0-255)
* `blue` (blue value, `integer`, range: 0-255)
* `alpha`(alpha/opacity value, `float`, range 0-1)
* `hue` (color hue, `integer`, range 0-360)
* `saturation` (color saturation, `integer`, range 0-100)
* `lightness` (color lightness, `integer`, range 0-100)
* `luminance`(color luminance, `integer`, range 0-255)
* `hex` (the hex value of the current color)

## Methods list:
* `newColor` 
* `getNew`
* `toCSS`

### `newColor`
Used to create a new object.
Example: 
```php
$color = ariColor::newColor( 'rgba(0, 33, 176, .62)' );
```
The `getNew` method has 2 arguments:
1. `$color`: can accept any color value (see below for examples)
2. `$mode`: the color mode. If undefined will be auto-detected.

Some example of acceptable formats for the color used in the 1st argument on the method:
```php
'black'
'darkmagenta'
'#000'
'#000000'
'rgb(0, 0, 0)'
'rgba(0, 0, 0, 1)'
'hsl(0, 0%, 0%)'
'hsla(0, 0%, 0%, 1)'
array( 'rgba' => 'rgba(0,0,0,1)' )
array( 'color' => '#000000' )
array( 'color' => '#000000', 'alpha' => 1 )
array( 'color' => '#000000', 'opacity' => '1' )
array( 0, 0, 0, 1 )
array( 0, 0, 0 )
array( 'r' => 0, 'g' => '0', 'b' => 0 )
array( 'r' => 0, 'g' => '0', 'b' => 0, 'a' => 1 )
array( 'red' => 0, 'green' => 0, 'blue' => 0 )
array( 'red' => 0, 'green' => 0, 'blue' => 0, 'alpha' => 1 )
array( 'red' => 0, 'green' => 0, 'blue' => 0, 'opacity' => 1 )

```
And more! This way you can use the saved values from all known frameworks.

### `getNew`
Used if we want to create a new object identical to the one we already have, but changing one of its properties.

The `getNew` method has 2 arguments:
1. `$property`: can accept any of the properties listed above
2. `$value`: the new value of the property.

Example 1: Darken a color by 10%
```php
// Create a new object using rgba as our original color
$color = ariColor::newColor( 'rgba(0, 33, 176, .62)' );
// Darken the color by 10%
$dark = $color->getNew( 'lightness', $color->lightness - 10 );
// return HEX color
return $dark->toCSS( 'hex' );
```
Or you could write the above simpler like this by combining 2 steps:
```php
$color = ariColor::newColor( 'rgba(0, 33, 176, .62)' );
return $color->getNew( 'lightness', $color->lightness - 10 )->toCSS( 'hex' )
```
Example 2: Remove any traces of green from an HSL color
```php
// Create a new color object using an HSL color as source
$color = ariColor::newColor( 'hsl(200, 33%, 82%)' );
// I don't like green, color, let's remove any traces of green from that color
$new_color = $color->getNew( 'green', 0 );
```

### `toCSS`
Returns a CSS-formatted color value.

The `toCSS` has a single argument:
1. `$mode`: can accept any of the values listed below (defaults to `hex` if undefined)

* `hex`
* `rgb`
* `rgba`
* `hsl`
* `hsla`

Example:
```php
// Create our instance
$color = ariColor::newColor( 'hsl(200, 33%, 82%)' );
// Get HEX color
$hex = $color->toCSS( 'hex' );
// Get RGB color
$rgb = $color->toCSS( 'rgb' );
// Get RGBA color
$rgba = $color->toCSS( 'rgba' );
// Get HSL color
$hsl = $color->toCSS( 'hsl' );
// Get HSLA color
$hsla = $color->toCSS( 'hsla' );

```

### Color sanitization:
All colors are sanitized so you could easily write a proxy function that will always return a sanitized color like this:
```php
/**
 * Sanitizes a CSS color.
 * 
 * @param $color  string   accepts all CSS-valid color formats
 * @return        string   the sanitized color
 */
function custom_color_sanitize( $color = '' ) {
	// If empty, return empty
	if ( '' == $color ) {
		return '';
	}
	// If transparent, return 'transparent'
	if ( is_string( $color ) && 'transparent' == trim( $color ) ) {
		return 'transparent';
	}
	// Instantiate the object
	$color_obj = ariColor::newColor( $color );
	// Return a CSS value, using the auto-detected mode
	return $color_obj->toCSS( $color_obj->mode );
}
```

You can even use a function like this one as a `sanitize_callback` in a customizer control :)