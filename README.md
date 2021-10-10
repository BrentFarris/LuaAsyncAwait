# Lua Async/Await
Async/Await for Lua in just a few lines of code

## Summary
Using Lua coroutines we can make async/await functionality which is quite useful for everything from HTTP requests to UI updates. The key to this is the `await` table that is passed around. The code will essentially freeze (`coroutine.yield`) on the line where you call `await(...)` until the `await.resolve(...)` function is called (which invokes `coroutine.resume`). When you call `await(...)` it will pass itself into the function call so that you can call either the `resolve` or `reject` functions. This also works with async functions calling other async functions and waiting on them.

## Async/Await in Lua
```lua
function async(routine, ...)
	local f = coroutine.create(function(await, ...) routine(await, ...) end)
	local await = { error = nil, result = nil, completed = false }
	local complete = function(arg, err)
		await.result = arg
		await.error = err
		await.completed = true
		coroutine.resume(f)
	end
	await.resolve = function(arg) complete(arg, nil) end
	await.reject = function(err) complete(nil, err) end
	await.__call = function(self, wait, ...)
		local lastResult = self.result
		self.completed = false
		wait(self, ...)
		if not self.completed then coroutine.yield(f, ...) end
		if self.error then assert(false, self.error) end
		self.completed = false
		local newResult = self.result
		self.result = lastResult
		return newResult
	end
	setmetatable(await, await)
	coroutine.resume(f, await, ...)
end
```

## Usage
```lua
local frameCounter = 0
--- Imagine this funciton is called every frame in a game engine
function update_frame()
	frameCounter = frameCounter + 1
end

function do_something()
	print("Starting async")
	async(do_async, 9)
	print("Async was started, I'm not waiting for it")
end

function do_async(await, countTo)
	print("Waiting 1000 frames")
	local msg = await(wait_for_frames, 1000)
	print(msg)
	for i=1, countTo do
		print("Printing number", i)
	end
	print("Async work complete")
end

function wait_for_frames(await, frames)
	while frameCounter < frames do
		--Nothing
	end
	await.resolve("All done here")
end

--[[ OUTPUT
	Starting async
	Waiting 1000 frames
	Async was started, I'm not waiting for it
	All done here
	1
	2
	3
	4
	5
	6
	7
	8
	9
]]
```
