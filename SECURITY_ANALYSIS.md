# Sauerbraten Client Security Analysis: Potential Cheating Vulnerabilities

## Executive Summary

This document identifies potential security vulnerabilities in the Sauerbraten game client that could be exploited for cheating. The analysis focuses on client-side modifications that could enable:
- Position manipulation (teleportation)
- Speed hacks
- Wall glitching (no-clip)
- Fly hacks
- Other movement-based exploits

## Table of Contents

1. [Overview](#overview)
2. [Vulnerability Categories](#vulnerability-categories)
3. [Detailed Vulnerability Analysis](#detailed-vulnerability-analysis)
4. [Exploitation Techniques](#exploitation-techniques)
5. [Current Mitigations](#current-mitigations)
6. [Recommendations](#recommendations)

## Overview

Sauerbraten is a multiplayer first-person shooter built on the Cube 2 engine. The game uses a client-server architecture where clients send position updates to the server, which then broadcasts them to other clients. The analysis reveals that the client-side code has significant control over movement and position data before transmission to the server.

### Key Files Analyzed

- `src/fpsgame/client.cpp` - Client-side networking and position updates
- `src/fpsgame/server.cpp` - Server-side validation and message handling
- `src/engine/physics.cpp` - Physics and collision detection
- `src/fpsgame/entities.cpp` - Entity interaction (teleports, jumps)
- `src/shared/ents.h` - Entity and physics structures

## Vulnerability Categories

### 1. Position Manipulation (Teleportation)

**Severity: HIGH**

**Location:** `src/fpsgame/client.cpp` lines 856-909 (`sendposition()`)

**Description:**
The client has full control over the position data sent to the server. The position is encoded in the `sendposition()` function and transmitted via the `N_POS` network message.

**Vulnerable Code:**
```cpp
static void sendposition(fpsent *d, packetbuf &q)
{
    putint(q, N_POS);
    putuint(q, d->clientnum);
    // ... 
    ivec o = ivec(vec(d->o.x, d->o.y, d->o.z-d->eyeheight).mul(DMF));
    // Position is directly sent without validation
```

**Exploitation Potential:**
- A modified client can set `d->o` (player position) to any arbitrary coordinates
- Teleportation to any map location is possible
- Short-range teleports (5-50 units) are especially hard to detect
- Could be used to:
  - Instantly reach flags in CTF
  - Escape combat situations
  - Access normally unreachable areas
  - Bypass map boundaries

**Server-Side Validation:** 
The server does NOT validate position changes for reasonableness. In `server.cpp` line 2927:
```cpp
cp->state.o = pos; // Position is blindly accepted
```

The only check is for velocity magnitude (line 2921):
```cpp
if(!ci->local && !m_edit && max(vel.magnitude2(), (float)fabs(vel.z)) >= 180)
    cp->setexceeded();
```

This checks velocity but NOT position displacement, allowing teleportation.

### 2. Speed Hacks

**Severity: HIGH**

**Location:** `src/shared/ents.h` line 67 (maxspeed), `src/fpsgame/client.cpp`

**Description:**
The client controls the physics simulation and velocity calculations before sending position updates.

**Vulnerable Code:**
```cpp
// From ents.h - physent structure
float maxspeed; // cubes per second, 100 for player
```

**Exploitation Potential:**
- Modify `maxspeed` value locally to exceed normal limits
- Increase velocity vector magnitude beyond 100 units/second
- Server has minimal velocity validation:
  - Only checks if `vel.magnitude2() >= 180` (squared magnitude)
  - This allows velocities up to ~13.4 units/second without triggering `setexceeded()`
  - Normal player speed is maxspeed=100 units/second, but this is client-controlled
- Could enable:
  - Rapid flag capture in CTF modes
  - Quick escapes and aggressive rushes
  - Dodging projectiles more effectively

**Current Mitigation:**
The server has a weak velocity check (`server.cpp` line 2921), but it only marks the player as exceeded, and doesn't prevent the movement or kick immediately.

### 3. Wall Glitching / No-Clip

**Severity: CRITICAL**

**Location:** `src/engine/physics.cpp` (collision detection)

**Description:**
Collision detection is performed CLIENT-SIDE before position updates are sent. The server trusts the client's reported position.

**Vulnerable Code:**
The entire collision system in `physics.cpp` runs on the client:
```cpp
bool ellipsecollide(physent *d, const vec &dir, const vec &o, ...)
bool plcollide(physent *d, const vec &dir) // collide with player or monster
```

**Exploitation Potential:**
- Disable or modify collision detection code
- Methods include:
  1. **Direct bypass**: Make collision functions always return false/no collision
  2. **Selective bypass**: Disable wall collision but keep floor collision
  3. **Clipping flag abuse**: Set gameclip flag inappropriately
  4. **Physics state manipulation**: Force `physstate` to values that ignore collision
  
**Example Modifications:**
```cpp
// Modified collision - always return false
bool ellipsecollide(physent *d, const vec &dir, ...) {
    return false; // No collision ever detected
}
```

**Exploitation Use Cases:**
- Walk through walls to access restricted areas
- Hide inside geometry to ambush players
- Escape when cornered
- Access map secrets and shortcuts
- In CTF: Take shortcuts through walls to capture flags faster

**Server-Side Issues:**
- No server-side collision validation
- Server only tracks position, not whether the path to that position was valid
- The `gameclip` flag (line 2928) is reported by CLIENT, not validated by server

### 4. Fly Hacks

**Severity: HIGH**

**Location:** `src/engine/physics.cpp`, `src/shared/ents.h`

**Description:**
Gravity and falling physics are computed client-side. The client controls the `falling` vector and `physstate`.

**Vulnerable Code:**
```cpp
// From ents.h
vec falling; // falling velocity
uchar physstate; // one of PHYS_* (FLOAT, FALL, SLIDE, etc.)
```

Client sends falling state in position packet (`client.cpp` lines 896-908).

**Exploitation Potential:**
- Set `falling` vector to zero or positive values to prevent falling
- Manipulate `physstate` to `PHYS_FLOAT` or other states
- Modify gravity constants or disable gravity application
- Could enable:
  - Flying to unreachable vantage points
  - Hovering above combat
  - Avoiding fall damage
  - Accessing rooftops and elevated positions

**Example Modification:**
```cpp
// In physics update code - zero out gravity
d->falling = vec(0, 0, 0); // No falling ever
d->physstate = PHYS_FLOAT; // Always floating
```

**Server Validation:**
The server accepts the falling vector from the client (lines 2902-2911 in `server.cpp`) but doesn't validate if it's physically possible given the player's position history.

### 5. Entity Interaction Exploits

**Severity: MEDIUM**

**Location:** `src/fpsgame/entities.cpp` (teleport and jumppad handling)

**Description:**
Teleport and jumppad interactions are triggered client-side and then reported to server.

**Vulnerable Code:**
```cpp
// entities.cpp line 296-307
case TELEPORT:
{
    if(d->lastpickup==ents[n]->type && lastmillis-d->lastpickupmillis<500) break;
    // Client triggers teleport
    teleport(n, d);
    break;
}
```

**Exploitation Potential:**
- Trigger teleports without being near teleport entity
- Abuse teleport cooldown by manipulating `lastpickupmillis`
- Fake teleport events to confuse server state tracking
- Spam jumppad effects for abnormal velocity boosts

**Server-Side Handling:**
Server accepts `N_TELEPORT` messages (line 2933-2944) but doesn't validate:
- Whether the player was actually near the teleport entity
- Whether the teleport destination is valid
- Timing constraints on teleport usage

### 6. Edit Mode Exploits

**Severity: MEDIUM (context-dependent)

**Location:** Throughout codebase, checked via `m_edit` macro

**Description:**
Edit mode has different physics and validation rules. Players in `CS_EDITING` state bypass certain restrictions.

**Vulnerable Code:**
```cpp
// server.cpp line 2921 - edit mode bypass
if(!ci->local && !m_edit && max(vel.magnitude2(), (float)fabs(vel.z)) >= 180)
    cp->setexceeded();
```

**Exploitation Potential:**
- If edit mode checks can be bypassed, players could:
  - Fly freely in normal game modes
  - Have no collision detection
  - Avoid velocity limits
- Requires either:
  - Server misconfiguration
  - Client manipulation of edit mode state
  - Exploiting race conditions in mode transitions

### 7. Velocity Vector Manipulation

**Severity: MEDIUM-HIGH**

**Location:** `src/fpsgame/client.cpp` lines 891-895 (velocity encoding)

**Description:**
The client controls the velocity vector direction and magnitude sent to server.

**Vulnerable Code:**
```cpp
uint vel = min(int(d->vel.magnitude()*DVELF), 0xFFFF);
// ...
float velyaw, velpitch;
vectoyawpitch(d->vel, velyaw, velpitch);
uint veldir = (velyaw < 0 ? 360 + int(velyaw)%360 : int(velyaw)%360) + clamp(int(velpitch+90), 0, 180)*360;
```

**Exploitation Potential:**
- Send incorrect velocity vectors that don't match actual movement
- This can confuse:
  - Server-side prediction
  - Other clients' interpolation
  - Hit detection systems
- Could make player appear to be moving differently than they are

## Exploitation Techniques

### Basic Teleport Exploit

```cpp
// In modified client - before sendposition() is called
void cheat_teleport(float x, float y, float z) {
    player1->o.x = x;
    player1->o.y = y;
    player1->o.z = z;
    player1->resetinterp(); // Prevent visual snapping
}
```

### Speed Hack Implementation

```cpp
// Modify physics constants
void cheat_speedhack(float multiplier) {
    player1->maxspeed = 100.0f * multiplier; // Normal is 100
    // Keep velocity just under detection threshold
    if(player1->vel.magnitude() * DVELF >= 180) {
        player1->vel.mul(179.0f / (player1->vel.magnitude() * DVELF));
    }
}
```

### No-Clip Exploit

```cpp
// Patch collision detection functions
bool ellipsecollide(physent *d, const vec &dir, ...) {
    if(d == player1 && noclip_enabled) {
        return false; // No collision for player
    }
    // Original collision detection code
}
```

### Fly Hack

```cpp
// In physics update loop
void cheat_flyhack(bool enabled) {
    if(enabled) {
        player1->falling = vec(0, 0, 0);
        player1->physstate = PHYS_FLOAT;
        player1->timeinair = 0;
    }
}
```

## Current Mitigations

### Existing Server-Side Checks

1. **Velocity Magnitude Check** (`server.cpp` line 2921):
   - Checks if `vel.magnitude2() >= 180`
   - Marks player as "exceeded" but doesn't immediately prevent movement
   - Threshold is quite high, allowing significant speed increases

2. **Client Ownership Validation** (`server.cpp` line 2901):
   - Verifies that position updates come from the correct client
   - Prevents other players from controlling your position
   - Does NOT prevent self-manipulation

3. **Edit Mode Restrictions** (various locations):
   - Some checks are disabled in edit mode
   - Prevents exploitation in edit-only servers
   - Not a mitigation for normal gameplay

### Limitations of Current Mitigations

- **No Position Displacement Validation**: Server doesn't check if position changes are physically possible
- **No Server-Side Physics**: Server trusts client's physics calculations
- **No Trajectory Validation**: Server doesn't verify if movement path is valid (collision-free)
- **No Anti-Cheat System**: No pattern detection or anomaly detection
- **Limited Velocity Checks**: Threshold too high, only checks magnitude not displacement
- **No Teleport Validation**: Server doesn't verify proximity to teleport entities

## Recommendations

### Short-Term Mitigations

1. **Add Position Displacement Validation**
   - Track previous position for each client
   - Calculate maximum possible displacement based on maxspeed and time delta
   - Reject position updates that exceed possible movement
   - Implementation: `server.cpp` around line 2927

```cpp
// Proposed validation
vec displacement = pos - cp->state.o;
float maxdisplacement = cp->maxspeed * (curtime / 1000.0f) * 1.2f; // 20% tolerance
if(displacement.magnitude() > maxdisplacement && !m_edit) {
    // Reject update or rubber-band player back
    disconnect_client(sender, DISC_MSGERR);
    return;
}
```

2. **Improve Velocity Validation**
   - Lower threshold from 180 to more reasonable value (e.g., 120-130)
   - Check velocity against position displacement for consistency
   - Implement immediate kick on repeated violations

3. **Add Server-Side Collision Checks**
   - Perform basic raycasting from old position to new position
   - Reject movements that pass through solid geometry
   - This is expensive but critical for preventing no-clip

4. **Teleport Proximity Validation**
   - Server should verify player is within reasonable distance of teleport entity
   - Check distance < 16 units (as used in `entities.cpp`)
   - Rate-limit teleport events per player

5. **Monitor for Anomalous Patterns**
   - Track "exceeded" counts per player
   - Auto-kick after threshold (e.g., 5 violations in 30 seconds)
   - Log suspicious behavior for admin review

### Long-Term Solutions

1. **Server-Side Physics Authority**
   - Move physics simulation to server
   - Client sends input (keys pressed, mouse movement)
   - Server calculates resulting position
   - Client only renders result
   - This is a major architectural change

2. **Anti-Cheat System**
   - Implement client integrity checking
   - Monitor for common cheat patterns
   - Statistical analysis of movement patterns
   - Integration with third-party anti-cheat solutions

3. **Enhanced Logging and Admin Tools**
   - Server-side replay system
   - Detailed movement logs for suspicious players
   - Real-time monitoring dashboard
   - Automated flagging of impossible movements

4. **Code Obfuscation**
   - While not a security measure, can slow reverse engineering
   - Limit effectiveness of public cheats
   - Should be combined with real security measures

5. **Regular Security Audits**
   - Periodic review of client-server protocol
   - Penetration testing with custom clients
   - Community bug bounty program

### Best Practices for Server Administrators

1. **Enable Logging**: Capture position update violations
2. **Monitor Players**: Watch for suspicious movement patterns
3. **Use Demos**: Record gameplay for review of reported cheaters
4. **Community Moderation**: Empower trusted players to report cheaters
5. **Regular Updates**: Keep server software updated with latest patches

## Conclusion

The Sauerbraten client architecture has significant security vulnerabilities that allow client-side manipulation of movement and position. The primary issue is that **physics and collision detection occur client-side**, with minimal server validation. This allows modified clients to:

- **Teleport** anywhere on the map
- **Move at excessive speeds** (speed hacks)
- **Ignore collision detection** (wall glitching/no-clip)
- **Fly freely** (fly hacks)
- **Manipulate entity interactions** (fake teleports/jumppads)

The fundamental problem is a **lack of server authority** over player physics. The server acts primarily as a relay, trusting client-reported positions without adequate validation. 

### Risk Assessment

| Vulnerability | Ease of Exploit | Impact | Detection Difficulty |
|--------------|----------------|--------|---------------------|
| Teleportation | Easy | Critical | Hard (small distances) |
| Speed Hacks | Easy | High | Medium |
| No-Clip | Medium | Critical | Hard |
| Fly Hacks | Easy | High | Medium |
| Entity Exploits | Medium | Medium | Hard |
| Edit Mode Abuse | Hard | High | Easy |

### Recommended Priority

1. **HIGH PRIORITY**: Position displacement validation
2. **HIGH PRIORITY**: Improved velocity checks  
3. **MEDIUM PRIORITY**: Server-side collision checks
4. **MEDIUM PRIORITY**: Teleport proximity validation
5. **LOW PRIORITY**: Enhanced logging and monitoring

Implementing these recommendations will significantly improve the security posture of Sauerbraten servers and reduce the effectiveness of common cheating techniques.

---

**Document Version**: 1.0  
**Date**: 2025-11-16  
**Author**: Security Analysis Team
