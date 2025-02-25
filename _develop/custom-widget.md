---
layout: develop
title: Writing Widgets

summary: The Fyne toolkit includes many standard widgets that make it easy to build basic applications. At some point you may want to use a widget that does not exist, and so the capability to use custom widgets will be useful. This page outlined how to write a new widget and include it in your application. We saw how widget state and logic is kept separate from the renderer for better testing and illustrated how to set up a renderer for your new widget.

order: 35
---

## Writing a Custom Widget

---

The standard widgets included with Fyne are designed to support standard user interactions and requirements.
As a GUI often has to provide custom functionality it may be neccessary to write a custom widget.
This article outlines how.

A widget is split into two areas - each implementing a standard interface - the `fyne.Widget` and the `fyne.WidgetRenderer`. The widget defines behaviour and state, with the renderer being used to define how it should be drawn to screen.

### fyne.Widget

A widget in Fyne is simply a stateful canvas object that has its rendering
definition separated from the main logic. As you can see from the `fyne.Widget`
interface there is not much that must be implemented.

```go
type Widget interface {
	CanvasObject

	CreateRenderer() WidgetRenderer
}

```

As a widget needs to be used like any other canvas item we inherit from the same interface. As a reminder the `fyne.CanvasObject` interface is defined as:

```go
type CanvasObject interface {
	// geometry
	Size() Size
	Resize(Size)
	Position() Position
	Move(Position)
	MinSize() Size

	// visibility
	Visible() bool
	Show()
	Hide()
}
```

Each widget definition will contain much more than the interface requires.
It is standard in a Fyne widget to export the fields which define behaviour
(just like the primitives defined in the `canvas` package).

For example, look at the `widget.Button` type:

```go
type Button struct {
	baseWidget
	Text  string
	Style ButtonStyle
	Icon  fyne.Resource

	OnTapped func()
}
```

As a standard widget it is able to make use of a private helper to handle the main `fyne.CanvasObject` boilerplate, but other than that it's a simple type definition.
You can see how each of these items store state about the widget behaviour but nothing about how it is rendered.

### fyne.WidgetRenderer

The widget renderer is responsible for managing a list of `fyne.CanvasObject` primitives that come together to create the design of our widget. It is much like
a `fyne.Container` with a custom layout and some additional theme handling.

Every widget must provide a renderer, but it is entirely possible to re-use a renderer from another widget - especially if your widget is a lightweight wrapper around another standard control.

```go
type WidgetRenderer interface {
	Layout(Size)
	MinSize() Size

	Refresh()
	ApplyTheme()
	BackgroundColor() color.Color
	Objects() []CanvasObject
	Destroy()
}
```

As you can see the `Layout(Size)` and `MinSize()` functions are similar to the
`fyne.Layout` interface, without the `[]fyne.CanvasObject` parameter - this is because a widget does need to be laid out but it controls which objects will be included.

The `BackgroundColor()` function sets the colour that will be drawn under the widget
and `ApplyTheme()` will be called if theme changes have occurred. The `Refresh()` method is triggered when the widget this renderer draws has changed so we may
need to make adjustments to how it looks. Lastly the `Destroy()` method is called when this renderer is no longer needed so it should clear any resources that would otherwise leak.

Compare again with the button widget - it's `fyne.WidgetRenderer` implementation is based on the following type:

```go
type buttonRenderer struct {
	icon  *canvas.Image
	label *canvas.Text

	objects []fyne.CanvasObject
	button  *Button
}
```

As you can see it has fields to cache the actual image and text canvas objects
for drawing. It keeps track of the slice of objects required by `fyne.WidgetRenderer` as a convenience.

Lastly it keeps a reference to tbe `widget.Button` for all state information.
In the `Refresh()` method it will update the graphical state based on any changes
in the underlying `widget.Button` type.

### Bring it together

The above interfaces need to be implemented in a new type that you define.
This typically takes the form of a struct that encapsulates the basic properties
of a `fyne.CanvasObject` and additionally any state that your widget requires.

In your widget's `CreateRenderer()` function you return a new instance of the
renderer defined above. The widget and driver code in Fyne will ensure that this
is cached accordingly - this method may be called many times (for example if a
widget is hidden and then shown). If `CreateRenderer()` is called again you
should return a new renderer instance as the old one may have been destroyed.
Take care not to keep any important state in your renderer - animation tickers
are well suited to that location but current values would not be.

