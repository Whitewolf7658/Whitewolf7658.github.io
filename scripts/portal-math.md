[← Back to Home](../README.md)

# Math & Coordinate Matrix Library (PortalMath)

This module was designed for my unified rigid transformation matrices, the relative coordinate mapping, and the continuous collision detection.

local PortalMath = {}
local FLIP = CFrame.Angles(0, math.pi, 0)
PortalMath.FLIP = FLIP
local EPSILON = 1e-6
function PortalMath.GetTransform(source,destination) return destination.CFrame * FLIP * source.CFrame:Inverse() end
function PortalMath.TransformCFrame(cf,s,d) return PortalMath.GetTransform(s,d) * cf end
function PortalMath.TransformPoint(p,s,d) return PortalMath.GetTransform(s,d):PointToWorldSpace(p) end
function PortalMath.TransformVector(v,s,d) return PortalMath.GetTransform(s,d):VectorToWorldSpace(v) end
function PortalMath.TransformDirection(v,s,d) if v.Magnitude<EPSILON then return Vector3.zero end return PortalMath.TransformVector(v,s,d) end
function PortalMath.ToPortalSpace(p,portal) return portal.CFrame:PointToObjectSpace(p) end
function PortalMath.VectorToPortalSpace(v,portal) return portal.CFrame:VectorToObjectSpace(v) end
function PortalMath.SignedDistance(p,portal) return PortalMath.ToPortalSpace(p,portal).Z end
function PortalMath.GetVisibleNormal(portal) return -portal.CFrame.LookVector end
function PortalMath.IsOnVisibleSide(p,portal) return PortalMath.SignedDistance(p,portal)>0 end
function PortalMath.IsInsideOpening(lp,portal,px,py)
	px,py=px or 0,py or 0
	return math.abs(lp.X)<=portal.Size.X*.5+px and math.abs(lp.Y)<=portal.Size.Y*.5+py
end
function PortalMath.SegmentPlaneIntersection(a,b,portal)
	local la,lb=portal.CFrame:PointToObjectSpace(a),portal.CFrame:PointToObjectSpace(b)
	local dz=lb.Z-la.Z
	if math.abs(dz)<EPSILON then return nil end
	local t=-la.Z/dz
	if t<0 or t>1 then return nil end
	local lp=la:Lerp(lb,t)
	return {Time=t,LocalPosition=lp,WorldPosition=portal.CFrame:PointToWorldSpace(lp),PreviousLocal=la,CurrentLocal=lb}
end
function PortalMath.GetCrossing(a,b,portal,px,py)
	local h=PortalMath.SegmentPlaneIntersection(a,b,portal)
	if not h or h.PreviousLocal.Z*h.CurrentLocal.Z>0 or not PortalMath.IsInsideOpening(h.LocalPosition,portal,px,py) then return nil end
	return h
end
function PortalMath.GetForwardCrossing(a,b,portal,px,py)
	local h=PortalMath.GetCrossing(a,b,portal,px,py)
	if not h or h.PreviousLocal.Z<=0 or h.CurrentLocal.Z>0 then return nil end
	return h
end
function PortalMath.TransformPhysics(cf,lv,av,s,d)
	local t=PortalMath.GetTransform(s,d)
	return {CFrame=t*cf,LinearVelocity=t:VectorToWorldSpace(lv),AngularVelocity=t:VectorToWorldSpace(av)}
end
function PortalMath.GetExitCFrame(cf,s,d,offset)
	return PortalMath.TransformCFrame(cf,s,d)+PortalMath.GetVisibleNormal(d)*(offset or .15)
end
return PortalMath


