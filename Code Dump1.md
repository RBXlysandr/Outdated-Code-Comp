A lot of this is older code of mine (about a year ago)... some of it uses my more outdated methods like the use of `_G` and so on.
This is a dump of a varied array of different code for different projects and applications

```lua
--!strict
local M = {}

M.DataTypeMap = {
	["number"] = "NumberValue",
	["boolean"] = "BoolValue",
	["string"] = "StringValue",
	["color3"] = "Color3Value",
	["cframe"] = "CFrameValue",
	["vector3"] = "Vector3Value",
	["instance"] = "ObjectValue",
}

M.GroupInfo = {
	GroupID = 69420,
	FacultyRankIDs = {
		255,
		254,
		253,
		252,
		251,
		250,
	},
	LowestRankToBeTesterID = 100, --// As in they must be this rank number or higher to get tester validation
}

local SimplifyerSign = {
	["S1"] = "K",
	["S2"] = "M",
	["S3"] = "B",
	["S4"] = "T",
	["S5"] = "Qa",
	["S6"] = "Qi",
	["S7"] = "Sx",
	["S8"] = "Sp",
	["S9"] = "Oc",
	["S10"] = "N",
	["S11"] = "Dc",
	["S12"] = "Ud",
	["S13"] = "Dd",
	["S14"] = "Td",
	["S15"] = "Qad",
	["S16"] = "Qid",
	["S17"] = "Sxd",
	["S18"] = "Spd",
	["S19"] = "Ocd",
	["S20"] = "Nod",
	["S21"] = "Vg",
	["S22"] = "Uvg",
	["S23"] = "Dvg",
	["S24"] = "Tvg",
	["S25"] = "Qavg"
}

local function FormatNum(num:number):string
	local formatted = string.format("%.0f",num)
	return string.reverse(formatted)
end

--// RTM lib

local function RTtoString(Color:any):string 
	local r = math.floor(Color.R*255)
	local g = math.floor(Color.G*255)
	local b = math.floor(Color.B*255)
	return tostring(r)..","..tostring(g)..","..tostring(b)
end

M["RTColor"] = function(String:string, Color:any):string  -- Converts text into RichText Markup format, returns the string as the color passed
	String = typeof(String) == "string" and String or tostring(String)
	Color = typeof(Color) == "string" and Color or RTtoString(Color)
	return '<font color="rgb('.. Color ..')">'.. String ..'</font>'
end

M["RTFont"] = function(String:string, Font:any):string  -- Converts text into RichText Markup format, returns the string as the font passed
	return '<font face="'..Font..'">'..String..'</font>'
end

M["RTSize"] = function(String:string, Size:any):string  -- Converts text into RichText Markup format, returns the string as the font size passed
	Size = typeof(Size) == "string" and Size or tostring(Size)
	return '<font size="'..Size..'">'..String..'</font>'
end

M["RTBold"] = function(String:string):string -- Converts text into RichText Markup format, returns the string as the font in bold
	return '<b>'..String..'</b>'
end

M["RTItalicize"] = function(String:string):string  -- Converts text into RichText Markup format, returns the string as the font in italics
	return '<i>'..String..'</i>'
end

M["RTStrikethrough"] = function(String:string):string  -- Converts text into RichText Markup format, returns the string as the font in strikeout
	return '<s>'..String..'</s>'
end

M["RTUnderline"] = function(String:string):string  -- Converts text into RichText Markup format, returns the string as the font in underline
	return '<u>'..String..'</u>'
end

M["RTSmallCaps"] = function(String:string):string  -- Converts text into RichText Markup format, returns the string as the font in full caps
	return '<sc>'..String..'</sc>'
end

--\\

--// Outputs and returns a string detailing traceback information following an error throw
M["HandleError"] = function(err:any):string
	warn("[Error]:",err,debug.traceback("[Traceback]:"))
	return tostring("[Error]: "..err.." "..debug.traceback("[Traceback]:"))
end

--// Runs passed function in a protected yielding call that will traceback-handle errors
M["XPcall"] = function(func,...):(any,any)
	local succ:any,err:any = xpcall(func,
		M.HandleError,
		...)
	return succ,err
end

--// Reverses the passed table and returns it
M["ReverseTable"] = function(Table:{any}):{any}
	for i = 1, math.floor(#Table/2) do
		Table[#Table - i + 1], Table[i] = Table[i], Table[#Table - i + 1]
	end
	return Table
end

--// Returns an abbreviated number as a string
M["SimplifyNumber"] = function(num: number, deci: number):string
	local numStr = FormatNum(num)
	local length = #numStr

	if not deci then
		deci = 1
	end

	if tonumber(numStr) then
		if length > 3 then
			local scale = math.floor((length - 1) / 3)
			local scaledNum = math.floor(num / (10^(scale*3 - deci)))
			return string.format("%."..deci.."f%s", scaledNum/10^deci, SimplifyerSign["S"..scale])
		else
			return tostring(num)
		end
	else
		warn(numStr.." is not a valid number. Please enter a number")
		return "NaN"
	end
end

--// Returns passed a as typeof in lower case letters
M["TypeofInLower"] = function(a:any):string
	return string.lower(typeof(a))
end

--// Returns a deep copy of a table
M["CopyTable"] = function(Table:{any}):{any}
	local Copy = {}
	for k,v in pairs(Table) do
		Copy[k] = typeof(v) == "table" and M["CopyTable"](v) or v
	end
	return Copy
end

--// Checks and returns a players rank status in a group
M["IsPlrTesterOrAdmin"] = function(plr:Player,GroupID:number?):(nil|string|number)
	local Rank:number? = plr:GetRankInGroup(GroupID or M.GroupInfo.GroupID)
	if not Rank then return end

	if table.find(M.GroupInfo.FacultyRankIDs,Rank::number) then 
		return "Admin",Rank
	elseif Rank::number >= M.GroupInfo.LowestRankToBeTesterID then
		return "Tester",Rank
	end
	return
end

--// Converts passed args to Vector3
M["ConvertToV3"] = function(a:any):Vector3 
	local Type = M.TypeofInLower(a)
	if Type == "instance" and a:IsA("Model") and a.PrimaryPart then a = a:GetPivot().Position
	elseif Type == "instance" and a:IsA("Model") and not a.PrimaryPart and a:FindFirstChildWhichIsA("BasePart") then a = a:FindFirstChildWhichIsA("BasePart").Position
	elseif Type == "instance" and a:IsA("BasePart") then a = a.Position
	elseif Type == "cframe" then a = a.Position
	elseif Type == "vector3" then return a
	else
		warn("ConvertToPos Error: Could not identify/convert",a)
	end
	return a
end

--// Converts a and b into a Vector3 if they are not already and returns the magnitude (AKA distance), optional arg XZOnly is for if you want it to exclude Y axis
M["GetMagnitude"] = function(a:any,b:any,XZOnly:any):number
	if XZOnly then
		a = M.ConvertToV3(a)
		b = M.ConvertToV3(b)
		return (Vector3.new(a.X,0,a.Z)-Vector3.new(b.X,0,b.Z)).Magnitude
	end
	return (M.ConvertToV3(a)-M.ConvertToV3(b)).Magnitude
end

--// Shallow searches passed table returning Index and Value if a match is found
--// Alternatives like Table[Index] or table.find() work but not if you want to search a mixed table and get the Index AND Value of the match
M["TableIndexMatchStr"] = function(Table:{any},Str:string):(any,any)
	for Index,Value in pairs(Table) do
		if tostring(Index) == Str then
			return Index,Value
		end
	end

	return
end

--// Passed folder object is recursively searched and creates a table with respective structure
M["ConvertFolderToTable"] = function(ParentFolder:any):{any}
	local NewTable = {}
	local function Recurse(UserDataTable,localTable)
		for _,Obj in pairs(UserDataTable) do
			local MatchedIndex,MatchedValue = M.TableIndexMatchStr(localTable,Obj.Name) -- Index > string, Value > userdata variable
			if MatchedIndex then
				if M.TypeofInLower(MatchedValue) == "table" then
					MatchedValue = Obj:GetChildren()
					Recurse(MatchedValue,MatchedValue)
				else
					localTable[MatchedIndex] = Obj.Value
				end
			end
		end
	end
	Recurse(ParentFolder:GetChildren(),NewTable)
	return NewTable
end

--// Passed table is recursively searched and creates a folder with respective structure, object equivalents are pathed in M.DataTypeMap
M["ConvertTableToFolder"] = function(DataTable:{any}, Parent:any):(any,any)
	local FirstFolder = nil
	local function Recurse(tab:{any}, localParent:any)
		for i,v in pairs(tab) do
			if M.TypeofInLower(v) == "table" then
				local subFolder = localParent:FindFirstChild(i)
				if not subFolder then --// if the key isn't found in default data (item created at runtime) create a valuebase for it
					subFolder = Instance.new("Folder")
					subFolder.Name = i
					subFolder.Parent = localParent
				end
				if not FirstFolder then FirstFolder = subFolder end
				Recurse(v, subFolder)
			elseif M.DataTypeMap[M.TypeofInLower(v)] then
				local valueBase = localParent:FindFirstChild(i)
				if not valueBase then --// if the key isn't found in default data (item created at runtime) create a valuebase for it
					local dataType = M.DataTypeMap[M.TypeofInLower(v)]
					valueBase = Instance.new(dataType)
					valueBase.Name = i
				end
				valueBase.Value = v
				valueBase.Parent = localParent
			end
		end
	end
	Recurse(DataTable, Parent)
	return FirstFolder,Parent
end



--// Commented because they wouldn't function out-of-the box here. They have dependencies in my game their native game
--// Also havent gotten to typechecking them and revising them completely'

--// Quick snippet of UI interaction
--[[

    Menu_CharSelectFrame.PurchaseCharacter_BasicCurrency[Client_InputMode]:Connect(function(i)
        if i.UserInputType ~= Enum.UserInputType.MouseButton1 and i.UserInputType ~= Enum.UserInputType.Touch and i.KeyCode ~= Enum.KeyCode.ButtonA then return end
        if not CharacterSelected then return end

        local CharacterDataFolder = M:GetSkinDataFolder(CharacterSelected.Name) or _G.REPS.Client_CharData:FindFirstChild(CharacterSelected.Name)
        local Cost = CharacterDataFolder.CurrencyCost.Value

        if DataFolder.BasicCurrency.Value < Cost then
            PremShop_PremShopMainFrame.Visible = not PremShop_PremShopMainFrame.Visible
            Menu_CharSelectFrame.Visible = not Menu_CharSelectFrame.Visible
            Notify("You need "..tostring(Cost-DataFolder.BasicCurrency.Value).." more "..M.CurrencyPhrasesMap["BasicCurrency"].."!",Color3.fromRGB(255,0,0))
            return
        end

        _G.EVENTS.BuyAsset:FireServer(CharacterSelected.Name,"Character")
    end)
--]]

--// Notification popups, the server or client can invoke the client to do a popup
--[[

    local function CheckForNotifOverlap(BorderPosY)
        for _,Frame in pairs(NotificationGuiScreenFrame.ActiveNotificationFrames:GetChildren()) do
            local FrameBorderPosY = Frame.AbsolutePosition.Y -- top border of frame
            if FrameBorderPosY-BorderPosY < 0 then
                return nil
            end
        end
        return true
    end

    local function Notify(Msg,Color)
        M:XPcall_Co(function()
            local Notif_MainFrameAbsSize,Notif_MainFrameAbsPos = NotificationGuiMainFrame.AbsoluteSize,NotificationGuiMainFrame.AbsolutePosition
            local Frame = Ass.UI.NotificationFrame:Clone()

            local LoadInAnimTime = .5

            Color = Color or Color3.fromRGB(255,255,255)
            Frame.Message.Text = Msg
            Frame.Size = UDim2.new(0,0,0,0)
            Frame.Position = UDim2.fromScale(-.5,0)
            Frame.Message.TextColor3 = Color
            Frame.Message.TextStrokeColor3 = M:StrokeColor(Color,.5)

            local V2 = game:GetService("TextService"):GetTextSize(Frame.Message.Text,21,Frame.Message.Font,Vector2.new(Notif_MainFrameAbsSize.X,Notif_MainFrameAbsSize.Y*.4))
            local NewSize = UDim2.new(0,Notif_MainFrameAbsSize.X,0,math.max(30,V2.Y))
            local NewPos = UDim2.new(0,Notif_MainFrameAbsPos.X,0,Notif_MainFrameAbsPos.Y)

            if not CheckForNotifOverlap(Notif_MainFrameAbsPos.Y+Notif_BaseFrameOffset+NewSize.Y.Offset) then
                repeat task.wait() until CheckForNotifOverlap(Notif_MainFrameAbsPos.Y+Notif_BaseFrameOffset+NewSize.Y.Offset)
                LoadInAnimTime = .05
            end

            Frame.Parent = NotificationGuiScreenFrame.ActiveNotificationFrames

            M["CreateSound"](11867106519,.25,plr,false,5,500,1,0,false,3,"UISFX",true) -- Notif Jingle

            Frame:TweenSizeAndPosition(NewSize,
                NewPos,"Out","Quad",LoadInAnimTime,false,function()
                    Frame:TweenPosition(UDim2.fromOffset(NewPos.X.Offset,NewPos.Y.Offset+Notif_MainFrameAbsSize.Y),"InOut","Linear",5,false,function()

                        M["CreateSound"](11867106620,.25,plr,false,5,500,.85,0,false,3,"UISFX",true) -- Notif Woosh

                        Frame:TweenSizeAndPosition(UDim2.new(0,0,0,0),UDim2.new(-.5,0,0,NewPos.Y.Offset+Notif_MainFrameAbsSize.Y),"In","Back",.5,false,function()
                            Frame:Destroy()
                        end)
                    end)
                end)
        end)      
    end

--]]

--// Lots of cool mouse stuff, like entering/leaving, clicking, tooltip gui, etc
--[[

    local function ResetMouse()
        UIFocus = nil
        Mouse_MainFrame.ToolTips:ClearAllChildren()
    end

    local function Mouse_AdjustMouseFrameToMouse()
        Mouse_MainFrame.Position = UDim2.new(0,Mouse.X-5,0,Mouse.Y+37)
    end

    local function SetAnchorPoint(Gui, DesiredAnchor)
        local Canvas = Gui.Parent
        local ParentSize = Canvas.AbsoluteSize

        local ParentPosition = Canvas.AbsolutePosition
        local ChildSize = Gui.AbsoluteSize
        local ChildPosition = Gui.AbsolutePosition

        ChildPosition = ChildPosition - ParentPosition

        local CorrectionOffsetX = ChildSize.X * DesiredAnchor.X
        local CorrectionOffsetY = ChildSize.Y * DesiredAnchor.Y

        local CorrectedUDim2 = UDim2.fromScale((ChildPosition.X + CorrectionOffsetX) / ParentSize.X, (ChildPosition.Y + CorrectionOffsetY) / ParentSize.Y)
        Gui.AnchorPoint = DesiredAnchor
        Gui.Position = CorrectedUDim2
    end

    local function Mouse_ButtonInteractDebris()
        for loop = 1,math.random(5,10) do
            local img = Ass.UI.ButtonClickDebrisFrame:Clone()
            _G.D:AddItem(img,.75)
            img.Rotation = math.random(-360,360)

            img.ZIndex = Mouse_MainFrame.ZIndex-1
            img.Parent = Mouse_MainFrame.Debris

            img:TweenSizeAndPosition(UDim2.new(0,0,0,0),UDim2.new(math.random(-500,500)/100,0,math.random(-500,500)/100,0),"Out","Quint",.75)
        end
    end

    local function Mouse_EnterButton(Button)
        local Magnitude = 1.25
        local OriginalSize,OrignalPosition = Button:GetAttribute("Mouse_LeftButtonOldSize"),Button:GetAttribute("Mouse_LeftButtonOldPosition")

        if Button:FindFirstAncestorWhichIsA("ScrollingFrame") then
            Magnitude -= (Magnitude-1)*2
        end

        Button:TweenSize(UDim2.fromScale(OriginalSize.X.Scale*Magnitude,OriginalSize.Y.Scale*Magnitude),"Out","Quint",.1,false)

        M["CreateSound"](1300447469,.5,plr,false,5,500,math.random(900,1100)/1000,0,false,3,"UISFX",true)
    end

    local function Mouse_ClickButton(Button,CircumventDebrisFunc)
        local Magnitude = .4
        local OriginalSize,OrignalPosition = Button:GetAttribute("Mouse_LeftButtonOldSize"),Button:GetAttribute("Mouse_LeftButtonOldPosition")

        Button:TweenSize(UDim2.fromScale(OriginalSize.X.Scale*Magnitude,OriginalSize.Y.Scale*Magnitude),"Out","Bounce",.05,true,function()
            Button:TweenSizeAndPosition(OriginalSize,OrignalPosition,"Out","Bounce",.05,true)
        end)
        M["CreateSound"](Button:GetAttribute("SoundToPlayID") or 956930900,.5,plr,false,5,500,Button:GetAttribute("SoundToPlayPlaybackSpeed") or 1.5,0,false,3,"UISFX"):Play()

        if not CircumventDebrisFunc then
            Mouse_ButtonInteractDebris()
        end
    end

    local function Mouse_LeaveButton(Button)
        Button:TweenSizeAndPosition(Button:GetAttribute("Mouse_LeftButtonOldSize"),Button:GetAttribute("Mouse_LeftButtonOldPosition"),"Out","Quint",.1,true)
    end

    local function Mouse_DisplayToolTip(TextObj,ToolTipObj,Text)
        Mouse_MainFrame.ToolTips:ClearAllChildren()
        if ToolTipObj:GetAttribute("Unique_AbilityName_NoChar") or ToolTipObj:GetAttribute("Unique_AbilityName_Char") and not plr.Character or ToolTipObj:GetAttribute("Unique_AbilityName_Char") and not plr.Character:FindFirstChild("MetaData")
        then
            local AbilityFolder = _G.REPS.Client_CharData:FindFirstChild(ToolTipObj:GetAttribute("Unique_CharName")).Abilities:FindFirstChild(ToolTipObj:GetAttribute("Unique_AbilityName_NoChar") or ToolTipObj:GetAttribute("Unique_AbilityName_Char"))
            if AbilityFolder then
                for Index,Func in pairs(M.TextStatModifierStringKeys) do
                    if string.match(Text,Index) then
                        Text = Func(Text,Index,AbilityFolder)
                    end
                end
            else
                warn("Stat Markdown Failure: Ability does not exist")
            end
        elseif ToolTipObj:GetAttribute("Unique_AbilityName_Char") then
            local AbilityFolder = plr.Character.MetaData.Abilities:FindFirstChild(ToolTipObj:GetAttribute("Unique_AbilityName_Char"))
            if AbilityFolder then
                for Index,Func in pairs(M.TextStatModifierStringKeys) do
                    if string.match(Text,Index) then
                        Text = Func(Text,Index,AbilityFolder,plr.Character.MetaData)
                    end
                end
            else
                warn("Stat Markdown Failure: Ability does not exist")
            end
        end
        local ToolTipUI = Ass.UI.ToolTipFrame:Clone()
        ToolTipUI.ToolTipText.Text = Text
        local ScaledTextSize_Abs = game:GetService("TextService"):GetTextSize(Text,ToolTipUI.ToolTipText.TextSize,ToolTipUI.ToolTipText.Font,Vector2.new(math.min(Client_ScreenSize.X/2,250),math.min(Client_ScreenSize.Y/2,1000)))+ Vector2.new(5,5)
        ToolTipUI:SetAttribute("ScaledTextSize_Abs",ScaledTextSize_Abs)
        ToolTipUI.Position = UDim2.new(0,0,0,0) + UDim2.fromOffset(35,-35)
        ToolTipUI.Parent = Mouse_MainFrame.ToolTips

        _G.TS:Create(ToolTipUI,ToolTip_ExpandStyle,{Size = UDim2.fromOffset(math.max(ScaledTextSize_Abs.X,50),math.max(ScaledTextSize_Abs.Y,25))}):Play()
    end

    local function Mouse_AdjustToolTipPos()
        for _,ToolTipUI in pairs(Mouse_MainFrame.ToolTips:GetChildren()) do
            local YPos = ToolTipUI.AbsolutePosition.Y
            local XPos = ToolTipUI.AbsolutePosition.X

            if YPos+ToolTipUI.AbsoluteSize.Y >= Client_ScreenSize.Y+GUIINSETY.Y then
                ToolTipUI.Position = UDim2.new(ToolTipUI.Position.X.Scale,ToolTipUI.Position.X.Offset,
                    ToolTipUI.Position.Y.Scale,0-ToolTipUI:GetAttribute("ScaledTextSize_Abs").Y
                )
            elseif YPos+GUIINSETY.Y  <= 0 then
                ToolTipUI.Position = UDim2.new(ToolTipUI.Position.X.Scale,ToolTipUI.Position.X.Offset,
                    ToolTipUI.Position.Y.Scale,0+ToolTipUI.Parent.AbsoluteSize.Y
                )
            end

            if XPos+ToolTipUI.AbsoluteSize.X >= Client_ScreenSize.X then
                ToolTipUI.Position = UDim2.new(ToolTipUI.Position.X.Scale,
                    0-ToolTipUI:GetAttribute("ScaledTextSize_Abs").X,
                    ToolTipUI.Position.Y.Scale,ToolTipUI.Position.Y.Offset
                )
            elseif XPos <= 0 then
                ToolTipUI.Position = UDim2.new(ToolTipUI.Position.X.Scale,
                    0+ToolTipUI.Parent.AbsoluteSize.X,
                    ToolTipUI.Position.Y.Scale,ToolTipUI.Position.Y.Offset
                )
            end
        end
    end

    local function Mouse_ConnectMouseEvents(Obj)
        if not Obj or not Obj.Parent or not Obj:IsA("GuiObject") then return end
        if plr.PlayerGui:FindFirstChild("TouchGui") and Obj:IsDescendantOf(plr.PlayerGui.TouchGui) then return end
        if table.find(Mouse_ConnectedUI,Obj) then
            warn("Already connected:",Obj)
            return
        end

        if Obj:IsA("TextLabel") or Obj:IsA("TextButton") then
            for TextToMatch,Func in pairs(RichText_KeyWordTriggers) do
                --Obj.RichText = true
                if Obj.RichText then
                    local Start,End = string.find(_sl(Obj.Text),_sl(TextToMatch))
                    if Start then
                        local Word = string.sub(Obj.Text,Start,End)
                        Obj.Text = string.gsub(Obj.Text,Word,Func)
                    end
                end
            end
        end


        if Obj:IsDescendantOf(Mouse_MainFrame) then return end

        if Obj:FindFirstChild(M.ToolTipNameKey) then 
            Obj.Active = true
        end

        if Obj:IsA("ImageButton") or Obj:IsA("TextButton") then

            if Obj.AnchorPoint.X <= 0 and Obj.AnchorPoint.Y <= 0 then -- Pass over Objs that already have assigned anchor points
                SetAnchorPoint(Obj,Vector2.new(.5,.5))
            end

            Obj:SetAttribute("Mouse_LeftButtonOldSize",Obj.Size)
            Obj:SetAttribute("Mouse_LeftButtonOldPosition",Obj.Position)

            Obj[Client_InputMode]:Connect(function(i)
                if i.UserInputType == Enum.UserInputType.Touch then
                    Mouse_MainFrame.Position = UDim2.fromOffset(Obj.AbsolutePosition.X,Obj.AbsolutePosition.Y)
                    if Obj:GetAttribute("IgnoreMouseClick") then return end
                    Mouse_ClickButton(Obj)
                elseif i.UserInputType == Enum.UserInputType.MouseButton1 or i.KeyCode == Enum.KeyCode.ButtonA then
                    if Obj:GetAttribute("IgnoreMouseClick") then return end
                    Mouse_ClickButton(Obj)
                end
            end)

            if Client_DeviceType ~= "Mobile" then
                Obj.MouseEnter:Connect(function()
                    UIFocus = Obj
                    Mouse_EnterButton(Obj)
                    if Obj:FindFirstChild(M.ToolTipNameKey) then
                        Mouse_DisplayToolTip(Obj,Obj[M.ToolTipNameKey],Obj[M.ToolTipNameKey].Value)
                    end
                end)
            else
                Obj.TouchLongPress:Connect(function()
                    UIFocus = Obj
                    Mouse_MainFrame.Position = UDim2.fromOffset(Obj.AbsolutePosition.X,Obj.AbsolutePosition.Y)
                    if not Obj:FindFirstChild(M.ToolTipNameKey) then return end
                    Mouse_DisplayToolTip(Obj,Obj[M.ToolTipNameKey],Obj[M.ToolTipNameKey].Value)
                end)
            end

            Obj.Destroying:Connect(function()
                if UIFocus ~= Obj then return end
                ResetMouse()
            end)

            Obj.MouseLeave:Connect(function()
                Mouse_LeaveButton(Obj)
                ResetMouse()
            end)

        elseif Obj:IsA("GuiObject") or Obj:IsA("TextLabel") or Obj:IsA("TextBox") then

            Obj[Client_InputMode]:Connect(function(i)
                if i.UserInputType == Enum.UserInputType.Touch then
                    Mouse_MainFrame.Position = UDim2.fromOffset(Obj.AbsolutePosition.X,Obj.AbsolutePosition.Y)
                elseif i.UserInputType == Enum.UserInputType.MouseButton1 or i.KeyCode == Enum.KeyCode.ButtonA then
                    -- if Obj:GetAttribute("IgnoreMouseClick") then return end
                    -- Mouse_ClickButton(Obj)
                end
            end)

            if Client_DeviceType ~= "Mobile" then
                Obj.MouseEnter:Connect(function()
                    UIFocus = Obj
                    if Obj:FindFirstChild(M.ToolTipNameKey) then
                        Mouse_DisplayToolTip(Obj,Obj[M.ToolTipNameKey],Obj[M.ToolTipNameKey].Value)
                    end
                end)
            else
                Obj.TouchLongPress:Connect(function()
                    UIFocus = Obj
                    Mouse_MainFrame.Position = UDim2.fromOffset(Obj.AbsolutePosition.X,Obj.AbsolutePosition.Y)
                    if not Obj:FindFirstChild(M.ToolTipNameKey) then return end
                    Mouse_DisplayToolTip(Obj,Obj[M.ToolTipNameKey],Obj[M.ToolTipNameKey].Value)
                end)
            end
            Obj.Destroying:Connect(function()
                if UIFocus ~= Obj then return end
                ResetMouse()
            end)
            Obj.MouseLeave:Connect(function()
                -- if Obj:FindFirstChild(M.ToolTipNameKey) then
                ResetMouse()
                --end
            end)

        end
    end
--]]

--// Do things if player loses X% of HP

--[[

            local DamageThresholds = {
                MetaData.MaxHealth.Value*.10,
                MetaData.MaxHealth.Value*.25,
                MetaData.MaxHealth.Value*.35,
            }
            local DamageThresholdsMap = {
                [1] = "Minor_Injured",
                [2] = "Medium_Injured",
                [3] = "Major_Injured",
            }

            local Netting = false
            local LastMaxHP = MetaData.MaxHealth.Value
            local LastHealth = MetaData.Health.Value
            MetaData.Health:GetPropertyChangedSignal("Value"):Connect(function()
                if Netting then return end
                Netting = true
                task.wait(.25)
                Netting = false


                local MaxHPDamage = MetaData.MaxHealth.Value-LastMaxHP
                local Damage = MetaData.Health.Value-LastHealth

                if Damage < 0 and MaxHPDamage-Damage ~= 0 then

                    local DamageIndexStr = nil

                    Damage=math.abs(Damage)

                    for Index,Threshold in pairs(DamageThresholds) do
                        if Damage > Threshold and Index<#DamageThresholds then
                            if Damage < DamageThresholds[Index+1] then
                                DamageIndexStr = DamageThresholdsMap[Index]
                                break
                            else 
                                continue 
                            end
                        elseif Damage > Threshold and Index>=#DamageThresholds then
                            DamageIndexStr = DamageThresholdsMap[Index]
                            break
                        elseif Damage <= Threshold then
                            DamageIndexStr = DamageThresholdsMap[Index]
                            break
                        end
                    end

                    if DamageIndexStr then
                        _G.EVENTS.ClientShakeCamera:Fire(DamageIndexStr)
                    else
                        warn("Could not calculate incoming damage threshold index")
                    end

                end

                LastMaxHP = MetaData.MaxHealth.Value
                LastHealth = MetaData.Health.Value

            end)
--]]


--// Some of the loading player code, serverside
--[[

local UserDataChangesCalls = {
    ["PropertyChangedSignal"] = {
        ["Wins"] = function(plr,DataFolder)
            AH:CheckForFirstWin(plr,DataFolder)
        end,
        ["MVPS"] = function(plr,DataFolder)
            AH:CheckForFirstMVP(plr,DataFolder)
        end,
    },
    ["ChildAddedOrRemoved"] = {
        ["OwnedCharacters"] = function(plr,DataFolder)
            AH:CollectorQuery(plr,DataFolder)
            AH:MasterCollectorQuery(plr,DataFolder)
        end,
        ["OwnedSkins"] = function(plr,DataFolder)
            AH:CollectorQuery(plr,DataFolder)
            AH:MasterCollectorQuery(plr,DataFolder)
        end,
        ["Queries"] = function(plr,DataFolder)
            AH:CollectorQuery(plr,DataFolder)
            AH:MasterCollectorQuery(plr,DataFolder)
        end,
    }
}

local function LoadPlayer(plr)
    print("Loading",plr,"data...")

    local PlrData = PFSDataStore:LoadProfileAsync(tostring(plr.UserId),"ForceLoad")

    if PlrData then -- if player data loaded

        PlrData:ListenToRelease(function()

            PlayerData[PlrData] = nil
            plr:Kick()
        end)

        if M:IsPlayerPresent(plr) then
            PlayerData[PlrData] = PlrData
            PlrData:AddUserId(plr.UserId) -- GDPR compliance
            PlrData:Reconcile()
        else-- if data loaded but player left
            warn("Data loading failure: "..plr.Name.." left during data-load process")
            PlrData:Release()
            return
        end

    else-- if data loading failed

        warn("Data loading failure: "..plr.Name.."'s data could not be loaded")
        if M:IsPlayerPresent(plr) then
            game:GetService("TeleportService"):TeleportAsync(game.PlaceId,{plr},M.TeleportOption_IfDataFails)
        end

        return
    end

    local _,DataFolder = M:Convert_TableToFolder(PlrData.Data,Meta.DataFolder:Clone())
    local ClientConnections = {}

    local FacultyStatus:nil | {any} = M:IsPlrTesterOrAdmin(plr)
    if FacultyStatus then
        local V = Instance.new("StringValue")
        V.Value = FacultyStatus[1]
        V.Name = "IsFaculty"
        V.Parent = DataFolder
    end

    HandleDefaultPlrChar(plr)

    ClientConnections[#ClientConnections+1]= _G.P.PlayerRemoving:Connect(function(player)
        if player ~= plr then return end
        for _,Con in pairs(ClientConnections) do
            Con:Disconnect()
            Con = nil
        end
        if PlrData then
            PlrData:Release()
        end
    end)

    local function UpdateChangesToUserDataInTable_Table(Table,Index,CorrespondingUserData)
        Table[Index] = {} -- Clear the table inside player data, will later be reconstructed as a data file
        for _,Obj in pairs(CorrespondingUserData:GetChildren()) do -- looping through folder with userdata objects in it
            Table[Index][Obj.Name] = Obj.Value -- assign shallow data to table
        end
    end
    local function UpdateChangesToUserDataInTable_NonTable(Table,Index,CorrespondingUserData)
        Table[Index] = CorrespondingUserData.Value -- update data of existing variable inside player data
    end

    local function AttachListenersToData(Table) -- recursively search and attach 
        for Index,Variable in pairs(Table) do

            local CorrespondingUserData = DataFolder:FindFirstChild(Index,true) -- match variable's index with an object inside the players DataFolder

            if CorrespondingUserData then

                if M:Typeof(Variable) == "table" then -- will be a folder obj
                    ClientConnections[#ClientConnections+1] = CorrespondingUserData.ChildAdded:Connect(function()
                        UpdateChangesToUserDataInTable_Table(Table,Index,CorrespondingUserData)
                        local Call = UserDataChangesCalls.ChildAddedOrRemoved[CorrespondingUserData.Name]
                        if Call then
                            Call(plr,DataFolder)
                        end
                    end)
                    ClientConnections[#ClientConnections+1] = CorrespondingUserData.ChildRemoved:Connect(function()
                        UpdateChangesToUserDataInTable_Table(Table,Index,CorrespondingUserData)
                        local Call = UserDataChangesCalls.ChildAddedOrRemoved[CorrespondingUserData.Name]
                        if Call then
                            Call(plr,DataFolder)
                        end
                    end)

                    AttachListenersToData(Variable) -- recurse this local table
                else -- Will not be a folder obj, will be an intvalue, numbervalue, stringvalue, boolvalue, etc
                    ClientConnections[#ClientConnections+1] = CorrespondingUserData:GetPropertyChangedSignal("Value"):Connect(function()
                        UpdateChangesToUserDataInTable_NonTable(Table,Index,CorrespondingUserData)
                        local Call = UserDataChangesCalls.PropertyChangedSignal[CorrespondingUserData.Name]
                        if Call then
                            Call(plr,DataFolder)
                        end
                    end)
                end



            end
        end

        return Table
    end

    AttachListenersToData(PlrData.Data)

    for id,Function in pairs(M.DevProducts.Gamepasses) do
        if not DataFolder.Queries:FindFirstChild(Function(nil,nil,nil,true)) and _G.MPS:UserOwnsGamePassAsync(plr.UserId,id) then -- specific check on load-in
            Function(plr,DataFolder,true)
        end
    end

    local QueryStr = "Cool Before Cool Was Cool"
    if not DataFolder.Queries:FindFirstChild(QueryStr) then
        M:CreateAsset(plr,DataFolder.Queries,QueryStr)
        _G.BS:AwardBadge(plr.UserId,AH.BadgesList[QueryStr])
    end

    QueryStr = "Welcome to Hero Battles!"
    if not DataFolder.Queries:FindFirstChild(QueryStr) then
        M:CreateAsset(plr,DataFolder.Queries,QueryStr)
        _G.BS:AwardBadge(plr.UserId,AH.BadgesList[QueryStr])
    end

    DataFolder.Parent = plr
    print(plr,"data loading successful!")

    return true
end

--]]

return M

```
