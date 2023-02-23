<script>
    hljs.highlightAll();
</script>
### Members from before ASP.NET Core 7
First, let's see what actions were possible before ASP.NET Core 7. I will give a shallow overview of the commonly used properties and methods of the `NavigationManager`.
#### string BaseUri
You can use the `BaseUri` property to get or set the base path of where the application is being presented. This is equivalent to the path that the <base /> tag in the head of the HTML document points to, but the path is actually resolved. This means that you get the string `"https://example.com/"` if your page is hosted there even though the `<base />` tag might only specify `"/"` as the base href.
#### string Uri
This is equivalent to the `window.location` property in JavaScript which we can use to get and set the absolute URI of the page.
#### void NavigateTo(...)
This method is also used to change the location to a specific URI, but this also has multiple useful overloads.
##### void NavigateTo(string uri, bool forceLoad)
The `forceLoad` parameter defines whether the navigation should simulate an HTTP reload or simply should change to the location without reloading.
##### void NavigateTo(string uri, bool forceLoad = false, bool replace = false)
The `replace` parameter defines whether the navigation should replace the existing part of the history. This means that you will lose your navigation history i.e. which pages you can navigate to with the back and forward buttons or gestures in the browser if you set it to true which can be desirable in certain scenarios like stateful MVVM-inspired Blazor applications.
##### void NavigateTo(string uri, NavigationOptions options)
The `options` parameter is simply an option that contains the two properties from the previous constructors as a contained class so that the same options can easily be used in multiple places.
#### string ToAbsoluteUri(String relativeUri)
Converts a relative URI to an absolute path in relation to the current path i.e. if you are at `"https://example.com/thepage"` and parses `"/subpath"` to it then you get `https://example.com/thepage/subpath`.
#### string ToBaseRelativePath(String uri)
Converts an absolute URI to a relative in relation path to the base path i.e. if your base path is `"https://example.com/mysite"` and you parse `"https://example.com/mysite/and/my/many/subpages"` to it then you get `"/and/my/many/subpages"`.
#### EventHandler<LocationChangedEventArgs> LocationChanged
You can listen to this event which will trigger once a location <u>has changed</u>. I emphasize that as it means you can't act on the change itself happening. From the event argument, you can see the `Location` that the navigation was to and whether this location change was _intercepted_ (`IsNavigationIntercepted`). _Intercepted_ means that some mechanism has changed the navigation. This is what Blazor does when you press a link on the page to navigate to a specific route instead of loading that page i.e. changing a cross-document navigation to a same-document one.

### Members introduced in ASP.NET Core 7
The following member were the ones added in ASP.NET Core 7 in order to enable more stateful navigation.
#### void RegisterLocationChangingHandler(...)
This method is (unlike `LocationChanged`) used to handle when the location <u>is changing</u>. This means that we can actually act on it before potentially navigating/leaving a page. The method takes a `Func` from a `LocationChangingContext` to a `ValueTask`. This means that we can act on the context and do some async work. An example could be the following:
```csharp
NavigationManager.RegisterLocationChangingHandler(async (context) =>
    {
        Console.WriteLine($"We are navigating to: {context.TargetLocation}");
        await AnimatedTransitionAsync();
    });
```
A nice thing we can do with the context is to stop the navigation using the `context.PreventNavigation()` method. This has been a very requested feature. It enables us to support canceling navigation to another page if the user has an unsubmitted form or some state that needs to be saved before leaving the page.
#### string HistoryEntryState
Another nice addition from ASP.NET Core 7 is the `HistoryEntryState` property. This has been added to many of the types that we have touched on above and enables us to send some state with our navigation without appending a query string (`?somekey=somevalue`) or a URI fragment (`#someValue`). The property has been added to the `NavigationManager` itself, but also to the `NavigationOptions`, `LocationChangingContext`, and `LocationChangedEventArgs` types. Setting the `HistoryEntryState` on the `NavigationOptions` enables us to append some information that can be acted on by reading the event argument or context in one of the two handlers. The state corresponds to the state that can also be parsed to the `navigate` method from the browser Navigation API where the state can be any serializable value. So that we can only parse a string through the `HistoryEntryState` is a simplification. But it can be justified as we can just serialize what state we need to parse as JSON or similar and then deserialize it again once read, essentially mimicking what the Navigation API does internally.

### The `NavigationManager` fitting into the Blazor ecosystem
When I inspect a _new_ feature or actually _any_ feature in Blazor I look at three key parameters for how well I think it fits with the values of Blazor: **Simplicity**, **Parity**, and **Familiarity**.
- **Simplicity** is when we make a simple abstraction over something complex to make the life of the developers easier.
- **Parity** is when we have the same functionality as other web frameworks (or more).
- **Familiarity** is when we have taken known concepts from the .NET ecosystem and made our solution match those.

If you have all three then you are lucky; If you have two then you have made a deliberate design decision; and if you have one then it is either too simple or you are religious.

_This is of cause only a very opinionated analysis._
#### Simplicity
Simplicity is one of the key reasons why someone new to Blazor would actually use a feature as it hides away overly complex functionality. This is done in some parts of the `NavigationManager`. One obvious one is the one we have mentioned above related to the `HistoryEntryState` that it is simply a `string` instead of being some complex object that is annotated as serializable. Another one that is very common in Blazor is the `LocationChangedEventArgs` which is simply a Data Transfer Object without any interop functionality. This is what is done for all event arguments across Blazor like `MouseEventArgs` or `ClipboardEventArgs`. This makes it very simple to work with, but advanced users might be missing more extensibility options.
#### Parity
The NavigationManager is obviously meant to be analogous to the browser Navigation API so it makes sense to compare it to the features in that. Below here we see the WebIDL definition of the Navigation interaface from the browser Navigation API.
```webidl
interface Navigation : EventTarget {
  sequence<NavigationHistoryEntry> entries();
  readonly attribute NavigationHistoryEntry? currentEntry;
  undefined updateCurrentEntry(NavigationUpdateCurrentEntryOptions options);
  readonly attribute NavigationTransition? transition;

  readonly attribute boolean canGoBack;
  readonly attribute boolean canGoForward;

  NavigationResult navigate(USVString url, optional NavigationNavigateOptions options = {});
  NavigationResult reload(optional NavigationReloadOptions options = {});

  NavigationResult traverseTo(DOMString key, optional NavigationOptions options = {});
  NavigationResult back(optional NavigationOptions options = {});
  NavigationResult forward(optional NavigationOptions options = {});

  attribute EventHandler onnavigate;
  attribute EventHandler onnavigatesuccess;
  attribute EventHandler onnavigateerror;
  attribute EventHandler oncurrententrychange;
};
```
We obviously see some parallels to the `NavigationManager` like the `currentEntry`, `navigate`, `onnavigatesuccess`, and `oncurrententrychange` members. We also see that there are multiple methods that Blazor has encapsulated by parameterizing the `NavigateTo` method i.e. `updateCurrentEntry`, `navigate`, and `reload`.

But we also see parts that are missing. We have two attributes `canGoBack` and `canGoForward` which would be easy to add like we have access to `currentEntry`, but they really only make sense to have if we were also able to call `back(...)` and `forward(...)`. Apart from this, we are also missing the `entries()` and `traverseTo(...)` methods that give access to the whole history of navigation in this session and enable us to navigate easily to a specific entry. This is only **one** part of the whole Navigation API so there are obviously more parts of the API that could have been accessible somehow, be that with direct wrappers or indirect abstractions like `NavigateTo` does.
#### Familiarity
For many, the key selling point of Blazor is that .NET backend developers can now also develop interactive websites using their knowledge of the .NET ecosystem. For this reason, it makes sense to use concepts from other .NET frameworks when appropriate. If we should relate the `NavigationManager` to some mechanism from another .NET framework then I would probably relate it to the `WebView2` control which is available in WPF and WinForms applications. This has a lot more methods for navigation and properties giving insight into the current state of the page like `GoBack()`, `GoForward()`, `NavigateToString(...)`, `CanGoBack`, `ZoomFactor`, and many more! As you can see WebView2 is a lot more featureful, but it is also a wrapper of an actual whole browsing experience so the comparison might not be all that fair as it also has access to things the `NavigationManager` couldn't possibly access in its context.

The naming of the methods or the properties isn't really consistent between the `NavigationManager` and the `WebView2` control. But a part that is consistent is the usage of `EventHandler`s for `LocationChanged`/`NavigationCompleted`. What doesn't really fit into the style is that `WebView2` also uses an `EventHander` for when a page is going to change called `NavigationStarting` whereas Blazor uses a `Func` that is evaluated before navigation by parsing it to the `RegisterLocationChangingHandler` as we have seen previously. I'm not quite sure why this distinguishment was made, but my guess would be that this is friendlier to async work whereas `event`s inherently has problems when used with async work.

#### Then, how did it do?
It did pretty well on the **simplicity** keeping things easily accessible and encapsulating multiple methods functionality in the `NavigateTo` method. It did okay in **parity** having methods and properties that are clearly mappable to the `Navigation` interface from the Navigation API. But there is still missing some functionality. But don't be discouraged we might get these in a future release of ASP.NET if people request this. The **familiarity** part could need some love and I would especially have liked for all events to use `EventHandler`'s and maybe even supply attribute-specific methods as well parallel to `addEventListener` and `removeEventListener` from the `EventTarget` interface which the `Navigation` interface extends in the browser specifications.

So if we were to put it somewhere on the scale that I presented earlier then I would say that the `NavigationManager` fulfills somewhere between 1 and 2 of my key parameters for fitting into Blazors values. This puts it somewhere between being too simple and having made deliberate design considerations (which is really good!). And if we would just have had a few more of the obvious missing functionality like `Back()` and `Forward()` navigation then I would definitely have leaned more towards it fulfilling parity as well. The additions from ASP.NET Core 7 have already pushed us towards this, so we are getting there.

### Scenario enabled using ASP.NET Core 7
I will now show a scenario that is enabled by the additions that came with ASP.NET Core 7. The scenario is "Canceling navigation to prevent loss of data."
In this scenario, we will make a simple form that can be submitted to an API. We want to display a pop-up if you try to navigate away from the form while there is text in it giving you the option to cancel the navigation. After all the code I have made a small video that shows the scenario in practice.

#### Modal Component
Let's first make a simple modal component that can work as a pop-up that forces the user to make a decision.
##### Shared/Modal.razor
```cshtml-razor
@if (!_isShown) return;

<div class="modal-container" @onclick="Close">
    <div class="modal-content" @onclick:stopPropagation=true>
        <h3>
            @Title
            <span @onclick="Close" class="close">X</span>
        </h3>
        <p>@ChildContent</p>
        <button @onclick=Accept>Accept</button>
    </div>
</div>
```
And then the associated code-behind which defines some simple logic for changing whether the modal is visible.
##### Shared/Modal.razor.cs
```csharp
public partial class Modal
{
    private bool _isShown { get; set; }
    private Action _accept { get; set; } = default!;

    [Parameter]
    public string Title { get; set; } = "";
    [Parameter]
    public RenderFragment? ChildContent { get; set; }

    public void Show(Action accept)
    {
        _accept = accept;
        _isShown = true;
        StateHasChanged();
    }

    private void Close()
    {
        _isShown = false;
    }
    private void Accept()
    {
        _accept();
        _isShown = false;
    }
}
```
And let's use a little styling to make the modal _"pretty"_.
##### Shared/Modal.razor.css
```css
.modal-container {
    position: fixed;
    z-index: 1;
    left: 0;
    top: 0;
    width: 100vw;
    height: 100vh;
    background-color: rgba(0, 0, 0, 0.2);
    display: flex;
    justify-content: center;
    align-items: center;
}

.modal-content {
    width: 600px;
    max-width: 80vw;
    padding: 30px;
    background-color: white;
    border-radius: 10px;
}

.close {
    float: right;
}
    .close:hover {
        font-weight: bold;
        cursor: pointer;
    }
```
#### Index page
Next, we will make the markdown for our actual page which will be pretty simple because we encapsulated the modal behavior in its own component.
##### index.razor
```cshtml-razor
@page "/"

<Modal @ref=PreventModal Title="You are about to leave">
    You have unsaved data,
    which you will lose if you navigate away from this page.
    Decline to save your data before leaving.
</Modal>

<label for="name">Fill in your name</label>
<input id="name" @bind-value=@name></input>

<button @onclick=Save>Save!</button>
```
And then the code-behind for the index page. The interesting part is in the `OnInitialized` method. It observes all navigation.
1. We start off by ignoring cases where the `name` field is empty or the parsed `HistoryEntryState` had been set to `"leave"`.
2. Then we show the `Modal` and if they accept that they will leave then we clear the name and navigate to the original `TargetLocation` and `HistoryEntryState` set to `"leave"`.
3. Finally, we prevent the navigation as we want to stay on the site until the user decides.

##### index.razor.cs
```csharp
public partial class Index
{
    private string name = "";
    private Modal? PreventModal;

    [Inject]
    public NavigationManager NavigationManager { get; set; } = default!;

    protected override void OnInitialized()
    {
        NavigationManager.RegisterLocationChangingHandler((context) =>
        {
            if (string.IsNullOrWhiteSpace(name) || context.HistoryEntryState is "leave")
            {
                return ValueTask.CompletedTask;
            }

            PreventModal!.Show(accept: () =>
            {
                name = "";
                NavigationManager.NavigateTo(
                    context.TargetLocation,
                    new NavigationOptions() { HistoryEntryState = "leave" }
                );
            });

            context.PreventNavigation();
            return ValueTask.CompletedTask;
        });
    }

    private void Save()
    {
        Console.WriteLine("We have saved!");
        name = "";
    }
}
```
Then we are done with the code part and can enjoy our result below.

#### Demo Video
<div class="center">
<video width="700" autoplay muted controls loop>
<source src="{site}/videos/navigation-manager.mp4" type="video/mp4">
A video showing a demo of the code written above.
</video>
</div>

### Conclusion
In this article, we have looked at what features the `NavigationManager` had before ASP.NET Core 7. We have looked at what was introduced in that version. We have discussed the qualities of the `NavigationManager` both the good and bad. And in the end, we have presented a small demo scenario with code that showcases some of the new features.

Feel free to reach out if you have any questions or comments to the post. And remember that parts of this is my subjective opinion.