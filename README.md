# AirMirror

## A proposal for differential IEEE 802.11 state auditing in wifit3

**Status:** design proposal; no implementation has started  
**Intended audience:** wifit3 maintainers and contributors  
**Initial scope:** authorized laboratory testing with two adapters minimum and three adapters recommended

## Executive summary

AirMirror is a proposed wifit3 campaign for detecting implementation inconsistencies in access points without first having to model the complete IEEE 802.11 state machine.

The central idea is paired differential testing. AirMirror sends a normal control exchange and a second exchange containing exactly one standards-backed, semantics-preserving variation. It then asks a deliberately narrow question:

> Did two exchanges that should mean the same thing produce different, repeatable protocol states?

This changes the oracle problem. A conventional fuzzer must know whether an arbitrary response is correct. AirMirror instead uses the target's response to the control exchange as a local baseline. It reports only stable differences between the control and mutation after repeating the experiment, randomizing trial order, and balancing variant assignments across the participating adapters.

An independent adapter verifies what was actually transmitted over the air. A trial is discarded if the observer did not receive the intended frame or if the observed frame differs from the planned frame. This separates target behavior from transmitter firmware, descriptor, or USB-path behavior.

Two adapters are sufficient for a minimum verified mode: one transmits both variants while the other observes, and their roles are exchanged in the next block. The preferred research setup uses three adapters: two maintain independent station contexts used for control and mutation while a third remains a dedicated observer. The two transmitters exchange variant assignments, and the observer role can later rotate across all three devices. Frames are tightly interleaved, not transmitted simultaneously.

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

`WlanArray` already manages several `WlanInterface` instances, assigns channels, merges their received traffic, and tracks signal information per card. AirMirror can reserve two members as independent station contexts and a third as the observer without introducing a second operating-system capture stack. A reduced two-adapter mode can execute both variants through one transmitter and exchange roles between blocks.

### 2.3 Scoped device state

The existing lease mechanism scopes channel selection, fake MAC configuration, BSSID matching, and ACK accounting. That is close to the lifecycle needed by a controlled trial and reduces the risk that a failed experiment leaves the selected adapter in an unexpected state.

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

AirMirror should name the two profiles explicitly in transcripts and reports.

#### Two-radio verified

- one transmitter executes both variants during a block;
- one independent observer verifies both variants;
- transmitter and observer exchange roles in the next block;
- the same physical TX path is used for control and mutation within a block.

This profile has the lowest hardware barrier and is sufficient to prove the observer and differential-oracle method.

#### Three-radio balanced

- two transmitters maintain separate station identities and protocol state;
- the third adapter observes both exchanges;
- control and mutation assignment is swapped between transmitters;
- the observer can be rotated in later blocks to expose receiver-specific blind spots;
- all transmissions remain sequential and closely interleaved.

This is the preferred profile for strong evidence because it controls transmitter bias without sacrificing an independent observer or repeatedly rearming one station context.

### 3.2 Terminology

- **Control:** the canonical exchange used as the local behavioral baseline.
- **Mutation:** the same exchange after applying one justified metamorphic transformation.
- **Metamorphic relation:** the reason control and mutation should have the same protocol meaning.
- **Observer:** an adapter that captures the transmitted request and target response but does not participate in the exchange.
- **Verified trial:** a trial in which the observer captured a request matching the expected normalized frame.
- **Response fingerprint:** the discrete, normalized protocol outcome used for comparison.
- **Transmitter crossover:** swapping control and mutation assignments between the two transmitting adapters.
- **Observer rotation:** moving the receive-only role between adapters in separate blocks to expose RX-specific blind spots.
- **Divergence:** a stable difference between control and mutation response distributions that survives the crossover required by the selected hardware profile.

## 4. Experimental design

The experimental design is more important than the number of supported mutations. A result is useful only if ordinary RF loss, stale station state, and adapter-specific behavior have been excluded as plausible explanations.

### 4.1 Trial lifecycle

A single trial should follow a strict lifecycle:

1. Select and lease the transmitter set and observer on the target channel.
2. Establish a known starting condition.
3. Assign a fresh or deliberately reused locally administered station MAC according to the experiment definition.
4. Build the control or mutation exchange from the same semantic input.
5. Record the exact intended bytes and relevant transmitter configuration.
6. Transmit the exchange.
7. Capture the request independently on the observer.
8. Reject the trial if OTA verification fails.
9. Capture the target response within a bounded window.
10. Normalize the response into discrete comparison fields.
11. Clean up or isolate the state before the next trial.
12. Store the complete trial transcript.

Each pair is repeated with randomized control/mutation order. In two-radio mode, transmitter and observer exchange roles for the next block. In three-radio mode, control and mutation assignments are swapped between the two transmitters while the independent observer remains fixed for the block. A stronger campaign can then rotate the observer and repeat the balanced crossover.

### 4.2 Why assignments and roles must be balanced

If adapter A always sends the control and adapter B always sends the mutation, a target difference can be caused by the adapters rather than the frame transformation. Different antennas, PHY defaults, retry behavior, transmit power, or firmware can all become hidden independent variables.

Two-radio mode avoids this by sending both variants through one transmitter during a block and then exchanging transmitter and observer roles. Three-radio mode uses the stronger balanced design: A sends control while B sends mutation, then A sends mutation while B sends control, with C independently observing both pairs. A divergence is not promoted unless it follows the transformation rather than the transmitting adapter.

Observer rotation is a second, optional control. If a difference disappears when a different chipset becomes the observer, the result may be an RX visibility issue rather than target behavior and must be reported as such.

### 4.3 Fresh identities and state isolation

Association-related tests create state in the target. Reusing one MAC can make a mutation inherit state from its control or vice versa. Always using new MACs can instead fill a target's station table and create a different artifact. Three-radio mode helps by keeping two station contexts armed independently, but it does not remove the need for explicit state isolation between paired trials.

Each experiment must therefore declare its identity policy:

- new matched MAC pair per repetition;
- one MAC with an explicit reset between variants;
- controlled reuse when the metamorphic relation concerns retransmission;
- bounded identity count with cleanup and cooldown.

The first relation should choose the simplest policy and avoid tests that require manipulating legitimate clients.

### 4.4 Repetition and stability

A single differing response is not evidence. RF loss, channel contention, retries, target load, and USB scheduling make isolated differences expected.

The first implementation should use a conservative reproducibility gate instead of pretending that one statistical test fits every experiment:

- compare only valid, observer-confirmed trials;
- require a minimum number of valid control and mutation trials;
- require the dominant response fingerprint of each variant to be stable;
- require the difference to recur after the crossover defined by the active hardware profile;
- report counts and raw outcomes, not only a binary vulnerability label;
- classify unstable results as inconclusive.

Exact thresholds should be versioned with the experiment definition and validated on known-good and deliberately faulty test targets. They should not be hidden UI constants.

## 5. The observer

The observer is the primary distinction between AirMirror and a conventional send-and-wait campaign.

### 5.1 What the observer proves

The observer should verify at least:

- transmitter/source address;
- receiver and BSSID addresses;
- frame type and subtype;
- sequence fields when they are part of the relation;
- frame body and Information Elements after excluding explicitly permitted hardware changes;
- that the frame was observed on the expected channel;
- that the target response corresponds to the current trial identity.

It does not prove that the target successfully decoded the frame. It proves that the planned test reached the air in an independently observable form. The distinction must remain explicit in reports.

### 5.2 Required WlanArray hook

Wifit3 currently suppresses frames transmitted by one of its own registered MAC addresses before they enter normal sink state. That is correct for scanning and attack campaigns: a second card hearing our own injected frame must not create a fake client or access point.

AirMirror needs a narrowly scoped observation path before this suppression. The proposed hook should:

- be campaign-scoped and removable;
- identify which physical member received the copy;
- run before own-transmission suppression;
- avoid mutating `WlanSink`;
- avoid adding beacons, clients, handshakes, or decloak evidence;
- allow a strict filter so unrelated traffic is not retained;
- preserve the normal scanner behavior when no observer is registered.

This infrastructure is valuable enough to be reviewed as a separate first PR. It is also the main point where AirMirror touches shared receive architecture.

### 5.3 Invalid trials

A trial must be marked invalid, rather than as a target failure, when:

- the observer does not capture the request;
- the captured request fails normalized byte comparison;
- the observer changes channel or disconnects;
- the transmitter reports an injection error;
- an unrelated response cannot be distinguished from the trial response;
- cleanup or initial-state establishment fails.

Invalid trials remain in the transcript for diagnosis but do not contribute to the control/mutation result.

## 6. Response fingerprints

PR1 should compare discrete protocol outcomes only. Candidate fields include:

- no response versus response;
- authentication status code;
- association status code;
- deauthentication or disassociation reason code;
- response subtype;
- normalized set of response IEs;
- advertised RSN and PMF capabilities;
- ACK observed versus not observed, where the hardware can expose it reliably.

The normalization function is part of the experiment definition. It must deliberately exclude values that naturally vary between otherwise equivalent exchanges, such as sequence numbers or timestamps, unless those fields are the subject of the experiment.

### 6.1 Timing is not an MVP oracle

Userland USB bulk transfers, host scheduling, firmware queues, channel contention, and RF retries make microsecond-level response comparisons unrealistic with the current generic interface. PR1 must not classify a target based on response latency or beacon phase.

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

At minimum, each run should record:

| Area | Recorded data |
| --- | --- |
| Target | BSSID, SSID if known, channel, advertised capabilities |
| Hardware | all participating chipsets, VID:PIDs, drivers, hardware profile, assignments, and role block |
| Experiment | relation identifier and version, parameters, repetition policy |
| Intent | canonical frame and transformed frame bytes |
| OTA verification | observed bytes, receiving card, match result |
| Response | raw response and normalized fingerprint |
| State | source MAC policy, trial order, cleanup result |
| Outcome | valid, invalid, control result, mutation result, or inconclusive |

The human-readable summary should avoid claiming a vulnerability automatically. A suitable result is:

> Stable differential behavior observed: control association received status X in 12/12 verified trials, while mutation received status Y in 11/12 verified trials. The result followed the transformation after control and mutation assignments were exchanged between transmitters. It also reproduced after observer rotation. Standards interpretation is still required.

PCAP or PCAPNG export is desirable, but the internal transcript must not depend on a capture format to preserve experiment metadata.

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

### Shared receive infrastructure

- campaign-scoped pre-suppression observer subscription;
- physical receiving-card identity;
- no changes to normal sink semantics when unused.

### AirMirror experiment layer

- experiment definition and preconditions;
- control and transformation builders;
- trial scheduling and randomized ordering;
- station identity policy;
- response matching and normalization;
- crossover and stability evaluation;
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

- one adapter can independently capture the other adapter's injected management frame;
- the captured MPDU can be matched to the intended bytes after documented normalization;
- two-radio transmitter/observer role exchange works on at least one selected pair;
- three-radio control/mutation crossover works with a dedicated observer on one selected adapter set;
- optional observer rotation exposes no unexplained receive-only differences in that set;
- own-frame observation can be exposed without contaminating `WlanSink`.

If this cannot be made reliable, the active proposal should stop rather than grow compensating heuristics.

### PR 1: observer and transcript infrastructure

- add the scoped raw observation hook;
- add hardware-free tests proving own transmissions remain absent from normal sink state;
- add transcript types with no active mutation catalog;
- demonstrate that behavior is unchanged when no observer is registered.

### PR 2: one differential experiment

- implement one standards-justified metamorphic relation;
- support two-radio verified mode as the minimum configuration;
- qualify one three-radio balanced configuration as the preferred evidence path;
- use discrete response fingerprints;
- randomize order, swap transmitter assignments, and record every role block;
- classify only stable, observer-confirmed differences;
- expose it only through authorized-lab mode.

### Later phases

- additional metamorphic relations;
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
- every counted trial has independent OTA request verification;
- control/mutation ordering is randomized and recorded;
- the result survives the crossover required by the selected two- or three-radio profile;
- three-radio reports distinguish transmitter crossover from optional observer rotation;
- unstable outcomes are reported as inconclusive;
- all normalization rules are explicit and tested;
- unit tests require no hardware;
- hardware tests use only an owned lab target;
- the report contains enough information for a human to reproduce the exchange.

Stop or redesign conditions:

- observer capture is too unreliable for the selected adapter pair or set;
- the result follows the transmitting adapter rather than the mutation;
- the selected relation cannot be defended as semantics-preserving;
- target state cannot be isolated without disruptive cleanup;
- the shared RX hook changes scanner or campaign behavior.

## 14. Non-goals

AirMirror PR1 is not:

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
**Mitigation:** discrete outcomes, repeated paired trials, randomized order, invalid-trial handling, and no timing oracle.

### Transmitter-specific behavior

**Risk:** the result is caused by one adapter.  
**Mitigation:** independent OTA verification, transmitter crossover, and optional observer rotation.

### Target state leakage

**Risk:** one trial changes the starting state of the next.  
**Mitigation:** explicit identity policy, bounded cleanup, randomized order, and fresh trial groups.

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
3. repeated paired trials with randomized ordering;
4. balanced transmitter crossover to expose TX bias, with optional observer rotation to expose RX bias;
5. evidence output instead of an automatic vulnerability claim;
6. implementation on directly controlled commodity USB Wi-Fi hardware.

This document does not claim exhaustive prior-art coverage or guaranteed novelty. The combination is a research hypothesis that should first be validated as an engineering method.

## 17. Questions for the maintainer

Before implementation, the proposal needs guidance on four points:

1. Does a lab-only differential state-auditing campaign fit wifit3's intended Layer 2 scope?
2. Should the pre-suppression observer hook and transcript model be proposed as an independent infrastructure PR?
3. Is a two-adapter minimum with a three-adapter recommended evidence profile acceptable for the first experiment?
4. Where should the hard authorized-lab policy live so campaigns cannot accidentally bypass it?

If the direction is in scope, the next step should be the Phase 0 bench feasibility check and a small observer-hook design, not a broad mutation catalog.

## Conclusion

AirMirror is intended to answer one defensible question at a time: did a target enter a different observable protocol state after a transformation that should not have changed the exchange's meaning?

Wifit3 can make that question unusually credible because it can run a two-radio verified profile at minimum and a three-radio balanced profile for stronger evidence. In the preferred setup, two directly controlled USB adapters maintain separate station contexts while a third verifies both exchanges over the air; control and mutation assignments are then crossed between the transmitters. The project does not need to become a general fuzzer to test the idea.

Starting with one relation, discrete outcomes, strict invalid-trial handling, and an explicit lab boundary keeps the proposal aligned with wifit3's auditing focus. If the method proves reliable, broader conformance packs, passive topology, MLO consistency checks, and automatic testcase reduction can be added later as separate, evidence-driven steps.
