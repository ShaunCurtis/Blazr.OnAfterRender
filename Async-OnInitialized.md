#  The Async OnInitialized

## Our First Page

This is a simple page to illustrate how the myth can be born.  You are making a database call to get some data and want to display *Loading* while it's happening.  You're *keeping it simple*, steering well clear of the *async* dark art.

What you code is this. 

What you get is a blank screen and then *Loaded*: no *Loading*.

```csharp
@page "/"

<PageTitle>The OnAfterRender Myth</PageTitle>

<h1>The OnAfterRender Myth</h1>

<div class="bg-dark text-white mt-5 m-2 p-2">
    <pre>@_state</pre>
</div>

@code {
    private string? _state = "New";

    protected override void OnInitialized()
    {
        _state = "Loading";
        TaskSync();
        _state = "Loaded";
    }

    // Emulate a synchronous blocking database operation
    private void TaskSync()
        => Thread.Sleep(1000);
}
```

### StateHasChanged

You go searching, find `StateHasChanged` and update your code.

But to no avail.

```caharp
    protected override void OnInitialized()
    {
        _state = "Loading";
        StateHasChanged();
        TaskSync();
        _state = "Loaded";
    }
```

### Task.Delay

You search further and find `await Task.Delay(1)`.  You start typing `await` and the Visual Studio editor adds an `async` to your method:

```csharp
    protected override async void OnInitialized()
```

You complete your change.

What you get is the opposite of what you had.  *Loading*, but no completion to `Loaded`.

```csharp
    protected override async void OnInitialized()
    {
        _state = "Loading";
        StateHasChanged();
        await Task.Delay(1);
        TaskSync();
        _state = "Loaded";
    }
```

### OnAfterRender

More searching reveals `OnAfterRender`.  You add it to your code.

Great, it now works as expected.

```csharp
    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
            StateHasChanged();
    }
```

You move on thinking *problem solved* and start to use the pattern elsewhere.   The myth is perpetuated, you've acquired some voodoo magic (which you may propogate), and learned nothing. 

### The Obvious Answer

The answer to the problem above is obvious to a more experienced coder: use `OnInitializedAsync` and async database operations.  But to the person above, that's days or weeks down the road.  Meanwhile, they've learned a "dirty" anti pattern to make it work for them.

## Delving into The Process

To delving into the process we need to add some debug statements to the code.   Breakpoints won't tell the true story: Blazor processes are asychronous.

```csharp
@page "/"
@using System.Diagnostics;

<PageTitle>Index</PageTitle>

@{
    Debug.WriteLine($"Render Component {_componentUid}.");
}

<h1>The OnAfterRender Myth</h1>

<div class="bg-dark text-white mt-5 m-2 p-2">
    <pre>@_state</pre>
</div>

@code {
    private string? _state = "New";
    private Guid _componentUid = Guid.NewGuid();

    protected override async void OnInitialized()
    {
        _state = "Loading";
        StateHasChanged();
        await Task.Delay(100);
        TaskSync();
        _state = "Loaded";
        Debug.WriteLine($"OnInitialized on Component {_componentUid}.");
    }

    protected override Task OnInitializedAsync()
    {
        Debug.WriteLine($"OnInitializedAsync on Component {_componentUid}.");
        return Task.CompletedTask;
    }

    protected override void OnParametersSet()
    {
        Debug.WriteLine($"OnParametersSet on Component {_componentUid}.");
    }

    protected override Task OnParametersSetAsync()
    {
        Debug.WriteLine($"OnParametersSetAsync on Component {_componentUid}.");
        return Task.CompletedTask;
    }

    protected override bool ShouldRender()
    {
        Debug.WriteLine($"ShouldRender on Component {_componentUid}.");
        return true;
    }


    private void TaskSync()
    => Thread.Sleep(1000);

    private async Task TaskAsync()
        => await Task.Yield();

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
        {
            Debug.WriteLine($"Fist OnAfterRender on Component {_componentUid}.");
            StateHasChanged();
        }
        else
            Debug.WriteLine($"Fist OnAfterRender on Component {_componentUid}.");
    }

    protected override Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
        {
            Debug.WriteLine($"Fist OnAfterRenderAsync on Component {_componentUid}.");
        }
        else
            Debug.WriteLine($"Fist OnAfterRenderAsync on Component {_componentUid}.");
        return Task.CompletedTask;
    }
}
```

What you get is this:

```text
OnInitialized Started on Component 88c36d17-a223-4cbd-b4b5-30f1738bd2c6.
```

We reach the `await Task.Delay(1);`.

`OnInitialized` yields back to the caller. As it returns a void there's nothing to await.  So it doesn't await the completion of `OnInitialized`, but continues on executing the lifecycle methods.  All the way to a componwnt render: which displays *Loading*.

```text
OnInitializedAsync Completed on Component 88c36d17-a223-4cbd-b4b5-30f1738bd2c6.
OnParametersSet Completed on Component 88c36d17-a223-4cbd-b4b5-30f1738bd2c6.
OnParametersSetAsync Completed on Component 88c36d17-a223-4cbd-b4b5-30f1738bd2c6.
Component Rendered 88c36d17-a223-4cbd-b4b5-30f1738bd2c6.
```

At this point we have a `OnInitialized` process totally out-of-sync with the main lifecycle.  The other lifecycle methods have completed.

At this poiunt there are two processes that need to run:

1. The completion of `OnInitialized` after a one millisecond delay.
2. The OnAfterRender events on the component.

The lifecycle methods normally take at least 1 millisecond, so the delay has completed, `OnInitialized` wins and continues loading the data and runs to completion.

```text
OnInitialized DataLoad Started on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
OnInitialized Completed on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
```

`OnAfterRender` starts and calls `StateHasChanged` and renders the component.

```text
First OnAfterRender Started on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
ShouldRender on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
Component Rendered 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
```

The first `OnAfterRender` sequence completes.

```text
First OnAfterRender Completed on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
First OnAfterRenderAsync Completed on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
```

Followed by the second round caused by the call to `StateHasChanged` in the first cycle 

```text
Subsequent OnAfterRender Completed on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
Subsequent OnAfterRenderAsync Completed on Component 95da5b4b-aa29-4bb7-983a-57a5a4f3de95.
```

If you extend the Task delay period out to say 100 milliseconds you get a different result. 


```text
OnInitialized Started on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
OnInitializedAsync Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
OnParametersSet Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
OnParametersSetAsync Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
Component Rendered 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
First OnAfterRender Started on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
ShouldRender on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
Component Rendered 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
First OnAfterRender Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
First OnAfterRenderAsync Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
Subsequent OnAfterRender Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
Subsequent OnAfterRenderAsync Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
```

The whole render process completes before `OnInitialized` completes and no *Loaded*.

```text
OnInitialized DataLoad Started on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
OnInitialized Completed on Component 7b6c57a2-ce89-4b73-bf3d-9a1940b2e4c8.
```
