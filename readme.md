#  The Blazor OnAfterRender Myth

I'll start this article with a bold, challenging [maybe even foolhardy] assertion:

> If your code in `OnAfterRender{Async}` isn't doing JS interop stuff, it's in the wrong place!


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

   