# Rodux API Reference

## Rodux.Store
The Store class is the core piece of Rodux. It is the state container that you create and use.

### Store.new
```
Store.new(reducer, [initialState, [middlewares]]) -> Store
```

Creates and returns a new Store.

* `reducer` is the store's root reducer function, and is invokved whenever an action is dispatched. It must be a pure function.
* `initialState` is the store's initial state. This should be used to load a saved state from storage.
* `middlewares` is a list of middleware to apply to the store.

The store will automatically dispatch an initialization action with a `type` of `@@INIT`.

!!! note
	The initialization action does not pass through any middleware prior to reaching the reducer.

### Store.changed
```lua
store.changed:connect(function(newState, oldState)
	-- do something with newState or oldState
end)
```

A [Signal](#Signal) that is fired when the store's state is changed up to once per frame.

!!! warning
	Multiple actions can be grouped together into one changed event!

!!! danger
	Do not yield within any listeners on `changed`; an error will be thrown.

### Store:dispatch
```
store:dispatch(action) -> nil
```

Dispatches an action. The action will travel through all of the store's middlewares before reaching the store's reducer.

Unless handled by middleware, `action` must contain a `type` field to indicate what type of action it is. No other fields are required.

### Store:getState
```
store:getState() -> table
```

Gets the store's current state.

!!! warning
	Do not modify this state! Doing so will cause **serious** bugs your code!

### Store:destruct
```
store:destruct() -> nil
```

Destroys the store, cleaning up its connections.

!!! danger
	Attempting to use the store after `destruct` has been called will cause problems.

### Store:flush
```
store:flush() -> nil
```

Flushes the store's pending actions, firing the `changed` event if necessary.

!!! info
	`flush` is called by Rodux automatically every frame and usually doesn't need to be called manually.

## Signal
The Signal class in Rodux represents a simple, predictable event that is controlled from within Rodux. It cannot be created outside of Rodux, but is used as `Store.changed`.

### Signal:connect
```
signal:connect(listener) -> { disconnect }
```

Connects a listener to the signal. The listener will be invoked whenever the signal is fired.

`connect` returns a table with a `disconnect` function that can be used to disconnect the listener from the signal.

## Helper functions
Rodux supplies some helper functions to make creating complex reducers easier.

### Rodux.combineReducers
A helper function that can be used to combine multiple reducers into a new reducer.

```lua
local reducer = combineReducers({
	key1 = reducer1,
	key2 = reducer2,
})
```

`combineReducers` is functionally equivalent to writing:

```lua
local function reducer(state, action)
	return {
		key1 = reducer1(state.key1, action),
		key2 = reducer2(state.key2, action),
	}
end
```

### Rodux.createReducer
```
Rodux.createReducer(initialState, actionHandlers) -> reducer
```

A helper function that can be used to create reducers.

Unlike JavaScript, Lua has no `switch` statement, which can make writing reducers that respond to lots of actions clunky.

Reducers often have a structure that looks like this:

```lua
local initialState = {}

local function reducer(state, action)
	state = state or initialState

	if action.type == "setFoo" then
		-- Handle the setFoo action
	elseif action.type == "setBar" then
		-- Handle the setBar action
	end

	return state
end
```

`createReducer` can replace the chain of `if` statements in a reducer:

```lua
local initialState = {}

local reducer = createReducer(initialState, {
	setFoo = function(state, action)
		-- Handle the setFoo action
	end,

	setBar = function(state, action)
		-- Handle the setBar action
	end
})
```

## Middleware
Rodux provides an API that allows changing the way that actions are dispatched called *middleware*. To attach middlewares to a store, pass a list of middleware as the third argument to `Store.new`.

A single middleware is just a function with the following signature:

```
(next) -> (store, action) -> result
```

That is, middleware is a function that accepts the next middleware to apply and returns a new function. That function takes the `Store` and the current action and can dispatch more actions, log to output, or do network requests!

A simple version of Rodux's `loggerMiddleware` is as easy as:

```lua
local function simpleLogger(next)
	return function(store, action)
		print("Dispatched action of type", action.type)

		return next(store, action)
	end
end
```

Rodux also ships with several middleware that address common use-cases.

### Rodux.loggerMiddleware
A middleware that logs actions and the new state that results from them.

`loggerMiddleware` is useful for getting a quick look at what actions are being dispatched. In the future, Rodux will have tools similar to [Redux's DevTools](https://github.com/gaearon/redux-devtools).

```lua
local store = Store.new(reducer, initialState, { loggerMiddleware })
```

### Rodux.thunkMiddleware
A middleware that allows thunks to be dispatched. Thunks are functions that perform asynchronous tasks or side effects, and can dispatch actions.

`thunkMiddleware` is comparable to Redux's [redux-thunk](https://github.com/gaearon/redux-thunk).

```lua
local store = Store.new(reducer, initialState, { thunkMiddleware })

store:dispatch(function(store)
	print("Hello from a thunk!")

	store:dispatch({
		type = "thunkAction"
	})
end)
```