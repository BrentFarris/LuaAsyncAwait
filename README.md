# LuaAsyncAwait
Async/Await for Lua in just a few lines of code

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
		local newResult = self.result
		self.result = lastResult
		return newResult
	end
	setmetatable(await, await)
	coroutine.resume(f, await, ...)
end
```
