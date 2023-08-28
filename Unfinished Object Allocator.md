This is some of my new-ish code, unfortunately I have yet to finish it as my time is spent elsewhere lately.

The function of this code is commonly known as 'object allocation' or 'part pooling'. For games that use a lot of the same object it is significantly more performant to initially create a resevoir of objects, 
pull from that resevoir for the objects when an object is needed to be retrieved, and procedurally create new objects on an as-needed basis.

```lua
--!strict
local ObjectAllocator = {}

------------- Services -------------
local Reps = game:GetService("ReplicatedStorage")

------------- Directories -------------

------------- Modules -------------

------------- Types -------------

------------- Variables -------------
local ClientAllocationFolder = Instance.new("Folder")
ObjectAllocator.Resevoir = {}
ObjectAllocator.Resevoirs = {}

local PositionItemsTo = CFrame.new(5000,5000,5000)


------------- Private Functions -------------
local function PositionItem(Item:Instance)
	if Item:IsA("BasePart") then
		Item.CFrame = PositionItemsTo
	elseif Item:IsA("Model") then
		if not Item.PrimaryPart then
			warn(tostring(Item).." did not have a PrimaryPart, make sure to fix that!")
			Item.PrimaryPart = Item:FindFirstChildWhichIsA("BasePart",true)
			
			if Item.PrimaryPart then
				Item:PivotTo(PositionItemsTo)
				Item.PrimaryPart = nil
			else
				warn(tostring(Item).." is a model but has no BasePart in it!")
			end
		end
	end
end

------------- Public Functions -------------

ObjectAllocator["Resevoir"].new = function(Name:string,FillItem:Instance,Parent:Instance?,FillAmount:number?,FillAsNeeded:boolean?)

	assert(not ObjectAllocator.Resevoirs[Name],"Error creating new allocation resevoir: namespace error, "..tostring(Name).." already exists!")

	local Resevoir = {}
	local NewAllocationContainer = Instance.new("Folder")
	NewAllocationContainer.Name = Name
	NewAllocationContainer.Parent = Parent or workspace

	Resevoir.Container = NewAllocationContainer
	Resevoir.FillAsNeeded = FillAsNeeded or true
	Resevoir.Locked = false
	Resevoir.Queue = {}
	Resevoir.Active = {}
	Resevoir.FillItem = FillItem:Clone()
	
	--// Fills up the resevoir with the designated FillItem instance, i.e. fill up the resevoir with 500 projectile parts to be used later
	--// They will be stored somewhere far off in the distance from rendering but in workspace in order to allocate memory and reduce networking demands
	function Resevoir:Fill(Amount:number):Instance?
		if Resevoir.Locked then print("Fill call failed: resevoir is locked!") return end
		
		for loop = 1,Amount do
			local Clone = Resevoir.FillItem:Clone()
			PositionItem(Clone)
			table.insert(Resevoir.Queue,Clone)
			Clone.Parent = Resevoir.Container
		end

		return Resevoir.Queue[1]::Instance
	end
	
	--// Returns the next part 
	function Resevoir:GetNext():any
		if Resevoir.Locked then print("GetNext call failed: resevoir is locked!") return end
		
		local Next:any = (Resevoir.Queue[1]) or (Resevoir.FillAsNeeded and Resevoir:Fill(1)) or (Resevoir:WaitForAvailable())
		table.insert(Resevoir.Active,Next)
		table.remove(Resevoir.Queue,1)
		return Next
	end

	function Resevoir:PutBack(Item:Instance)
		if Resevoir.Locked then print("PutBack call failed: resevoir is locked!") return end
		PositionItem(Item)
		table.insert(Resevoir.Queue,Item)
		return true
	end

	function Resevoir:WaitForAvailable():Instance
		repeat task.wait() until Resevoir.Queue[1] and not Resevoir.Locked
		return Resevoir.Queue[1]
	end
	
	function Resevoir:ReplaceFillItem(Item:Instance)
		if Resevoir.Locked then print("ReplaceFillItem call failed: resevoir is locked!") return end
		
		if Resevoir.FillItem and Resevoir.FillItem.Parent then
			Resevoir.FillItem:Destroy()
			return true
		else
			warn("Could not destroy fill item: it did not exist!")
		end
	end

	function Resevoir:Destroy()
		Resevoir.Locked = true
		
		for _,Item:Instance in pairs(Resevoir.Queue) do
			Item:Destroy()
		end
		for _,Item:Instance in pairs(Resevoir.Active) do
			Item:Destroy()
		end
		Resevoir.FillItem:Destroy()
		Resevoir.Container:Destroy()
		Resevoir = nil::any --// Had to assert type 'any' to silence typecheck warning
	end

	ObjectAllocator.Resevoirs[Name] = Resevoir

	return Resevoir
end

ObjectAllocator["Resevoir"].Get = function(Name:string)
	return ObjectAllocator.Resevoirs[Name]
end

------------- Events -------------

------------- Init -------------


return ObjectAllocator
```
