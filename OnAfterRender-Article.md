# Blazor's OnAfterRender

`OnAfterRender{Async}` is often used, along with calls to `StateHasChanged`, to try and fix rendering problems in Blazor components.  Code that starts in `OnInitialized{Async}/OnParametersSet{Async}` migrates into `OnAfterRender{Async}` to make it work, often with a call to `StateHasChanged` at the end to get the component UI to reflect the component state.  It's used as if it's part of the component "lifecycle" methods.

It isn't: this article aims to enlighten you on what it really is, and how you should be using it in component design.

## Repository

The code is in the [Blazr.OnAfterRender.Demo Project](https://github.com/ShaunCurtis/Blazr.OnAfterRender/tree/master/Blazr.OnAfterRender.Demo) in the [Blazr.OnAfterRender Repository](https://github.com/ShaunCurtis/Blazr.OnAfterRender)

## Definitions

I use the term *Component Lifecycle* as described in this [Microsoft Learn Article](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/lifecycle).  Specifically those methods called by `SetParametersAsync` - `OnInitialized{Async}/OnParametersSet{Async}`.

## StateHasChanged

`StateHasChanged` doesn't render the component.  It simply places a `RenderFragment` on the Renderer's queue.  There is no direct execution linkage between calling `StateHasChanged` and the renderer running the `RenderFragment` placed on the queue.  The queue is serviced by a totally separate process running on the `Synchronisation Context`.  If you execute a block of synchronous code with a call to `StateHasChanged` in the middle, the component can't render until after that synchronous block completes. 

## So What is OnAfterRender?

Here's the implementation in `ComponentBase`.

```csharp
    Task IHandleAfterRender.OnAfterRenderAsync()
    {
        var firstRender = !_hasCalledOnAfterRender;
        _hasCalledOnAfterRender = true;

        OnAfterRender(firstRender);

        return OnAfterRenderAsync(firstRender);

        // Note that we don't call StateHasChanged to trigger a render after
        // handling this, because that would be an infinite loop. The only
        // reason we have OnAfterRenderAsync is so that the developer doesn't
        // have to use "async void" and do their own exception handling in
        // the case where they want to start an async task.
    }
```

It's an interface implementation for `IHandleAfterRender`.

Look through the `ComponentBase` code: you will find no call to `OnAfterRenderAsync`, no direct linkage from `OnInitialized{Async}/OnParametersSet{Async}`.

So how does it get called?

When the current render event is complete, the renderer checks if any affected components implement `IHandleAfterRender`.  If they do, it calls `OnAfterRenderAsync`.

In essence, it's an event, raised by the Renderer at the completion of a render cycle on the component.  The lifecycle methods may trigger a render by calling `StateHasChanged`, but they have no control on when it executes.  This distinction is important, because many people code it as if the two are linked and sequential.

To demonstrate here's a simple page.  It contains a `Task.Delay` to mimic an async call to get some data.

```csharp
@page "/"
<h3>Test</h3>
@{
    Console.WriteLine("Rendered");
}

@code {
    protected override async Task OnInitializedAsync()
    {
        Console.WriteLine("OnInitializedAsync-1");
        await Task.Delay(1);
        Console.WriteLine("OnInitializedAsync-2");
    }

    protected override Task OnAfterRenderAsync(bool firstRender)
    {
        Console.WriteLine("OnAfterRenderAsync");
        return Task.CompletedTask;
    }
}
```

The output looks like this.  Note that everything completed before the two `OnAfterRenderAsync` events were called.

```text
OnInitializedAsync-1
Rendered
OnInitializedAsync-2
Rendered
OnAfterRenderAsync
OnAfterRenderAsync
```

Now if we make a simple change - make the delay 100ms instead of 1 emulating a slower data get - we get a different result.  Now the first `OnAfterRenderAsync` occurs before the continuation.

```csharp
OnInitializedAsync-1
Rendered
OnAfterRenderAsync
OnInitializedAsync-2
Rendered
OnAfterRenderAsync
```

The why is in how long processes take to run, task scheduling and prioritization: too complex a subject for this discussion.  But it happens, and there are logical reasons why it does so.

Any code run in `OnAfterRenderAsync` dependant on the data retrieved in `OnInitializedAsync` may or may not work, depending on the amount of processing time it takes to get it.

## My Rules

If you run lifecycle code in `OnAfterRender`, you are almost certainly running code that mutates the state of the component in an event that is fired AFTER the component has rendered.  You need to re-render to get those state changes reflected in the UI.  Not logical.

`OnAfterRender{Async}`:

1. Should only contain JsInterop code.
2. Should not mutate the state of the component.
3. Should never call `StateHasChanged`. 

To be blunt: Get your logic right and there are very few occasions where you need to use `OnAfterRender{Async}` at all.

You can even short-circuit it to save few CPU cycles.

```csharp
@page "/"
@implements IHandleAfterRender

//...

@code {
//...

    Task IHandleAfterRender.OnAfterRenderAsync()
        => Task.CompletedTask;
}
```

## An Example

This example is based on this Microsoft Learn article that decribes how to stream image data into a page.

https://learn.microsoft.com/en-us/aspnet/core/blazor/images?view=aspnetcore-7.0#stream-image-data

It has both C# logic and JsInterop logic.

### Image Component

First the markup and UI logic.

1. `_loading` provides the display control.  Note we only hide `img`: we can still interact with it in JsInterop.
2. `_uid` provides a unique `id` field for `img`.

```csharp
@inject IJSRuntime JS
@inject IHttpClientFactory ClientFactory

<img hidden="@_loading" id="@_uid" />
@if (_loading)
{
    <span>Loading...</span>
}

@{
    Console.WriteLine($"{_uid} => Rendered");
}

@code {
    [Parameter] public string? Url { get; set; }

    private bool _loading;
    private string _uid = Guid.NewGuid().ToString();
```

Next we need to stream the image.  The Url may change, so this code runs in `OnParametersSetAsync` to reload the image if necessary.    `_imageToRender` is set to true when the process completes.

```csharp

    private bool _imageToRender;
    private Stream? _stream;
    private string? _currentUrl;

    protected async override Task OnParametersSetAsync()
    {
        // Detect any Url changes and update on change
        Console.WriteLine($"{_uid} => OnParametersSetAsync");
        if (_currentUrl != this.Url)
        {
            Console.WriteLine($"{_uid} => OnParametersSetAsync - handling new Url");
            _loading = true;
            using var http = this.ClientFactory.CreateClient();
            _stream = await http.GetStreamAsync(this.Url);
            // First remder happens here on the async yield
            _loading = false;
            _currentUrl = this.Url;
            _imageToRender = true;
            Console.WriteLine($"{_uid} => OnParametersSetAsync - Streaming complete");
            // second render happens on completion
        }
    }
```

`OnAfterRenderAsync` contains the JsInterop code.  It does break my Rule 2 - *Should not mutate the state of the component*.  However, it only mutates component data not linked to rendering: no need for `StateHasChanged`.  *There are always exceptions: when you break rules, understand why*.

```csharp
    protected async override Task OnAfterRenderAsync(bool firstRender)
    {
        Console.WriteLine($"{_uid} => OnAfterRenderAsync");

        // Only update the image if we've had a change and a completed stream to use
        if (_imageToRender && _stream is not null)
        {
            Console.WriteLine($"{_uid} => OnAfterRenderAsync - Handle Image");
            var streamRef = new DotNetStreamReference(_stream);
            await JS.InvokeVoidAsync("setImage", _uid, streamRef);
            // clean up and set flags
            _stream.Dispose();
            streamRef.Dispose();
            streamRef = null;
            _stream = null;
            _imageToRender = false;
        }
    }
```

The demo page loads four images.  It also shows the `OnAfterRenderAsync` short circuit in action.

```csharp
@page "/"
@implements IHandleAfterRender

<PageTitle>Index</PageTitle>

<h1>Hello, Images!</h1>

@foreach (var imageSrc in ImageSources)
{
    <Image Url="@imageSrc" />
}

@code {
    private List<string> ImageSources = new(){
        "https://picsum.photos/200/300",
        "https://picsum.photos/200/300",
        "https://picsum.photos/200/300",
        "https://picsum.photos/200/300"
    };

    Task IHandleAfterRender.OnAfterRenderAsync()
        => Task.CompletedTask;
}
```

The output from one of the image components.

```text
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnParametersSetAsync
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnParametersSetAsync - handling new Url
// => Note 1
27daeb9f-653f-4a54-a528-97e5a7ea37cc => Rendered
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnAfterRenderAsync
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnParametersSetAsync - Streaming complete
27daeb9f-653f-4a54-a528-97e5a7ea37cc => Rendered
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnAfterRenderAsync
27daeb9f-653f-4a54-a528-97e5a7ea37cc => OnAfterRenderAsync - Handle Image
```

1. At 1, `OnParametersSetAsync` yields as the image is retrieved asynchronously, and it calls `StateHasChanged`.  
2. The code beyond `await http.GetStreamAsync(this.Url);` is scheduled as a continuation.
3. The Renderer gets thread time to run the render [and in this case OnAfterRenderAsync].
4. Once the image download completes, the continuation completes.  
5. The final [ComponentBase] lifecycle call to `StateHasChanged` is made.  
6. The Renderer has thread time and completes the render. `OnAfterRenderAsync` handles the image.   

## Summing up

Hopefully I've demonstrated why you should rarely need to use `OnAfterRender` in components.  I don't like defining bad practices: but I consider my rules good practice.  I have seen comments from other experienced Blazor programmers who say that they rarely use `OnAfterRender`.  I use it in less than 1% of the components I write.

I don't use `ComponentBase`.  My base component doesn't even implement `IHandleAfterRender`  I implement it where I need it.  

Where to run your code:

1. Synchronous logic that needs to complete before any rendering takes place should be in either `OnInitialized` if you only run it at initialization, or `OnParametersSet` if it needs to react to parameter changes.
 
2. Asynchronous logic that needs to complete before any rendering takes place should to be run in `SetParametersAsync`.  The pattern to use is this:
 
```csharp
public override async Task SetParametersAsync(ParameterView parameters)
{
    parameters.SetParameterProperties(this);

    // Your logic

    await base.SetParametersAsync(ParameterView.Empty);
}
```

## Appendix

Example service definition:

```csharp
// Add services to the container.
builder.Services.AddRazorPages();
builder.Services.AddServerSideBlazor();
builder.Services.AddSingleton<WeatherForecastService>();
builder.Services.AddHttpClient();

var app = builder.Build();
```

Host page:

```html
@page "/"
@using Microsoft.AspNetCore.Components.Web
@namespace Blazr.OnAfterRender.Demo.Pages
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <base href="~/" />
    <link rel="stylesheet" href="css/bootstrap/bootstrap.min.css" />
    <link href="css/site.css" rel="stylesheet" />
    <link href="Blazr.OnAfterRender.Demo.styles.css" rel="stylesheet" />
    <link rel="icon" type="image/png" href="favicon.png"/>
    <component type="typeof(HeadOutlet)" render-mode="ServerPrerendered" />
</head>
<body>
    <component type="typeof(App)" render-mode="ServerPrerendered" />

    <div id="blazor-error-ui">
        <environment include="Staging,Production">
            An error has occurred. This application may no longer respond until reloaded.
        </environment>
        <environment include="Development">
            An unhandled exception has occurred. See browser dev tools for details.
        </environment>
        <a href="" class="reload">Reload</a>
        <a class="dismiss">🗙</a>
    </div>

    <script src="_framework/blazor.server.js"></script>
    <script>
        window.setImage = async (imageElementId, imageStream) => {
            const arrayBuffer = await imageStream.arrayBuffer();
            const blob = new Blob([arrayBuffer]);
            const url = URL.createObjectURL(blob);
            const image = document.getElementById(imageElementId);
            image.onload = () => {
                URL.revokeObjectURL(url);
            }
            image.src = url;
        }
    </script>
</body>
</html>
```