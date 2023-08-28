This is some of my new-ish code, unfortunately I have yet to finish it as my time is spent elsewhere lately.

This is a small and very unrefined utilities module used by the unfinished Data Replicator, Part Allocator & other.

```lua
--!strict
local Utilities = {}

------------- Services -------------
local Reps = game:GetService("ReplicatedStorage")
local HTTPS = game:GetService("HttpService")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")

------------- Directories -------------
local SharedModules = Reps.SharedAssets.Scripts


------------- Modules -------------
local DataStructures = require(SharedModules.DataStructures)

------------- Types -------------

export type Listener = {
	__newindex:("function")?,
	__index:({}|"function")?,
	Connected:boolean,
	Disconnect:"function",
}

export type MetaTableMetaData = {
	AncestorIndex:string?,
	ListenFor:string?,
	Path:{string}?,
	MetaTable:Listener
}

------------- Variables -------------

local IsServer = RunService:IsServer()

------------- Private Functions -------------

local function NewListener(...):Listener

	local NewMetaTable = setmetatable(...)

	assert(not NewMetaTable["Disconnect"],'Cannot have unique metatable key "Disconnect"!')
	assert(not NewMetaTable["Connected"],'Cannot have unique metatable key "Connected"!')

	rawset(NewMetaTable,"Disconnect",function()
		rawset(NewMetaTable,"Connected",false)
		setmetatable(NewMetaTable,nil)
		NewMetaTable = nil
	end)

	rawset(NewMetaTable,"Connected",true)

	return NewMetaTable
end

local function DeepCopyTable(Table:{}):{}
	local Copy = {}
	for Index,Value in pairs(Table) do
		if typeof(Value) == "table" then
			Copy[Index] = DeepCopyTable(Value)
		else
			Copy[Index] = Value
		end
	end

	return Copy
end

------------- Public Functions -------------

local Utilities = {}

Utilities["PrintTable"] = function(Table:{},Recursive:boolean?)
	print(debug.traceback(),HTTPS:JSONEncode(Table))
end

Utilities["DeepCopyTable"] = function(Table:{}):{}
	return DeepCopyTable(Table)
end

Utilities["Expect"] = function(Function,Type,...):any
	local Result = Function(...)
	if not Result then
		repeat task.wait() Result = Function(...) until Result
	end
	return Result
end

local StallFor_DefaultStallTimeout = 5
Utilities["StallFor"] = function(Table:any,Index:any,Timeout:number?)
	local Stamp:number = tick()
	if not Table[Index] then
		repeat task.wait() until 
		Table[Index] or 
			(tick()-Stamp >= (Timeout or StallFor_DefaultStallTimeout)) and 
			((warn("Stall timeout reached, query: "..tostring(Index)..", "..typeof(Index),(tick()-Stamp >= (Timeout or StallFor_DefaultStallTimeout)))) or true)
	end
	return Table[Index]
end

Utilities["GetTablePath"] = function(Table,TargetIndex:string|number):{}?

	local function Recurse(InnerTable, Path):{}?
		Path = Path or {} -- Initialize path if it's not provided

		for Index,Value in pairs(InnerTable) do
			-- Add current key to the path
			table.insert(Path, Index)
			if TargetIndex == Index then
				-- Output the trace data
				return Path -- Return the path if target value is found
			elseif type(Value) == "table" then
				-- Recursively traverse the nested table
				local foundPath = Recurse(Value, Path)
				if foundPath then
					return foundPath -- Return the path if found
				end
			end

			-- Remove the current key from the path
			table.remove(Path)
		end

		return nil -- Return nil if target value is not found in the current branch
	end
	return Recurse(Table,{})
end

Utilities["FollowPath"] = function(AncestorTable:{},Path:{string}):{}? --// Path is a table of Indexes (strings)
	local CurrentTable = AncestorTable
	for _,NextTableIndex in pairs(Path) do
		CurrentTable = CurrentTable[NextTableIndex] or CurrentTable
	end
	return CurrentTable
end

--[[
Utilities["Unpack"] = function(Table:{any}):any
	local Gathered = {}
	for 
end
--]]

Utilities["ListenToIndex"] = function(Table, Callback,MetaData:MetaTableMetaData?):Listener
	--// A callback must be passed, otherwise listening is illogical
	assert(Callback,"A callback function must be provided!")

	local MetaData:MetaTableMetaData = MetaData or {}::MetaTableMetaData

	local NewMetaTable:Listener = NewListener(Table,{
		__newindex = function(self, Index, Value)
			rawset(self,Index,Value)
			if not MetaData or not MetaData.ListenFor or (MetaData.ListenFor and MetaData.ListenFor == Index) then
				--// If Key is provided then it's indicative a specific item in the Table is meant to be listened to and thusly the callback executed
				--// If Key is left out then callback will be run regardless of what item was changed
				Callback(Index, Value, MetaData)
			end
		end,
	})

	MetaData.MetaTable = NewMetaTable

	return NewMetaTable
end

Utilities["ListenToChange"] = function(Table, Callback,MetaData:MetaTableMetaData?):Listener
	--// A callback must be passed, otherwise listening is illogical
	assert(Callback,"A callback function must be provided!")

	local MetaData:MetaTableMetaData = MetaData or {}::MetaTableMetaData

	--// Copy Table
	local Proxy = {}

	for i,v in pairs(Table) do
		Proxy[i] = v
	end

	--// Set all to nil so non-empty tables will trigger
	for i,_ in pairs(Table) do
		Table[i] = nil
	end

	local NewMetaTable:Listener = NewListener(Table, {
		__index = Proxy,
		__newindex = function(self, Index, Value)
			--// Check to make sure the new value is different than the last one
			--// This is because an index's value being set to the exact same value it already has will trigger this again
			if Proxy[Index] == Value then return end
			Proxy[Index] = Value

			if not MetaData or not MetaData.ListenFor or (MetaData.ListenFor and MetaData.ListenFor == Index) then
				--// If Key is provided then it's indicative a specific item in the Table is meant to be listened to and thusly the callback executed
				--// If Key is left out then callback will be run regardless of what item was changed
				Callback(Index, Value, MetaData)
			end
		end,
	})

	MetaData.MetaTable = NewMetaTable

	return NewMetaTable
end


--// Recursively searches the entire table for other tables to listen to
Utilities["ListenAll"] = function(Table,Callback,Path:{string}?):{Listener}
	local Listeners = {} --//  List of all listeners created

	local function Recurse(Tab)
		for Index,Value in pairs(Tab) do
			if typeof(Value) == "table" then

				Path = {unpack(Path or {}),unpack(Utilities.GetTablePath(Table,Index) or {})} --// 

				table.insert(Listeners,Utilities.ListenToChange(Value,Callback,{Path=Path}::MetaTableMetaData))
				Recurse(Value)
			end
		end
	end

	Recurse(Table) --// Begin recurse at hierarchy

	table.insert(Listeners,Utilities.ListenToChange(Table,Callback))

	return Listeners --//  Return all listeners created so they can be disconnected later
end

Utilities["Tween"] = function(ObjectToTween:Instance,TweenInf:TweenInfo,Goals:{}):Tween
	return TweenService:Create(ObjectToTween,TweenInf,Goals)
end

------------- Public Functions ------------- Checks -------------

Utilities["IsPlayerPresent"] = function(plr:Player?):boolean
	return plr and plr.Parent and plr.Parent == game.Players or false
end

------------- Init -------------


return Utilities

```
