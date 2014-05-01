# MotionKit

*The RubyMotion layout and styling gem.*

1. Non-polluting
2. Simple, easy to remember syntax
3. ProMotion/RMQ/SugarCube-compatible
4. Styles and layouts are compiled
5. Crossplatform compatibility: iOS, OSX
6. Crossframework compatibility:
   - [UIKit][readmore-uikit]
   - [ApplicationKit][readmore-applicationkit]
   - [Joybox][readmore-joybox]
   - [SpriteKit][readmore-spritekit]
   - [CoreAnimation][readmore-coreanimation]
   - [NSMenu/NSMenuItem][readmore-nsmenu]
7. Written by [the authors][authors] of [ProMotion][] and [Teacup][]

[authors]: CONTRIBUTORS.md
[Colin]: https://github.com/colinta
[Jamon]: https://github.com/jamonholmgren
[ProMotion]: https://github.com/clearsightstudio/ProMotion
[RMQ]: https://github.com/infinitered/rmq
[Teacup]: https://github.com/rubymotion/teacup

[readmore-uikit]:          https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#uikit
[readmore-applicationkit]: https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#applicationkit
[readmore-joybox]:         https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#joybox
[readmore-spritekit]:      https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#spritekit
[readmore-coreanimation]:  https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#coreanimation
[readmore-nsmenu]:         https://github.com/rubymotion/motion-kit/blob/master/READMORE.md#nsmenu

## What happened to Teacup??

You can [read all about](#goodbye-teacup) why Colin decided that Teacup needed to
be replaced with a new project, rather than upgraded or refactored.


## Usage

From your controller you will instantiate a `MotionKit::Layout` instance, and
request views from it.  `layout.view` is the root view, and it's common to
assign this to `self.view` in your `loadView` method.  You'll also want to hook
up your instance variables, using `layout.get(:id)` or using instance variables.

```ruby
class LoginController < UIViewController
  def loadView
    @layout = LoginLayout.new
    self.view = @layout.view

    @button = @layout.get(:button)  # This will be created in our layout (below)
    @button = @layout.button  # Alternatively you can use instance variables and accessor methods
  end

  def viewDidLoad
    @button.on(:touch) { my_code } # Mix with some SugarCube for sweetness!
    rmq(@button).on(:touch) { my_code } # and of course RMQ works just as well
  end
end
```


### Lay out your subviews with a clean DSL

In a layout class, the `layout` method is expected to create the view hierarchy,
and it should also take care of frames and styling.  You can apply styles here,
and it's handy to do so when you are creating a quick mock-up, or a very small
app.  But in a real application, you'll want to include a Stylesheet module, so
your layout isn't cluttered with all your styling code.

Here's a layout that just puts a label and a button in the middle of the screen:

```ruby
class SimpleLayout < MotionKit::Layout
  view :button

  def layout
    add UILabel, :label

    # the `view` method above creates a read-only accessor that will build the
    # layout if it hasn't been built already, so you can call `layout.button`
    # before `layout.view` and you won't get nil, and layout.view will be built.
    @button = add UIButton, :button

    # If you want Key-Value Observation compliance you can use the setter:
    self.button = add UIButton, :button
  end

  def label_style
    text 'Hi there! Welcome to MotionKit'
    font UIFont.fontWithName('Comic Sans', size: 24)
    sizeToFit
    center [CGRectGetMidX(superview.bounds), CGRectGetMidY(superview.bounds)]
    text_alignment UITextAlignmentCenter
    text_color UIColor.whiteColor

    # if you prefer to use shorthands from another gem, you certainly can!
    background_color rmq.color.white  # from RMQ
    background_color :white.uicolor   # from SugarCube
  end

  def button_style
    # this will call 'setTitle(forState:)' via a styling method
    title 'Press it!'
    center [CGRectGetMidX(superview.bounds), CGRectGetMidY(superview.bounds) + 50]
  end

end
```

Nice, that should be pretty easy to follow, right?  M'kay, in this next, more
complicated layout we'll create a login page, with a 'Login' button and inputs
for username and password.  In this example, I will assign the frame in the
`layout` method, instead of in the `_style` methods.  This is purely an
aesthetic choice.  Some people like to have their frame code in the layout
method, others like to put it in the _style methods.  I point it out only as an
available feature.

```ruby
class LoginLayout < MotionKit::Layout
  # we write our `_style` methods in a module
  include LoginStyles

  def layout
    # we know it's easy to add a subview, with a stylename...
    add UIImageView, :logo

    # but even better, pass the 'frame' in, too:
    add UIImageView, :logo do
      frame [[0, 0], [320, 568]]
    end
    # hardcoded dimensions!? there's got to be a better way...

    # This frame argument will be handed to the 'MotionKit::Layout#frame'
    # method, which can accept lots of shorthands.  Let's use one to scale the
    # imageview so that it fills the width, but keeps its aspect ratio.
    add UIImageView, :logo do
      frame [[0, 0], ['100%', :scale]]
    end
    # 'scale' uses sizeToFit and the other width/height property to keep the
    # aspect ratio the same. Neat, huh?

    add UIView, :button_container do
      # Like I said, the frame method is very powerful, there are lots of
      # ways it can help with laying out your rects, and it will even try to
      # apply the correct autoresizingMask for you; the from_bottom method will
      # set the UIAutoresizingMask to "FlexibleTop", and using '100%' in the
      # width will ensure the frame stays the width of its parent.
      frame from_bottom(height: 50, width: '100%')
      frame from_bottom(h: 50, w: '100%')  # is fine, too

      # same as above; assumes full width
      frame from_bottom(height: 50)

      # similar to Teacup, views added inside a block are added to that
      # container.  You can reference the container with 'superview' or
      # 'parent'.
      add UIButton, :login_button do
        background_color superview.background_color

        # 'parent' is not instance of a view, it's a special object that
        # acts like a placeholder for various values; if you want to assign
        # *any* superview property, use 'superview' instead.  'parent' is mostly
        # useful for setting the frame.
        frame [[ 10, 5 ], [ 50, parent.height - 10 ]]
      end
    end

    # 'container' is a generic view method, and will return 'UIView' on iOS and
    # 'NSView' on OS X.  Totally optional, but it helps keep your layout files
    # consistent across multiple platforms.
    add container, :inputs do
      frame x: 0, y: 0, width: '100%', height: '100% - 50'

      # setting autoresizing_mask should handle rotation events; this overrides
      # any automatic mask settings that occurred in 'frame'
      autoresizing_mask :pin_to_top, :flexible_height, :flexible_width

      # we'll use 'sizeToFit' to calculate the height
      add UITextField, :username_input do
        frame [[10, 10], ['100% - 10', :auto]]
      end
      add UITextField, :password_input do
        frame below(:username_input, margin: 8)
      end
    end
  end
end
```


### Styles are compiled, simple, and clean

In MotionKit, when you define a method that has the same name as a view
stylename with the suffix "_style", that method is called and is expected to
style that view.

```ruby
class LoginLayout < MK::Layout

  def layout
    add UIImageView, :logo do
      frame [[0, 0], ['100%', :scale]]
    end
    add UIView, :button_container do
      # ...
    end
  end

  def logo_style
    image UIImage.imageNamed('logo')
  end

  def button_container_style
    background_color UIColor.clearColor
  end

end
```

So as an additional code-cleanup step, why not put those methods in a module,
and include them in your layout! Sounds clean and organized to me! You can
include multiple stylesheets this way, just be careful around name collisions.

```ruby
module LoginStyles

  def login_button_style
    background_color '#51A8E7'.uicolor
    title 'Log In'
    # `layer` returns a CALayer, which in turn becomes the new context inside
    # this block!
    layer do
      corner_radius 7.0
      shadow_color '#000000'.cgcolor
      shadow_opacity 0.9
      shadow_radius 2.0
      shadow_offset [0, 0]
    end
  end

end

# back in our LoginLayout class
class LoginLayout
  include LoginStyles

  def layout
    # ...
  end

end
```


### How do styles get applied?

If you've used RMQ's Stylers, you'll recognize a very similar pattern here. In
RMQ the 'style' methods are handed a 'Styler' instance, which wraps access to
the view.  In MotionKit we make use of `method_missing` to call these methods
indirectly.  But other than that, the design is very similar.

```ruby
  def login_label_style
    text 'Press me'  # this gets delegated to UILabel#text
  end

  # It's not hard to add extensions for common tasks, like setting the "normal"
  # title on a UIButton
  def login_button_style
    title 'Press me'
    # this gets delegated to UIButtonLayout#title(title), which in turn calls
    # button.setTitle(title, forState: UIControlStateNormal)
  end
```

MotionKit offers shortcuts and mini-DSLs for frames, auto-layout, and
miscellaneous helpers.  But if a method is not defined, it is sent to the view
after a little introspection. If you call a method like `title_color value`, MotionKit
will try to call:

- `setTitle_color(value)`
- `title_color=(value)`
- `title_color(value)`
- (try again, converting to camelCase)
- `setTitleColor(value)`
- `titleColor=(value)`
- `titleColor(value)`
- (failure:) `raise NoMethodError`

```ruby
  def login_button_style
    background_color UIColor.clearColor  # this gets converted to `self.target.backgroundColor = ...`
  end
```

You can easily add your own helpers to MotionKit's existing Layout classes. They
are all named consistenly, e.g. `MotionKit::UIViewLayout`, e.g.
`MotionKit::UILabelLayout`.  Just open up these classes and hack away.

```ruby
module MotionKit
  # these helpers will only be applied to instances of UILabel and UILabel
  # subclasses
  class UILabelLayout

    # style methods can accept any number of arguments, and a block. The current
    # view should be referred to via the method `target`
    def color(color)
      target.textColor = color
    end

    # If a block is passed it is your responsibility to call `context(val,
    # &block)`, if that is appropriate.  I'll use `UIView#layer` as an example,
    # but actually if you pass a block to a method that returns an object, that
    # block will be called with that object as the context.
    def layer(&block)
      context(target.layer, &block)
    end

    # Sure, you can add flow-control mechanisms if that's your thing!
    #
    # You can use the block to conditionally call code; on iOS there are
    # orientation helpers `portrait`, `landscape`, etc that apply styles based
    # on the current orientation.
    def sometimes(&block)
      if rand > 0.5
        yield
      end
    end

  end
end
```

For your own custom classes, you can provide a Layout class by calling the
`targets` method in your class body.

```ruby
# Be sure to extend an existing Layout class, otherwise you'll lose a lot of
# functionality.  Often this will be `MK::UIViewLayout` on iOS and
# `MK::NSViewLayout` on OS X.
class CustomLayout < MK::UIViewLayout
  targets CustomView

  def fore_color(value)
    target.foregroundColor = value
  end

end
```


### Frames

There are lots of frame helpers for NSView and UIView subclasses.  It's cool
that you can set position and sizes as percents, but scroll down to see examples
of setting frames *based on any other view*.  These are super useful!  Most of
the ideas, method names, and some code came straight out of
[geomotion](https://github.com/clayallsopp/geomotion).  It's not *quite as
powerful* as geomotion, but it's close!

One advantage over geomotion is that many of these frame helpers accept a view
or view name, so that you can place the view relative to that view.

```ruby
# most direct way to set the frame, using pt values
frame [[0, 0], [320, 568]]

# using sizes relative to superview
frame [[5, 5], ['100% - 10pt', '100% - 10pt']]
# the 'pt' suffix is optional, and ignored.  in the future we could add support
# for other suffixes - would that even be useful?  probably not...

# other available methods:
origin [5, 5]
x 5  # aka left(..)
right 5  # right side of the view is 5px from the left side of the superview
bottom 5  # bottom of the view is 5px from the top of the superview
size ['100% - 10', '100% - 10']
width '100% - 10'  # aka w(...)
height '100% - 10'  # aka h(...)

size ['90%', '90%']
center ['50%', '50%']

########
# +--------------------------------------------------+
# |from_top_left       from_top        from_top_right|
# |                                                  |
# |from_left          from_center          from_right|
# |                                                  |
# |from_bottom_left   from_bottom   from_bottom_right|
# +--------------------------------------------------+
```

You can position the view *relative to other views*, either the superview or any
other view.  You must pass the return value to `frame`.

```ruby
# If you don't specify a view to base off of, the view is positioned relative to
# the superview:
frame from_bottom_right(size: [100, 100])  # 100x100 in the BR corner
frame from_bottom(size: ['100%', 32])  # full width, 32pt height
frame from_top_right(left: 5)

# But if you pass a view or symbol as the first arg, the position will be
# relative to that view
from_top_right(:info_container, left: 5)


########
#          above
#          +---+
#  left_of |   | right_of
# (before) |   | (after)
#          +---+
#          below

# these methods *require* another view.
frame above(:foo, up: 8)

frame above(:foo, up: 8)
frame before(:foo, left: 8)
frame relative_to(:foo, down: 5, right: 5)

# it's not common, but you can also pass a view to any of these methods
foo = self.get(:foo)
frame from_bottom_left(foo, up: 5, left: 5)
```


### Autoresizing mask

You can pass symbols like `autoresizing_mask :flexible_width`, or use
symbols that have more intuitive meaning than the usual
`UIViewAutoresizingFlexible*` constants.  These work in iOS and OS X.

```ruby
autoresizing_mask :flexible_right, :flexible_bottom, :flexible_width
autoresizing_mask :pin_to_left, :rigid_top  # 'rigid' undoes a 'flexible' setting
autoresizing_mask :pin_to_bottom, :flexible_width
autoresizing_mask :fill_top

flexible_left:       Sticks to the right side
flexible_width:      Width varies with parent
flexible_right:      Sticks to the left side
flexible_top:        Sticks to the bottom
flexible_height:     Height varies with parent
flexible_bottom:     Sticks to the top

rigid_left:          Left side stays constant (undoes :flexible_left)
rigid_width:         Width stays constant (undoes :flexible_width)
rigid_right:         Right side stays constant (undoes :flexible_right)
rigid_top:           Top stays constant (undoes :flexible_top)
rigid_height:        Height stays constant (undoes :flexible_height)
rigid_bottom:        Bottom stays constant (undoes :flexible_bottom)

fill:                The size increases with an increase in parent size
fill_top:            Width varies with parent and view sticks to the top
fill_bottom:         Width varies with parent and view sticks to the bottom
fill_left:           Height varies with parent and view sticks to the left
fill_right:          Height varies with parent and view sticks to the right

pin_to_top_left:     View stays in top-left corner, size does not change.
pin_to_top:          View stays in top-center, size does not change.
pin_to_top_right:    View stays in top-right corner, size does not change.
pin_to_left:         View stays centered on the left, size does not change.
pin_to_center:       View stays centered, size does not change.
pin_to_right:        View stays centered on the right, size does not change.
pin_to_bottom_left:  View stays in bottom-left corner, size does not change.
pin_to_bottom:       View stays in bottom-center, size does not change.
pin_to_bottom_right: View stays in bottom-left corner, size does not change.
```


### Constraints / Auto Layout

Inside a `constraints` block you can use the same helpers as above, but you'll
be using Auto Layout instead.  This is the recommended way to set your frames,
but beware, Auto Layout can be frustrating...

```ruby
constraints do
  from_top_left x: 5, y: 5
  # or
  from_top_left down: 5, right:5
end
```

But of course with constraints you can setup *relationships* between views. The
named views work especially well here.

```ruby
constraints do
  from_top_left x: 5, y:5
  width.equals(:foo).minus(10)
  height.equals(:foo).minus(10)
  # that's repetitive, so just set 'size'
  size.equals(:foo).minus(10)
  size.equals(:foo).minus([10, 15])  # 10pt thinner, 15pt shorter
end
```

Just like with frame helpers you can use `:names` to refer to other views, but
get this: the views need not be created yet!  This is because when you setup a
constraints block, it isn't resolved immediately; the symbols are resolved at
the end.  This feature uses the `deferred` method behind the scenes to
accomplish this.

```ruby
# believe it or not, this code works! :-)
add UIView, :foo do
  constraints do
    width.equals(:bar).plus(10)
  end
end

add UIView, :bar do
  constraints do
    width.equals(:foo).minus(10)

    # if you have multiple views with the same name:
    width.equals(last(:foo)).minus(10)
    width.equals(first(:foo)).minus(10)
  end
end
```


### Some handy tricks

#### Orientation specific styles

These are available on iOS.

```ruby
add UIView, :container do
  portrait do
    frame from_top(width: '100%', height: 100)
  end
  landscape do
    frame from_top_left(width: 300, height: 100)
  end
end
```

#### Update views via 'initial', 'reapply', and 'deferred'

If you call 'layout.reapply!', your style methods will be called again (but
NOT the `layout` method). You very well might want to control what methods get
called on later invocations, or only on the *initial* layout.

This is more for being able to initialize values, or to handle orientation, than
anything else.  There is not much performance increase/decrease if you just
reapply styles every time, but you might not want to have your frame or colors
reset, if you've done some animation.

```ruby
def login_button_style
  # only once, when the `layout` is created
  initial do
  end

  # on later invocations
  reapply do
  end

  # style every time
  title 'Press Me'
end
```

Or, you might need to set a frame or other property based on a view that hasn't
been created yet. In this case, you can use `deferred` to have a block of code
run after the current layout is completed.

```ruby
def login_button_style
  deferred do
    frame below(last(:label), height: 20)
  end
end
```

#### Apply styles via module

```ruby
module AppStyles

  def rounded_button
    layer do
      corner_radius 7
      masks_to_bounds true
    end
  end

end

class LoginLayout < MotionKit::Layout
  include AppStyles

  def layout
    add button, :login_button
  end

  def login_button_style
    self.rounded_button
    title 'Login'
  end

end
```

#### Use 'container' classes

You saw the `container` method above, right?  Well those are easy to create.  So
if you've got a 'button' subclass that you just love, you can assign it to a
layout class method.

```ruby
module AppViews

  def button
    MySpecialButton
  end

end

class MyLayout < MotionKit::Layout
  extend AppViews

  def layout
    add button  # adds MySpecialButton instance
  end

end
```


# Contributing

We welcome your contributions! Please be sure to run the specs before you do,
and consider adding support for both iOS and OS X.

To run the specs for both platforms, you will need to run `rake spec` twice:

```
> rake spec  # runs iOS specs
> rake spec platform=osx  # OS X specs
```


# Goodbye Teacup
###### by [colinta][Colin]

If you've worked with XIB/NIB files, you might know that while they can be
cumbersome to deal with, they have the great benefit of keeping your controllers
free of layout and styling concerns. [Teacup][] brought some of this benefit, in
the form of stylesheets, but you still built the layout in the body of your
controller file.

Plus Teacup is a beast! Imported stylesheets, orientation change events,
auto-layout support. It's got a ton of features, but with that comes a lot of
complexity. This has led to an unfortunate situation - I'm the *only* person who
understands the code base! This was never the intention of Teacup. It started
out as, and was always meant to be, a community project, with contributions
coming from all of its users.

When [ProMotion][] and later [RMQ][] were released, they both included their own
styling mechanisms. Including Teacup as a dependency would have placed a huge
burden on their users, and they would have had to ensure compatibility. Since
Teacup does a lot of method swizzling on base classes, this is not a trivial
undertaking.

If you use RMQ or ProMotion already, you'll find that MotionKit fits right in.
We designed it to be something that can easily be brought into an existing
project, too; it does not extend any base classes (the cross platform support is
handled by the [style handlers][handlers]), and it's completely opt-in.

Unlike Teacup, you won't have your styles reapplied due to orientation changes,
but it's *really* easy to set that up, as you'll see.

Big thanks to everyone who contributed on this project!  I hope it serves you
as well as Teacup, and for even longer into the future.

Sincerely,

Colin T.A. Gray
Feb 13, 2014
