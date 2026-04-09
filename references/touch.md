---
name: imba-touch
description: @touch event reference — custom pointer event combining pointerdown/move/up, modifiers (.fit, .sync, .moved, .pin, .reframe), touch interface properties, drag examples.
---

# @touch Event

Custom event combining `pointerdown` → `pointermove` → `pointerup` in one handler. More convenient than raw pointer events.

## Touch Interface

| Property | Description |
|----------|-------------|
| `active?` | True if touch is still active |
| `ended?` | True if touch has ended |
| `x` | Final X coordinate (after modifiers) |
| `y` | Final Y coordinate (after modifiers) |
| `dx` | Horizontal movement delta |
| `dy` | Vertical movement delta |
| `clientX` | X in DOM content coordinates |
| `clientY` | Y in DOM content coordinates |
| `pointerId` | Unique identifier for the pointer |
| `pointerType` | Device type (mouse, pen, touch) |
| `altKey` / `ctrlKey` / `shiftKey` / `metaKey` | Modifier keys |

## Modifiers

### .fit — coordinate space mapping

Converts x,y relative to an element box into interpolated values between start and end.

```imba
# Fit to element box, 0–100 range
<self @touch.fit(self,0,100)=(x=e.x,y=e.y)> "x:{x} y:{y}"

# Fit with step (snap to grid)
<self @touch.fit(self,2)=(x=e.x)> "snapped"

# Fit to ranges [start,end] per axis
<div.box @touch.fit([0,-5],[1,5],0.1)=handler>
```

```imba
tag Slider
	min = -50
	max = 50
	step = 1
	value = 0

	<self @touch.fit(min,max,step)=(value = e.x)>
		<.thumb[l:{100 * (value - min) / (max - min)}%]> <b> value

imba.mount do <>
	<Slider min=0 max=1 step=0.1>
	<Slider min=-100 max=100 step=1>
```

### .sync — sync coordinates to object

```imba
const pos = {x:0,y:0}
imba.mount do <>
	<div[w:60px x:{pos.x} y:{pos.y}].box @touch.sync(pos)> 'drag'
	<div[w:60px x:{pos.y} y:{pos.x}].box> 'flipped'
```

Custom property names (second and third args):

```imba
const data = {a:0,b:0}
imba.mount do <>
	<div[w:80px].box @touch.sync(data,'a','b')> 'drag'
	<label> "a:{data.a} b:{data.b}"
```

### .moved — threshold before activating

```imba
# Won't trigger until moved 30px in any direction
<self @touch.moved(30px)=(x=e.x,y=e.y)>

# Axis-specific threshold
<self @touch.moved(30px,'x')=(x=e.x,y=e.y)>   # horizontal only
<self @touch.moved(30px,'y')=(x=e.x,y=e.y)>   # vertical only
<self @touch.moved(30px,'up')=(x=e.x,y=e.y)>
<self @touch.moved(30px,'down')=(x=e.x,y=e.y)>
<self @touch.moved(30px,'left')=(x=e.x,y=e.y)>
<self @touch.moved(30px,'right')=(x=e.x,y=e.y)>
```

### .pin — lock position to start point

### .reframe — recompute element rect on each move

## Full Examples

### Draggable element

```imba
tag drag-me
	css d:block pos:relative p:3 m:1
		bg:white bxs:sm rd:sm cursor:default
		@touch scale:1.02
		@move scale:1.05 rotate:2deg zi:2 bxs:lg

	def build
		x = y = 0

	def render
		<self[x:{x} y:{y}] @touch.moved.sync(self)> 'drag me'

imba.mount do <div.grid>
	<drag-me>
	<drag-me>
```

### Resizable split panel

```imba
tag Panel
	split = 70

	<self[d:flex pos:absolute inset:0]>
		<div[bg:teal2 flex-basis:{split}%]>
		<div[fls:0 w:2 bg:teal3 @touch:teal5]
			@touch.pin.fit(self,0,100,2)=(split=e.x)>
		<div[bg:teal1 flex:1]>

imba.mount do <Panel>
```

### Canvas painting

```imba
const dpr = window.devicePixelRatio

tag app-paint
	size = 500

	def draw e
		let path = e.$path ||= new Path2D
		path.lineTo(e.x * dpr,e.y * dpr)
		$canvas.getContext('2d').stroke(path)

	def render
		<self[d:block overflow:hidden bg:blue1]>
			<canvas$canvas[size:{size}px]
				width=size*dpr height=size*dpr @touch.fit(self)=draw>

imba.mount <app-paint>
```

### Marquee selection

```imba
tag Marquis
	def handle t
		x = t.x0 + Math.min(t.dx,0)
		y = t.y0 + Math.min(t.dy,0)
		w = Math.abs(t.dx)
		h = Math.abs(t.dy)
	<self[inset:0 of:hidden]@touch.flag('drag').reframe(self)=handle>
		<[x:{x}px y:{y}px w:{w}px h:{h}px]>
			css bg:blue3/10 bd:blue4 o:0 ..drag:1
```
