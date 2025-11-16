# Sauerbraten Security Research Documentation

## Overview

This directory contains security research documentation for the Sauerbraten game client. The research identifies potential client-side vulnerabilities that could be exploited for cheating in multiplayer gameplay.

## Documents

### 1. SECURITY_ANALYSIS.md
**Comprehensive security vulnerability analysis**

This is the main document that provides:
- Detailed analysis of 7 major vulnerability categories
- Technical explanations of how each vulnerability works
- Assessment of severity and exploitability
- Current mitigation analysis
- Recommendations for fixes

**Key Findings:**
- **Teleportation**: Client can move to any position without server validation
- **Speed Hacks**: Client controls movement speed with weak server checks
- **No-Clip/Wall Glitching**: Collision detection is client-side only
- **Fly Hacks**: Gravity and physics are client-controlled
- **Entity Exploits**: Teleport/jumppad triggers not validated server-side

### 2. EXPLOIT_EXAMPLES.md
**Proof-of-concept code examples**

This document provides concrete code examples showing:
- How each vulnerability could be exploited
- Actual code that could be added to a modified client
- Combined exploits for maximum impact
- Anti-detection techniques
- Step-by-step exploitation workflow

**⚠️ WARNING**: These examples are for educational and defensive security purposes only. Do not use them to cheat.

## Purpose

This research was conducted to:

1. **Identify Security Weaknesses**: Document potential attack vectors
2. **Educate Developers**: Help developers understand exploitation techniques
3. **Guide Improvements**: Provide concrete recommendations for fixes
4. **Inform Admins**: Help server administrators detect and prevent cheating
5. **Academic Study**: Contribute to game security research

## Key Vulnerabilities

### Critical Issues

| Issue | Severity | Root Cause |
|-------|----------|------------|
| Position Manipulation | **CRITICAL** | No server-side position validation |
| Wall Glitching (No-Clip) | **CRITICAL** | Client-side collision detection |
| Speed Hacks | **HIGH** | Client controls physics with weak validation |
| Fly Hacks | **HIGH** | Client controls gravity and falling |
| Entity Exploits | **MEDIUM** | No proximity validation for teleports |

### Root Cause Analysis

The fundamental issue is **client-side authority over physics**:

```
Current Architecture (Vulnerable):
┌────────┐                    ┌────────┐
│ Client │                    │ Server │
│        │                    │        │
│ Physics├───Position────────►│        │
│ Collision                   │ Relay  │
│ Movement│◄──Position────────┤        │
└────────┘   (broadcast)      └────────┘

The client calculates everything and just tells
the server where it is. Server blindly trusts it.
```

```
Secure Architecture (Recommended):
┌────────┐                    ┌────────┐
│ Client │                    │ Server │
│        │                    │        │
│ Input  ├───Keys/Mouse──────►│ Physics│
│ Render │                    │ Collision
│        │◄──Position─────────┤ Movement│
└────────┘   (authoritative)  └────────┘

The server simulates physics based on client input.
Client cannot lie about position.
```

## Recommendations Summary

### Immediate Actions (Can Be Done Now)

1. **Add Position Displacement Validation**
   - Track last position for each client
   - Reject impossible movements based on time delta and maxspeed
   - Implementation: ~50 lines of code in `server.cpp`

2. **Improve Velocity Checks**
   - Lower threshold from 180 to 120-130
   - Add immediate kick on repeated violations
   - Implementation: ~20 lines of code

3. **Teleport Proximity Validation**
   - Verify player is near teleport entity
   - Rate-limit teleport usage
   - Implementation: ~30 lines of code

4. **Enhanced Logging**
   - Log position violations
   - Track "exceeded" counts per player
   - Auto-kick after threshold
   - Implementation: ~40 lines of code

**Estimated Total Effort**: 2-3 days for experienced developer

### Long-Term Solution

**Server-Side Physics Authority** (Major Architectural Change)
- Move physics simulation to server
- Client sends only input (keys, mouse)
- Server calculates and returns position
- Prevents all position-based cheats

**Estimated Effort**: 2-3 months for full implementation

## Implementation Priority

### Phase 1: Quick Wins (Week 1)
- [ ] Add position displacement validation
- [ ] Improve velocity threshold
- [ ] Add violation logging
- [ ] Implement auto-kick on repeated violations

### Phase 2: Enhanced Validation (Week 2-3)
- [ ] Teleport proximity checking
- [ ] Basic server-side collision raycasting
- [ ] Rate limiting for entity interactions
- [ ] Admin monitoring tools

### Phase 3: Long-Term (Month 2-3)
- [ ] Design server-authoritative architecture
- [ ] Implement server-side physics
- [ ] Client input prediction/compensation
- [ ] Comprehensive testing

### Phase 4: Maintenance (Ongoing)
- [ ] Regular security audits
- [ ] Pattern detection for new exploits
- [ ] Community reporting system
- [ ] Updates and patches

## Testing the Mitigations

### Test Cases

1. **Teleport Detection**
   ```
   Test: Player position changes by 1000 units in 50ms
   Expected: Server rejects update and kicks player
   ```

2. **Speed Hack Detection**
   ```
   Test: Player moves 500 units in 1 second (maxspeed=100)
   Expected: Server rejects update (max allowed ~120 units/sec)
   ```

3. **Wall Penetration Detection**
   ```
   Test: Player moves from (100,100,10) to (100,200,10) through wall
   Expected: Server raycast detects wall, rejects update
   ```

4. **Fake Teleport Detection**
   ```
   Test: Player sends N_TELEPORT when 100 units from teleporter
   Expected: Server rejects (proximity check failed)
   ```

### Regression Testing

After implementing fixes:
- Ensure normal gameplay is not affected
- Test legitimate edge cases (lag, jump pads, teleporters)
- Verify false positive rate is acceptable
- Test performance impact of server-side validation

## For Server Administrators

### Current Detection Methods

1. **Manual Observation**
   - Watch for impossible movements
   - Player suddenly appearing elsewhere
   - Moving faster than possible
   - Walking through walls

2. **Demo Recording**
   - Record suspicious players
   - Review for proof of cheating
   - Share with community

3. **Community Reports**
   - Encourage players to report cheaters
   - Investigate reported players
   - Maintain ban list

### Recommended Tools

1. **Server-Side Logging**
   - Enable verbose logging
   - Monitor for position anomalies
   - Track repeat offenders

2. **Admin Spectator Mode**
   - Observe suspicious players
   - Look for patterns
   - Gather evidence

3. **Third-Party Anti-Cheat**
   - Consider integration with existing solutions
   - Community-developed anti-cheat plugins

## For Developers

### Code Locations

Key files to review for implementing fixes:

- `src/fpsgame/server.cpp`: Server-side message handling
  - Line 2895-2930: N_POS message handling (position updates)
  - Line 2933-2944: N_TELEPORT message handling
  - Line 2921: Current velocity check

- `src/fpsgame/client.cpp`: Client-side networking
  - Line 856-909: Position packet encoding
  - Line 911-917: Position sending

- `src/engine/physics.cpp`: Physics and collision
  - Line 463-517: Collision detection functions
  - Client-side collision (needs server-side equivalent)

- `src/shared/ents.h`: Entity structures
  - Line 62-112: physent structure (physics entity)
  - Contains position, velocity, physics state

### Code Review Checklist

When implementing fixes:
- [ ] Server validates all client-provided positions
- [ ] Position changes are checked against time and maxspeed
- [ ] Entity interactions verify proximity
- [ ] Repeated violations result in kicks
- [ ] Logging is comprehensive for debugging
- [ ] Performance impact is acceptable
- [ ] False positives are minimized
- [ ] Normal gameplay is not affected

## Ethical Considerations

This research is published with the goal of **improving security**, not enabling cheating.

**Responsible Disclosure:**
- Vulnerabilities documented here are present in the current public codebase
- No 0-day exploits are being disclosed
- Fixes are recommended and should be prioritized
- Community is encouraged to contribute patches

**Please Use Responsibly:**
- Do not use these techniques to cheat in online games
- Respect other players and server administrators
- Contribute fixes rather than exploits
- Report new vulnerabilities responsibly

## Contributing

If you have:
- Additional vulnerability discoveries
- Proof-of-concept mitigations
- Test cases for validation
- Performance optimization ideas

Please consider contributing:
1. Open an issue describing the security concern
2. Submit patches with fixes
3. Review and test proposed mitigations
4. Help with documentation

## References

### Related Research
- Game Security Best Practices
- Client-Server Game Architectures
- Anti-Cheat Technologies
- Network Protocol Security

### Further Reading
- Valve Anti-Cheat (VAC) whitepaper
- Riot Vanguard technical overview
- Game hacking prevention strategies
- Authoritative server design patterns

## License

This security research documentation is provided for educational purposes under the same license as the Sauerbraten source code (ZLIB license).

## Contact

For security concerns or questions about this research:
- Open a GitHub issue (for non-sensitive matters)
- Contact repository maintainers directly (for sensitive disclosures)

---

**Document Version**: 1.0  
**Last Updated**: 2025-11-16  
**Status**: Initial Research Phase
