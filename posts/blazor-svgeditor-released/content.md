<script>
    hljs.highlightAll();
</script>
### Features
Before we look at how we can use the package, we will go through some of the central features of the editor, accompanied by some small videos that show those features.

You can find the project on GitHub: [https://github.com/KristofferStrube/Blazor.SVGEditor](https://github.com/KristofferStrube/Blazor.SVGEditor)

You can find the package on NuGet: [KristofferStrube.Blazor.SVGEditor](https://www.nuget.org/packages/KristofferStrube.Blazor.SVGEditor)

And you can demo all the below features in the online demo site: [https://kristofferstrube.github.io/Blazor.SVGEditor/](https://kristofferstrube.github.io/Blazor.SVGEditor/)

#### Setting input and getting updates
The editor parses the XML structure that SVGs are defined in to be able to edit every detail of an SVG. To make it possible for the library consumer to populate this initial value, the `Input` is exposed as a Parameter that needs to be set when the component is used. The component has another Parameter `InputUpdated` that you can set to listen for when changes happen to the underlying SVG code. This can be used to get a live view of the underlying code while you edit it.

<video width="600px" src="{site}/videos/svg-editor-listen-to-changes.mp4" autoplay muted controls loop></video>

#### Editing shapes
The editor enables us to update all shapes. Among these shapes are lines, polygons, rectangles, and paths. Paths are especially complex to parse as they contain many instructions that can be used in many combinations.

<video width="600px" src="{site}/videos/svg-editor-edit-shapes.mp4" autoplay muted controls loop></video>

#### Creating new shapes
The editor supports creating new shapes through its context menu and supports unique flows for easy creation for each of these shapes.

<video width="600px" src="{site}/videos/svg-editor-create-shapes.mp4" autoplay muted controls loop></video>

#### Panning and zooming
Sometimes some fine details in the SVG would be easier to edit up close either because they are very small or because you need high precision. To support this, we enable you to zoom and move around the canvas using the scroll wheel and the mouse's middle button. These interactions are only supported for desktop as they rely on you having a mouse or at least some way to scroll. If you want to contribute to the project, then I have an open issue to add support for touch devices: <a href="https://github.com/KristofferStrube/Blazor.SVGEditor/issues/11" target="_blank">Blazor.SVGEditor Issue #11</a>

<video width="600px" src="{site}/videos/svg-editor-panning-and-zooming.mp4" autoplay muted controls loop></video>

#### Multi-select and area selection
The editor also enables the user to select multiple elements when editing by holding the `CTRL` key down while selecting. To select multiple items in an area, the user can hold down the left mouse button and drag to mark the desired elements, similar to selecting multiple files in the Windows file explorer.

<video width="600px" src="{site}/videos/svg-editor-multi-select.mp4" autoplay muted controls loop></video>

#### Color selection
Using the context menu, the users can change the fill color of all shapes by picking a color from a modal. They can likewise change the color of the strokes (the outline of shapes) and other details related to the stroke.

<video width="600px" src="{site}/videos/svg-editor-color-selection.mp4" autoplay muted controls loop></video>

#### Grouping and Ungrouping
SVG elements can be grouped using the `<g>` tag. When elements are grouped, the editor moves them together and disables the ability to edit them individually. After selecting one or more elements, you can create a new group using the context menu. You can likewise ungroup elements that are grouped together using the context menu.

<video width="600px" src="{site}/videos/svg-editor-grouping.mp4" autoplay muted controls loop></video>

#### Copy, paste, remove, and re-order
The last core part of the editor is the ability to copy, paste, remove, and re-order elements. You can mark one or more elements and press copy to paste their content into your clipboard. Then you can click paste to insert a copy. If you had any item marked when you pasted, it would make sure to add the element just above that; otherwise, it will paste it in front of all items. You can likewise select any amount of items and press remove to remove them. The last function is under the menu called move, which gives options for moving elements back or forward in the render hierarchy. 

<video width="600px" src="{site}/videos/svg-editor-copy-paste-remove.mp4" autoplay muted controls loop></video>

#### Miscellaneous features
Some other features are also available in the editor but need more refinement. Among these are support for editing and creating linear gradients, editing and creating animations, and optimizing paths via the context menu and anchor movement interactions.

### Customization
Now that we have seen what the editor can do out of the box let's see what we can do with it if we extend it. If you want to follow these steps yourself, you should start by following the [Getting Started section in the repository README](https://github.com/KristofferStrube/Blazor.SVGEditor#getting-started). If you don't want to follow these steps and just want to browse the nice videos, then that is also okay. 

The example customization we will make is a network editor consisting of nodes and connectors. Commonly known as a graph. To start, we create the following page to set up a minimal editor that doesn't support any elements, has no options for adding new elements, and has a subset of all our possible context menu features.

```razor
<div style="height:80vh">
    <SVGEditor Input=@Input
               InputUpdated="(string s) => { Input = s; StateHasChanged(); }"¨
               SnapToInteger=true
               SupportedElements=SupportedElements
               AddNewSVGElementMenuItems=AddNewSVGElementMenuItems
               ActionMenuItems=ActionMenuItems />
</div>

@code {
    protected string Input = @"";

    protected List<SupportedElement> SupportedElements { get; set; } = new()
    {
    };

    protected List<SupportedAddNewSVGElementMenuItem> AddNewSVGElementMenuItems { get; set; } = new()
    {
    };

    protected List<ActionMenuItem> ActionMenuItems { get; set; } = new() {
        new(typeof(StrokeMenuItem), (_, data) => data is Shape shape && !shape.IsChildElement),
        new(typeof(MoveMenuItem), (_, data) => data is Shape shape && !shape.IsChildElement),
        new(typeof(GroupMenuItem), (_, data) => data is Shape shape && !shape.IsChildElement),
        new(typeof(UngroupMenuItem), (_, data) => data is G g && !g.IsChildElement),
        new(typeof(RemoveMenuItem), (svgEditor, data) => data is Shape && !svgEditor.DisableRemoveElement),
        new(typeof(CopyMenuItem), (svgEditor, data) => data is Shape && !svgEditor.DisableCopyElement),
        new(typeof(PasteMenuItem), (svgEditor, _) => !svgEditor.DisablePasteElement),
    };
}
```

#### Node.cs
We will first make the classes and components needed to control, render, and add new nodes. This will build on the existing code we have created for editing Circles. We first add a class called `Node` that will handle how we interact with nodes and how we create new ones. We will later add more logic to this class to enrich its interaction with connectors.

```csharp
public class Node : Circle
{
    public Node(IElement element, SVGEditor svg) : base(element, svg)
    {
        string? id = element.GetAttribute("id");
        if (id is null || svg.Elements.Any(e => e.Id == id))
        {
            Id = Guid.NewGuid().ToString();
        }
    }
}
```

The class extends `Circle` as it shares most of its logic with it. The first thing we do in the constructor is to check if the SVG definition that the `Node` is constructed from has a unique `Id`, and if it doesn't, then assign a new `Id` to it so that we have a way to reference each node individually.

Next we override the `Presenter` property to customize how we present the `Node` with a `NodeEditor` instead of using the inherited `CircleEditor`.
```csharp
public override Type Presenter => typeof(NodeEditor);
```
We also override the `R` property that defines the radius of the `Circle` so that it is always `50` but still enables the `R` to be set to something else.
```csharp
public new double R { get => 50; set => base.R = value; }
```
Next, we override the `Stroke` property of the `Circle` so that it also updates the `Fill`. This aims to limit the user's options for editing the nodes to give a more homogeneous visual look.

```csharp
public override string Stroke
{
    get => base.Stroke;
    set
    {
        base.Stroke = value;
        int[] parts = value[1..].Chunk(2).Select(part => int.Parse(part, System.Globalization.NumberStyles.HexNumber)).ToArray();
        Fill = "#" + string.Join("", parts.Select(part => Math.Min(255, part + 50).ToString("X2")));
    }
}
```
Our goal is to set the `Fill` to a color that is a bit lighter than the newly set `Stroke` color. We know the `Stroke` will always be set using a 6-character long hex value prepended by a pound sign `'#'`. To increase the color brightness, we chunk the 6 characters into chunks of 2 characters, each representing one of the color parts, i.e., red, green, and blue. We parse these chunks as hex values and use these parsed values to construct the `Fill` color. To do this, we just join the parts back together as 2-digit hex values but increase the intensity of each color by 50, capped at 255 as that is the largest possible 2-digit hex value.

The last part we add to the `Node` class right now is a method for adding a new `Node`.
```csharp
public static new void AddNew(SVGEditor SVG)
{
    IElement element = SVG.Document.CreateElement("CIRCLE");
    element.SetAttribute("data-elementtype", "node");

    Node node = new(element, SVG)
    {
        Changed = SVG.UpdateInput,
        Stroke = "#28B6F6",
        R = 50
    };

    (node.Cx, node.Cy) = SVG.LocalDetransform(SVG.LastRightClick);

    SVG.ClearSelectedShapes();
    SVG.SelectShape(node);
    SVG.AddElement(node);
}
```
We first create the underlying XML element that the `Node` will use as its data underlying data structure. We set the attribute `data-elementtype` on this to `"node"` so that we can distinguish it from other `<circle>` tags.

Then we construct a new `Node` from this that will trigger the `SVGEditors`´s `UpdateInput` method whenever it changes and set its `Stroke` color and radius to our defaults.

We also set the center of the new `Node` to be the last place that the user has right-clicked so that it will appear near where they last used the context menu. The `SVGEditor` automatically keeps track of this position in the screen coordinate system. We might have panned around the SVG or zoomed in, so to get the original position in the coordinate system of the SVG, we parse the screen coordinate through the `SVGEditor` method `LocalDetransform` which can transform any point from the screen coordinate system to the SVG coordinate system.

Lastly, we deselect all selected shapes, select the new `Node`, and add it to the `SVGEditor` using the `AddElement` method.

#### NodeEditor.razor
We defined that the `Node` should be presented by a `NodeEditor` component. We want something very close to how we present a `Circle`, but without the capability to edit the radius. That gives us the following component:
```razor
@using BlazorContextMenu
@using KristofferStrube.Blazor.SVGEditor.ShapeEditors
@using KristofferStrube.Blazor.SVGEditor.Extensions
@inherits ShapeEditor<Node>

<ContextMenuTrigger MenuId="SVGMenu" WrapperTag="g" Data=@SVGElement MouseButtonTrigger="SVGElement.ShouldTriggerContextMenu ? MouseButtonTrigger.Right : (MouseButtonTrigger)4">
    <g transform="translate(@SVGElement.SVG.Translate.x.AsString() @SVGElement.SVG.Translate.y.AsString()) scale(@SVGElement.SVG.Scale.AsString())">
        <circle @ref=ElementReference
        @onfocusin="FocusElement"
        @onfocusout="UnfocusElement"
        @onpointerdown="SelectAsync"
        @onkeyup="KeyUp"
                tabindex="@(SVGElement.IsChildElement ? -1 : 0)"
                cx=@SVGElement.Cx.AsString()
                cy=@SVGElement.Cy.AsString()
                r=@SVGElement.R.AsString()
                stroke="@SVGElement.Stroke"
                stroke-width="@SVGElement.StrokeWidth"
                stroke-linecap="@SVGElement.StrokeLinecap.AsString()"
                stroke-linejoin="@SVGElement.StrokeLinejoin.AsString()"
                stroke-dasharray="@SVGElement.StrokeDasharray"
                stroke-dashoffset="@SVGElement.StrokeDashoffset.AsString()"
                fill="@SVGElement.Fill"
                style="filter:brightness(@(SVGElement.Selected ? "0.8" : "1"))">
        </circle>
    </g>
</ContextMenuTrigger>
```
Another thing we have changed is that the circle tag applies a filter to darken its fill if selected. It extends the abstract `ShapeEditor` component that implements all its necessary logic.

#### AddNewNodeMenuItem.razor
We also need to define a simple menu item that we can use to add new nodes. 

```razor
@using BlazorContextMenu

<Item OnClick="_ => Node.AddNew(SVGEditor)">
    <div class="icon">⚫</div> New Node
</Item>

@code {
    [CascadingParameter]
    public required SVGEditor SVGEditor { get; set; }

    [Parameter]
    public required object Data { get; set; }
}
```
The component will get a reference to the `SVGEditor` that it is used in through a `CascadingParameter`. It also gets initialized with reference to the item that was last right-clicked through the `Data` `Parameter`, if there was any. We don't use the `Data` property in our sample, but one way we could have used this would be to check if the last right-clicked element was a `Node`, and if it were, then use the same `Stroke` color. Instead, we simply invoke the `AddNew` method when the menu item is clicked.

Let's add these new components and classes to our sample razor page and see how it looks. We add the `Node` class to our list of supported elements and specify that it should handle representing any tag that is a circle with the `data-elementtype` set to `"node"`.
```csharp
protected List<SupportedElement> SupportedElements = new()
{
    new(typeof(Node), element => element.TagName is "CIRCLE" && element.GetAttribute("data-elementtype") == "node"),
};
```
And we add the menu item for adding a new `Node` to our list of menu items for the "Add New" sub-menu and specify that the menu item should always be presented.
```csharp
protected List<SupportedAddNewSVGElementMenuItem> AddNewSVGElementMenuItems = new()
{
    new(typeof(AddNewNodeMenuItem), (_,_) => true),
};
```

Now when we run the project, we will be able to view, edit and create nodes.

<video width="600px" src="{site}/videos/svg-editor-just-nodes.mp4" autoplay muted controls loop></video>

#### Connector.cs
Now we are ready to add our connectors. We start of with creating a new class that extends the existing `Line` class.
```csharp
public class Connector : Line
{
    public Connector(IElement element, SVGEditor svg) : base(element, svg)
    {
        UpdateLine();
    }
}
```
When it is initialized, we call a method that will update the position of the line. We also define a custom presenter for the `Connector` that slightly changes how it is presented and how a user can interact with it.
```csharp
public override Type Presenter => typeof(ConnectorEditor);
```
We will not set the connector's position directly by moving its endpoints. Instead, this will be controlled by which nodes it is connected to. We define two properties for this that will simplify how we can access and update them later. We first define which `Node` the connecter starts in.
```csharp
public Node? From
{
    get
    {
        var from = (Node?)SVG.Elements.FirstOrDefault(e => e is Node && e.Id == Element.GetAttribute("data-from"));
        _ = from?.RelatedConnectors.Add(this);
        return from;
    }
    set
    {
        if (From is { } from)
        {
            _ = from.RelatedConnectors.Remove(this);
        }
        if (value is null)
        {
            _ = Element.RemoveAttribute("data-from");
        }
        else
        {
            Element.SetAttribute("data-from", value.Id);
            _ = value.RelatedConnectors.Add(this);
        }
        Changed?.Invoke(this);
    }
}
```
When we get the `From` `Node`, it finds the `Node` among all registered elements that also match the `Id` with the content of the `data-from` attribute of the connecter. After this, it adds this connector to a `HashSet` called `RelatedConnectors` on the `Node`.  If we set the `From` `Node`, we remove the previous `From` `Node` from the `HashSet` and either remove the attribute if it was set to `null` or set it to the `Id` of the new `Node` and again adds the connector to the `HashSet`. We also invoke `Changed` as we have changed some a part of the SVG definition when we set this property. We will show what we have added to the `Node` class to support the `RelatedConnectors` `HashSet` in just a bit, but let's first show the code for the `To` property as that is almost identical to the `From` node.
```csharp
public Node? To
{
    get
    {
        var to = (Node?)SVG.Elements.FirstOrDefault(e => e is Node && e.Id == Element.GetAttribute("data-to"));
        _ = to?.RelatedConnectors.Add(this);
        return to;
    }
    set
    {
        if (To is { } to)
        {
            _ = to.RelatedConnectors.Remove(this);
        }
        if (value is null)
        {
            _ = Element.RemoveAttribute("data-to");
        }
        else
        {
            Element.SetAttribute("data-to", value.Id);
            _ = value.RelatedConnectors.Add(this);
        }
        Changed?.Invoke(this);
    }
}
```
Let's see what we have added to the `Node` class to support the relationship between the connectors and nodes. 
We first add the `HashSet` that we mentioned to the `Node` class.
```csharp
public HashSet<Connector> RelatedConnectors { get; } = new();
```
We also override the `HandlePointerMove` method that every `Shape` implements to ensure that the related connectors are updated when the `Node` is moved.
```csharp
public override void HandlePointerMove(PointerEventArgs eventArgs)
{
    base.HandlePointerMove(eventArgs);
    if (SVG.EditMode is EditMode.Move)
    {
        foreach (Connector connector in RelatedConnectors)
        {
            connector.UpdateLine();
        }
    }
}
```
All related connectors should also be removed if a `Node` is removed. To support this, we can override the `BeforeBeingRemoved` method, which is called before any `ISVGElement` is removed from the editor.
```csharp
public override void BeforeBeingRemoved()
{
    foreach (Connector connector in RelatedConnectors)
    {
        SVG.RemoveElement(connector);
    }
}
```
Next we will go back to the `Connector` class. We start of with implementing the method for adding a new `Connector`.
```csharp
public static void AddNew(SVGEditor SVG, Node from)
{
    IElement element = SVG.Document.CreateElement("LINE");
    element.SetAttribute("data-elementtype", "connector");

    Connector connector = new(element, SVG)
    {
        Changed = SVG.UpdateInput,
        Stroke = "black",
        StrokeWidth = "5",
        From = from
    };
    SVG.EditMode = EditMode.Add;

    SVG.ClearSelectedShapes();
    SVG.SelectShape(connector);
    SVG.AddElement(connector);
}
```
We create a new XML element in the form of a line. We set the `data-elementtype` to `"connector"` on this, which we will use later to specify that it should only be the `Connector` class that handles this element. Next, we create a new `Connector` instance and set some defaults. This `AddNew` method differs from the one we specified by the `Node`. This method also takes a `Node` as a parameter. We use this to select which `Node` the `Connector` should go from so that we can finish the addition of the `Connector` with very few interactions. In the end, we do the same thing as for the `Node` `AddNew` method and clear all selected shapes, set the new `Connector` as the single selected shape, and add the new `Connector` to the `SVGEditor`.

To give the user some feedback while selecting a second `Node` to connect to, we want to update where the `Connector` points to follow the mouse until the user has decided on a `Node`. To control this we again override the `HandlePointerMove` method.

```csharp
public override void HandlePointerMove(PointerEventArgs eventArgs)
{
    if (SVG.EditMode is EditMode.Add)
    {
        (X2, Y2) = SVG.LocalDetransform((eventArgs.OffsetX, eventArgs.OffsetY));
        SetStart((X2, Y2));
    }
}
```
We don't want the `Connector` to handle a pointer move in any other case than when we are adding a new one. In this case, we set the end position equal to the pointer position, which is parsed to the method. After this, we update the position of the start position so that it is oriented towards where the cursor is currently. Let's see how we do that now. This includes a tiny bit of trigonometry, so be warned.
```
    public void SetStart((double x, double y) towards)
    {
        double differenceX = towards.x - From!.Cx;
        double differenceY = towards.y - From!.Cy;
        double distance = Math.Sqrt((differenceX * differenceX) + (differenceY * differenceY));

        if (distance > 0)
        {
            X1 = From!.Cx + (differenceX / distance * 50);
            Y1 = From!.Cy + (differenceY / distance * 50);
        }
    }
```
We first calculate the difference between the point we are drawing toward and the center of the `Node` we draw from for the `x` and `y` axes, respectively. Then we calculate the Euclidean distance between the two same points using the Pythagorean theorem. If the cursor is any distance from the `From` `Node`, we update the start of the line. We wish to start the line at the edge of the `From` `Node` circle and find the point closest to the cursor. We already have a vector that points to the cursor `(differenceX, differenceY)`, so we start with normalizing it by dividing it by its length, which is our `distance`, and then we multiply it with `50` to make that vector point out to the edge of the `Node` as that has a radius of `50`. Then we add this re-scaled vector to the center of the `From` `Node`.

Before continuing, let's demo this to see that it updates the line to point to the edge of the `From` `Node`. To show this, I've added a `@ref` to the `SVGEditor` in our sample page, programmatically initialized a new `Node`, and started the addition of a `Connector` like just as a temporary change.
```csharp
private SVGEditor SVGEditor { get; set; } = default!;

protected override void OnAfterRender(bool firstRender)
{
    if (!firstRender) return;

    Node.AddNew(SVGEditor);
    var node = (Node)SVGEditor.Elements.First();
    Connector.AddNew(SVGEditor, node);
}
```
<video width="600px" src="{site}/videos/svg-editor-connector-going-from-node.mp4" autoplay muted controls loop></video>

Next, we need to be able to select a `Node` while adding a `Connector` to end the interaction. `Shape`s cannot be selected when other elements are being created by default, so we first need to override this inherited logic of the `NodeEditor`. We add the following to the code section of our `NodeEditor` component:
```csharp
public new async Task SelectAsync(MouseEventArgs eventArgs)
{
    if (SVGElement.SVG.EditMode is EditMode.Add && SVGElement.SVG.SelectedShapes.Any(s => s is Connector))
    {
        SVGElement.SVG.SelectedShapes.Add(SVGElement);
    }
    else
    {
        await base.SelectAsync(eventArgs);
    }
}
```
We wish to change the behavior of the `SelectAsync` method which is invoked when the pointer is pressed down on a `Node`, so we `new` the `SelectAsync` method in the component. In this, we have two cases. If the `SVGEditor` is in `Add` mode and any `Connector` is currently selected, then we add the `Node` to the list of selected shapes. Else we call the base implementation of the `SelectAsync` method.

Now we can select a `Node` while a `Connector` is being added. Then we need to handle when the pointer is being raised while the `Connecter` is added. We override the `HandlePointerUp` on the `Connector` class for this.
```
public override void HandlePointerUp(PointerEventArgs eventArgs)
{
    if (SVG.EditMode is EditMode.Add
        && SVG.SelectedShapes.FirstOrDefault(s => s is Node node && node != From) is Node { } to)
    {
        if (to.RelatedConnectors.Any(c => c.To == From || c.From == From))
        {
            Complete();
        }
        else
        {
            To = to;
            SVG.EditMode = EditMode.None;
            UpdateLine();
        }
    }
}
```
We only care about the case where we are adding a `Connector` so we first check for this and then check if we can find any selected shape that is a `Node`. If we have matched this, we then do one of two things. If the two nodes that are going to be connected already are connected, then we call `Complete`, which we will implement to remove this unfinished `Connector` in a bit. We do this so that two nodes can't be connected more than once. If there wasn't already a `Connection` present between these two nodes then we see the `To` `Node`, reset the `SVGEditor` mode, and update its placement.

Before going to the `UpdateLine` method, let's see the `Complete` method we used.
```csharp
public override void Complete()
{
    if (To is null)
    {
        SVG.RemoveElement(this);
        Changed?.Invoke(this);
    }
}
```
This method can also be called from the context menu to enable the user to cancel creating a new `Connector` after starting. So in these two cases, we remove the temporary `Connector` if `To` is not set.

Now let's see the `UpdateLine` method that we have referenced a couple of times now.

```csharp
public void UpdateLine()
{
    if (From is null || To is null)
    {
        (X1, Y1) = (X2, Y2);
        return;
    }

    double differenceX = To.Cx - From.Cx;
    double differenceY = To.Cy - From.Cy;
    double distance = Math.Sqrt((differenceX * differenceX) + (differenceY * differenceY));

    if (distance < 100)
    {
        (X1, Y1) = (X2, Y2);
    }
    else
    {
        SetStart((To.Cx, To.Cy));
        X2 = To.Cx - (differenceX / distance * 50);
        Y2 = To.Cy - (differenceY / distance * 50);
    }
}
```
If the `From` or `To` `Node` is not set, we set the start of the line equal to the end position so that it will not appear.

Then we do something very close to what we have seen previously by finding the vector from the `To` `Node` to the `From` `Node`. Then we calculate the length of this vector. And if the length of the vector is less than `100`, then we know that they are so close that the `Connector` should not be rendered, so we once again set the start equal to the end. Else we set the start position to point towards the `To` `Node` and then calculate the point on the edge of the circle of the `To` `Node` closest to the `From` `Node` exactly like we did in the `SetStart` method for the `From` `Node`.

#### ConnectorEditor.razor
We also need to add the custom `ConnectorEditor` we previously referenced in the `Connector`. We have just copied the `LineEditor` component and made some adjustments for this.
```razor
@using BlazorContextMenu
@using KristofferStrube.Blazor.SVGEditor.Extensions;
@using KristofferStrube.Blazor.SVGEditor.ShapeEditors;
@inherits ShapeEditor<Connector>

<ContextMenuTrigger MenuId="SVGMenu" WrapperTag="g" Data=@SVGElement MouseButtonTrigger="SVGElement.ShouldTriggerContextMenu ? MouseButtonTrigger.Right : (MouseButtonTrigger)4">
    <g style="@(SVGElement.SVG.EditMode is EditMode.Add ? "pointer-events:none;" : "")" transform="translate(@SVGElement.SVG.Translate.x.AsString() @SVGElement.SVG.Translate.y.AsString()) scale(@SVGElement.SVG.Scale.AsString())">
        <line @ref=ElementReference
        @onfocusin="FocusElement"
        @onfocusout="UnfocusElement"
        @onpointerdown="SelectAsync"
        @onkeyup="KeyUp"
              tabindex="@(SVGElement.IsChildElement ? -1 : 0)"
              x1=@SVGElement.X1.AsString()
              y1=@SVGElement.Y1.AsString()
              x2=@SVGElement.X2.AsString()
              y2=@SVGElement.Y2.AsString()
              stroke="@SVGElement.Stroke"
              stroke-width="@SVGElement.StrokeWidth"
              stroke-linecap="@SVGElement.StrokeLinecap.AsString()"
              stroke-linejoin="@SVGElement.StrokeLinejoin.AsString()"
              stroke-dasharray="@SVGElement.StrokeDasharray"
              stroke-dashoffset="@SVGElement.StrokeDashoffset.AsString()"
              fill="@SVGElement.Fill">
        </line>
    </g>
</ContextMenuTrigger>
```
We have changed two things for the markup part of the `ConnectorEditor`. We have added a `style` tag to the `<g>` that encloses the `<line>` itself. The style tag makes it so that it will not handle any pointer events if the editor is currently in the `Add` mode. This is to ensure that we can click through the line when we are adding a new `Connector` so that it doesn't block any `Node` we would like to click. Secondly, we have removed the two anchors that normally appear when a `Line` is selected to make the UI less noisy. We have also overridden the `OnAfterRenderAsync` method in the code section of the component. 
```csharp
protected override async Task OnAfterRenderAsync(bool firstRender)
{
    if (firstRender)
    {
        SVGElement.UpdateLine();
    }
    await base.OnAfterRenderAsync(firstRender);
}
```
This ensures that all connectors get their position updates when first rendered.

#### AddNewConnectorMenuItem.razor
The last component we need is the context menu item for when we want to add a new `Connector`.
```razor
@using BlazorContextMenu

@if (Data is Node node)
{
    <Item OnClick="_ => Connector.AddNew(SVGEditor, node)">
        <div class="icon">〰</div> New Connector
    </Item>
}

@code {
    [CascadingParameter]
    public required SVGEditor SVGEditor { get; set; }

    [Parameter]
    public required object Data { get; set; }
}
```
This menu item is a bit more complex than the one we made for adding a `Node` as we only want it to render if we opened the context menu from a `Node`. So we check the `Data` property to get the `Node` that was right-clicked if there was any, and parse that to the `AddNew` method if the menu item is clicked.

Lastly we need to update the configuration in our sample page to include these new handlers and components. We first add the `Connector` class to the list of `SupportedElements` and specify that it can handle any `<line>` tag with a `data-elementtype` attribute equal to `"connector"`.
```csharp
protected List<SupportedElement> SupportedElements { get; set; } = new()
{
    new(typeof(Node), element => element.TagName is "CIRCLE" && element.GetAttribute("data-elementtype") == "node"),
    new(typeof(Connector), element => element.TagName is "LINE" && element.GetAttribute("data-elementtype") == "connector"),
};
```
And then, we add the menu item for adding new connectors and define that it should only count as a menu item if the context menu started from a `Node` as we defined in the component itself.

```csharp
protected List<SupportedAddNewSVGElementMenuItem> AddNewSVGElementMenuItems { get; set; } = new()
{
    new(typeof(AddNewNodeMenuItem), (_,_) => true),
    new(typeof(AddNewConnectorMenuItem), (_,data) => data is Node)
};
```

Now let's get to the final demo with all the parts working together.

<video width="600px" src="{site}/videos/svg-editor-graph-editor.mp4" autoplay muted controls loop></video>

You might have noticed that the area select tool also had a new orange color in this sample video. We changed this by adding the following scoped CSS styling to our sample page.
```css
::deep .anchor-primary {
    stroke: orange;
}

::deep .box {
    fill: orange;
    fill-opacity: 0.5;
    stroke: orange;
}
```
You can try the demo shown in the video yourself here: [kristofferstrube.github.io/Blazor.SVGEditor/CustomElements](https://kristofferstrube.github.io/Blazor.SVGEditor/CustomElements/)
And you can see the source code for the whole sample project, including some other demo pages, here: [Demo project source code](https://github.com/KristofferStrube/Blazor.SVGEditor/tree/120101d2fce7f0f561697045f82133501836d823/samples/KristofferStrube.Blazor.SVGEditor.WasmExample)

### Future work
I will continue to develop on the project whenever I find some feature or nice interaction that I would like to add. I currently have three open issues if you would to contribute yourself: <a href="https://github.com/KristofferStrube/Blazor.SVGEditor/issues/5" target="_blank">#5</a>, <a href="https://github.com/KristofferStrube/Blazor.SVGEditor/issues/11" target="_blank">#11</a>, and  <a href="https://github.com/KristofferStrube/Blazor.SVGEditor/issues/12" target="_blank">#12</a>. One project that I plan to use the package in already is a demo for my [Web Audio Blazor wrapper](https://github.com/KristofferStrube/Blazor.WebAudio). The demo will be a network of sound sources, processors, and outputs that can be wired together to make some simple audio editing from the browser using Blazor.

### Conclusion
At the start of the article, we went through all the different features of the Blazor.SVGEditor package. Features include parsing SVG definitions, adding, selecting, editing, moving, removing shapes, and navigating the canvas using the mouse. Then we went through a sample project where we added some custom handlers for SVG elements to the editor and made a network editor with these. While developing this customization, we also saw how we could still use many of the features in the package by inheriting from existing shapes. If you have any feedback or questions relating to the article, feel free to contact me on any of my social media profiles.