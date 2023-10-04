# Blazor's OnAfterRender

`OnAfterRender{Async}` is an enigmatic method in `ComponentBase`.  Many people use it as if were one of the component "lifecycle" methods: called after "OnInitialized{Async}/OnParametersSet{Async}".  It isn't.

This article is to enlighten you on what it really is and how you should be using it in component design.

## What is it?

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

Note that it's an interface implementation for `IHandleAfterRender`.  A base component implementing `IComponent` doesn't need to implement it.

If you look through the `ComponentBase` code you will find no call to `OnAfterRenderAsync`.  There's no direct linkage between it and "OnInitialized{Async}/OnParametersSet{Async}".

So how does it get called?

When the Renderer detects that the component's render output has been rendered in the browser, it checks to see if the component implements `IHandleAfterRender`.  If it does it, then it calls `OnAfterRenderAsync`.

In essence, it's an event, raised by the Renderer at the completion of a render cycle on the component.  The lifecycle methods may ultimately trigger it by calling `StateHasChanged`, but they have no control on when it's executed.  This distinction is important, because many people code it as if the two are linked and sequential.

To demonstrate here's a simple page.  It contains a `Task.Delay` to mimic a quick async call to get some data.

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

Any code run in `OnAfterRenderAsync` that is dependant on the data retrieved in `OnInitializedAsync` may or may not work, depending on the amount of processing time it takes to get it.

## My Rules

`OnAfterRender{Async}`:

1. Should only contain JsInterop code.
2. Should not mutate the state of the component.
3. Should never call `StateHasChanged`. 

In almost all circumstances, logic that responds to `SetParametersAsync` belongs in `OnInitialized{Async}/OnParametersSet{Async}`.  If you structure your logic correctly it will fit.

## An Example

This example is based on this Microsoft article that decribes how to stream image data into a page.

https://learn.microsoft.com/en-us/aspnet/core/blazor/images?view=aspnetcore-7.0#stream-image-data

It has both C# logic and JsInterop logic.

### Image Component

As we are handling images, we encapsulate the logic into a component.


