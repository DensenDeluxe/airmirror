# AirMirror

## A proposal for differential IEEE 802.11 state auditing in wifit3

**Status:** design proposal; no implementation has started  
**Intended audience:** wifit3 maintainers and contributors  
**Initial scope:** authorized laboratory testing with two adapters minimum and three adapters recommended

## Executive summary

AirMirror is a proposed wifit3 campaign for detecting implementation inconsistencies in access points without first having to model the complete IEEE 802.11 state machine.

The central idea is paired differential testing. AirMirror sends a normal control exchange and a second exchange whose only intended treatment difference is one standards-backed, semantics-preserving variation. Identity, order, sequence handling, transmitter path, and prior target state remain controlled or explicitly balanced nuisance variables. AirMirror then asks a deliberately narrow question:

> Did two exchanges that should mean the same thing produce different, repeatable protocol states?

This changes the oracle problem. A conventional fuzzer must know whether an arbitrary response is correct. AirMirror instead uses the target's response to the control exchange as a local baseline. It reports only stable differences between the control and mutation after repeating the experiment, randomizing trial order, and balancing variant assignments across the participating adapters.

An independent adapter verifies what was actually transmitted over the air. A trial is invalid if the required observer set did not receive the intended frame or if its observations conflict under the declared policy. This detects many transmitter, descriptor, and USB-path deviations before they can be mistaken for target behavior; it does not eliminate PHY, retry, RF-path, or target-reception uncertainty.

OTA observation alone does not prove that the target received a request. A response is direct evidence that the target processed the exchange. A no-response outcome is eligible for comparison only when an applicable unicast request received a recipient ACK attributable to the target under the trial-isolation rules. Without either form of target-reception evidence, the outcome is `target-reception-unconfirmed`, not a divergence.

Two adapters are sufficient for a minimum verified mode: one transmits both variants while the other observes, and their roles are exchanged in the next block. The preferred research setup uses three adapters: two maintain independent station contexts used for control and mutation while a third remains a dedicated observer. The two transmitters exchange variant assignments, and the observer role can later rotate across all three devices. Frames are tightly interleaved, not transmitted simultaneously.

Four and five adapters do not change the core oracle. They are candidate diagnostic profiles whose value must be demonstrated during calibration. Four radios permit two transmitters plus two independent observers. Five radios can provide a three-observer ensemble or reserve additional radios as passive sentinels on affiliated channels. More receivers may expose capture-path failures, but they do not automatically increase causal or statistical confidence. These are scaling profiles, not MVP requirements.

The first implementation is intentionally small:

- one target and one channel;
- two adapters as the minimum verified mode;
- three adapters as the preferred balanced mode with separate control, mutation, and observer roles;
- one carefully justified metamorphic relation;
- discrete response fields only;
- no timing oracle;
- no general-purpose fuzzing;
- no automatic testcase minimization;
- no MLO or 6 GHz dependency;
- no promise that every wifit3 chipset is immediately suitable;
- an explicit authorized-lab gate for all active trials.

The proposal is designed around capabilities that wifit3 already has: direct userland USB drivers, raw frame injection, fake MAC support, interface leasing, ACK accounting, and a `WlanArray` that combines observations from multiple adapters.

## 1. Motivation

### 1.1 The oracle problem

Protocol fuzzing can generate a large number of unusual frames, but detecting a meaningful failure is difficult. Crashes and reboots are visible. Subtle state-machine errors are not.

For example, an access point may:

- accept an exchange it should reject;
- reject a valid exchange only after a harmless representation change;
- return a different status code;
- advertise a different security configuration in its response;
- leave behind a different association state;
- handle one station identity differently from another because of stale state.

Determining whether each response is correct normally requires a detailed model of the relevant IEEE 802.11 clauses, amendments, optional capabilities, and target configuration. That model is expensive to build and easy to get wrong.

AirMirror does not attempt to solve the complete specification-oracle problem. It starts from metamorphic relations: transformations for which control and mutation are expected to have the same protocol meaning. A stable response difference is therefore useful evidence even before the full internal state of the target is known.

### 1.2 The attribution problem

Over-the-air testing introduces a second uncertainty: the planned test frame may not be the frame that reached the target.

A transmitting adapter may:

- reject the frame before transmission;
- rewrite sequence or retry fields;
- insert, remove, or alter fields;
- use a different source address than requested;
- report successful USB submission even though no usable RF transmission occurred;
- behave differently after changing its armed fake MAC;
- apply chipset- or firmware-specific filtering.

Without an independent receiver, a missing response can be incorrectly attributed to the target. This is especially important in wifit3 because it intentionally bypasses the operating system's normal Wi-Fi stack and drives heterogeneous USB chipsets directly.

AirMirror makes independent over-the-air verification part of the validity condition of each trial, rather than an optional debugging aid.

## 2. Why wifit3 is a suitable platform

AirMirror is not proposed as a generic framework that happens to be embedded in wifit3. Its design depends on wifit3-specific strengths.

### 2.1 Direct radio control

Wifit3 owns the injection path from the frame bytes down to each supported USB device. This makes it possible to retain the intended frame, record the selected device and configuration, and compare the intent with an independently captured over-the-air copy.

### 2.2 Multiple adapters as one session

`WlanArray` already manages several `WlanInterface` instances, assigns channels, merges their received traffic, and tracks signal information per card. AirMirror can reserve two members as independent station contexts and one or more additional members as observers without introducing a second operating-system capture stack. A reduced two-adapter mode can execute both variants through one transmitter and exchange roles between blocks.

### 2.3 Scoped device state

The existing lease mechanism scopes channel selection, fake MAC configuration, BSSID matching, and ACK accounting for one interface. That is close to the per-device lifecycle needed by a controlled trial, but AirMirror also needs atomic logical ownership of the complete transmitter and observer set.

An array-level coordinator lock must claim every required device before changing any of them, then stop hopping, tune, arm identities and ACK accounting, and register observers in staged order. Physical device configuration is not atomic. Failure at any step must release every logical claim, restore every reachable device in deterministic reverse order, and mark disconnected devices as lost. A trial must never start from a partially configured set.

### 2.4 Existing protocol primitives

Wifit3 already parses authentication and association traffic, RSN capabilities, PMF state, status and reason codes, and several forms of SSID evidence. AirMirror should reuse these typed packets and builders instead of introducing a parallel Scapy-based packet stack.

### 2.5 Honest limitation: chipset diversity

The number of supported chipsets is not automatically an advantage. It is also a source of experimental variance. Driver paths differ, firmware behavior differs, and not every adapter will be appropriate for reproducible injection.

The MVP should qualify a small reference set instead of claiming universal compatibility. The preferred qualification target is a three-adapter set, because it exercises separate station contexts and independent observation. A two-adapter pair remains the minimum operational configuration. Additional devices can be admitted through explicit capability and repeatability tests later.

## 3. Core concept

An AirMirror experiment consists of a control exchange, a mutation exchange, independent observation, repetition, balanced variant assignment, and optional observer rotation.

```text
Minimum verified mode: two adapters

  Block 1: Adapter A transmits Control and Mutation; Adapter B observes
  Block 2: Adapter B transmits Control and Mutation; Adapter A observes

Preferred balanced mode: three adapters

  Pair 1: Adapter A = Control,  Adapter B = Mutation, Adapter C = Observer
  Pair 2: Adapter A = Mutation, Adapter B = Control,  Adapter C = Observer

  Optional receiver-bias rotation:

  Block 1: A/B transmit, C observes
  Block 2: B/C transmit, A observes
  Block 3: C/A transmit, B observes
```

Control and mutation must not be transmitted simultaneously in the initial design. Simultaneous transmission would add collisions and may cause the two trials to affect the same global AP state. They are paired and closely interleaved, with randomized order and a controlled reset or fresh station identity between trials.

The three-adapter mode is not about microsecond synchronization. Its value is stable separation of responsibilities: two independently armed station contexts, no fake-MAC rearming within a paired comparison, and a third radio that verifies both requests and responses.

### 3.1 Hardware profiles

AirMirror should name the active hardware profile explicitly in transcripts and reports.

#### Two-radio verified

- one transmitter executes both variants during a block;
- one independent observer verifies both variants;
- transmitter and observer exchange roles in the next block;
- the same physical TX path is used for control and mutation within a block.

This profile has the lowest hardware barrier and is sufficient to demonstrate feasibility of the observer and differential-oracle method.

#### Three-radio balanced

- two transmitters maintain separate station identities and protocol state;
- the third adapter observes both exchanges;
- control and mutation assignment is swapped between transmitters;
- the observer can be rotated in later blocks to expose receiver-specific blind spots;
- all transmissions remain sequential and closely interleaved.

This is the preferred profile for strong evidence because it balances and exposes transmitter bias without sacrificing an independent observer or repeatedly rearming one station context.

#### Four-radio corroborated

- two transmitters run the same balanced crossover as three-radio mode;
- two observers are independently assigned to capture every request and target response;
- observers should preferably differ by chipset family, antenna position, or both;
- one observer may be positioned to verify transmitter egress while the other better approximates reception near the target;
- per-observer results remain separate instead of being collapsed into the global deduplicated stream.

Four radios are a candidate high-confidence diagnostic setup, not a proven sweet spot. The second observer may reduce invalidations caused by a single missed capture and can reveal stable RX filtering differences between chipsets. Phase 0 measurements must establish whether that benefit survives added USB load, RF coupling, placement effects, and correlated receiver loss.

#### Five-radio ensemble

The primary five-radio layout keeps two balanced transmitters and uses three observers. This permits corroboration across heterogeneous RX paths and makes a persistent single-observer blind spot visible without abandoning independent OTA verification.

A later alternative is a cross-channel sentinel layout:

- two transmitters and one primary observer remain on the target channel;
- two passive sentinels watch known affiliated BSSIDs or links on other channels;
- sentinels record cross-BSSID disappearance, capability changes, or link resets temporally associated with a target-channel trial.

The sentinel layout cannot establish causality from temporal proximity alone. Any candidate cross-channel effect needs repeated negative controls and a design that separates target-wide behavior from unrelated channel activity. The layout also depends on a trustworthy topology model and sufficient supported radios and bands. It is therefore future work, especially for 6 GHz and MLO.

#### Full role rotation

For confirmation of an already interesting result, AirMirror can test every unordered transmitter pair while all remaining adapters observe. Four adapters produce six transmitter pairs and five produce ten. Each pair still swaps control and mutation assignments before moving to the next block.

This full rotation can help separate the transformation effect from TX- and RX-card effects only when placement, time blocks, identity allocation, and reset conditions are controlled. It also increases experiment duration and target-state drift. It is a confirmation mode, not the normal discovery path.

The scheduler and transcript should therefore model role collections from the beginning:

```text
transmitters: [A, B]
observers: [C, D, E]
```

rather than hard-coding exactly one control card, one mutation card, and one observer card.

### 3.2 Terminology

- **Control:** the canonical exchange used as the local behavioral baseline.
- **Mutation:** the same exchange after applying one justified metamorphic transformation.
- **Metamorphic relation:** the reason control and mutation should have the same protocol meaning.
- **Observer:** an adapter that captures the transmitted request and target response but does not participate in the exchange.
- **Observer ensemble:** two or more observers whose per-card copies remain independently attributable.
- **Verified trial:** a trial in which at least the required observer set captured a request matching the expected normalized frame.
- **Corroborated trial:** a verified trial independently confirmed by two or more observers under the experiment's evidence policy.
- **Target-reception evidence:** a matched target response or, for an eligible unicast request, a recipient ACK attributable to the target under the trial-isolation rules.
- **Negative control:** a comparison expected to remain equivalent, such as control-versus-control through both transmitter paths, used to measure the method's false-divergence baseline.
- **Washout:** a defined transition and verification procedure that restores or replaces target and station state between trials.
- **Response fingerprint:** the discrete, normalized protocol outcome used for comparison.
- **Transmitter crossover:** swapping control and mutation assignments between the two transmitting adapters.
- **Observer rotation:** moving the receive-only role between adapters in separate blocks to expose RX-specific blind spots.
- **Divergence:** a stable difference between control and mutation response distributions that satisfies the complete frozen analysis plan, including its controls, state-isolation, reception-evidence, crossover, ordering, and invalidity requirements.

## 4. Experimental design

The experimental design is more important than the number of supported mutations. A result is useful only if ordinary RF loss, stale station state, and adapter-specific behavior have been excluded as plausible explanations.

### 4.1 Trial lifecycle

A single trial should follow a strict lifecycle:

1. Load a versioned experiment definition and analysis plan.
2. Atomically reserve, then configure the complete transmitter and observer set.
3. Establish and verify the declared starting condition.
4. Assign a fresh or deliberately reused locally administered station MAC according to the experiment definition.
5. Build the control or mutation exchange from the same semantic input.
6. Record the exact intended bytes, transmitter configuration, and trial-scoped ACK-correlation state.
7. Transmit the exchange without any overlapping AirMirror transmission.
8. Capture the request independently on every assigned observer.
9. Apply the declared request-validity policy; retain invalid attempts in the transcript.
10. Capture the target response within a bounded window and apply the declared response-capture policy.
11. Determine target-reception evidence from a matched response or, where applicable, a correlated recipient ACK.
12. Normalize the response into discrete comparison fields.
13. Perform and verify the declared cleanup or washout before the next trial.
14. Store the complete trial transcript.

Each pair is repeated with randomized control/mutation order. In two-radio mode, transmitter and observer exchange roles for the next block. In three-radio mode, control and mutation assignments are swapped between the two transmitters while the independent observer remains fixed for the block. Four- and five-radio modes keep every observer copy separate while applying the same transmitter crossover. A stronger campaign can then rotate roles and repeat the balanced crossover.

A missing response is never sufficient by itself. If the request is not eligible for an ACK, or neither a response nor a recipient ACK attributable to the target is observed, the result is `target-reception-unconfirmed`. In particular, silence after a broadcast request cannot be interpreted as target behavior in the MVP.

### 4.2 Why assignments and roles must be balanced

If adapter A always sends the control and adapter B always sends the mutation, a target difference can be caused by the adapters rather than the frame transformation. Different antennas, PHY defaults, retry behavior, transmit power, or firmware can all become hidden independent variables.

Two-radio mode avoids this by sending both variants through one transmitter during a block and then exchanging transmitter and observer roles. Three-radio mode uses the stronger balanced design: A sends control while B sends mutation, then A sends mutation while B sends control, with C independently observing both pairs. A divergence is not promoted unless it follows the transformation rather than the transmitting adapter.

Observer rotation is a second, optional control. If a difference disappears when a different chipset becomes the observer, the result may be an RX visibility issue rather than target behavior and must be reported as such.

### 4.3 Fresh identities and state isolation

Association-related tests create state in the target. Reusing one MAC can make a mutation inherit state from its control or vice versa. Always using new MACs can instead fill a target's station table and create a different artifact. Three-radio mode helps by keeping two station contexts armed independently, but it does not remove the need for explicit state isolation between paired trials.

Each experiment must therefore declare its identity and carry-over policy:

- new matched MAC pair per repetition;
- one MAC with an explicit reset between variants;
- controlled reuse when the metamorphic relation concerns retransmission;
- bounded identity count with cleanup and cooldown;
- the exact reset or washout operation and how its completion is verified;
- the maximum number of attempts before the starting condition must be re-established.

The first relation should choose the simplest policy and avoid tests that require manipulating legitimate clients. A complete comparison block should include both variant orders (`control→mutation` and `mutation→control`, abbreviated `CM/MC`) and the transmitter crossover appropriate to the hardware profile. Identity allocation, variant order, and transmitter assignment should be independently balanced or randomized so none becomes a proxy for elapsed time or accumulated target state.

When the target cannot expose a verifiable reset condition, the experiment must use fresh matched identities and bounded blocks, then classify evidence that changes across blocks as carry-over-sensitive or inconclusive. A crossover alone controls the physical transmitter; it does not control MAC identity, trial order, or target state.

### 4.4 Negative controls

Before interpreting a control-versus-mutation comparison, AirMirror should run control-versus-control through the same transmitter paths, observer policy, identity allocation, scheduling, and washout procedure. This measures natural false divergence, adapter-path differences, and normalization failures without relying on the proposed transformation.

The experiment definition must state how many negative-control blocks are required and the maximum acceptable divergence and invalidity imbalance. If the negative control fails, the corresponding mutation block is inconclusive. A mutation-versus-mutation repeat can be added when it helps distinguish transformation instability from control-path instability.

### 4.5 Repetition and stability

A single differing response is not evidence. RF loss, channel contention, retries, target load, and USB scheduling make isolated differences expected.

The first implementation should use a conservative, preregistered reproducibility gate instead of pretending that one statistical test fits every experiment. Every versioned experiment definition must declare its comparison cells, minimum sample counts, stopping rules, and acceptance thresholds before active trials begin.

- compare only valid, observer-confirmed trials with target-reception evidence where the outcome depends on silence;
- require a minimum number of valid trials in every variant × transmitter × order/block cell used by the inference;
- require the dominant response fingerprint of each variant to be stable;
- require the difference to recur after the crossover defined by the active hardware profile;
- retain the complete per-observer visibility matrix instead of reducing it to a single Boolean;
- report invalid attempts by variant, transmitter, observer, order, and block;
- declare maximum total invalidity and maximum between-cell invalidity imbalance;
- classify asymmetric observer loss, ACK loss, or early stopping as inconclusive rather than silently discarding it;
- report counts and raw outcomes, not only a binary vulnerability label;
- classify unstable results as inconclusive.

Phase 0 should calibrate exact thresholds on known-good and deliberately faulty test targets. The thresholds, dominant-fingerprint rule, and stopping rule must be versioned with the experiment definition rather than hidden as UI constants. AirMirror does not need one universal statistical test, but it does need a reproducible analysis plan for each relation.

## 5. The observer layer

Independent observation is the primary distinction between AirMirror and a conventional send-and-wait campaign. The minimum profile has one observer; larger profiles retain a separate observation from every assigned card.

### 5.1 What the observer proves

Each observer should verify at least:

- transmitter/source address;
- receiver and BSSID addresses;
- frame type and subtype;
- sequence fields when they are part of the relation;
- frame body and Information Elements after excluding explicitly permitted hardware changes;
- that the frame was observed on the expected channel;
- that the target response corresponds to the current trial identity.

An observer does not prove that the target successfully decoded the frame. It proves that the planned test reached the air in an independently observable form. The distinction must remain explicit in reports.

### 5.2 Multi-observer corroboration

More observers improve evidence only when their copies remain attributable and their failure modes are not treated as automatically independent. Three co-located adapters on one USB controller do not justify a naive majority vote.

Every experiment definition must separate the rule that makes a trial usable from the label that describes its evidence. It should declare:

- required and optional observer identities;
- a request-validity rule such as `primary`, `any`, `quorum(k)`, or `all-required`;
- a response-capture rule such as `any`, `quorum(k)`, or `all-required`;
- the matching and conflict rules for retransmitted requests and responses;
- the target-reception policy used when the target produces no response.

An `any` policy can reduce loss from one receiver but admits a trial through different observation paths. A quorum policy demands stronger corroboration but may create variant-dependent invalidity. Both choices are legitimate when declared in advance and accompanied by the full per-observer matrix.

AirMirror should expose evidence labels rather than hide the underlying observations:

- **verified:** the minimum observer policy confirmed the intended request;
- **corroborated:** at least two observers independently confirmed it;
- **heterogeneous-corroborated:** observers from at least two chipset families confirmed it;
- **observer-conflict:** observers produced incompatible valid observations or stable response visibility differences.

`heterogeneous-corroborated` describes the capture paths; it is not automatically a higher statistical confidence level. `observer-conflict` is not resolved by majority vote. It is retained as evidence of a possible RX-path, placement, retransmission-matching, or capture-normalization problem. A conflict may also motivate a separate receiver-differential experiment, but it must not be silently converted into a target finding.

### 5.3 Required WlanArray hook

Wifit3 currently suppresses frames transmitted by one of its own registered MAC addresses before they enter normal sink state. That is correct for scanning and attack campaigns: a second card hearing our own injected frame must not create a fake client or access point.

AirMirror needs a narrowly scoped observation path before this suppression and before cross-card deduplication removes per-observer identity. The proposed hook should:

- be campaign-scoped and removable;
- identify which physical member received every copy;
- run before own-transmission suppression;
- avoid mutating `WlanSink`;
- avoid adding beacons, clients, handshakes, or decloak evidence;
- allow a strict filter so unrelated traffic is not retained;
- support an observer collection rather than a single hard-coded card;
- preserve the normal scanner behavior when no observers are registered.

This infrastructure is valuable enough to be reviewed as a separate first PR. It is also the main point where AirMirror touches shared receive architecture.

### 5.4 Invalid trials

A trial must be marked invalid, rather than as a target failure, when:

- the required observer policy does not capture the request;
- the captured request fails normalized byte comparison;
- an assigned required observer changes channel or disconnects;
- the transmitter reports an injection error;
- an unrelated response cannot be distinguished from the trial response;
- cleanup or initial-state establishment fails.

Invalid trials remain in the transcript for diagnosis but do not contribute to the control/mutation result. Optional observers may miss a frame without invalidating the trial, but their absence and role must still be recorded.

Observer loss can itself depend on the mutation, transmitter, order, or USB load. AirMirror must therefore report attempt and invalidation counts by variant × transmitter × observer × order/block, including optional-observer misses. A result is inconclusive when invalidity exceeds the experiment's limit or is materially imbalanced between comparison cells. Discarding failed observations without this accounting would create selection bias.

## 6. Response fingerprints

The initial differential experiment should compare discrete protocol outcomes only. Candidate fields include:

- no response versus response, only when the negative outcome has target-reception evidence;
- authentication status code;
- association status code;
- deauthentication or disassociation reason code;
- response subtype;
- normalized set of response IEs;
- advertised RSN and PMF capabilities.

The normalization function is part of the experiment definition. It must deliberately exclude values that naturally vary between otherwise equivalent exchanges, such as sequence numbers or timestamps, unless those fields are the subject of the experiment.

### 6.1 Target reception and negative outcomes

Request verification and target reception answer different questions. An observer-matched request proves that the intended MPDU existed over the air. It does not prove that the target decoded it.

For an eligible unicast request, AirMirror may use a recipient MAC-layer ACK as reception evidence when ACK accounting can be scoped to one serialized trial on a qualified device. An ACK frame identifies its receiver address, not its transmitter. AirMirror may therefore attribute it to the selected target only when the request was unicast to that target, the ACK matches the trial station address and response interval, and no other frame from that source address or AirMirror transmission overlaps the interval. The strongest policy uses a unique source address per trial. Reuse requires a calibrated drain and guard interval longer than the device's retry and delayed-delivery window, because a late ACK from an earlier request is otherwise indistinguishable. The tally must be cleared or snapshotted immediately before transmission, and unsupported hardware must report ACK evidence as unavailable rather than false. A matched target response also proves reception and does not require a separate ACK.

An ACK confirms MAC-layer reception, not successful semantic parsing of the request. If a trial has neither a response nor a recipient ACK attributable to the target, its negative outcome is `target-reception-unconfirmed` and cannot contribute to a divergence. Broadcast and multicast requests are not ACKed, so silence after them cannot serve as target evidence in the MVP.

### 6.2 Timing is not an MVP oracle

Userland USB bulk transfers, host scheduling, firmware queues, channel contention, and RF retries make microsecond-level response comparisons unrealistic with the current generic interface. The initial differential experiment must not classify a target based on response latency or beacon phase.

The TSF timestamp embedded by an AP can still be useful for future passive analysis because it originates at the AP, not at host receipt time. That is a separate research direction and should not be mixed with the initial differential engine.

## 7. Selecting the first metamorphic relation

The credibility of AirMirror depends on selecting transformations that are genuinely semantics-preserving. "Interesting-looking" mutations are not sufficient.

Every relation should document:

1. the relevant standards requirement;
2. the preconditions under which the transformation is equivalent;
3. fields allowed to differ in the response;
4. fields expected to remain invariant;
5. initial-state and cleanup requirements;
6. whether the relation applies to all targets or only advertised capability profiles;
7. known vendor extensions that may invalidate the assumption.

### 7.1 Candidate relation

A possible first candidate is inserting an unknown but well-formed optional vendor-specific element into an otherwise identical management request. The expectation would be that a receiver ignores an element it does not implement and preserves the surrounding protocol outcome.

This is only a candidate. It must be checked against the exact IEEE 802.11 element-ordering and unknown-element processing requirements before implementation. A private element cannot be assumed harmless merely because it looks optional.

### 7.2 What should not be the first relation

PMF-capable versus PMF-required is not a clean first metamorphic relation. Those flags intentionally change security semantics, so different results can be standards-compliant. PMF remains an excellent later conformance pack, but it needs an explicit specification oracle rather than only differential comparison.

Likewise, arbitrary IE reordering is unsuitable unless the standard explicitly permits the selected elements to be reordered. Many management-frame elements have normative ordering rules.

## 8. Transcript and evidence model

AirMirror should produce an evidence bundle, not a simple success banner. The bundle should make an experiment independently reviewable and replayable.

Before the first active trial, it should freeze an experiment manifest containing the relation version, normalization rules, identity and washout policy, observer policies, comparison cells, minimum valid counts, invalidity limits, dominant-fingerprint threshold, stopping rule, and required negative controls. The transcript should identify that exact manifest.

At minimum, each run should record:

| Area | Recorded data |
| --- | --- |
| Target | BSSID, SSID if known, channel, advertised capabilities |
| Hardware | all participating chipsets, VID:PIDs, drivers, hardware profile, assignments, and role block |
| Experiment | relation and analysis-plan identifiers and versions, parameters, repetition and stopping policy |
| Intent | canonical frame and transformed frame bytes |
| OTA verification | request and response policies, observed bytes, receiving card, match result, and evidence label for every observer |
| Target reception | matched response, attributable recipient ACK, unavailable, or unconfirmed, including the correlation window and attribution basis |
| Response | raw response and normalized fingerprint |
| State | source MAC policy, trial order, block, reset or washout action, and verification result |
| Controls | control-versus-control outcomes and applicable baseline thresholds |
| Validity | attempts and invalidations stratified by variant, transmitter, observer, order, and block |
| Outcome | valid, invalid, control result, mutation result, or inconclusive |

The human-readable summary should avoid claiming a vulnerability automatically. A suitable result is:

> Stable differential behavior observed: control association received status X in 12/12 valid trials, while mutation received status Y in 11/12 valid trials. Required control-versus-control blocks remained equivalent, invalidity stayed within the declared limits, and the result followed the transformation after transmitter crossover and CM/MC ordering. Every counted no-response outcome had target-reception evidence. Standards interpretation is still required.

PCAP or PCAPNG export is desirable, but the internal transcript must not depend on a capture format to preserve experiment metadata.

For multi-observer profiles, the transcript should retain an observation matrix rather than only a merged capture:

```text
                 Observer C   Observer D   Observer E
Control request     seen         seen         missed
Control response    seen         seen          seen
Mutation request    seen         missed        seen
Mutation response   seen         seen          seen
```

The matrix must include the normalized bytes, FCS status when available, RSSI, channel, and trial-matching decision behind every `seen` value. Packet loss is often correlated, so the number of observers is not itself a statistical confidence score.

## 9. Safety and authorization model

AirMirror is intended for devices and networks owned by the operator or covered by explicit authorization.

The application should enforce a real separation between passive operation and active laboratory trials.

### 9.1 Field-safe mode

Field-safe behavior may build passive topology and capability information, but it must not initiate an AirMirror exchange. The restriction should be enforced in the campaign entry point, not only stated in documentation.

### 9.2 Authorized-lab mode

Active differential tests require an explicit lab-mode activation. The UI should make this state continuously visible. Initial relations should also impose:

- a single explicitly selected BSSID;
- conservative rate limits;
- bounded repetition counts;
- no broadcast mutation campaigns;
- no deauthentication of legitimate clients;
- no credential collection;
- no end-to-end traffic generation;
- no deliberate crash or reboot probing.

The first implementation should fail closed if authorization mode, observer availability, or target selection becomes ambiguous.

## 10. Relationship to passive identity and decloaking work

Recent decloaking work already provides much of the data-acquisition substrate for a future passive Air Identity Graph:

- Multiple-BSSID SSID hints;
- Reduced Neighbor Report hints;
- OWE transition relationships;
- FILS and short-SSID information;
- collision-aware short-SSID candidate resolution;
- evidence-strength handling;
- sibling BSSID and per-card RSSI information.

That should not be reimplemented inside AirMirror. A later passive topology model could promote these hints into explicit nodes and edges with source, confidence, and conflicting-evidence handling. It could then identify likely physical AP, radio, BSS, and security-profile relationships.

However, this graph is not required to prove the differential-testing concept. Combining it with the first active PR would increase review scope and obscure the central experiment. It should remain a separate track.

## 11. Proposed integration boundaries

Names are provisional, but the responsibility split should remain narrow.

### Shared coordination infrastructure

- one array-level coordinator lock that atomically acquires logical ownership of the complete transmitter and observer set;
- staged physical configuration only after every ownership claim succeeds;
- release of every logical claim after success, cancellation, disconnect, or exception;
- best-effort restoration of every reachable device and an explicit lost-device state for disconnected members;
- no partially configured device set visible to a running campaign.

### Shared receive infrastructure

- campaign-scoped pre-suppression observer subscriptions;
- physical receiving-card identity;
- per-observer copies before global deduplication;
- no changes to normal sink semantics when unused.

### AirMirror experiment layer

- experiment definition and preconditions;
- control and transformation builders;
- trial scheduling and randomized ordering;
- station identity policy;
- reset and washout verification;
- response matching and normalization;
- negative-control, crossover, CM/MC, and stability evaluation;
- separate request-validity, response-capture, target-reception, and evidence-label policies;
- stratified invalidity accounting and preregistered stopping rules;
- transcript creation.

### UI layer

- explicit authorized-lab activation;
- target and supported-adapter validation;
- progress and invalid-trial counts;
- evidence summary and export.

The experiment layer should depend on typed wifit3 packets and public `WlanArray`/lease behavior. It should not reach into chipset registers or contain per-driver workarounds.

## 12. Proposed delivery sequence

### Phase 0: bench feasibility

Before a feature PR, confirm with a small local spike that:

- logical ownership of the complete device set can be acquired atomically, with fault-injected tests for every staged setup and cleanup step;
- one adapter can independently capture the other adapter's injected management frame;
- the captured MPDU can be matched to the intended bytes after documented normalization;
- an eligible unicast request can be tied to a per-trial recipient ACK attributable to the target, including tests for delayed ACK delivery, unique and reused source identities, and the declared guard interval;
- two-radio transmitter/observer role exchange works on at least one selected pair;
- three-radio control/mutation crossover works with a dedicated observer on one selected adapter set;
- control-versus-control blocks through both transmitter paths establish an acceptably low false-divergence baseline;
- invalidation rates can be measured by variant, transmitter, observer, order, and block without silently dropping attempts;
- CM/MC ordering and the declared reset or washout procedure prevent an apparent result from following trial order;
- optional observer rotation exposes no unexplained receive-only differences in that set;
- own-frame observation can be exposed without contaminating `WlanSink`.

Four- and five-radio profiles should be admitted only after separate calibration shows a diagnostic benefit over the three-radio design under controlled placement and USB topology. If the core method cannot be made reliable, the active proposal should stop rather than grow compensating heuristics.

### PR 1: observer and transcript infrastructure

- add atomic logical multi-interface reservation, staged configuration, and deterministic claim release;
- add the scoped raw observation hook;
- add hardware-free tests proving own transmissions remain absent from normal sink state;
- add transcript types with no active mutation catalog;
- demonstrate that behavior is unchanged when no observer is registered and that partial setup cannot escape cleanup.

### PR 2: one differential experiment

- implement one standards-justified metamorphic relation;
- support two-radio verified mode as the minimum configuration;
- qualify one three-radio balanced configuration as the preferred evidence path;
- use discrete response fingerprints;
- freeze a versioned analysis plan and run control-versus-control baselines;
- randomize CM/MC order, swap transmitter assignments, verify washout, and record every role block;
- require target-reception evidence for every counted no-response outcome;
- classify only stable, observer-confirmed differences with acceptable stratified invalidity;
- expose it only through authorized-lab mode.

### Later phases

- additional metamorphic relations;
- four-radio corroboration with two heterogeneous observers;
- five-radio observer ensembles and full role-rotation confirmation;
- cross-channel sentinels for known affiliated BSSIDs;
- explicit PMF and RSN conformance packs;
- credential-free patch-status tests where legally and ethically appropriate;
- passive Air Identity Graph;
- automatic reduction of multi-frame reproducers;
- MLO and cross-link consistency testing after supported 6 GHz operation exists;
- timing-based experiments only on hardware that exposes suitable timestamps.

## 13. Acceptance criteria for the MVP

The MVP is successful if it can demonstrate the measurement method, even if it finds no vulnerability.

Required properties:

- normal scanning and sink state are unchanged outside an active trial;
- logical ownership of transmitter and observer sets is acquired atomically and released after every exit path, while reachable devices are restored and disconnected devices are marked lost;
- every counted trial has independent OTA request verification;
- every counted no-response outcome has target-reception evidence satisfying the frozen response-or-ACK attribution policy;
- multi-observer profiles preserve per-card copies and evidence labels;
- request validity, response capture, and evidence grading remain separate declared policies;
- control/mutation ordering is balanced as CM/MC and recorded;
- control-versus-control baselines pass through every transmitter path used for inference;
- the declared reset or washout condition is verified between dependent trials;
- the result survives the crossover required by the selected two- or three-radio profile;
- three-radio reports distinguish transmitter crossover from optional observer rotation;
- invalid attempts and observer misses are reported per comparison cell, and imbalanced loss produces an inconclusive result;
- unstable outcomes are reported as inconclusive;
- all normalization rules are explicit and tested;
- unit tests require no hardware;
- hardware tests use only an owned lab target;
- the report contains enough information for a human to reproduce the exchange.

Stop or redesign conditions:

- observer capture is too unreliable for the selected adapter pair or set;
- target reception cannot be confirmed for a response-silence comparison;
- control-versus-control produces a stable divergence or exceeds its invalidity limit;
- invalidity is materially imbalanced between variants or experiment cells;
- the result follows the transmitting adapter rather than the mutation;
- the result follows identity allocation, CM/MC order, or accumulated target state;
- the selected relation cannot be defended as semantics-preserving;
- target state cannot be isolated without disruptive cleanup;
- logical ownership cannot be released reliably or partial physical setup cannot be detected and handled;
- the shared RX hook changes scanner or campaign behavior.

## 14. Non-goals

The initial AirMirror scope is not:

- a replacement for a coverage-guided firmware fuzzer;
- a random malformed-frame generator;
- a complete IEEE 802.11 conformance suite;
- a remote crash scanner;
- a client-tracking system;
- a timing side-channel framework;
- an MLO implementation;
- a claim that a divergence is automatically a security vulnerability;
- a promise of identical behavior across every supported adapter.

Keeping these non-goals explicit is necessary for a reviewable contribution.

## 15. Risks and mitigations

### RF and USB noise

**Risk:** packet loss and scheduling jitter create false differences.  
**Mitigation:** discrete outcomes, repeated paired trials, CM/MC ordering, target-reception evidence for silence, invalid-trial accounting, and no timing oracle.

### Multi-adapter resource contention

**Risk:** four or five concurrent bulk-RX streams can overload a shared USB controller or hub, while closely placed transmitters can desensitize observers through RF coupling.

**Mitigation:** record USB bus topology, power arrangement, adapter placement, per-card loss rates, and RSSI; test coordinated setup and cleanup faults; qualify larger sets independently instead of assuming that more radios always improve evidence.

### Observer-dependent selection

**Risk:** one variant is more likely to be discarded because an observer or ACK path misses it, leaving a biased set of apparently valid trials.

**Mitigation:** freeze request and response policies before the run, retain every attempt, report invalidity by comparison cell, cap total and differential invalidity, and classify imbalance as inconclusive.

### Correlated observer evidence

**Risk:** multiple observers share RF placement, antennas, USB infrastructure, or parser behavior, making corroboration labels look stronger than the underlying evidence.

**Mitigation:** preserve per-card observations, describe hardware and placement, treat heterogeneous corroboration as descriptive, and calibrate larger profiles against the three-radio baseline.

### Transmitter-specific behavior

**Risk:** the result is caused by one adapter.  
**Mitigation:** independent OTA verification, transmitter crossover, and optional observer rotation.

### Target state leakage

**Risk:** one trial changes the starting state of the next.  
**Mitigation:** explicit identity policy, CM/MC blocks, verifiable reset or washout, bounded identity use, fresh matched identities where needed, and negative controls through both transmitter paths.

### Incorrect metamorphic assumption

**Risk:** control and mutation are not actually equivalent.  
**Mitigation:** clause-level justification, documented preconditions, and one relation at a time.

### Excessive project scope

**Risk:** the proposal turns into a general fuzzing platform.  
**Mitigation:** separate observer infrastructure, one-relation MVP, and explicit deferred work.

### Safety regression

**Risk:** active behavior becomes reachable during ordinary scanning.  
**Mitigation:** a hard authorized-lab gate, selected-target scope, and fail-closed campaign entry.

## 16. Related work and differentiation

AirMirror builds on, rather than dismisses, existing wireless testing research.

- [wifuzzit](https://github.com/0xd012/wifuzzit) demonstrated model-based stateful fuzzing of access points and stations.
- [WPAxFuzz](https://github.com/efchatz/WPAxFuzz) supports management, control, data, and SAE fuzzing.
- [Owfuzz](https://github.com/alipay/Owfuzz) provides interactive over-the-air testing and broad frame coverage using openwifi or injection-capable adapters.
- [Fragile Frames](https://papers.mathyvanhoef.com/wisec2025.pdf) demonstrated practical, credential-free, minimally invasive tests for FragAttacks patch adoption and showed that carefully designed OTA oracles can support real-world measurement.
- [StateFi](https://wisec26.events.cispa.de/program-accepted-papers/) showed that management-frame state transitions expose stable behavioral structure.
- [SSID Confusion](https://papers.mathyvanhoef.com/wisec2024.pdf) demonstrated that network identity and authenticated protocol state can diverge in modern Wi-Fi deployments.
- [CHAOS](https://www.ndss-symposium.org/ndss-paper/chaos-exploiting-station-time-synchronization-in-802-11-networks/) showed that management-frame ordering and TSF behavior expose additional structure, although AirMirror does not propose a timing oracle for its MVP.
- [Timestamps Unchained](https://wisec26.events.cispa.de/program-accepted-papers/) illustrates why precise timing experiments require chipset-specific timestamp exposure rather than generic host receive timing.

The proposed distinction is the combination of:

1. a semantics-preserving differential oracle;
2. independent over-the-air verification of every counted request;
3. explicit target-reception evidence before silence is treated as an outcome;
4. control-versus-control baselines and repeated CM/MC trial blocks;
5. balanced transmitter crossover to expose TX bias, with optional observer rotation to expose RX bias;
6. an extensible per-observer evidence matrix and stratified invalidity accounting for larger radio sets;
7. evidence output instead of an automatic vulnerability claim;
8. implementation on directly controlled commodity USB Wi-Fi hardware.

This document does not claim exhaustive prior-art coverage or guaranteed novelty. The combination is a research hypothesis that should first be validated as an engineering method.

## 17. Questions for the maintainer

Before implementation, the proposal needs guidance on six points:

1. Does a lab-only differential state-auditing campaign fit wifit3's intended Layer 2 scope?
2. Should the pre-suppression observer hook and transcript model be proposed as an independent infrastructure PR?
3. Is a two-adapter minimum with a three-adapter recommended evidence profile acceptable for the first experiment?
4. Where should the hard authorized-lab policy live so campaigns cannot accidentally bypass it?
5. Should the observer API model a collection from the first PR even though four- and five-radio profiles remain future work?
6. Should atomic logical multi-interface ownership extend the existing lease API or remain an AirMirror-owned coordinator until another campaign needs it?

If the direction is in scope, the next step should be the Phase 0 bench feasibility check and a small observer-hook design, not a broad mutation catalog.

## Conclusion

AirMirror is intended to answer one defensible question at a time: did a target produce different, repeatable observable behavior after a transformation that should not have changed the exchange's meaning?

Wifit3 can make that question unusually credible because it can run a two-radio verified profile at minimum and a three-radio balanced profile for stronger evidence. In the preferred setup, two directly controlled USB adapters maintain separate station contexts while a third verifies both exchanges over the air; control and mutation assignments are then crossed between the transmitters. A result still requires a clean control-versus-control baseline, CM/MC blocks, verified state isolation, and target-reception evidence before silence is counted. Four- and five-radio profiles can later add diagnostic observer views, full role rotation, or cross-channel sentinels without changing the core differential oracle, but their benefit must be calibrated rather than assumed. The project does not need to become a general fuzzer to test the idea.

Starting with one relation, discrete outcomes, coordinated multi-device ownership, strict invalid-trial accounting, and an explicit lab boundary keeps the proposal aligned with wifit3's auditing focus. If the method proves reliable, broader conformance packs, passive topology, MLO consistency checks, and automatic testcase reduction can be added later as separate, evidence-driven steps.
