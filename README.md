-- PARKOUR+TANQUE compact | run 38 fixo | NC duro | cam torreta | anti fling/queda
local Players=game:GetService("Players")
local RS=game:GetService("RunService")
local UIS=game:GetService("UserInputService")
local TS=game:GetService("TweenService")
local PS=game:GetService("PhysicsService")
local player=Players.LocalPlayer
local env=(getgenv and getgenv()) or _G
if env.ParkourCleanup then pcall(env.ParkourCleanup) end
pcall(function() loadstring(game:HttpGet("https://scriptblox.com/raw/Universal-Script-Fe-Ugc-Animations-242778"))() end)

local CLIMB="rbxassetid://121075390792786"
local IDS={
	idle="rbxassetid://89598513816669",walk="rbxassetid://87493540718854",run="rbxassetid://127998320083590",
	jump=CLIMB,climb=CLIMB,fall=CLIMB,crawl="rbxassetid://109318260839099",
	crouchIdle="rbxassetid://82327422558552",crouchWalk="rbxassetid://99452665172764",
	flip="rbxassetid://76842517659641",slide="rbxassetid://79078317067813",sit="rbxassetid://112314297001065",
	laugh="rbxassetid://98252592747900",
}
local TANK_ANIM={
	Tiro="rbxassetid://133488754387819",WalkTank="rbxassetid://73379618463325",
	IdleTank="rbxassetid://107822660492695",TransformTank="rbxassetid://101155521231914",
}

local RUN_S,RUN_M=16,38
local CRAWL_SP,CROUCH_SP,CLIMB_SP,CLIMB_SIDE,CLIMB_OFF=12,8,12,8,0.8
local HEAD_CHK,WALL_D,JUMP_H,JUMP_V,FLIP_T,FLIP_S=1.8,2.2,22,20,0.35,1.2
local SLIDE_MIN,SLIDE_BST,SLIDE_T0,SLIDE_T1=14,1.1,0.6,1.5
local CROUCH_HB,SLIDE_HB=0.7,0.5
local SHIFT_OFF=Vector3.new(1.75,0,0)
local NC_D,NC_K,NC_MAX,NC_SNAP=5.5,42,80,0.92
local T_TURN,T_YAW,T_YSPD=100,90,48
local MAX_MP,MAX_MR,MAX_AP,MAX_AR=30,24,26,16
local P_SM,H_K,H_SM,H_UP,H_DN=9.5,14,10,35,28
local TJ_F,TJ_D,ALT_S,ALT_MN,ALT_MX=52,0.22,0.55,-3,10
local BASE_H,MIN_H,TF_S,MAX_SNAP,FALL_DELAY=3.15,2.35,3,6.5,0.12
local S_MAX,S_MID,S_NEAR,S_TOUCH=90,35,12,3.5
local S_SAFE,S_MIDV,S_FAR,S_MIN=-10,-22,-45,-18
local GHOST,GUARD_LOOSE,LOOSE_R,LOOSE_LIN,LOOSE_ANG=true,true,30,250,120
local VEL_GUARD,MAX_HZ,MAX_UP,MAX_ANG,SNAP,BLOCK_FALL=true,120,120,40,8,true

local function mk(c,p,par)
	local o=Instance.new(c)
	for k,v in pairs(p) do o[k]=v end
	if par then o.Parent=par end
	return o
end
local function corner(p,r) return mk("UICorner",{CornerRadius=UDim.new(0,r or 8)},p) end
local function stroke(p,c,t,tr) return mk("UIStroke",{Color=c,Thickness=t or 1,Transparency=tr or 0.3},p) end
local function tween(o,pr,t) TS:Create(o,TweenInfo.new(t,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),pr):Play() end
local conns,running={},true
local function bind(s,f) local c=s:Connect(f) conns[#conns+1]=c return c end

local char,hum,root,ready,mode,ocupado
local tracks,tankTracks={},{}
local current,airCurrent,airTime=nil,nil,0
local crawlOn,crouching,shiftLock=false,false,false
local climbing,vaulting,sliding=false,false,false
local isRunning,wantRun,blockSit,sitting,onNC,onWater,laughing=false,false,false,false,false,false,false
local climbN,climbStart,lastExit,flipStart,lastSlide=Vector3.yAxis,0,0,0,0
local slideStart,slideDir,slideSp0,slideDur,slideNow,slideAir=0,Vector3.zero,0,1,0,0
local curSpeed,orig,baseSpeed,baseHip=RUN_S,{},16,0
local rayP=RaycastParams.new()
rayP.FilterType=Enum.RaycastFilterType.Exclude
rayP.RespectCanCollide=false
local tankYaw,yaw,tankPitch,tankRoll,tankPOff,tankROff=0,0,0,0,0,0
local LIFT,tankHover={tanque=0},BASE_H
local tankVel,tankAtt,tankOri,tankJump,tankJCD=nil,nil,nil,false,0
local cannonOn,cannonPitch,devMode,autoH,lastGY,smVy=false,0.5,false,false,nil,0
local mTF,mTB,mTL,mTR,mL,mR,mU,mD=false,false,false,false,false,false,false,false
local updateButtons,lastCF,lastVel=nil,nil,nil
local turretCam,savedCamType,savedCamSubject=false,nil,nil
local cam=workspace.CurrentCamera

local CG="ParkourChar"
pcall(function()
	PS:RegisterCollisionGroup(CG)
	PS:CollisionGroupSetCollidable(CG,CG,false)
	PS:CollisionGroupSetCollidable(CG,"Default",true)
end)
local function setGroup(p) if p:IsA("BasePart") then pcall(function() p.CollisionGroup=CG end) end end
local function applyLocal(c)
	if not c then return end
	for _,d in ipairs(c:GetDescendants()) do setGroup(d) end
	c.DescendantAdded:Connect(setGroup)
end
local flingCache={}
local function trackFling(ch)
	if flingCache[ch] then return end
	local set={}
	flingCache[ch]=set
	for _,d in ipairs(ch:GetDescendants()) do if d:IsA("BasePart") then set[d]=true end end
	bind(ch.DescendantAdded,function(d) if d:IsA("BasePart") then set[d]=true end end)
	bind(ch.DescendantRemoving,function(d) set[d]=nil end)
	bind(ch.AncestryChanged,function() if not ch:IsDescendantOf(game) then flingCache[ch]=nil end end)
end
local function hookFling(pl)
	if pl==player then return end
	bind(pl.CharacterAdded,trackFling)
	if pl.Character then trackFling(pl.Character) end
end
for _,pl in ipairs(Players:GetPlayers()) do hookFling(pl) end
bind(Players.PlayerAdded,hookFling)
local function stripOther(c)
	if not c then return end
	for _,d in ipairs(c:GetDescendants()) do
		if d:IsA("BasePart") then d.CanCollide=false; setGroup(d) end
	end
	c.DescendantAdded:Connect(function(d)
		if d:IsA("BasePart") then d.CanCollide=false; setGroup(d) end
	end)
end
local function refreshGhost()
	for _,pl in ipairs(Players:GetPlayers()) do
		local c=pl.Character
		if c then if pl==player then applyLocal(c) else stripOther(c) end end
	end
end
local function setupPl(pl)
	pl.CharacterAdded:Connect(function(c)
		task.defer(function()
			if pl==player then applyLocal(c) else stripOther(c) end
		end)
	end)
	if pl.Character then
		if pl==player then applyLocal(pl.Character) else stripOther(pl.Character) end
	end
end
for _,pl in ipairs(Players:GetPlayers()) do setupPl(pl) end
Players.PlayerAdded:Connect(setupPl)

local function lockFall()
	if BLOCK_FALL and hum then
		pcall(function()
			hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown,false)
			hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll,false)
		end)
	end
end
local function restoreHB()
	if not (root and hum and orig.size) then return end
	root.Size=orig.size
	hum.HipHeight=orig.hip or baseHip
	if orig.partCollide then
		for p,can in pairs(orig.partCollide) do
			if p and p.Parent then pcall(function() p.CanCollide=can end) end
		end
	end
end
local function isWaterHit(hit)
	if not hit then return false end
	if hit.Material==Enum.Material.Water then return true end
	if hit.Instance:IsA("BasePart") then
		local n=string.lower(hit.Instance.Name)
		if n:find("water") or n:find("agua") or n:find("água") or n:find("swim") or n:find("lake") or n:find("ocean") or n:find("pool") then return true end
		if hit.Instance.Material==Enum.Material.Water then return true end
	end
	return false
end
local function detectWater()
	if not (root and hum and char) then return false end
	if hum.FloorMaterial==Enum.Material.Water or hum:GetState()==Enum.HumanoidStateType.Swimming then return true end
	local p=RaycastParams.new()
	p.FilterType=Enum.RaycastFilterType.Exclude
	p.FilterDescendantsInstances={char}
	p.RespectCanCollide=false
	local h=workspace:Raycast(root.Position+Vector3.new(0,1,0),Vector3.new(0,-6,0),p)
	return isWaterHit(h) or isWaterHit(workspace:Raycast(root.Position,Vector3.new(0,2,0),p))
end

local gui=mk("ScreenGui",{Name="ParkourTankGui",ResetOnSpawn=false,DisplayOrder=50})
pcall(function() gui.Parent=(gethui and gethui()) or game.CoreGui end)
if not gui.Parent then gui.Parent=player:WaitForChild("PlayerGui") end
local GREEN=Color3.fromRGB(0,255,140)
local COR_ON,COR_OFF=Color3.fromRGB(0,170,100),Color3.fromRGB(30,30,40)
local mira=mk("Frame",{Size=UDim2.new(0,3,0,3),Position=UDim2.new(0.5,-1.5,0.5,-1.5),BackgroundColor3=GREEN,BorderSizePixel=0,Visible=false},gui)
corner(mira,99)
mk("Frame",{Size=UDim2.new(0,14,0,2),Position=UDim2.new(0.5,-7,0.5,-1),BackgroundColor3=GREEN,BackgroundTransparency=0.3,BorderSizePixel=0},mira)
mk("Frame",{Size=UDim2.new(0,2,0,14),Position=UDim2.new(0.5,-1,0.5,-7),BackgroundColor3=GREEN,BackgroundTransparency=0.3,BorderSizePixel=0},mira)
local dot=mk("Frame",{Size=UDim2.fromOffset(6,6),AnchorPoint=Vector2.new(0.5,0.5),Position=UDim2.fromScale(0.5,0.5),BackgroundColor3=Color3.new(1,1,1),BackgroundTransparency=0.2,BorderSizePixel=0,Visible=false},gui)
corner(dot,99)

-- estilo parkour (círculo)
local function stylePark(b)
	b.BackgroundColor3=COR_OFF
	b.BackgroundTransparency=0.15
	b.TextColor3=Color3.new(1,1,1)
	b.Font=Enum.Font.GothamBold
	b.AutoButtonColor=false
	corner(b,99)
	stroke(b,Color3.new(1,1,1),1.5,0.6)
end

local pBtns={}
local function mkPB(key,label,col,row,cb)
	local b=mk("TextButton",{
		Name=key,Size=UDim2.fromOffset(62,62),Position=UDim2.new(1,-140+col*68,0.1,row*68),
		Text=label,TextScaled=true
	},gui)
	stylePark(b)
	local pad=Instance.new("UIPadding",b)
	pad.PaddingLeft=UDim.new(0,6); pad.PaddingRight=UDim.new(0,6)
	pad.PaddingTop=UDim.new(0,18); pad.PaddingBottom=UDim.new(0,18)
	bind(b.MouseButton1Click,cb)
	pBtns[key]=b
end

local function mkTB(text,pos,size,cb)
	local b=mk("TextButton",{Size=size or UDim2.fromOffset(62,62),Position=pos,Text=text,TextSize=14,Visible=false},gui)
	stylePark(b)
	if cb then bind(b.MouseButton1Click,cb) end
	return b
end

local bTank=mk("TextButton",{Size=UDim2.fromOffset(90,62),Position=UDim2.new(0,14,0,18),Text="TANQUE",TextSize=14},gui)
stylePark(bTank)
local bVoltar=mkTB("Voltar",UDim2.new(1,-175,0,18),UDim2.fromOffset(90,62))
local bDev=mkTB("⚙️",UDim2.new(1,-250,0,18),UDim2.fromOffset(62,62))
local bAuto=mkTB("AUTO",UDim2.new(1,-325,0,18),UDim2.fromOffset(62,62))
local bCanhao=mkTB("Canhão",UDim2.new(1,-175,0,90),UDim2.fromOffset(90,62))
local bTiro=mkTB("Tiro",UDim2.new(1,-250,0,90),UDim2.fromOffset(62,62))
local bTurretCam=mkTB("Cam",UDim2.new(1,-325,0,90),UDim2.fromOffset(62,62))

local function mbtn(txt,pos)
	local b=mkTB(txt,pos,UDim2.fromOffset(62,62))
	b.TextSize=22
	return b
end
local btnTF=mbtn("▲",UDim2.new(0,88,1,-228))
local btnTL=mbtn("←",UDim2.new(0,20,1,-160))
local btnTR=mbtn("→",UDim2.new(0,156,1,-160))
local btnTB=mbtn("▼",UDim2.new(0,88,1,-92))
local btnL=mbtn("◄",UDim2.new(1,-172,1,-160))
local btnR=mbtn("►",UDim2.new(1,-102,1,-160))
local btnU=mbtn("↑",UDim2.new(0,18,1,-290))
local btnD=mbtn("↓",UDim2.new(0,18,1,-220))
local function hold(b,set)
	b.MouseButton1Down:Connect(function() set(true) end)
	b.MouseButton1Up:Connect(function() set(false) end)
	b.MouseLeave:Connect(function() set(false) end)
end
hold(btnTF,function(v) mTF=v end); hold(btnTB,function(v) mTB=v end)
hold(btnTL,function(v) mTL=v end); hold(btnTR,function(v) mTR=v end)
hold(btnL,function(v) mL=v end); hold(btnR,function(v) mR=v end)
hold(btnU,function(v) mU=v end); hold(btnD,function(v) mD=v end)

local pitchLabel=mk("TextLabel",{Size=UDim2.new(0,160,0,16),Position=UDim2.new(0.5,-80,1,-118),BackgroundTransparency=1,Text="Inclinação: 0°",TextColor3=Color3.fromRGB(0,220,255),Font=Enum.Font.GothamBold,TextSize=12,Visible=false},gui)
local function dbtn(t,pos)
	local b=mkTB(t,pos,UDim2.fromOffset(62,34))
	b.TextSize=11
	return b
end
local bPF1=dbtn("Frente +",UDim2.new(0.5,-140,1,-96))
local bPF2=dbtn("Frente ++",UDim2.new(0.5,-72,1,-96))
local bPB1=dbtn("Trás +",UDim2.new(0.5,10,1,-96))
local bPB2=dbtn("Trás ++",UDim2.new(0.5,78,1,-96))
local altLabel=mk("TextLabel",{Size=UDim2.new(0,160,0,16),Position=UDim2.new(0.5,-80,1,-72),BackgroundTransparency=1,Text="Altura: 0.00",TextColor3=Color3.fromRGB(255,215,0),Font=Enum.Font.GothamBold,TextSize=12,Visible=false},gui)
local function abtn(t,pos)
	local b=mkTB(t,pos,UDim2.fromOffset(52,34))
	b.TextSize=11
	return b
end
local bE1=abtn("Esq +",UDim2.new(0.5,-178,1,-52))
local bE2=abtn("Esq ++",UDim2.new(0.5,-120,1,-52))
local bAM=abtn("Alt -",UDim2.new(0.5,-52,1,-52))
local bAP=abtn("Alt +",UDim2.new(0.5,6,1,-52))
local bD1=abtn("Dir +",UDim2.new(0.5,70,1,-52))
local bD2=abtn("Dir ++",UDim2.new(0.5,128,1,-52))

task.spawn(function()
	while running and gui and gui.Parent do
		pcall(refreshGhost)
		lockFall()
		task.wait(0.5)
	end
end)

local overlap=OverlapParams.new()
overlap.FilterType=Enum.RaycastFilterType.Exclude
overlap.MaxParts=150
overlap.RespectCanCollide=true
bind(RS.Stepped,function()
	if GHOST then
		for _,set in pairs(flingCache) do
			for p in pairs(set) do
				if p and p.Parent and p.CanCollide then p.CanCollide=false end
			end
		end
	end
	if GUARD_LOOSE and root and root.Parent then
		local ok,found=pcall(workspace.GetPartBoundsInRadius,workspace,root.Position,LOOSE_R,overlap)
		if ok and found then
			for _,p in ipairs(found) do
				if not p.Anchored and p.CanCollide and (p.AssemblyLinearVelocity.Magnitude>LOOSE_LIN or p.AssemblyAngularVelocity.Magnitude>LOOSE_ANG) then
					p.CanCollide=false
				end
			end
		end
	end
end)

local function flatSp()
	local v=root.AssemblyLinearVelocity
	return Vector3.new(v.X,0,v.Z).Magnitude
end
local function applyHB(m)
	if mode=="tanque" or not (root and hum and orig.size) then return end
	local size,hip,f=orig.size,orig.hip,nil
	if m=="crouch" then f=CROUCH_HB elseif m=="slide" then f=SLIDE_HB end
	if f then
		local ny=orig.size.Y*f
		size=Vector3.new(orig.size.X,ny,orig.size.Z)
		hip=orig.hip+(orig.size.Y-ny)/2
	elseif m=="climb" then size=Vector3.new(orig.size.X,orig.size.Y,orig.size.Z*0.6)
	elseif m=="normal" then size,hip=orig.size,orig.hip end
	tween(root,{Size=size},0.15)
	pcall(function() tween(hum,{HipHeight=hip},0.15) end)
end
local function playAir(n)
	if airCurrent==n then return end
	if airCurrent and tracks[airCurrent] then tracks[airCurrent]:Stop(0.08) end
	airCurrent=n
	if n and tracks[n] then tracks[n]:Play(0.08) end
end
local function playGnd(n)
	if current==n and (not n or (tracks[n] and tracks[n].IsPlaying)) then return end
	if current and tracks[current] and current~=n then tracks[current]:Stop(0.1) end
	current=n
	if n and tracks[n] then tracks[n]:Play(0.1) end
end
local function stopAirHard()
	airTime=0
	if airCurrent and tracks[airCurrent] then tracks[airCurrent]:Stop(0) end
	airCurrent=nil
	if tracks.fall and tracks.fall.IsPlaying then tracks.fall:Stop(0) end
	if tracks.jump and tracks.jump.IsPlaying then tracks.jump:Stop(0) end
end
local function loadPark()
	tracks={}
	local an=hum:FindFirstChildOfClass("Animator") or Instance.new("Animator",hum)
	local cfg={
		idle={IDS.idle,Enum.AnimationPriority.Movement,true},walk={IDS.walk,Enum.AnimationPriority.Action,true},
		run={IDS.run,Enum.AnimationPriority.Action,true},jump={IDS.jump,Enum.AnimationPriority.Action2,true},
		fall={IDS.fall,Enum.AnimationPriority.Action2,true},crawl={IDS.crawl,Enum.AnimationPriority.Action2,true},
		crouchIdle={IDS.crouchIdle,Enum.AnimationPriority.Action3,true},crouchWalk={IDS.crouchWalk,Enum.AnimationPriority.Action3,true},
		climb={IDS.climb,Enum.AnimationPriority.Action4,true},flip={IDS.flip,Enum.AnimationPriority.Action4,false},
		slide={IDS.slide,Enum.AnimationPriority.Action4,true},sit={IDS.sit,Enum.AnimationPriority.Action4,true},
		laugh={IDS.laugh,Enum.AnimationPriority.Action4,false},
	}
	for n,c in pairs(cfg) do
		local a=Instance.new("Animation"); a.AnimationId=c[1]
		local ok,t=pcall(function() return an:LoadAnimation(a) end)
		if ok and t then t.Looped=c[3]; t.Priority=c[2]; tracks[n]=t end
	end
	if tracks.laugh then
		tracks.laugh.Stopped:Connect(function()
			laughing=false
			if updateButtons then updateButtons() end
		end)
	end
end
local function loadTank()
	tankTracks={}
	local an=hum:FindFirstChildOfClass("Animator") or Instance.new("Animator",hum)
	for n,id in pairs(TANK_ANIM) do
		local a=Instance.new("Animation"); a.AnimationId=id
		local ok,t=pcall(function() return an:LoadAnimation(a) end)
		if ok and t then
			t.Priority=Enum.AnimationPriority.Action4
			t.Looped=(n~="Tiro" and n~="TransformTank")
			tankTracks[n]=t
		end
	end
end
local function castWall(o,d)
	local r=workspace:Raycast(o,d,rayP)
	if not r then return nil end
	if r.Instance:IsA("TrussPart") or math.abs(r.Normal.Y)>0.35 then return nil end
	local m=r.Instance:FindFirstAncestorOfClass("Model")
	if m and m~=workspace and m:FindFirstChildOfClass("Humanoid") then return nil end
	return r
end
local function blockedAbove()
	return workspace:Raycast(root.Position,Vector3.new(0,3,0),rayP)~=nil
end

-- CORRIDA FIXA 38 (sem acelerar devagar)
local function applySpeed()
	local target
	if crouching then target=CROUCH_SP
	elseif crawlOn then target=CRAWL_SP
	elseif wantRun then target=RUN_M
	else target=RUN_S end
	if hum.WalkSpeed~=target then hum.WalkSpeed=target end
	curSpeed=target
	isRunning=wantRun and not crouching and not crawlOn
end

local function updateGnd(dt)
	if sitting or laughing then return end
	onWater=detectWater()
	local mov=hum.MoveDirection.Magnitude>0.1
	local gnd=hum.FloorMaterial~=Enum.Material.Air or onNC
	if onWater then
		playGnd(nil)
		if airCurrent~="fall" then playAir("fall") end
		if tracks.fall then tracks.fall:AdjustSpeed(mov and 1 or 0.35) end
		return
	end
	if not gnd then
		playGnd(nil); airTime+=dt
		local vy=root.AssemblyLinearVelocity.Y
		if not airCurrent then
			if vy>4 then playAir("jump")
			elseif airTime>FALL_DELAY then playAir("fall") end
		elseif airCurrent=="jump" and vy<-3 then playAir("fall") end
		return
	end
	stopAirHard()
	local f=tracks.flip
	if f and f.IsPlaying and tick()-flipStart>0.3 then f:Stop(0.1) end
	local want=crouching and (mov and "crouchWalk" or "crouchIdle")
		or crawlOn and "crawl"
		or mov and (isRunning and "run" or "walk")
		or "idle"
	playGnd(want)
	local t=tracks[want]
	if t then
		if want=="crawl" then t:AdjustSpeed(mov and math.clamp(hum.WalkSpeed/16,0.5,3) or 0)
		elseif want=="walk" then t:AdjustSpeed(math.clamp(hum.WalkSpeed/16,0.6,1.6))
		elseif want=="run" then t:AdjustSpeed(1.15) end
	end
end

-- NC mais duro (parece bloco com colisão)
local function supportNC(dt)
	onNC=false
	if mode or not root or not hum or climbing or vaulting or sliding or sitting then return end
	local v=root.AssemblyLinearVelocity
	if v.Y>10 then return end
	local p=RaycastParams.new()
	p.FilterType=Enum.RaycastFilterType.Exclude
	p.FilterDescendantsInstances={char}
	p.RespectCanCollide=false
	local off=hum.HipHeight+root.Size.Y*0.5
	local origin=root.Position+Vector3.new(0,0.6,0)
	local look=root.CFrame.LookVector
	local right=root.CFrame.RightVector
	local best,bestY=nil,math.huge
	for _,offv in ipairs({Vector3.zero,look*0.9,-look*0.9,right*0.7,-right*0.7}) do
		local hit=workspace:Raycast(origin+offv,Vector3.new(0,-(off+NC_D),0),p)
		if hit and hit.Normal.Y>=0.45 and not isWaterHit(hit) then
			local md=hit.Instance:FindFirstAncestorWhichIsA("Model")
			if not (md and md:FindFirstChildOfClass("Humanoid")) then
				if hit.Instance:IsA("Terrain") or (hit.Instance:IsA("BasePart") and not hit.Instance.CanCollide and hit.Instance.Transparency<0.95) then
					local y=hit.Position.Y
					if y<bestY then bestY=y; best=hit end
				end
			end
		end
	end
	if not best then return end
	local targetY=bestY+off
	local err=targetY-root.Position.Y
	if err<-1.5 or err>NC_D then return end
	onNC=true
	stopAirHard()
	pcall(function()
		local st=hum:GetState()
		if st==Enum.HumanoidStateType.Freefall or st==Enum.HumanoidStateType.Jumping or st==Enum.HumanoidStateType.FallingDown then
			hum:ChangeState(Enum.HumanoidStateType.Running)
		end
	end)
	-- gruda forte no chão (sensação de colisão real)
	if math.abs(err)>0.02 then
		local k=math.clamp(dt*28,0,1)*NC_SNAP
		root.CFrame=root.CFrame+Vector3.new(0,err*k,0)
	end
	local vv=root.AssemblyLinearVelocity
	local vy=vv.Y
	if err>0.01 then
		vy=math.max(vy,math.clamp(err*NC_K,0,NC_MAX))
	else
		vy=math.clamp(vy,-2,6)
		if vy<0 then vy=0 end
	end
	root.AssemblyLinearVelocity=Vector3.new(vv.X,vy,vv.Z)
end

local softRay=RaycastParams.new()
softRay.FilterType=Enum.RaycastFilterType.Exclude
softRay.RespectCanCollide=false
local function softLand()
	if mode or not root or not ready or not char or not hum then return end
	if onWater or climbing or vaulting or sliding or sitting then return end
	local v=root.AssemblyLinearVelocity
	if v.Y>S_MIN then return end
	softRay.FilterDescendantsInstances={char}
	local origin=root.Position
	local look,right=root.CFrame.LookVector,root.CFrame.RightVector
	local bestDist,bestHit=S_MAX+1,nil
	for _,off in ipairs({Vector3.zero,look*1.5,-look*1.5,right*1.2,-right*1.2}) do
		local hit=workspace:Raycast(origin+off+Vector3.new(0,1,0),Vector3.new(0,-(S_MAX+2),0),softRay)
		if hit and hit.Normal.Y>0.2 and not isWaterHit(hit) then
			local md=hit.Instance:FindFirstAncestorWhichIsA("Model")
			if not (md and md:FindFirstChildOfClass("Humanoid")) then
				local dist=origin.Y-hit.Position.Y
				if dist<bestDist and dist>=0 then bestDist=dist; bestHit=hit end
			end
		end
	end
	if not bestHit or bestDist>S_MAX then return end
	local maxVy
	if bestDist<=S_TOUCH then
		maxVy=S_SAFE
		if v.Y<-40 then
			root.AssemblyLinearVelocity=Vector3.new(v.X*0.85,0,v.Z*0.85)
			local ty=bestHit.Position.Y+(hum.HipHeight+root.Size.Y*0.5)
			if root.Position.Y>ty+0.15 then root.CFrame=root.CFrame+Vector3.new(0,(ty-root.Position.Y)*0.35,0) end
			pcall(function()
				if hum:GetState()==Enum.HumanoidStateType.Freefall then hum:ChangeState(Enum.HumanoidStateType.Landed) end
			end)
			return
		end
	elseif bestDist<=S_NEAR then maxVy=S_SAFE+((bestDist-S_TOUCH)/(S_NEAR-S_TOUCH))*(S_MIDV-S_SAFE)
	elseif bestDist<=S_MID then maxVy=S_MIDV+((bestDist-S_NEAR)/(S_MID-S_NEAR))*(S_FAR-S_MIDV)
	else maxVy=S_FAR+((bestDist-S_MID)/(S_MAX-S_MID))*(-80-S_FAR) end
	local over=v.Y-maxVy
	if over<0 then
		local st=math.clamp((-over)/40,0.35,1)
		local ny=v.Y+(-over)*(0.55+0.45*st)
		if ny>maxVy then ny=maxVy end
		if bestDist<S_NEAR then ny=maxVy end
		root.AssemblyLinearVelocity=Vector3.new(v.X,ny,v.Z)
	end
end

local exitClimb,endSlide,setTurretCam,setTUI,limparT

function endSlide()
	if not sliding then return end
	sliding=false; lastSlide=tick()
	if tracks.slide then tracks.slide:Stop(0.15) end
	hum.AutoRotate=not shiftLock
	local bl=blockedAbove(); crouching=bl
	applyHB(bl and "crouch" or "normal")
	current=nil
	if updateButtons then updateButtons() end
end
local function startSlide()
	local v=root.AssemblyLinearVelocity
	local flat=Vector3.new(v.X,0,v.Z)
	local dir=flat.Magnitude>1 and flat.Unit or Vector3.new(root.CFrame.LookVector.X,0,root.CFrame.LookVector.Z).Unit
	sliding=true; slideStart=tick(); slideDir=dir
	slideSp0=math.min(flat.Magnitude*SLIDE_BST,RUN_M*SLIDE_BST); slideNow=slideSp0
	local k=math.clamp((slideSp0-RUN_S)/(RUN_M-RUN_S),0,1)
	slideDur=SLIDE_T0+(SLIDE_T1-SLIDE_T0)*k; slideAir=0
	playGnd(nil); playAir(nil); hum.AutoRotate=false; applyHB("slide")
	if tracks.slide then tracks.slide:Play(0.1) end
	if updateButtons then updateButtons() end
end
local function updateSlide(dt)
	local t=tick()-slideStart
	local frac=t/slideDur
	if hum.FloorMaterial==Enum.Material.Air and not onNC then slideAir+=dt else slideAir=0 end
	if frac>=1 or slideAir>0.2 or (t>0.25 and flatSp()<3) then endSlide() return end
	local sp=slideSp0*(1-frac)^1.5; slideNow=sp
	if sp<5 then endSlide() return end
	local v=root.AssemblyLinearVelocity
	root.AssemblyLinearVelocity=Vector3.new(slideDir.X*sp,v.Y,slideDir.Z*sp)
	root.CFrame=CFrame.lookAt(root.Position,root.Position+slideDir)
end
local function canSlide()
	return ready and not mode and not climbing and not crouching and not crawlOn and not sliding and not sitting and not onWater
		and (hum.FloorMaterial~=Enum.Material.Air or onNC)
		and hum.MoveDirection.Magnitude>0.1 and flatSp()>=SLIDE_MIN and tick()-lastSlide>0.5
end
function exitClimb(j)
	if not climbing then return end
	climbing,vaulting=false,false
	hum.PlatformStand=false; hum.AutoRotate=not shiftLock
	if tracks.climb then tracks.climb:Stop(0.1) end
	lastExit=tick(); applyHB(crouching and "crouch" or "normal"); airTime=0
	if j then
		local n=climbN; hum.Jump=false
		root.CFrame=CFrame.lookAt(root.Position,root.Position+n)
		root.AssemblyLinearVelocity=n*JUMP_H+Vector3.new(0,JUMP_V,0)
		hum:ChangeState(Enum.HumanoidStateType.Freefall)
		local f=tracks.flip
		if f then flipStart=tick(); f:Play(0.05,1,FLIP_S); f.TimePosition=FLIP_T end
	end
	current=nil
end
local function startClimb(hit)
	climbing,vaulting=true,false
	climbN=Vector3.new(hit.Normal.X,0,hit.Normal.Z).Unit
	climbStart=tick()
	playGnd(nil); playAir(nil)
	if tracks.flip then tracks.flip:Stop(0.05) end
	hum.PlatformStand=true; hum.AutoRotate=false; applyHB("climb")
	if tracks.climb then tracks.climb:Play(0.12); tracks.climb:AdjustSpeed(1) end
end
local function tryClimb()
	if mode or crouching or sitting or onWater or tick()-lastExit<0.6 then return end
	local md=hum.MoveDirection
	if md.Magnitude<0.1 then return end
	local dir=Vector3.new(md.X,0,md.Z).Unit*WALL_D
	local low=castWall(root.Position,dir)
	if not low then return end
	local high=castWall(root.Position+Vector3.new(0,HEAD_CHK,0),dir)
	if not high or low.Normal:Dot(high.Normal)<0.8 then return end
	startClimb(low)
end
local function startVault(chest)
	vaulting=true
	local into=-climbN
	local from=Vector3.new(chest.Position.X,root.Position.Y+HEAD_CHK+3,chest.Position.Z)+into*0.5
	local down=workspace:Raycast(from,Vector3.new(0,-10,0),rayP)
	if not down or down.Position.Y<root.Position.Y-0.5 then exitClimb(false) return end
	local target=Vector3.new(down.Position.X,down.Position.Y+orig.height+0.15,down.Position.Z)+into*0.3
	local tw=TS:Create(root,TweenInfo.new(0.3,Enum.EasingStyle.Quad,Enum.EasingDirection.Out),{CFrame=CFrame.lookAt(target,target+into)})
	tw.Completed:Connect(function()
		if not climbing then return end
		root.AssemblyLinearVelocity=Vector3.zero
		exitClimb(false)
	end)
	tw:Play()
end
local function updateClimb(dt)
	if vaulting then return end
	local n,into=climbN,-climbN
	local chest=castWall(root.Position,into*2.5)
	if not chest then exitClimb(false) return end
	local md=hum.MoveDirection
	local right=into:Cross(Vector3.yAxis)
	local up,side=0,0
	if md.Magnitude>0.1 then up=md:Dot(into); side=md:Dot(right) end
	up=math.abs(up)<0.25 and 0 or (up>0 and 1 or -1)
	side=math.abs(side)<0.35 and 0 or (side>0 and 1 or -1)
	if side~=0 and not castWall(root.Position+right*side*1.3,into*2.5) then side=0 end
	if up>0 and not castWall(root.Position+Vector3.new(0,HEAD_CHK,0),into*2.5) then startVault(chest) return end
	if up<0 and workspace:Raycast(root.Position,Vector3.new(0,-(orig.height+0.6),0),rayP) then exitClimb(false) return end
	local pos=root.Position+Vector3.new(0,up*CLIMB_SP*dt,0)+right*side*CLIMB_SIDE*dt
	pos=pos+n*(CLIMB_OFF-(pos-chest.Position):Dot(n))
	root.CFrame=CFrame.lookAt(pos,pos+into)
	root.AssemblyLinearVelocity=Vector3.zero
	root.AssemblyAngularVelocity=Vector3.zero
	if tracks.climb then tracks.climb:AdjustSpeed((up~=0 or side~=0) and 1 or 0) end
end
local function tryJumpOff()
	if mode=="tanque" then return end
	if sliding then endSlide() return end
	if climbing and not vaulting and tick()-climbStart>0.35 then exitClimb(true) end
end
local function setCrouch(on)
	if mode or on==crouching or climbing or sliding or sitting then return end
	if not on and blockedAbove() then return end
	crouching=on; applyHB(on and "crouch" or "normal"); updateButtons()
end
local function setSit(on)
	if on==sitting then return end
	sitting=on
	if on then
		playGnd(nil); playAir(nil)
		if tracks.sit then tracks.sit.Looped=true; tracks.sit:Play(0.15) end
	else
		if tracks.sit then tracks.sit:Stop(0.15) end
		current=nil
	end
end
local function applyBlockSit()
	if not hum then return end
	pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Seated,not blockSit) end)
	if blockSit then
		hum.Sit=false
		if hum.SeatPart then pcall(function() hum.SeatPart:Sit(nil) end) end
		setSit(false)
	end
end
local function setShift(on)
	shiftLock=on; dot.Visible=on
	if hum then
		hum.CameraOffset=on and SHIFT_OFF or Vector3.zero
		if not on and not climbing and not sliding and not mode then hum.AutoRotate=true end
	end
	if not on and UIS.MouseEnabled then UIS.MouseBehavior=Enum.MouseBehavior.Default end
	if updateButtons then updateButtons() end
end
local function doLaugh()
	if mode or not ready or climbing or sliding or sitting or laughing then return end
	local t=tracks.laugh; if not t then return end
	laughing=true
	playGnd(nil); stopAirHard()
	t.Looped=false; t:Play(0.1)
	if updateButtons then updateButtons() end
end

function setTurretCam(on)
	cam=workspace.CurrentCamera
	if on then
		if not turretCam then savedCamType=cam.CameraType; savedCamSubject=cam.CameraSubject end
		turretCam=true
		cam.CameraType=Enum.CameraType.Scriptable
		bTurretCam.BackgroundColor3=COR_ON
	else
		turretCam=false
		cam.CameraType=savedCamType or Enum.CameraType.Custom
		cam.CameraSubject=savedCamSubject or hum
		bTurretCam.BackgroundColor3=COR_OFF
	end
end
local function updateTurretCamera()
	if not (turretCam and mode=="tanque" and root and root.Parent) then return end
	cam=workspace.CurrentCamera
	cam.CameraType=Enum.CameraType.Scriptable
	local pitchDeg=(0.5-cannonPitch)*40
	local yawCF=CFrame.Angles(0,math.rad(tankYaw+yaw),0)
	local pitchCF=CFrame.Angles(math.rad(pitchDeg),0,0)
	local eye=root.Position+Vector3.new(0,2.8,0)+yawCF.LookVector*0.6
	local look=(yawCF*pitchCF).LookVector
	cam.CFrame=CFrame.new(eye,eye+look)
end
bTurretCam.MouseButton1Click:Connect(function()
	if mode=="tanque" then setTurretCam(not turretCam) end
end)

bind(RS.RenderStepped,function()
	if turretCam and mode=="tanque" then updateTurretCamera() return end
	if not shiftLock or not ready or not hum or hum.Health<=0 then return end
	if mode=="tanque" or climbing or vaulting or sliding or sitting then return end
	if UIS.MouseEnabled then UIS.MouseBehavior=Enum.MouseBehavior.LockCenter end
	hum.AutoRotate=false
	local look=workspace.CurrentCamera.CFrame.LookVector
	local flat=Vector3.new(look.X,0,look.Z)
	if flat.Magnitude>0.01 then root.CFrame=CFrame.new(root.Position,root.Position+flat) end
end)

local function tCrawl() if not mode and not sitting then crawlOn=not crawlOn; updateButtons() end end
local function tCrouch()
	if mode or not ready or sitting then return end
	if sliding then endSlide() return end
	setCrouch(not crouching)
end
local function doSlide()
	if mode or not ready or sitting then return end
	if sliding then endSlide() return end
	if canSlide() then startSlide() end
end
local function tShift() setShift(not shiftLock) end
local function tRun() if mode then return end; wantRun=not wantRun; applySpeed(); updateButtons() end
local function tNoSit() blockSit=not blockSit; applyBlockSit(); updateButtons() end

mkPB("crawl","CRAWL",0,0,tCrawl)
mkPB("crouch","AGACHAR",1,0,tCrouch)
mkPB("slide","SLIDE",0,1,doSlide)
mkPB("shift","SHIFTLOCK",1,1,tShift)
mkPB("run","CORRER",0,2,tRun)
mkPB("nosit","NÃO SENTAR",1,2,tNoSit)
mkPB("laugh","RIR",0,3,doLaugh)
updateButtons=function()
	pBtns.crawl.BackgroundColor3=crawlOn and COR_ON or COR_OFF
	pBtns.crouch.BackgroundColor3=crouching and COR_ON or COR_OFF
	pBtns.slide.BackgroundColor3=sliding and COR_ON or COR_OFF
	pBtns.shift.BackgroundColor3=shiftLock and COR_ON or COR_OFF
	pBtns.run.BackgroundColor3=wantRun and COR_ON or COR_OFF
	pBtns.nosit.BackgroundColor3=blockSit and COR_ON or COR_OFF
	pBtns.laugh.BackgroundColor3=laughing and COR_ON or COR_OFF
	bCanhao.BackgroundColor3=cannonOn and COR_ON or COR_OFF
	bAuto.BackgroundColor3=autoH and COR_ON or COR_OFF
	bDev.BackgroundColor3=devMode and COR_ON or COR_OFF
	bTurretCam.BackgroundColor3=turretCam and COR_ON or COR_OFF
end
updateButtons()

bind(UIS.InputBegan,function(i,gp)
	if gp then return end
	if i.KeyCode==Enum.KeyCode.Z then tCrawl()
	elseif i.KeyCode==Enum.KeyCode.C then tCrouch()
	elseif i.KeyCode==Enum.KeyCode.X then doSlide()
	elseif i.KeyCode==Enum.KeyCode.V then tShift()
	elseif i.KeyCode==Enum.KeyCode.LeftShift or i.KeyCode==Enum.KeyCode.RightShift then tRun()
	elseif i.KeyCode==Enum.KeyCode.B then tNoSit()
	elseif i.KeyCode==Enum.KeyCode.R then doLaugh() end
end)

local function newRP()
	local p=RaycastParams.new()
	p.FilterType=Enum.RaycastFilterType.Exclude
	p.RespectCanCollide=false
	local f={char}
	for _,pl in ipairs(Players:GetPlayers()) do
		if pl.Character and pl.Character~=char then f[#f+1]=pl.Character end
	end
	p.FilterDescendantsInstances=f
	return p
end
local function validG(hit)
	if not hit or not hit.Instance or hit.Normal.Y<0.18 or isWaterHit(hit) then return false end
	local m=hit.Instance:FindFirstAncestorWhichIsA("Model")
	if m and m:FindFirstChildOfClass("Humanoid") then return false end
	if hit.Instance:IsA("BasePart") and hit.Instance.Transparency>=0.95 then return false end
	return true
end
local function castDown(o,dist,p)
	local from=o+Vector3.new(0,1.15,0)
	for _=1,6 do
		local hit=workspace:Raycast(from,Vector3.new(0,-(dist+1.15),0),p)
		if not hit then return nil end
		if validG(hit) then return hit end
		local m=hit.Instance:FindFirstAncestorWhichIsA("Model")
		if m and m:FindFirstChildOfClass("Humanoid") then
			local f=p.FilterDescendantsInstances; f[#f+1]=m; p.FilterDescendantsInstances=f
		else from=hit.Position-Vector3.new(0,0.04,0) end
	end
	return nil
end
local function sampleG(rp,look,right,rd,p)
	local offs={Vector3.zero,look*2.5,-look*2.5,right*1.5,-right*1.5}
	local hits,sum,hc,cnt={},0,nil,0
	for i,off in ipairs(offs) do
		local h=castDown(rp+off,rd,p)
		hits[i]=h
		if h then cnt+=1; sum+=h.Position.Y; if i==1 then hc=h end end
	end
	if cnt==0 then return nil,hits,nil end
	local gy=sum/cnt
	if hc then gy=hc.Position.Y*0.6+gy*0.4 end
	return gy,hits,hc
end
local function calcHover()
	local b=BASE_H
	if hum and root and hum.RigType==Enum.HumanoidRigType.R15 then
		b=math.max(baseHip*0.75+root.Size.Y*0.4,BASE_H-0.15)
	end
	return math.max(b+(LIFT.tanque or 0),MIN_H)
end
local function ensurePhys()
	if not root then return end
	if not tankAtt or not tankAtt.Parent then
		if tankVel then tankVel:Destroy(); tankVel=nil end
		if tankOri then tankOri:Destroy(); tankOri=nil end
		tankAtt=Instance.new("Attachment"); tankAtt.Parent=root
	end
	if not tankVel or not tankVel.Parent then
		tankVel=Instance.new("LinearVelocity")
		tankVel.Attachment0=tankAtt
		tankVel.RelativeTo=Enum.ActuatorRelativeTo.World
		tankVel.VelocityConstraintMode=Enum.VelocityConstraintMode.Vector
		tankVel.ForceLimitMode=Enum.ForceLimitMode.PerAxis
		tankVel.MaxAxesForce=Vector3.zero
		tankVel.VectorVelocity=Vector3.zero
		tankVel.Parent=root
	end
	if not tankOri or not tankOri.Parent then
		tankOri=Instance.new("AlignOrientation")
		tankOri.Attachment0=tankAtt
		tankOri.Mode=Enum.OrientationAlignmentMode.OneAttachment
		tankOri.MaxTorque=math.max(root.AssemblyMass,1)*260000
		tankOri.Responsiveness=28
		tankOri.RigidityEnabled=false
		tankOri.Parent=root
	end
end
local function lockUp(yd)
	ensurePhys()
	tankOri.Enabled=true; tankOri.RigidityEnabled=true
	tankOri.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,math.rad(yd or tankYaw),0)
	tankVel.MaxAxesForce=Vector3.new(1e6,1e6,1e6)
	tankVel.VectorVelocity=Vector3.zero
	root.AssemblyLinearVelocity=Vector3.zero
	root.AssemblyAngularVelocity=Vector3.zero
end
local function groundStep(dt,yd,hvel)
	ensurePhys()
	tankOri.Enabled=true; tankOri.RigidityEnabled=false; tankOri.Responsiveness=28
	local p=newRP()
	local ycf=CFrame.Angles(0,math.rad(yd),0)
	local look,right=ycf.LookVector,ycf.RightVector
	local rd=math.max(tankHover+7,11)
	local rp=root.Position
	local gy,hits,hc=sampleG(rp,look,right,rd,p)
	local gnd=false
	if gy and (rp.Y-gy)<=MAX_SNAP+tankHover then gnd=true; lastGY=gy else gy=nil; lastGY=nil end
	local hF,hB,hR,hL=hits[2],hits[3],hits[4],hits[5]
	local aP,aR=0,0
	if gnd and hF and hB then
		local d=(hF.Position-hB.Position).Magnitude
		if d>0.1 then aP=math.deg(math.asin(math.clamp((hF.Position.Y-hB.Position.Y)/d,-1,1))) end
	elseif gnd and hc then aP=math.deg(math.asin(math.clamp(-hc.Normal:Dot(look),-1,1))) end
	if gnd and hR and hL then
		local d=(hR.Position-hL.Position).Magnitude
		if d>0.1 then aR=math.deg(math.asin(math.clamp((hR.Position.Y-hL.Position.Y)/d,-1,1))) end
	elseif gnd and hc then aR=-math.deg(math.asin(math.clamp(hc.Normal:Dot(right),-1,1))) end
	aP=math.clamp(aP,-MAX_AP,MAX_AP); aR=math.clamp(aR,-MAX_AR,MAX_AR)
	local tP=math.clamp(aP+tankPOff,-(MAX_AP+MAX_MP),MAX_AP+MAX_MP)
	local tR=math.clamp(aR+tankROff,-(MAX_AR+MAX_MR),MAX_AR+MAX_MR)
	local k=1-math.exp(-(math.abs(tankPOff)>0.5 and P_SM*1.5 or P_SM)*dt)
	tankPitch+=(tP-tankPitch)*k
	tankRoll+=(tR-tankRoll)*k
	tankOri.CFrame=CFrame.new(rp)*CFrame.Angles(0,math.rad(yd),0)*CFrame.Angles(math.rad(tankPitch),0,0)*CFrame.Angles(0,0,math.rad(tankRoll))
	local raw=0
	if gnd and not tankJump then
		local cosT=math.max(math.cos(math.rad(tankPitch))*math.cos(math.rad(tankRoll)),0.6)
		local err=(gy+tankHover/cosT)-rp.Y
		if rp.Y<gy+0.25 then raw=math.clamp(err*(H_K*1.3),-H_DN,H_UP)
		else
			if math.abs(err)<0.05 then err=0 end
			raw=math.clamp(err*H_K,-H_DN,H_UP)
		end
	end
	smVy+=(raw-smVy)*(1-math.exp(-H_SM*dt))
	if gnd or tankJump then
		tankVel.MaxAxesForce=Vector3.new(1e6,1e6,1e6)
		local fy=hvel:Dot(look)*math.tan(math.rad(math.clamp(tankPitch,-MAX_AP,MAX_AP)))*0.7+smVy
		if tankJump then fy=math.max(fy,TJ_F) end
		tankVel.VectorVelocity=Vector3.new(hvel.X,fy,hvel.Z)
	else
		tankVel.MaxAxesForce=Vector3.new(1e6,0,1e6)
		tankVel.VectorVelocity=Vector3.new(hvel.X,0,hvel.Z)
		smVy=0
	end
	return gnd
end
local function applyLift(d)
	LIFT.tanque=math.clamp((LIFT.tanque or 0)+d,ALT_MN,ALT_MX)
	altLabel.Text=string.format("Altura: %.2f",LIFT.tanque)
	tankHover=calcHover()
	if hum then hum.HipHeight=math.max(orig.hip or baseHip,0) end
end
local function applyPitch(d)
	tankPOff=math.clamp(tankPOff+d,-MAX_MP,MAX_MP)
	pitchLabel.Text=string.format("Inclinação: %+.0f°",tankPOff)
end
local function applyRoll(d) tankROff=math.clamp(tankROff+d,-MAX_MR,MAX_MR) end
bAM.MouseButton1Click:Connect(function() applyLift(-ALT_S) end)
bAP.MouseButton1Click:Connect(function() applyLift(ALT_S) end)
bPF1.MouseButton1Click:Connect(function() applyPitch(-5) end)
bPF2.MouseButton1Click:Connect(function() applyPitch(-11) end)
bPB1.MouseButton1Click:Connect(function() applyPitch(5) end)
bPB2.MouseButton1Click:Connect(function() applyPitch(11) end)
bE1.MouseButton1Click:Connect(function() applyRoll(-4) end)
bE2.MouseButton1Click:Connect(function() applyRoll(-8) end)
bD1.MouseButton1Click:Connect(function() applyRoll(4) end)
bD2.MouseButton1Click:Connect(function() applyRoll(8) end)
local function autoAlign(dt)
	if not autoH or mode~="tanque" then return end
	local ap,cl=math.abs(tankPitch),LIFT.tanque or 0
	if ap>12 then
		local tl=math.clamp(ap*0.055,0,1.8)
		local df=tl-cl
		if math.abs(df)>0.03 then applyLift(math.clamp(df*2.5*dt,-0.1,0.1)) end
	elseif math.abs(cl)>0.05 then
		applyLift(-math.sign(cl)*0.08*dt*3)
	end
end
local function tryTJump()
	if mode~="tanque" or tankJump or tankJCD>0 or not root then return end
	local hit=castDown(root.Position,tankHover+3.5,newRP())
	if not hit or hit.Normal.Y<0.3 then return end
	if (root.Position.Y-hit.Position.Y)>(tankHover+2) then return end
	tankJump=true; tankJCD=0.55
	if tankVel then
		tankVel.MaxAxesForce=Vector3.new(1e6,1e6,1e6)
		local c=tankVel.VectorVelocity
		tankVel.VectorVelocity=Vector3.new(c.X,TJ_F,c.Z)
	end
	task.delay(TJ_D,function() tankJump=false end)
end
UIS.InputBegan:Connect(function(i,g)
	if g then return end
	if i.KeyCode==Enum.KeyCode.Space or i.KeyCode==Enum.KeyCode.ButtonA then tryTJump() end
end)

function setTUI(on)
	bVoltar.Visible=on; bDev.Visible=on; bCanhao.Visible=on; bTiro.Visible=on; mira.Visible=on; bTurretCam.Visible=on
	btnTF.Visible=on; btnTB.Visible=on; btnTL.Visible=on; btnTR.Visible=on
	btnL.Visible=on; btnR.Visible=on; btnU.Visible=on; btnD.Visible=on
	bTank.Visible=not on
	for _,b in pairs(pBtns) do b.Visible=not on end
	local s=on and devMode
	bAuto.Visible=s; altLabel.Visible=s
	bAM.Visible=s; bAP.Visible=s; bE1.Visible=s; bE2.Visible=s; bD1.Visible=s; bD2.Visible=s
	pitchLabel.Visible=s; bPF1.Visible=s; bPF2.Visible=s; bPB1.Visible=s; bPB2.Visible=s
	if not on then setTurretCam(false) end
end
bDev.MouseButton1Click:Connect(function()
	devMode=not devMode
	setTUI(mode=="tanque")
	updateButtons()
end)
bAuto.MouseButton1Click:Connect(function()
	autoH=not autoH
	updateButtons()
end)
local function setCan(v)
	cannonOn=v
	updateButtons()
end
bCanhao.MouseButton1Click:Connect(function()
	if mode=="tanque" and not ocupado then setCan(not cannonOn) end
end)

function limparT()
	if tankVel then tankVel:Destroy(); tankVel=nil end
	if tankAtt then tankAtt:Destroy(); tankAtt=nil end
	if tankOri then tankOri:Destroy(); tankOri=nil end
	tankPitch,tankRoll,tankPOff,tankROff=0,0,0,0
	tankJump,tankJCD,lastGY,smVy=false,0,nil,0
	setCan(false); setTurretCam(false)
	for _,t in pairs(tankTracks) do pcall(function() t:Stop(0.1) end) end
end
local function ativarT()
	if ocupado or not ready or not hum or hum.Health<=0 then return end
	ocupado=true
	if climbing then exitClimb(false) end
	if sliding then endSlide() end
	if sitting then setSit(false); hum.Sit=false end
	if laughing and tracks.laugh then tracks.laugh:Stop(0); laughing=false end
	crouching,crawlOn,wantRun=false,false,false
	playGnd(nil); playAir(nil); restoreHB()
	mode="tanque"; setTUI(true)
	hum.AutoRotate=false; hum.PlatformStand=true; hum.WalkSpeed=0; hum.JumpPower=0
	pcall(function()
		for _,st in ipairs({
			Enum.HumanoidStateType.Ragdoll,Enum.HumanoidStateType.FallingDown,
			Enum.HumanoidStateType.GettingUp,Enum.HumanoidStateType.Climbing,
			Enum.HumanoidStateType.Seated,Enum.HumanoidStateType.Freefall,
			Enum.HumanoidStateType.Flying,Enum.HumanoidStateType.Jumping
		}) do hum:SetStateEnabled(st,false) end
		hum:ChangeState(Enum.HumanoidStateType.Physics)
	end)
	baseHip=orig.hip or hum.HipHeight
	tankHover=calcHover()
	hum.HipHeight=orig.hip or baseHip
	tankPitch,tankRoll,tankPOff,tankROff=0,0,0,0
	local look=root.CFrame.LookVector
	tankYaw=math.deg(math.atan2(-look.X,-look.Z))
	root.Anchored=true
	root.AssemblyLinearVelocity=Vector3.zero
	root.AssemblyAngularVelocity=Vector3.zero
	lockUp(tankYaw)
	root.CFrame=CFrame.new(root.Position)*CFrame.Angles(0,math.rad(tankYaw),0)
	local t=tankTracks.TransformTank
	if t then
		pcall(function()
			t.Looped=false; t:AdjustSpeed(TF_S); t:Play(0.05)
			task.wait(math.max(t.Length/TF_S,0.25)); t:Stop(0.1)
		end)
	else task.wait(0.25) end
	restoreHB(); tankHover=calcHover()
	local hit=castDown(root.Position+Vector3.new(0,2,0),tankHover+12,newRP())
	local y=hit and (hit.Position.Y+tankHover) or root.Position.Y
	root.CFrame=CFrame.new(root.Position.X,y,root.Position.Z)*CFrame.Angles(0,math.rad(tankYaw),0)
	lastGY=hit and hit.Position.Y or nil; smVy=0
	root.Anchored=false; lockUp(tankYaw); tankOri.RigidityEnabled=false
	pcall(function()
		hum:SetStateEnabled(Enum.HumanoidStateType.Jumping,true)
		hum:SetStateEnabled(Enum.HumanoidStateType.Freefall,true)
	end)
	lockFall(); ocupado=false
end
local function voltarN()
	if ocupado or mode~="tanque" then return end
	ocupado=true; setCan(false); setTurretCam(false)
	mTF,mTB,mTL,mTR,mL,mR,mU,mD=false,false,false,false,false,false,false,false
	if tankTracks.IdleTank then tankTracks.IdleTank:Stop(0.1) end
	if tankTracks.WalkTank then tankTracks.WalkTank:Stop(0.1) end
	if tankTracks.Tiro and tankTracks.Tiro.IsPlaying then tankTracks.Tiro:Stop(0.1) end
	root.Anchored=true
	root.AssemblyLinearVelocity=Vector3.zero
	root.AssemblyAngularVelocity=Vector3.zero
	lockUp(tankYaw)
	local t=tankTracks.TransformTank
	if t then
		pcall(function()
			t.Looped=false; t:Play(0.05,1,-TF_S)
			t.TimePosition=math.max(t.Length-0.03,0)
			task.wait(math.max(t.Length/TF_S,0.25)); t:Stop(0.1)
		end)
	else task.wait(0.25) end
	root.Anchored=false; limparT(); mode=nil; setTUI(false); restoreHB()
	hum.PlatformStand=false; hum.AutoRotate=not shiftLock
	hum.WalkSpeed=baseSpeed; hum.JumpPower=50; hum.HipHeight=orig.hip or baseHip
	pcall(function()
		for _,st in ipairs({
			Enum.HumanoidStateType.GettingUp,Enum.HumanoidStateType.Climbing,
			Enum.HumanoidStateType.Freefall,Enum.HumanoidStateType.Jumping
		}) do hum:SetStateEnabled(st,true) end
		hum:SetStateEnabled(Enum.HumanoidStateType.Seated,not blockSit)
		hum:ChangeState(Enum.HumanoidStateType.GettingUp)
	end)
	lockFall(); wantRun=false; ocupado=false; updateButtons()
end
bTank.MouseButton1Click:Connect(ativarT)
bVoltar.MouseButton1Click:Connect(voltarN)
bTiro.MouseButton1Click:Connect(function()
	if mode~="tanque" or ocupado or not hum then return end
	ocupado=true; setCan(false)
	if tankTracks.IdleTank then tankTracks.IdleTank:Stop(0.1) end
	if tankTracks.WalkTank then tankTracks.WalkTank:Stop(0.1) end
	local t=tankTracks.Tiro
	if t then pcall(function() t.Looped=false; t:AdjustSpeed(1); t:Play(0.15); task.wait(math.max(t.Length,1)); t:Stop(0.15) end) end
	ocupado=false
end)

local function onChar(c)
	ready=false; mode=nil
	climbing,vaulting,sliding,crouching=false,false,false,false
	isRunning,wantRun,sitting,onNC,onWater,laughing=false,false,false,false,false,false
	current,airCurrent,airTime=nil,nil,0
	lastCF,lastVel=nil,nil
	setTurretCam(false)
	char=c; hum=c:WaitForChild("Humanoid"); root=c:WaitForChild("HumanoidRootPart")
	task.wait(0.4)
	orig={size=root.Size,hip=hum.HipHeight,partCollide={}}
	orig.height=(hum.RigType==Enum.HumanoidRigType.R15) and (orig.hip+orig.size.Y/2) or 3
	for _,d in ipairs(c:GetDescendants()) do
		if d:IsA("BasePart") then orig.partCollide[d]=d.CanCollide end
	end
	baseSpeed=hum.WalkSpeed; baseHip=hum.HipHeight
	rayP.FilterDescendantsInstances={c}
	rayP.RespectCanCollide=false
	overlap.FilterDescendantsInstances={c}
	loadPark(); loadTank(); applyLocal(c); applyBlockSit(); lockFall()
	bind(hum:GetPropertyChangedSignal("Jump"),function() if hum.Jump then tryJumpOff() end end)
	hum.Jumping:Connect(tryTJump)
	bind(hum:GetPropertyChangedSignal("Sit"),function()
		if mode then return end
		if blockSit then hum.Sit=false; setSit(false); return end
		setSit(hum.Sit)
	end)
	bind(hum.StateChanged,function(_,n)
		if mode then return end
		if n==Enum.HumanoidStateType.Seated then
			if blockSit then
				hum.Sit=false
				pcall(function() hum:ChangeState(Enum.HumanoidStateType.GettingUp) end)
				setSit(false)
			else setSit(true) end
		elseif sitting and n~=Enum.HumanoidStateType.Seated then setSit(false) end
	end)
	if shiftLock then hum.CameraOffset=SHIFT_OFF end
	setTUI(false); ready=true; updateButtons()
end

bind(RS.Heartbeat,function(dt)
	if not (ready and char and char.Parent and hum and hum.Health>0) then return end
	if VEL_GUARD and root and root.Parent and not ocupado then
		if hum.Sit or hum.SeatPart then lastCF,lastVel=root.CFrame,root.AssemblyLinearVelocity
		elseif mode~="tanque" then
			local v,w=root.AssemblyLinearVelocity,root.AssemblyAngularVelocity
			local hz=Vector3.new(v.X,0,v.Z).Magnitude
			if hz>MAX_HZ or v.Y>MAX_UP or w.Magnitude>MAX_ANG then
				root.AssemblyLinearVelocity=lastVel or Vector3.zero
				root.AssemblyAngularVelocity=Vector3.zero
				if lastCF and (root.Position-lastCF.Position).Magnitude>SNAP then root.CFrame=lastCF end
			else lastCF,lastVel=root.CFrame,v end
		else lastCF,lastVel=root.CFrame,root.AssemblyLinearVelocity end
	end
	if ocupado then
		if mode=="tanque" and root and not root.Anchored then lockUp(tankYaw) end
		return
	end
	if tankJCD>0 then tankJCD=math.max(0,tankJCD-dt) end
	if not mode then softLand(); supportNC(dt); onWater=detectWater() or onWater end
	if blockSit and hum.Sit then hum.Sit=false; setSit(false) end
	if mode=="tanque" then
		hum.AutoRotate=false; hum.PlatformStand=true; hum.WalkSpeed=0
		local held=mTF or mTB or mTL or mTR
		local turn=(mTR and 1 or 0)-(mTL and 1 or 0)
		if turn~=0 then yaw=yaw+turn*T_YSPD*dt; if cannonOn then setCan(false) end end
		yaw=math.clamp(yaw,-T_YAW,T_YAW)
		local t=tankTracks.Tiro
		if cannonOn and t then
			if not t.IsPlaying then
				if tankTracks.IdleTank then tankTracks.IdleTank:Stop(0.1) end
				if tankTracks.WalkTank then tankTracks.WalkTank:Stop(0.1) end
				t.Looped=true; t:Play(0.1); t:AdjustSpeed(0)
			end
			local up=mU or UIS:IsKeyDown(Enum.KeyCode.Up)
			local dn=mD or UIS:IsKeyDown(Enum.KeyCode.Down)
			if up and not dn then cannonPitch=math.clamp(cannonPitch-dt*0.9,0,1)
			elseif dn and not up then cannonPitch=math.clamp(cannonPitch+dt*0.9,0,1) end
			t.TimePosition=cannonPitch*t.Length
		elseif t and t.IsPlaying then t:Stop(0.12) end
		if mL then tankYaw+=T_TURN*dt elseif mR then tankYaw-=T_TURN*dt end
		local fwd=(mTF and 1 or 0)-(mTB and 1 or 0)
		groundStep(dt,tankYaw,CFrame.Angles(0,math.rad(tankYaw),0).LookVector*(fwd*16))
		autoAlign(dt)
		if not cannonOn then
			if (held and (mTF or mTB)) or (not held and hum.MoveDirection.Magnitude>0.05) then
				if tankTracks.IdleTank then tankTracks.IdleTank:Stop(0.1) end
				if tankTracks.WalkTank and not tankTracks.WalkTank.IsPlaying then tankTracks.WalkTank:Play(0.12) end
			else
				if tankTracks.WalkTank then tankTracks.WalkTank:Stop(0.1) end
				local idle=tankTracks.IdleTank
				if idle then
					if not idle.IsPlaying then idle:Play(0.1) end
					idle:AdjustSpeed(0)
					idle.TimePosition=math.clamp((yaw+T_YAW)/(T_YAW*2),0,1)*idle.Length
				end
			end
		end
		return
	end
	if sitting or laughing then return end
	if vaulting then root.AssemblyLinearVelocity=Vector3.zero; return end
	if climbing then updateClimb(dt) return end
	if sliding then updateSlide(dt) return end
	tryClimb()
	if not climbing then applySpeed(); updateGnd(dt) end
end)

bind(UIS.JumpRequest,tryJumpOff)
bind(player.CharacterAdded,function(c) task.spawn(onChar,c) end)
if player.Character then task.spawn(onChar,player.Character) end

env.ParkourCleanup=function()
	running=false
	setTurretCam(false)
	for _,c in ipairs(conns) do pcall(function() c:Disconnect() end) end
	limparT()
	pcall(function()
		if root then root.Anchored=false end
		if hum then
			hum.PlatformStand=false; hum.AutoRotate=true; hum.CameraOffset=Vector3.zero
			hum.WalkSpeed=baseSpeed; hum.HipHeight=orig.hip or baseHip
			hum:SetStateEnabled(Enum.HumanoidStateType.Seated,true)
			hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown,true)
			hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll,true)
		end
		if root and orig.size then root.Size=orig.size end
	end)
	if UIS.MouseEnabled then UIS.MouseBehavior=Enum.MouseBehavior.Default end
	for _,t in pairs(tracks) do pcall(function() t:Stop(0) end) end
	for _,t in pairs(tankTracks) do pcall(function() t:Stop(0) end) end
	gui:Destroy()
end
env.AntiFlingCleanup=env.ParkourCleanup
print("[Parkour+Tanque] OK | run 38 fixo | NC duro | botões iguais | cam torreta")
