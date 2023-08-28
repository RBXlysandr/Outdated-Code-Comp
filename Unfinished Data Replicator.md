This is some of my new-ish code, unfortunately I have yet to finish it as my time is spent elsewhere lately.

The function of this extremely unrefined code is a Data Replicator. Data is hosted on the server and replicated to any clients. 
It's meant to be super lightweight and accessible compared to ReplicaService which is big, comprehensive, and complicated. The intended use-case is mostly just player data replication to other clients.

```lua
--!strict
local DataReplicator = {}

------------- Services -------------
local Players = game:GetService("Players")
local Reps = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

------------- Directories -------------

local SharedModules = Reps.SharedAssets.Scripts


------------- Modules -------------

local Util = require(SharedModules.Utilities)
local DataStructures = require(SharedModules.DataStructures)

------------- Types -------------

type PlayerData = DataStructures.PlayerData

------------- Variables -------------

local SETTINGS_DEBUG_PrintDataActivity = true

local IsServer = RunService:IsServer()
local LocalPlayer = not IsServer and Players.LocalPlayer
local Relay:RemoteEvent = Reps.Events:WaitForChild("DataReplicatorRelay",30) or assert(Reps.Events:FindFirstChild("DataReplicatorRelay"),"Data Replicator Relay missing from Events folder!")
local Server_ClientListeners = {}
local Server_ServerListeners = {}

--// Client: find/wait for the components folder
--// Server: find/create components folder
local ComponentsFolder:Folder
local DataCallInvoke:RemoteFunction

if IsServer then
	ComponentsFolder = (Reps.SharedAssets:FindFirstChild("DataReplicator_ComponentsFolder") or Instance.new("Folder",Reps.SharedAssets))::Folder
	ComponentsFolder.Name = "DataReplicator_ComponentsFolder"

	DataCallInvoke = (ComponentsFolder:FindFirstChild("DataReplicator_Invoker") or Instance.new("RemoteFunction",ComponentsFolder))::RemoteFunction
	DataCallInvoke.Name = "DataReplicator_Invoker"
else
	ComponentsFolder = (Reps.SharedAssets:WaitForChild("DataReplicator_ComponentsFolder",math.huge))::Folder
	DataCallInvoke = (ComponentsFolder:WaitForChild("DataReplicator_Invoker",math.huge))::RemoteFunction
end

DataReplicator.Datas = {}
DataReplicator.Datas.PlayerDatas = {}
DataReplicator.DataRelay = Relay

------------- Private Functions -------------

--// Seperate handlers for different types of data, PlayerData is a special and sensitive data type and must be replicated differently
local function OnDataAdded(ParentIndex:string,DataKey:string,Profile:{any})
	assert(DataReplicator.DataHandlers[ParentIndex],"Missing data handler for: "..tostring(ParentIndex).."!")
	return DataReplicator.DataHandlers[ParentIndex](DataKey,Profile)
end

local function RelayToClients(ParentIndex:string,TableIndex:string,ChangedIndex:string,ChangedValue:any,Path:{string}?)
	for UserId:number,ListenToKey:string in pairs(Server_ClientListeners[ParentIndex]) do
		local plr = Players:GetPlayerByUserId(UserId)
		if not plr then warn("Couldn't find player by userid "..tostring(UserId))  continue end
		--// Omit relay to player if they are listening to a specific index change, and the current changed index do not match that specific index
		if ListenToKey and ListenToKey ~= "" and ChangedIndex ~= ListenToKey then warn(ChangedIndex,ListenToKey,"not their index") continue end
		(ComponentsFolder:FindFirstChild(ParentIndex) :: RemoteEvent):FireClient(plr,TableIndex,ChangedIndex,ChangedValue,Path)
	end
end

------------- Public Functions -------------


------------- Public Functions > Invocations -------------

if IsServer then

	--// Server gets a read/writeable return of the queried data. Only the server can write to server-sided data
	DataReplicator["GetData"] = function(DataKey:string|Player,ParentIndex:string?):(PlayerData?,string?)
		if not DataKey and not ParentIndex then warn("Missing arguments for player data get") return end

		--// Convert scope keys to strings
		ParentIndex = (ParentIndex or typeof(DataKey) == "Instance" and DataKey:IsA("Player") and "PlayerDatas" or "PlayerDatas")::string --// Convert parent index, if it's omitted and DataKey is a player than it's likely PlayerDatas index, default is "PlayerDatas"
		DataKey = ((typeof(DataKey) == "Instance" and DataKey:IsA("Player") and tostring(DataKey.UserId)) or DataKey)::string --// Convert data key, if it's a player then it will be the player's userid as the key
		assert(ParentIndex and DataKey,"Get data parent/datakey index failure")

		--// A stall if the data is not currently present
		Util.StallFor(DataReplicator.Datas,ParentIndex)
		Util.StallFor(DataReplicator.Datas[ParentIndex],DataKey)
		
		return DataReplicator.Datas[ParentIndex][DataKey]::PlayerData,DataKey::string
	end
else
	
	DataReplicator["Call"] = function(FunctionName:string,...)
		return DataCallInvoke:InvokeServer(FunctionName,...)
	end

	DataReplicator["GetData"] = function(DataKey:string|Player,ParentIndex:string?):(PlayerData?,string?)
		if not DataKey and not ParentIndex then warn("Missing arguments for player data get") return end

		--// Convert scope keys to strings
		ParentIndex = (ParentIndex or typeof(DataKey) == "Instance" and DataKey:IsA("Player") and "PlayerDatas" or "PlayerDatas")::string --// Convert parent index, if it's omitted and DataKey is a player than it's likely PlayerDatas index, default is "PlayerDatas"
		DataKey = ((typeof(DataKey) == "Instance" and DataKey:IsA("Player") and tostring(DataKey.UserId)) or DataKey)::string --// Convert data key, if it's a player then it will be the player's userid as the key
		assert(ParentIndex and DataKey,"Get data parent/datakey index failure")
		
		--// If the queried data isn't in the client's own database then this will be nil, if nil then request the servers copy of it
		local Data = DataReplicator.Datas[ParentIndex][DataKey] or DataReplicator.Call("GetData",DataKey,ParentIndex)

		return Data::PlayerData,DataKey::string
	end
end

--// Exclusively invoked from client (via RemoteFunction [DataCallInvoke]) to get a listener for specific data changes, 
--// i.e. a friend of a client wants to listen to changes to their friends data, they must subscribe here
DataReplicator["Listen"] = function(ListenerId:"Player UserId: number",DataKey:string|Player,ListenToKey:string?):RemoteEvent?

	assert(ListenerId,"A valid player UserId is required for the 'Listen' invocation")

	local Data:PlayerData?,DataKey:string? = DataReplicator.GetData(DataKey)

	if not Data or not DataKey then warn("Could not subscribe listener to data: "..tostring(DataKey)) return  end

	--// Omitting the optional ListenToKey arg will cause changes of ANY index to trigger event
	--// If one wants to listen to a specific change then pass ListenToKey as that specific index change
	Server_ClientListeners[DataKey][ListenerId] = ListenToKey or ""

	return ComponentsFolder:FindFirstChild(DataKey)::RemoteEvent
end

----------------------------------------------------

--// Seperate handlers for different types of data, PlayerData is a special and sensitive data type and must be replicated differently
DataReplicator.DataHandlers = {
	["PlayerDatas"] = function(DataKey:string,PlrData)

		--// Setup Debris table, storing the 'metatable listeners' to be disconnected on player leave
		Server_ServerListeners[DataKey] = {}
		
		--// TODO this returns the raw table, not the metatable
		--// it should return the metatable 
		local ProxyData = Util.DeepCopyTable(PlrData)

		local function Recurse(Table,TableIndex:string,ParentIndex:string)

			for SubTableIndex,SubTable in pairs(Table) do
				if typeof(SubTable) ~= "table" then continue end
				Table[SubTableIndex] = Recurse(SubTable,SubTableIndex,TableIndex) --// Basically converts it into a metatable
				
				for i,v in pairs(Table[SubTableIndex]) do
					warn(i,v)
				end
			end

			--// Path is critical as it gives a direct path to the changed item in the data
			local Path:{string}? = {"PlayerDatas",DataKey,unpack(Util.GetTablePath(PlrData,TableIndex) or {})}

			local MetaTable = setmetatable({},{
				Unique_MetaTable_RawIndex = TableIndex,
				Unique_MetaTable_RawParentIndex = ParentIndex,
				Unique_MetaTable_RawAncestorTable = PlrData,
				Unique_MetaTable_RawPath = Path,
				__index = function(self,QueriedIndex)
					if SETTINGS_DEBUG_PrintDataActivity then warn("__index triggered:",QueriedIndex) end
					return rawget(Table,QueriedIndex)
				end,
				__newindex = function(self,ChangedIndex:string,ChangedValue:any)
					if rawget(Table,ChangedIndex) == ChangedValue then return end --// Make sure the value has ACTUALLY changed to something different
					if SETTINGS_DEBUG_PrintDataActivity then
						warn("__newindex triggered:",ChangedIndex,ChangedValue,rawget(getmetatable(self),"Unique_MetaTable_RawIndex"))
					end
					rawset(Table,ChangedIndex,ChangedValue)
					--// This is where changes are sent to listening clients so they can reconcile data accordingly
					RelayToClients(DataKey,TableIndex,ChangedIndex,ChangedValue,Path)
				end,
			})::any --// cast to 'any' to silence typecheck warning

			--// Add to listener debris table as aforementioned
			--table.insert(Server_ServerListeners[DataKey],MetaTable)

			return MetaTable
		end

		--// ProxyData is proxy of the player's Profile, it is the buffer betwixt server and client, changes are made indirectly to the actual Profile (or 'Datastore') via proxy data
		--// An example of proxy-data usage shown at bottom of script
		local a = Recurse(ProxyData,DataKey,"PlayerDatas")

		warn("_")
		
		for i,v in pairs(a) do
			warn(i,v)
		end
		
		local Relay = Instance.new("RemoteEvent")
		Relay.Name = DataKey
		Relay.Parent = ComponentsFolder

		Server_ClientListeners[DataKey] = {}
		DataReplicator.Datas.PlayerDatas[DataKey] = ProxyData

		return ProxyData
	end,
}

DataReplicator["AddData"] = function(ParentIndex:string,DataKey:string,Profile:{any})
	local Parent = DataReplicator.Datas[ParentIndex]
	assert(Parent,tostring(ParentIndex).." is an invalid data table!")
	return OnDataAdded(ParentIndex,DataKey,Profile)
end

DataReplicator["OnPlayerRemoving"] = function(plr:Player)
	local DataKey = tostring(plr.UserId)

	--// Unassign all data entries the player has
	--// Disconnect all metatables from listening to player data
	for _,MetaTable in pairs(Server_ServerListeners[DataKey] or {}) do
		setmetatable(MetaTable,nil)
	end

	Server_ServerListeners[DataKey] = nil
	if ComponentsFolder:FindFirstChild(DataKey) then ComponentsFolder:FindFirstChild(DataKey):Destroy() end
	Server_ClientListeners[DataKey] = nil
	DataReplicator.Datas.PlayerDatas[DataKey] = nil

	print(tostring(plr).." has left, removed all data entries for: "..(IsServer and "Server" or "Client"))
end

------------- Events -------------


------------- Init -------------

--// Init should only ever be called once, executed in ServerConsole (for the server) or ClientConsole (for client)
DataReplicator.Init = function()

	Players.PlayerRemoving:Connect(DataReplicator.OnPlayerRemoving)

	if IsServer then

		--// The client invokes server to retrieve something, a listener, data, etc
		--// All functions in DataReplicator that the client needs executed by the server is run through here. YIELDS
		DataCallInvoke.OnServerInvoke = function(plr:Player,FunctionCall:string,...)
			--// Special exceptions if statement for functions that need special passed args while maintaining ease-of-use when calling these functions
			--// I.e. with this vs without: on the client;  DataReplicator.Call("Listen",PlayerToListenTo) vs DataReplicator.Call("Listen",ListeningPlayer,PlayerToListenTo)
			--// TODO solve the issue of needing this if statement
			if FunctionCall == "Listen" then
				return DataReplicator[FunctionCall](plr.UserId,...)
			else
				return DataReplicator[FunctionCall](...)
			end
		end

	else


		--[[
		local Data = DataReplicator.GetData(LocalPlayer)
		
		assert(Data,"Ummmmmmmmm there should be data here lol")
		
		warn(Data.DailyLogin.LastLoginStamp)
		
		DataReplicator.Datas.PlayerDatas[tostring(LocalPlayer.UserId)] = Data::any --// Not sure why but must be indicated as any otherwise there's an annoying typecheck warning

		--// Subscribe local player to their own data changes, this event connection updates all of the players data anytime a change is made
		--// Other 'Listen' callbacks do not need these specific internals, they can be and do whatever
		DataReplicator.Call("Listen",LocalPlayer).OnClientEvent:Connect(function(TableIndex,ChangedIndex,ChangedValue,Path)
			local Table:{}? = Util.FollowPath(DataReplicator.Datas,Path)
			if not Table then 
				warn("Could not follow table path:",TableIndex,ChangedIndex,ChangedValue,Path and unpack(Path))
				return
			end
			Table[ChangedIndex] = ChangedValue
			if SETTINGS_DEBUG_PrintDataActivity then
				warn(TableIndex,ChangedIndex,ChangedValue)
			end
		end)
--]]
	end
end

--[[ --// Use example of data Setting via server

local Data = DataReplicator.GetData(plr)

Data.Money += 5000
--// This would data Set and signal to the client of this change and their records would be reconciled accordingly
--// The actual saving is handled by ProfileService

--// If a client wanted to Get their- or someone elses data
local Data = DataReplicator.Call("GetData",plr)

print(Data.Money) --> Prints: 5000

--// If a client wanted to Listen to a data change in their- or someone elses data

--// Listen example
local Con = DataReplicator.Call("Listen",LocalPlayer,"TotalKills").OnClientEvent:Connect(function(TableIndex,ChangedIndex,ChangedValue,Path)
	... --// Do stuff
end)


--]]

return DataReplicator
```
