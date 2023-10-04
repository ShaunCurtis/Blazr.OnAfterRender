# Blazor's OnAfterRender Excution Order Issues

This Repo demonstrates some differences in the execution order of the component lifecycle methods.

## Demo Component

This simple component demonstrates a common async process, such as loading a record or record set from a data source or API.  `Debug.WriteLine` statements throughout the code document the execution order. 

```csharp
@page "/"
@using System.Diagnostics;

<PageTitle>Index</PageTitle>

@{
    Log($"Component Rendered.");
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
        Log($"OnInitializedAsync Started.");

        _state = "Loading";

        Log($"OnInitializedAsync DataLoad Started.");

        //await TaskYieldAsync();
        await TaskDelayAsync();

        _state = "Loaded";

        Log($"OnInitializedAsync Completed.");
    }

    protected override void OnInitialized()
        => Log($"OnInitialized Completed.");

    protected override void OnParametersSet()
        => Log($"OnParametersSet Completed.");

    protected override Task OnParametersSetAsync()
    {
        Log($"OnParametersSetAsync Completed.");
        return Task.CompletedTask;
    }

    protected override bool ShouldRender()
    {
        Log($"ShouldRender");
        return true;
    }

    private void TaskSync()
    => Thread.Sleep(1000);

    private async Task TaskYieldAsync()
        => await Task.Yield();

    private async Task TaskDelayAsync()
    => await Task.Delay(1);

    protected override void OnAfterRender(bool firstRender)
    {
        if (firstRender)
            Log($"First OnAfterRender Completed.");
        else
            Log($"Subsequent OnAfterRender Completed.");
    }

    protected override Task OnAfterRenderAsync(bool firstRender)
    {
        if (firstRender)
            Log($"First OnAfterRenderAsync Completed.");
        else
            Log($"Subsequent OnAfterRenderAsync Completed.");

        return Task.CompletedTask;
    }

    private void Log(string message)
    {
        Console.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} - {message} on Component {_componentUid}.");
        Debug.WriteLine($"Thread: {Thread.CurrentThread.ManagedThreadId} - {message} on Component {_componentUid}.");
    }
}
```

## Blazor Server

Run this code.

The lifecycle methods start to run as expected.

```text
Thread: 19 - OnInitialized Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 19 - OnInitializedAsync Started on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 19 - OnInitializedAsync DataLoad Started on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

At this point `TaskYieldAsync` yields to `OnInitializedAsync` which yields to the internal `ComponentBase` method `RunInitAndSetParametersAsync`. This in turn yields to the SC: the renderer renders the component.  

```text
Thread: 19 - Component Rendered e45633be-c5a1-451b-99a2-ce64f128a65c.
```

Once the render completes, it invokes the `OnAfterRender` on the SC.  The queue has two tasks:

1. The continuation from `OnInitializedAsync`.
2. An instance of `OnAfterRender` for the component.
 
The `OnInitializedAsync` continuation runs to completion [there are no further awaits]. 

```text
Thread: 17 - OnInitializedAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 17 - OnParametersSet Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 17 - OnParametersSetAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

The lifecycle methods run to completion and the internal `ComponentBase` method `CallOnParametersSetAsync` completes and makes the final call to `StateHasChanged`.  The SC prioritizes this over the `OnAfterRender` task and renders the component. 

```text
Thread: 17 - ShouldRender on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 17 - Component Rendered e45633be-c5a1-451b-99a2-ce64f128a65c.
```

The component renders and a second `OnAfterRender` is queued onto the SC. 

There are now two `OnAfterRender` tasks which run to completion.

```
Thread: 20 - First OnAfterRender Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 20 - First OnAfterRenderAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 16 - Subsequent OnAfterRender Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
Thread: 16 - Subsequent OnAfterRenderAsync Completed on Component e45633be-c5a1-451b-99a2-ce64f128a65c.
```

## Task Delay Vs Task Yield

If we replace the `Task.Yield()` with a `Task.Delay(1)` do we get the same result?

Yes.  The sequence is shown below.

```text
Thread: 22 - OnInitialized Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 22 - OnInitializedAsync Started. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 22 - OnInitializedAsync DataLoad Started. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 22 - Component Rendered. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 23 - OnInitializedAsync Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 23 - OnParametersSet Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 23 - OnParametersSetAsync Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 23 - ShouldRender on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 23 - Component Rendered. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 20 - First OnAfterRender Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 20 - First OnAfterRenderAsync Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 16 - Subsequent OnAfterRender Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
Thread: 16 - Subsequent OnAfterRenderAsync Completed. on Component cde895a1-8f00-4d8c-a394-e1604b5dde1e.
```

## A Longer Delay

What about a longer delay that emulates a slow database transaction or API call.

The result below shows the sequence with a 100ms delay. 

```text
Thread: 18 - OnInitialized Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 18 - OnInitializedAsync Started. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 18 - OnInitializedAsync DataLoad Started. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 18 - Component Rendered. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
```
The same, but now a change.  

The `OnAfterRender` associated with the first render happens ahead of the `OnInitializedAsync` continuation because it's still being awaited - the timer hasn't expired [or in the real world, the database transaction hasn't yet returned a result].

```text
Thread: 16 - First OnAfterRender Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 16 - First OnAfterRenderAsync Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
```
And then the continuation and second `OnAfterRender`.

```text
Thread: 17 - OnInitializedAsync Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 17 - OnParametersSet Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 17 - OnParametersSetAsync Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 17 - ShouldRender on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 17 - Component Rendered. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 19 - Subsequent OnAfterRender Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
Thread: 19 - Subsequent OnAfterRenderAsync Completed. on Component 7ca13883-716e-4c6f-9065-2995950d67ee.
```

## Blazor WASM

In Blazor WASM the code runs in the following order regardless of whether you use `Yield`, short `Delay` or long `Delay`.

```text
Thread: 1 - OnInitialized Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - OnInitializedAsync Started. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - OnInitializedAsync DataLoad Started. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - Component Rendered. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - First OnAfterRender Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - First OnAfterRenderAsync Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - OnInitializedAsync Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - OnParametersSet Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - OnParametersSetAsync Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - ShouldRender on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - Component Rendered. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - Subsequent OnAfterRender Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
Thread: 1 - Subsequent OnAfterRenderAsync Completed. on Component dd0380c6-c7bd-4326-a5c8-378594dc6b26.
```

## Conclusions

The results strongly suggest:

1. There's a difference in execution between Server and WASM.
 
2. The time it takes the asynchronous processes to run is critical to when `OnAfterRender` is run in Server.  If the process is complete when the SC completes the first render, the lifecycle methods will either run to the next yield, or completion before the first `OnAfterRender` executes.  If the process is still running, the initial `OnAfterRender` will execute immediately.  You can complicate the issue further by adding multiple awaits spread across the methods.

The changeover point with this code on my development system, today, with a westerly blowing at 14 knots was 7ms.  The sun was out which may or may not make a difference!

3. In WASM the order is the same regardless of which method you use, or the length of the delay.


> Consider JS Interop.  When can you expect any JSinterop code in `OnAfterRender` to have executed?  In Server you may think that because you've run a render in `OnInitalizedAsync`, fields/objects obtained through the JSInterop should be available immedaitely.  They may not be.
