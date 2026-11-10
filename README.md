# Distributed-multisensor-ISAC

Investigation and simulation of distributed multi-sensor integrated sensing and
communication (MS-ISAC).

## Scope

This repository currently proposes a generic architecture, not an implemented
simulator or a validated system. The goal is to serve communication users while
multiple spatially separated sensors cooperatively observe and track targets.
The same radio infrastructure and scheduled time-frequency resources support
both functions; targets need not be connected communication users.

## Proposed architecture

```text
                  Communication users
                    ^           |
                    | data / channel feedback
                    v           v
       +---------------------------------------+
       | Distributed ISAC nodes (1 ... N)       |
       | Radio + antenna array                 |
       | Communication processing              |
       | Local sensing + uncertainty estimation|
       +---------------------------------------+
           |          ^                   |
           |          | echoes            | observations,
           | illumination                 | channel quality,
           v          |                   |
                 Targets / environment    | health reports
                                          v
       +---------------------------------------+
       | Edge fusion and coordination          |
       | Time/geometry alignment + association |
       | Multi-target tracking + uncertainty   |
       | Resource and node-role scheduler      |
       +---------------------------------------+
                  |                 |
                  | tracks          | schedules, beams,
                  v                 | power, node roles
              Applications          +----> ISAC nodes
```

The diagram represents logical roles: fusion and scheduling may run on one edge
server, be distributed across nodes, or use a hierarchy of local fusion clusters.
The transport between nodes and the edge is a capacity- and latency-limited
fronthaul/backhaul, which may be wired or wireless.

| Component | Responsibility |
| --- | --- |
| ISAC node | Transmit scheduled waveforms, receive communication signals and target echoes, estimate local measurements, and report quality and health. |
| Communication user | Exchange payload data and provide channel feedback where supported. |
| Fusion service | Align measurements to a common time and coordinate frame, associate observations with targets, and produce tracks with uncertainty. |
| Coordinator | Select participating nodes, transmit/receive roles, beams, power, and time-frequency allocations under communication and sensing constraints. |
| Application | Consume tracks and communication services; specify sensing region, update rate, accuracy, and communication quality-of-service requirements. |

### Operating assumptions and interfaces

- Each node has a known position and orientation, calibrated antenna parameters,
  and timestamps with characterized clock uncertainty. Nodes may have different
  sensing, bandwidth, antenna, and processing capabilities.
- Monostatic operation uses a node's own transmitted waveform and echoes.
  Bistatic/multistatic operation uses other nodes' illumination and requires
  knowledge of the transmitted waveform, transmitter identity and geometry, and
  timing/frequency alignment. Simultaneous transmission and reception requires
  sufficient isolation/self-interference cancellation; otherwise schedule
  separate transmit and receive intervals.
- A local observation report carries node/transmitter IDs, measurement time,
  coordinate frame, available range/delay, Doppler and angle measurements,
  uncertainty, and quality indicators. A bistatic delay measures a
  transmitter-target-receiver path, not a monostatic range.
- A control message carries a schedule version, validity interval, participating
  nodes, waveform/reference configuration, resource allocations, and power/beam
  settings. Late or invalid schedules must not authorize conflicting transmissions.
- Measurement-level fusion is the baseline. Raw-sample fusion is optional and
  requires substantially more transport capacity and tighter synchronization.
  Coherent combining additionally requires phase calibration; it is not assumed
  for ordinary measurement-level fusion.
- Authenticate control/reporting endpoints and protect transport. Limit access
  to target tracks and raw sensing data and define retention before deployment.

## How it works

Each coordination interval follows a closed sensing-communication loop:

1. **Discover and synchronize.** Register node capabilities, geometry, clock
   quality, and transport availability. Exclude measurements that cannot be
   aligned reliably, or account for their additional uncertainty.
2. **Set objectives.** Collect user traffic/channel estimates and application
   sensing requirements, such as a region of interest and track update deadline.
3. **Allocate resources.** Select useful sensor geometry and node roles; allocate
   time, frequency, beams, and power. Protect minimum communication service and
   sensing updates where feasible, subject to interference, hardware, spectrum,
   and transport limits. Explicitly report infeasible requirements.
4. **Transmit and observe.** Deliver communication payloads while using known
   waveform components for sensing. Coordinate illumination to reduce mutual
   interference; receive echoes only in supported radio operating modes.
5. **Process locally.** Estimate delay/range, Doppler, and angle where observable,
   reject low-quality detections, and attach timestamps and uncertainty. Local
   tracking can reduce reporting bandwidth when necessary.
6. **Fuse and track.** Transform observations into the common frame, account for
   monostatic/bistatic geometry, associate detections, and update target position
   and velocity estimates. Handle missed detections and clutter; avoid treating
   correlated observations or shared-prior local tracks as independent evidence.
7. **Adapt.** Publish tracks with age and uncertainty, measure communication and
   sensing performance, and adjust the next schedule. Predict tracks to the
   decision time rather than treating delayed reports as current observations.

The scheduler can use a weighted communication/sensing objective, or prioritize
one service subject to minimum constraints on the other. Weights, requirements,
and infeasibility handling are explicit experiment parameters, not universal
constants. Better sensing may improve beam selection, but uncertain or stale
tracks must not replace communication channel feedback without validation.

## Expected behavior

These are hypotheses and design policies to evaluate, not measured results.

| Condition | Intended response and trade-off |
| --- | --- |
| More well-placed sensors | Complementary views can improve coverage, localization, and occlusion resilience, at the cost of coordination, interference, and reporting overhead. Extra nodes do not guarantee improvement. |
| Increasing communication load | Allocate more resources to payloads while preserving feasible sensing deadlines; sensing accuracy or update rate may decrease. Declare degraded service when both requirements cannot be met. |
| Maneuvering or newly detected target | Increase observation rate or recruit nodes with useful geometry, consuming resources that could otherwise serve traffic. |
| Occlusion or weak echoes | Use alternative viewpoints and predict tracks through short gaps; uncertainty grows without fresh observations and stale tracks eventually expire. |
| Clock drift or poor calibration | Increase measurement uncertainty or reject affected reports; disable coherent cooperation until synchronization/calibration recovers. |
| Congested or delayed transport | Prefer compact measurements/local tracks, enforce report-age limits, and reduce cooperation rather than fusing stale data as fresh. |
| Node or coordinator failure | Remove unavailable nodes and reallocate if coordination remains available. On control loss, use only a pre-authorized, unexpired fallback schedule; otherwise stop cooperative transmissions and mark outputs degraded. |
| Clutter or conflicting reports | Gate and associate measurements using quality and uncertainty, retain ambiguity where necessary, and avoid overconfident fused tracks. |

## Evaluation plan

A future simulator should make the following parameters configurable:

- Node count, positions, antenna/radio capabilities, carrier frequency, bandwidth,
  waveform, transmit-power limits, and scheduling interval.
- User locations and traffic, target trajectories, propagation/echo models,
  clutter, noise, occlusion, and mutual/self-interference.
- Clock/calibration errors, report quantization, transport capacity and delay,
  packet loss, and node/control outages.
- Fusion method, uncertainty model, resource policy, service requirements, and
  random seeds.

Start with a single-node baseline, then compare uncoordinated multi-node operation
and coordinated measurement-level fusion under identical geometry, traffic,
channel realizations, and total time-frequency and transmit-energy budgets.
Account separately for coordination/reporting overhead. Include ideal
synchronization as a reference, then introduce realistic impairments.

Measure communication throughput, latency, outage and fairness; sensing detection
probability at a specified false-alarm rate, localization/velocity error, track
continuity and update age; and system transport load, energy consumption, and
recovery time. Report errors together with missed/false targets, rather than only
errors on successful detections. Sweep traffic, node geometry/count, target motion,
synchronization error, transport delay, and failures across repeated seeded runs.

Basic checks should include single-node operation, no targets, no user traffic,
complete occlusion, delayed/out-of-order reports, and loss of a node or coordinator.
Verify allocation limits, exclusion of stale reports, consistent coordinate/time
handling, and explicit degraded/infeasible status. There is currently no executable
code, build process, or automated test suite in this repository.
