# Async Await Inside of Lua
Async/Await for Lua in just a few lines of code.

Using Lua coroutines we can make async/await functionality which is quite useful for everything from HTTP requests to UI updates. The key to this is the `await` table that is passed around. The code will essentially freeze (`coroutine.yield`) on the line where you call `await(...)` until the `await.resolve(...)` function is called (which invokes `coroutine.resume`). When you call `await(...)` it will pass itself into the function call so that you can call either the `resolve` or `reject` functions. This also works with async functions calling other async functions and waiting on them.

## Lua Async/Await Complete Code
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
function test_async()
        print("Starting async")
        async(do_async, 6)
        print("Async was started, I'm not waiting for it")
end

local fakeNet = nil
-- The async function to be called from anywhere in the game
function do_async(await, countTo)
        local req = {
                done = function(res) end,
                sim = coroutine.create(function()
                        local clock = os.clock
                        local t0 = clock()
                        local fakeDelay = t0 + 3.0 -- Seconds (slow network :P)
                        while clock() - t0 < fakeDelay do
                                coroutine.yield()
                        end
                        -- Got response from network
                        fakeNet.done("Hello from the fake internet")
                end)
        }
        fakeNet = req
        print("Doing fake network request")
        -- The code stops here and waits for async.resolve or async.reject before continuing
        local msg = await(wait_for_net, req)
        print("Message from fake network: "..msg)
        print("Async work complete")
end

test_async()

local clock = os.clock
local d0 = clock()
local delay = d0 + 5.0
while clock() - d0 < delay do
        -- Just keep the program alive
        coroutine.resume(fakeNet.sim)
end

--[[ OUTPUT
Starting async
Doing fake network request
Going to wait on the network to finish...
Async was started, I'm not waiting for it
-- 3 seconds pass
Message from fake network: Hello from the fake internet
Async work complete
]]
```
