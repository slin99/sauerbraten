# Sauerbraten Client Security Vulnerabilities - Quick Reference

## Executive Summary

The Sauerbraten game client has **7 major vulnerability categories** that allow client-side manipulation for cheating. The root cause is **client-side authority over physics and movement** with minimal server-side validation.

## Vulnerability Quick Reference

### 🔴 CRITICAL: Position Manipulation (Teleportation)

**What it allows:** Player can teleport anywhere on the map  
**How it works:** Client directly controls position sent to server  
**Server validation:** None - server blindly accepts position  
**Code location:** `client.cpp:856-909`, `server.cpp:2927`  
**Exploitation ease:** ⭐⭐⭐⭐⭐ (Very Easy)  
**Detection difficulty:** Hard for short distances, Easy for long distances

**Example Attack:**
```cpp
player1->o = vec(target_x, target_y, target_z); // Any position
// Server accepts it without question
```

---

### 🔴 CRITICAL: Wall Glitching (No-Clip)

**What it allows:** Walk through walls, floors, and ceilings  
**How it works:** Collision detection is client-side only  
**Server validation:** None - server never checks collision  
**Code location:** `engine/physics.cpp` (entire collision system)  
**Exploitation ease:** ⭐⭐⭐⭐ (Easy)  
**Detection difficulty:** Hard - requires position analysis

**Example Attack:**
```cpp
bool ellipsecollide(...) {
    return false; // Disable all collision detection
}
```

---

### 🟠 HIGH: Speed Hacks

**What it allows:** Move faster than normal (2-10x speed)  
**How it works:** Client controls maxspeed and physics  
**Server validation:** Weak - checks velocity magnitude only  
**Code location:** `ents.h:67`, `server.cpp:2921`  
**Exploitation ease:** ⭐⭐⭐⭐⭐ (Very Easy)  
**Detection difficulty:** Medium - noticeable by players

**Example Attack:**
```cpp
player1->maxspeed = 300.0f; // Normal is 100
// Stay under server threshold: vel.magnitude2() < 180
```

**Server check weakness:**
```cpp
// Only checks if velocity is too high, not displacement
if(max(vel.magnitude2(), fabs(vel.z)) >= 180)
    cp->setexceeded(); // Weak - doesn't kick immediately
```

---

### 🟠 HIGH: Fly Hacks

**What it allows:** Fly freely, ignore gravity, hover  
**How it works:** Client controls gravity and falling physics  
**Server validation:** None - server accepts falling vector from client  
**Code location:** `ents.h:64`, `client.cpp:896-908`  
**Exploitation ease:** ⭐⭐⭐⭐⭐ (Very Easy)  
**Detection difficulty:** Medium - obvious if hovering

**Example Attack:**
```cpp
player1->falling = vec(0, 0, 0); // No gravity
player1->physstate = PHYS_FLOAT; // Floating state
// Server accepts this without validation
```

---

### 🟡 MEDIUM: Entity Interaction Exploits

**What it allows:** Trigger teleports/jumppads without proximity  
**How it works:** Client triggers entity interactions  
**Server validation:** None - doesn't check proximity  
**Code location:** `entities.cpp:254-307`, `server.cpp:2933-2957`  
**Exploitation ease:** ⭐⭐⭐ (Moderate)  
**Detection difficulty:** Hard

**Example Attack:**
```cpp
teleport(teleport_id, player1); // No proximity check
// Can teleport from anywhere on map
```

**Server issue:**
```cpp
// Server accepts N_TELEPORT without validating
// player was near the teleport entity
case N_TELEPORT:
    sendf(-1, 0, "ri4x", N_TELEPORT, pcn, teleport, teledest, ...);
```

---

### 🟡 MEDIUM: Edit Mode Exploits

**What it allows:** Bypass restrictions in edit mode  
**How it works:** Edit mode disables certain checks  
**Server validation:** Mode-dependent  
**Code location:** Throughout codebase (`m_edit` checks)  
**Exploitation ease:** ⭐⭐ (Hard - requires mode abuse)  
**Detection difficulty:** Easy if mode is wrong

**Example Issue:**
```cpp
// Velocity check bypassed in edit mode
if(!ci->local && !m_edit && vel >= 180)
    cp->setexceeded();
// If m_edit can be faked, no velocity check
```

---

### 🟡 MEDIUM: Velocity Vector Manipulation

**What it allows:** Confuse hit detection and prediction  
**How it works:** Send false velocity direction  
**Server validation:** None - velocity direction not validated  
**Code location:** `client.cpp:891-895`  
**Exploitation ease:** ⭐⭐⭐ (Moderate)  
**Detection difficulty:** Very Hard

**Example Issue:**
```cpp
// Client can send any velocity direction
// Server doesn't check if it matches actual movement
vectoyawpitch(d->vel, velyaw, velpitch);
// This is client-controlled, not validated
```

---

## Vulnerability Matrix

| Vulnerability | Severity | Ease | Impact | Detection |
|--------------|----------|------|--------|-----------|
| Teleportation | 🔴 Critical | Very Easy | Critical | Hard |
| No-Clip | 🔴 Critical | Easy | Critical | Hard |
| Speed Hacks | 🟠 High | Very Easy | High | Medium |
| Fly Hacks | 🟠 High | Very Easy | High | Medium |
| Entity Exploits | 🟡 Medium | Moderate | Medium | Hard |
| Edit Mode Abuse | 🟡 Medium | Hard | High | Easy |
| Velocity Manipulation | 🟡 Medium | Moderate | Low | Very Hard |

## Attack Vectors Summary

### Most Dangerous Combinations

1. **CTF Flag Rush** (Critical Impact)
   - No-clip through walls to flag
   - Speed hack to move quickly
   - Teleport back to base
   - **Time to capture flag:** < 10 seconds

2. **Invisible Assassin** (High Impact)
   - Teleport behind enemy
   - No-clip through walls for surprise
   - Speed hack for quick escape
   - **Effectiveness:** Very High

3. **Immortal Player** (High Impact)
   - Fly hack to stay out of reach
   - Speed hack to dodge
   - No-clip to hide in walls
   - **Survivability:** Nearly 100%

## Server-Side Validation Status

| What Server Checks | Status |
|-------------------|--------|
| Position displacement | ❌ Not checked |
| Collision/line-of-sight | ❌ Not checked |
| Physics consistency | ❌ Not checked |
| Teleport proximity | ❌ Not checked |
| Entity interaction range | ❌ Not checked |
| Movement trajectory | ❌ Not checked |
| Velocity magnitude | ⚠️ Weak (threshold too high) |
| Client ownership | ✅ Checked |

**Result:** Server validates almost nothing related to movement!

## Root Cause: Client-Side Physics

```
┌─────────────────────────────────────────┐
│  CLIENT (Full Control)                  │
├─────────────────────────────────────────┤
│  • Position calculation                 │
│  • Collision detection                  │
│  • Physics simulation                   │
│  • Gravity/falling                      │
│  • Entity interactions                  │
│  • Movement speed                       │
└─────────────────┬───────────────────────┘
                  │
                  │ N_POS packet
                  │ (position, velocity, state)
                  ▼
┌─────────────────────────────────────────┐
│  SERVER (Minimal Validation)            │
├─────────────────────────────────────────┤
│  ✓ Check client ownership               │
│  ⚠️ Check velocity magnitude (weak)     │
│  ❌ No position validation               │
│  ❌ No collision validation              │
│  ❌ No physics validation                │
│  • Relay position to other clients      │
└─────────────────────────────────────────┘
```

**Problem:** Client has **absolute authority** over position and movement.

## Recommended Fixes (Priority Order)

### 🔥 Priority 1: Position Displacement Validation
**Impact:** Prevents teleportation, limits speed hacks  
**Effort:** Low (1 day)  
**Code:** ~50 lines in `server.cpp`

```cpp
// Proposed fix
vec displacement = pos - cp->state.o;
float maxdisplacement = maxspeed * timedelta * 1.2f;
if(displacement.magnitude() > maxdisplacement) {
    // Reject or rubber-band
}
```

### 🔥 Priority 2: Improved Velocity Checks
**Impact:** Better speed hack detection  
**Effort:** Low (0.5 days)  
**Code:** ~20 lines in `server.cpp`

```cpp
// Lower threshold and immediate action
if(max(vel.magnitude2(), fabs(vel.z)) >= 130) { // Was 180
    disconnect_client(sender, DISC_SPEEDHACK);
}
```

### 🔥 Priority 3: Teleport Proximity Validation
**Impact:** Prevents fake teleports  
**Effort:** Low (1 day)  
**Code:** ~30 lines in `server.cpp`

```cpp
// Verify player near teleport entity
if(player_pos.dist(teleport_pos) > 16.0f) {
    // Reject teleport
}
```

### 🔶 Priority 4: Basic Collision Raycast
**Impact:** Prevents obvious no-clip  
**Effort:** Medium (3-5 days)  
**Code:** ~200 lines (raycast implementation)

```cpp
// Check if path from old_pos to new_pos is clear
if(!raycast_clear(old_pos, new_pos)) {
    // Reject movement, rubber-band player
}
```

### 🔶 Priority 5: Enhanced Logging
**Impact:** Better detection and forensics  
**Effort:** Low (1 day)  
**Code:** ~50 lines throughout

```cpp
// Log suspicious movements
if(suspicious_movement) {
    logf("Player %d: suspicious movement (%f units)", cn, displacement);
}
```

### 🔷 Priority 6: Server-Authoritative Physics (Long-term)
**Impact:** Prevents ALL position-based cheats  
**Effort:** Very High (2-3 months)  
**Code:** Architectural redesign

```
Client sends: Key presses, mouse movement
Server calculates: Position, velocity, collision
Client receives: Authoritative position
```

## Implementation Roadmap

### Week 1: Quick Wins
- ✅ Position displacement validation
- ✅ Improved velocity thresholds
- ✅ Auto-kick on violations
- ✅ Basic logging

**Expected Impact:** 60% reduction in effective cheating

### Week 2-3: Enhanced Validation
- ⬜ Teleport proximity checks
- ⬜ Rate limiting
- ⬜ Collision raycasting
- ⬜ Admin tools

**Expected Impact:** 80% reduction in effective cheating

### Month 2-3: Architectural Changes
- ⬜ Server-side physics design
- ⬜ Implementation
- ⬜ Client prediction
- ⬜ Testing

**Expected Impact:** 99% reduction in position-based cheating

## Testing Checklist

### Exploit Tests (Should Be Blocked)
- [ ] Teleport 1000 units instantly
- [ ] Move at 5x normal speed
- [ ] Walk through solid wall
- [ ] Fly/hover in mid-air
- [ ] Trigger teleport from far away
- [ ] Spam jumppad for super velocity

### Legitimate Gameplay (Should Work)
- [ ] Normal walking/running
- [ ] Jumping on jump pads
- [ ] Using teleporters normally
- [ ] Network lag compensation
- [ ] Fast movements (rocket jumps, etc.)
- [ ] Spectator mode movement

### Edge Cases (Should Handle Gracefully)
- [ ] High latency (200ms+)
- [ ] Packet loss
- [ ] Legitimate speed boosts
- [ ] Map entity interactions
- [ ] Mode transitions

## Quick Detection Guide (For Admins)

### Signs of Teleporting
- Player disappears and reappears elsewhere
- Instant position changes in demo playback
- Captures flag impossibly fast
- Appears behind you suddenly

### Signs of Speed Hacking
- Player moves much faster than normal
- Covers long distances too quickly
- Outruns everyone consistently
- Can't be caught when fleeing

### Signs of No-Clip
- Player seen inside walls/floors
- Takes impossible shortcuts
- Appears in unreachable areas
- Exits/enters buildings through walls

### Signs of Fly Hacking
- Player hovering in air
- Floating after jumping
- Climbing invisible stairs
- On roofs without access

## Resources

- **Full Analysis:** See `SECURITY_ANALYSIS.md`
- **Code Examples:** See `EXPLOIT_EXAMPLES.md`
- **Implementation Guide:** See `SECURITY_README.md`

## Summary

The Sauerbraten client has **fundamental security flaws** due to client-side physics authority. Simple modifications allow:
- ✅ Instant teleportation
- ✅ Walking through walls
- ✅ Flying/hovering
- ✅ Speed hacking

**The only complete fix is server-authoritative physics**, but short-term validations can reduce exploitation significantly.

---

**Document Version:** 1.0  
**Last Updated:** 2025-11-16  
**Severity:** 🔴 CRITICAL
