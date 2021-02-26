# Fluxor snippets for development in Visual Studio

To install, clone this repo and add the folder to your Code Snippets Manager in Visual Studio.

## Explanation

These snippets are intended to encourage a cohesive structure for actions within a Fluxor store: a single file containing the action, reduce method, and effect handler all in the same place. The `Action` gets sent to the dispatcher and the `Reduce` method performs the immediate state update logic. Meanwhile the `Effect.HandleAsync` method takes care of running the asynchronous logic and dispatches either a `Success.Action` or a `Failed.Action` to indicate the result.

### fleffectaction

The fleffectaction generates the boilerplate required to implement an asynchronous workflow (eg, an API call). It's meant to be inserted within a static class named after your action. For best results, follow these steps:

1. Immediately perform a Rename Refactor to rename `TState` to the name of your externally defined state class. This updates the TState type parameter throughout the generated code. Afterwards, you can delete the empty `class TState { }` line at the start of the snippet. The only reason this step is here is because VS 2019 doesn't seem to support placing multiple cursors on the file after inserting the snippet.
2. Add any properties to your action that you need to send in order to update state or in order to perform the async logic.
3. Implement the Reduce method to synchronously update your local state based on the Action.
4. Define the return values from your async logic on the `Success.Action` class (eg, data returned from an API).
5. Implement the `Effect.HandleAsync` method that performs asynchronous logic. Construct and dispatch a `Success.Action` if the result was successful, and a `Failed.Action` if there was a problem.

### flpureaction

This snippet is used for actions that can be handled synchronously on the client and do not have any side effects. As a result, the snippet will only insert the `Action` class and the `Reduce` method, without the extra `Effect`, `Success`, or `Failed` classes. Otherwise, implementing the workflow is identical to implementing the full async workflow above.

## Additional Advice
**Important**: It's highly recommended that you manually log and handle all exceptions within any effect methods that you write. This is to avoid having the error completely crash the page, and also to make sure you are collecting accurate stack trace information.  
**Important**: Instead of returning IEnumerable in your success actions, it's preferable to use IReadOnlyList. This ensures that your data does not silently have a dependency on a service that could be disposed after the effect handler has returned.  
**Note**: Fluxor plays very nicely with C# 9. This lets you drop a lot of object cloning boilerplate due to having the `with` keyword and `record` type syntax.  
**Note**: Continue to use your best discretion regarding when it might be a good time to split up this file or extract common classes. This is meant to be a template to get started with adding new features, rather than a strict requirement for your solution structure.

## Simple example

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Fluxor;
using Microsoft.Extensions.Logging;

namespace App.Search.Store.Actions
{
    public static class FetchSearchResults
    {
        public class Action
        {
            public Action(string searchText)
            {
                SearchText = searchText;
            }
            public string SearchText { get; }
        }

        [ReducerMethod]
        public static AppSearchState Reduce(AppSearchState state, Action action) =>
            new AppSearchState(searchText: action.SearchText, isLoading: true, results: state.Results);

        public class Effect
        {
            private readonly ILogger<Effect> logger;
            private readonly IState<AppSearchState> state;
            private readonly IApiClient apiClient;

            public Effect(ILogger<Effect> logger, IState<AppSearchState> state, IApiClient apiClient)
            {
                this.logger = logger;
                this.state = state;
                this.apiClient = apiClient;
            }

            [EffectMethod]
            public async Task HandleAsync(Action action, IDispatcher dispatcher)
            {
                try
                {
                    var results = await apiClient.SearchAsync(action.SearchText);
                    dispatcher.Dispatch(new Success.Action(results.ToList()));
                }
                catch (Exception ex)
                {
                    logger.LogError(ex, "An error occurred handling the action");
                    dispatcher.Dispatch(new Failed.Action("Action failed"));
                }
            }
        }

        public static class Success
        {
            public class Action
            {
                public Action(IReadOnlyList<string> results)
                {
                    Results = results;
                }
                public IReadOnlyList<string> Results { get; }
            }

            [ReducerMethod]
            public static AppSearchState Reduce(AppSearchState state, Action action)
            {
                return new AppSearchState(searchText: state.SearchText, isLoading: false, results: action.Results);
            }
        }

        public static class Failed
        {
            public class Action
            {
                public Action(string errorMessage)
                {
                    ErrorMessage = errorMessage;
                }

                public string ErrorMessage { get; }
            }

            [ReducerMethod]
            public static AppSearchState Reduce(AppSearchState state, Action action)
            {
                return new AppSearchState(searchText: state.SearchText, isLoading: false, results: Enumerable.Empty<string>());
            }

            // Optional: You may want the Failed.Action to have its own side effects, such as displaying an error toast.
            public class Effect
            {
                [EffectMethod]
                public async Task HandleAsync(Action action, IDispatcher dispatcher)
                {

                }
            }
        }

    }
}
```
