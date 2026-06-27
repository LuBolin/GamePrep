# Game Security and Defensive Anti-Cheat

See also: [[ue_networking_and_replication]], [[systems_programming_for_game_interviews]], [[advanced_gameplay_patterns_specialist]], [[ue_network_prediction_rollback_proof]], [[ue_hands_on_projects]].

**Priority:** P2 for broad Unreal interviews, P1 for networked gameplay, backend, competitive multiplayer and engine/platform roles.  
**Expected depth:** D2-D3 for generalists; D4 for anti-cheat/security-facing roles.  
**Safety boundary:** This chapter explains threat models, detection surfaces and defensive design. It deliberately avoids operational instructions for cheating, bypassing anti-cheat, credential theft, DLL injection recipes or exploit development.

## 1. Security Model: The Client Is Presentation Plus Intent

For a networked game, the client is a useful but untrusted participant. It owns local input capture, prediction and presentation. The server owns authoritative durable gameplay state: health, inventory, score, cooldowns, economy, match outcome and anti-abuse telemetry. Unreal's replication model already pushes in this direction: server authority and owning-connection rules determine which side may mutate or request state. [SRC-NET-001] [SRC-NET-002]

Strong rule:

> Do not ask "how do I hide the truth from the client?" first. Ask "why does the client need this data, at what precision, for how long, and can the server validate the action without trusting the client's result?"

## 2. Cheat Engine-Style Memory Tampering

Cheat Engine is a public memory/debugging/modification tool. Its own site describes it as a tool for private/educational use and warns users not to violate game/application terms. Public release notes also show memory scanning, disassembly, symbol and injection-adjacent capabilities. [SRC-SEC-007]

Defensive model, high level:

- A local attacker can inspect and alter client process memory.
- Obfuscating a client-side value may slow casual tampering but does not make it authoritative.
- If currency, ammo, cooldowns, position, hit results or inventory are accepted from the client as truth, memory editing can become gameplay cheating.
- Anti-tamper and commercial anti-cheat may detect classes of tampering, but game design must still minimise trusted client outcomes.

Better defences:

1. Server owns durable state and recomputes outcomes from bounded intent.
2. Client values are presentation caches, not source of truth.
3. Server validates rate, sequence, ownership, cooldown, resources, distance, line of sight and state transitions.
4. Important rewards and economy operations use operation IDs and idempotency.
5. Telemetry records impossible transitions and repeated near-threshold behaviour.
6. Sensitive checks run in packaged/dedicated-server conditions, not only PIE.

## 3. DLL Injection and Module Tampering, Defensively

A DLL is a dynamically linked library loaded into a process so its code/data can be used at runtime. Windows documents DLLs and DLL search/loading behaviour as normal platform features. The defensive security concern is that unexpected modules or load paths can alter code execution, inspect memory, hook functions or draw overlays. [SRC-SYS-004]

Defensive discussion should stay at this level:

- Restrict plugin/module load paths and signed/trusted deployment.
- Avoid writable/executable memory by default; change page permissions deliberately and briefly when a legitimate JIT/shader/tool path requires it. VirtualAlloc/VirtualProtect are page-level tools, not normal gameplay allocation APIs. [SRC-SYS-001]
- Use platform-supported code signing, anti-cheat SDKs and store/platform policy where applicable.
- Log loaded modules/build IDs in diagnostic builds when policy allows.
- Treat local module integrity as one layer, not proof that gameplay state can be trusted.

Do not present "how to inject a DLL" as interview study material. For game interviews, the useful answer is threat modelling plus server-authoritative design.

## 4. Replay Attacks

A replay attack resends a previously valid message/action so it is accepted again. TLS protects the transport between endpoints, but application-level replay can still exist when your server accepts stale, duplicated or reordered commands as new business operations. TLS 1.3 provides modern channel security, and OWASP recommends current TLS configuration, but gameplay/economy operations still need application semantics. [SRC-SEC-003] [SRC-SEC-004]

Defensive contract:

| Field | Purpose |
|---|---|
| authenticated player/session | bind command to identity |
| operation ID or monotonic sequence | detect duplicates and gaps |
| server-issued nonce/challenge when needed | prevent reuse outside intended window |
| bounded timestamp/tick | reject stale actions |
| idempotency key for rewards/purchases | safe retry without duplicate effect |
| MAC/signature for backend service calls | detect tampering across trusted services |
| server-side state validation | prove the action is currently legal |

For gameplay RPCs, Unreal ownership and authority are necessary but not sufficient. A valid Server RPC can still carry stale or duplicated intent. Add sequence/operation IDs to gameplay systems that grant rewards, consume resources, submit match results or commit irreversible transitions. [SRC-NET-001]

## 5. ESP, Aimbot and Information Control

ESP-style cheats exploit information available on the client: enemy positions, hidden entities, loot, projectiles, traces, skeletal transforms or debug data. Aimbots exploit the client's ability to observe targets and submit plausible input/aim.

Defence is layered:

- **Relevancy and interest management:** do not replicate what a client cannot reasonably need now. Actor relevancy and Replication Graph-style interest management are performance features that also reduce unnecessary information exposure. [SRC-NET-001] [SRC-NET-015]
- **Precision minimisation:** replicate coarse or delayed data when full precision is not needed.
- **Server-side visibility/range checks:** validate line of sight, distance, target state, team rules and weapon constraints for consequential actions.
- **Input plausibility:** compare aim deltas, reaction time, target acquisition, recoil/spread policy and repeated near-perfect events.
- **Replay review:** store compact evidence for suspicious events so humans/tools can inspect context.
- **Anti-overlay/tamper layers:** platform/commercial anti-cheat can help, but do not rely on it as the only control.

Do not promise to "stop all ESP" in a client-rendered multiplayer game. If the client must render an enemy, it has some information about that enemy. The design goal is minimising unnecessary information, validating consequential actions and making abuse expensive to sustain.

## 6. TCP, UDP and Game Synchronisation

TCP provides a reliable byte stream with ordering and congestion control. It is suitable for patchers, login, web/backend APIs, chat and low-frequency reliable services. RFC 9293 is the current TCP specification. [SRC-SEC-001]

UDP provides datagrams with minimal protocol mechanism and does not guarantee delivery, ordering or duplicate protection. RFC 768 explicitly leaves applications that need ordered reliable delivery to use TCP. [SRC-SEC-002]

Realtime games often build reliability, ordering and prioritisation above UDP because not every update should wait behind an older lost packet. State snapshots, input commands, unreliable cosmetic events, reliable inventory changes and backend transactions have different delivery semantics.

| Model | Core idea | Good fit | Risk |
|---|---|---|---|
| State synchronisation | server sends current state/snapshots; clients interpolate/predict | broad UE replication, shooters, action games | bandwidth, correction, stale state |
| Input/frame synchronisation | clients exchange deterministic inputs per frame/tick | lockstep strategy/fighting/rollback-ready games | determinism and stall/rollback complexity |
| Rollback | predict, restore old state and resimulate after late input | deterministic competitive action | side-effect control and state capture cost |

Gaffer on Games' state synchronisation material is a practical source for snapshot/state design and priority schemes, while UE's built-in replication is a higher-level engine framework with its own authority/relevancy/RPC contracts. Use the vocabulary carefully. [SRC-SEC-006] [SRC-NET-001]

## 7. Symmetric Crypto, Asymmetric Crypto and HTTPS

Symmetric cryptography uses the same secret key for encryption/decryption or message authentication. It is fast and used for bulk data once endpoints share keys. Asymmetric cryptography uses public/private keys for identity, key exchange or signatures; it is slower but solves distribution and authentication problems. OWASP's cryptographic guidance stresses using well-reviewed algorithms/libraries, authenticated modes, strong keys and correct key management rather than inventing custom crypto. [SRC-SEC-005]

HTTPS is HTTP over TLS. TLS gives server authentication through certificates, negotiated keys, confidentiality and integrity for the channel when configured correctly. It does not automatically make a malicious client honest, prevent application replay of a valid command, or validate game rules. [SRC-SEC-003] [SRC-SEC-004]

Man-in-the-middle risk appears when an attacker can intercept/alter traffic and the client accepts the wrong endpoint or insecure transport. Use:

- TLS 1.2/1.3 with modern configuration and certificate validation;
- no certificate-validation bypass in production builds;
- HSTS/pinning only with an operational rotation plan;
- signed backend-to-backend requests where service boundaries need message-level proof;
- server validation for gameplay/economy semantics after transport security succeeds.

## 8. Anti-Cheat Design Layers

| Layer | What it can do | What it cannot prove alone |
|---|---|---|
| Server authority | reject illegal outcomes and own durable state | prevent all client information disclosure |
| Network relevancy | reduce unnecessary replicated data | hide data the client must render |
| Protocol sequencing | block duplicates/stale commands | prove human input |
| Telemetry/anomaly scoring | find suspicious patterns | avoid false positives without review |
| Binary integrity/tamper checks | detect classes of local modification | guarantee an uncompromised client |
| Commercial anti-cheat | add platform/process/kernel-level signals where allowed | replace secure game rules |
| Human review/replay | resolve context-heavy cases | scale without good evidence packets |

Privacy, legality and platform policy matter. Anti-cheat often observes sensitive local process/device data. A strong interview answer mentions user trust, false-positive handling, appeals, regional law, platform terms and data minimisation.

## 9. Strong Interview Answer Patterns

**How would you fight Cheat Engine editing health?**

> Health must be server-authoritative. The client can cache predicted/presented health, but damage/heal/resource transitions are computed on the server from validated intent or server events. I would add sequence/operation IDs for effects that can retry, replicate results through normal state, and log impossible transitions. Client memory tamper may change UI, but it should not change authoritative health.

**How do you fight ESP?**

> First reduce information: relevancy, culling, team/visibility rules and precision where possible. Then validate actions on the server with line-of-sight/range/weapon/state checks. Finally add telemetry and replay review for suspicious target acquisition and tracking. I would not claim perfect prevention once the client must render the target.

**How do you fight aimbot?**

> Server-side hit validation is mandatory. Then I compare input/aim patterns over time: acquisition speed, smoothness, target switching, recoil/spread consistency, impossible visibility and repeated high-confidence near-threshold events. Detection feeds review or graduated action, not a single magic threshold.

**What does HTTPS solve for a game?**

> It protects the channel to backend services from passive reading and many active MITM attacks when configured and validated correctly. It does not validate gameplay rules, stop a malicious client, or prevent application-level replay. I still need server-side command validation, idempotency and secure key handling.

## 10. Hands-On Verification

Extend Project 3 with a Security and Replay Lab:

1. Add sequence/operation IDs to fire, reward and inventory commands.
2. Write tests for duplicate, stale, out-of-order and wrong-owner commands.
3. Build a server-side hit validation matrix: range, LOS, ammo, cooldown, target alive/team/invulnerable.
4. Add relevancy experiments showing how much hidden target data each client receives.
5. Add a telemetry record for suspicious aim/hit events without storing excessive personal data.
6. Create a replay-review packet for one rejected and one accepted event.
7. Verify backend calls use HTTPS/TLS without certificate bypass in the target build.
8. Document which data remains visible to the client and why.

## Source Map

- UE authority/relevancy/RPCs: [SRC-NET-001] [SRC-NET-002] [SRC-NET-015]
- TCP/UDP/sync models: [SRC-SEC-001] [SRC-SEC-002] [SRC-SEC-006]
- TLS/HTTPS/crypto: [SRC-SEC-003] [SRC-SEC-004] [SRC-SEC-005]
- Cheat Engine and local tamper context: [SRC-SEC-007]
- DLL/modules/page permissions: [SRC-SYS-001] [SRC-SYS-004]
