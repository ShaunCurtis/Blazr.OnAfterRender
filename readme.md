# Blazor's OnAfterRender

This Repo demonstrates why you can't trust the execution order of the component lifecycle methods.

## Demo Component

This simple component demonstrates a common async process, such as loading a record or record set from a data source or API.  `Debug.WriteLine` statements throughout the code document the execution order. 

```csharp
@page "/"
@using System.Diagnostics;

<PageTitle>Index</PageTitle>

@{
    Debug.WriteLine($"Component Rendered {_componentUid}.");
}

<h1>The OnAfterRender Myth</h1>

<div class="bg-dark text-white mt-5 m-2 p-2">
    <pre>@_state</pre>
</div>

@code {
    private string? _state = "New";
    private Guid _componentUid = Guid.NewGuid();

    protected override async Task OnInitializedAsync()
    {
        Debug.WriteLine($"OnInitializedAsync Started on Component {_componentUid}.");
        _state = "Loading";
        Debug.WriteLine($"OnInitializedAsync DataLoad Started on Component {_componentUid}.");
        await TaskYieldAsync();
        //await TaskDelayAsync();
        _state = "Loaded";
        Debug.WriteLine($"OnInitializedAsync Completed on Component {_componentUid}.");
    }

    protected override void OnInitialized()
        => Debug.WriteLine($"OnInitialized Completed on Component {_componentUid}.");

    protected override void OnParametersSet()
    =>  Debug.WriteLine($"OnParametersSet Completed on Component {_componentUid}.");

    protected override Task OnParametersSetAsync()
    {
        Debug.WriteLine($"OnParametersSetAsync Completed on Component {_componentUid}.");
        return Task.CompletedTask;
    }

    protected override bool ShouldRender()
    {
        Debug.WriteLine($"ShouldRender on Component {_componentUid}.");
        return true;
    }

    private void TaskSync()
    => Thread.Sleep(1000);

    private async Task TaskYieldAsync()
        => await Task.Yield();

    private async Task TaskDelayAsync()
    => await Task.Delay(6);

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
            Debug.WriteLine($"First OnAfterRender Completed on Component {_componentUid}.");
        else
            Debug.WriteLine($"Subsequent OnAfterRender Completed on Component {_componentUid}.");
    }

    protected override Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            Debug.WriteLine($"First OnAfterRenderAsync Completed on Component {_componentUid}.");
        else
            Debug.WriteLine($"Subsequent OnAfterRenderAsync Completed on Component {_componentUid}.");

        return Task.CompletedTask;
    }
}
```

Run this code.

The lifecycle methods start to run as expected.

```text
OnInitialized Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
OnInitializedAsync Started on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
OnInitializedAsync DataLoad Started on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

At this point `TaskYieldAsync` yields to `OnInitializedAsync` which yields to the internal `ComponentBase` method `RunInitAndSetParametersAsync`. This in turn yields to the SC: the renderer renders the component.  

```text
Component Rendered e45633be-c5a1-451b-99a2-ce64f128a65c.
```

Once the render completes, it invokes the `OnAfterRender` on the SC.  The queue has two tasks:

1. The continuation from `OnInitializedAsync`.
2. An instance of `OnAfterRender` for the component.
 
The `OnInitializedAsync` continuation runs to completion [there are no further awaits]. 

```text
OnInitializedAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
OnParametersSet Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
OnParametersSetAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

The lifecycle methods run to completion and the internal `ComponentBase` method `CallOnParametersSetAsync` completes and makes the final call to `StateHasChanged`.  The SC prioritizes this over the `OnAfterRender` task and renders the component. 

```text
ShouldRender on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Component Rendered e45633be-c5a1-451b-99a2-ce64f128a65c.
```

The component renders and a second `OnAfterRender` is queued onto the SC. 

There are now two `OnAfterRender` tasks which run to completion.

```
First OnAfterRender Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
First OnAfterRenderAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Subsequent OnAfterRender Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Subsequent OnAfterRenderAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

## Task Delay Vs Task Yield

If we replace the `Task.Yield()` with a `Task.Delay(1)` do we get the same result?

Yes.  The sequence is shown below.

```text
OnInitialized Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
OnInitializedAsync Started on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
OnInitializedAsync DataLoad Started on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
Component Rendered 2103c005-cc57-497b-ab3d-0e2894fe173e.
OnInitializedAsync Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
OnParametersSet Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
OnParametersSetAsync Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
ShouldRender on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
Component Rendered 2103c005-cc57-497b-ab3d-0e2894fe173e.
First OnAfterRender Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
First OnAfterRenderAsync Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
Subsequent OnAfterRender Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
Subsequent OnAfterRenderAsync Completed on Component 2103c005-cc57-497b-ab3d-0e2894fe173e.
```

## A Longer Delay

What about a longer delay that emulates a slow database transaction or API call.

The result below shows the sequence with a 100ms delay. 

```text
OnInitialized Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
OnInitializedAsync Started on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
OnInitializedAsync DataLoad Started on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
Component Rendered dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
```
The same, but now a change.  

The `OnAfterRender` associated with the first render happens ahead of the `OnInitializedAsync` continuation because it's still being awaited - the timer hasn't expired [or in the real world, the database transaction hasn't yet returned a result].

```text
First OnAfterRender Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
First OnAfterRenderAsync Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
```
And then the continuation and second `OnAfterRender`.

```text
OnInitializedAsync Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
OnParametersSet Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
OnParametersSetAsync Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
ShouldRender on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
Component Rendered dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
Subsequent OnAfterRender Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
Subsequent OnAfterRenderAsync Completed on Component dea25bbc-7504-4a2c-a7ed-e4f2172c5c3b.
```

## Conclusions

The results strongly suggest that the time it takes the asynchronous processes to run is critical to when `OnAfterRender` is run.  If the process is complete when the SC completes the first render, the lifecycle methods will either run to the next yield, or completion before the first `OnAfterRender` executes.  If the process is still running, the initial `OnAfterRender` will execute immediately.  You can complicate the issue further by adding multiple awaits spread across the methods.

The changeover point with this code on my development system, today, with a westerly blowing at 14 knots was 7ms.  The sun was out which may or may not make a difference!

If you run code in `OnAfterRender{Async}`, there is no guarantee when it will run.

Consider any JS Interop operations.  When can you expect any JSinterop code in `OnAfterRender` to have executed?  You may think that because you've run a render in `OnInitalizedAsync`, fields/objects obtained through the JSInterop should be available immedaitely.  They may not be.





