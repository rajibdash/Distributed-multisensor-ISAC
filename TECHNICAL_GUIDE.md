# Distributed Multisensor ISAC: Technical and Simulation Guide

This technical guide consolidates the system model, estimation methods, coordination
principles, usecases and measurement example in the source paper into a simulation first
reference. It combines implementation oriented explanations with selected figures 
reproduced from the reserach paper.

## 0. Objective

This 6G-DISAC document aims to bring the integrated sensing and communication (ISAC) vision to life by 
adopting a holistic perspective with widely distributed intelligent infrastructure compatible with real 
world integration constraints and future wireless network flexibility requirements. The project addresses 
both fundamental and practical challenges, such as tracking connected UEs and **passive objects**, performing 
ISAC with many distributed base stations, efficient distributed signal processing and machine learning for 
semantic ISAC, incorporating extremely large MIMO technologies and reconfigurable intelligent surfaces, 
and intelligent sensing activation.

Here, 6G-DISAC will focus on the distributed implementation of ISAC, unlocking real sensing applications 
and providing a multi perspective view of networks in space and time for tangible communication gains. 
It will bridge theoretical approaches with operational and standards compatible distributed joint 
communication and sensing.

6G-DISAC brings together world leading actors across the value chain, including operators, network vendors, 
verticals, SMEs, research laboratories, and academic institutions. Together, the 6G-DISAC partners will define 
use cases, requirements, and design an innovative network architecture. They will also develop novel physical 
layer waveforms, distributed sensing and communications methods, optimised resource allocation methods, and 
protocols. Several demonstrations will validate the key 6G-DISAC concepts. 

This project will revolutionise various applications, from extended reality and robot human interaction to 
vehicular safety functions and **improving communication key performance indicators with sensing aided 
communications**.

## 1. Executive summary

Integrated sensing and communications (ISAC) reuses a mobile communications
network's transmitters, receivers, spectrum, OFDM waveform, synchronization,
computing, and data transport to detect and track objects(localizing of passive 
objects) that do not carry a radio tag. **Distributed multisensor ISAC (MS-ISAC)** 
treats spatially separated radio nodes as a cooperative multistatic radar network. 
It's sensing links jointly illuminate and observe a target there by delay, Doppler, 
and optionally direction measurements from several geometries are fused into a 3D 
target state.

So, RAN Advanced ISAC networks will rely on distributed infrastructure, multisensor uplink and 
downlink, or meshed sidelink access. In this way, ISAC develops into a MS-MIMO (multisensor 
multiple input multiple output) network which constitutes a **distributed MIMO** radar network. 
Multisensor link coordination and synchronization are becoming crucial today. Many multisensor 
access and signaling techniques find their communication counterpart in MU-MIMO and cooperative 
multilink communications (CoMP transmission and reception) and can be adopted from there as well.

What are central engineering ideas here briefly:

1. Coordinate multiple transmit/receive links and their time-frequency
   resources, rather than treating each sensing link as an isolated radar.
2. Use **Cooperative Passive Coherent Location (CPCL)** to reconstruct the
   communication waveform as a correlation reference and estimate target
   delay/Doppler relative to the direct transmitter-to-receiver (LOS) path.
3. Use a complex, model-based delay-Doppler estimator when the OFDM resource
   grid is sparse or nonuniform; zero-filling missing resource elements before
   an FFT can create uncontrolled artifacts.
4. Exploit spatial diversity for detection and well-conditioned geometry for
   localization. More sensors do not guarantee accurate location if their
   bistatic ellipses intersect at shallow angles (high geometric dilution of
   precision, GDoP).
5. Separate local coherent sensing from higher-level data fusion: precise
   timing/phase coherence is needed within a sensing link, while site-to-site
   fusion of already estimated states can use looser absolute synchronization.

## 2. System concepts and vocabulary

| Term | Meaning |
|---|---|
| ISAC | Integrated Sensing and Communications; here, communications-centered radar sensing using the mobile radio interface. |
| MS-ISAC | Multisensor ISAC with multiple spatially distributed illuminators and/or receivers. |
| Monostatic | Transmitter and receiver are collocated or nearly collocated. |
| Bistatic / multistatic | A target is illuminated and observed from separated Tx/Rx locations; multistatic uses multiple links. |
| eToF | Excess time of flight: target-path delay relative to the direct Tx-to-Rx path. |
| eDoppler | Target-related Doppler relative to the Tx/Rx reference path, with sign dependent on convention. |
| BTP | Bistatic target path: Tx → target → Rx, including target-related scattering. |
| CIR / FRF | Channel impulse response / frequency-response function between a Tx and Rx. |
| CPCL | Cooperative Passive Coherent Location; sensing based on a reconstructed communication waveform and differential LOS/BTP measurements. |
| SLC / IntraSC / InterSC | Sensor-level cooperation / within-site or cloud cooperation / between sites or clouds. |

This is **distributed MIMO radar**, not merely a collocated MIMO array. A
network of \(N\) nodes can form \(N\) monostatic and \(N^2-N\) bistatic
measurements if each directed Tx/Rx pairing is measured. Reciprocal or
homogeneous links may make some measurements redundant; heterogeneous
forward/backward links can provide up to \(N^2\) independent measurements.
Extra links provide diversity, redundancy, outlier checks, and additional
geometric constraints.

### Three deployment patterns

1. **Infrastructure-only:** Fixed gNB/RRU illuminators and receive-only
   “sniffer” nodes report to a centralized/cloud radio pool. It is suitable for
   persistent surveillance of roads, ports, rail yards, utilities, campuses, and
   other critical infrastructure. It avoids putting a UE in the sensing loop.
2. **UE-assisted UL/DL:** A UE may be a sensor or illuminator, with the
   infrastructure as its complementary endpoint. DL naturally supports
   broadcast illumination to multiple listening UEs; UL transmissions from
   spatially separated UEs naturally provide multiple bistatic measurements.
   UE availability, mobility, transmit power, and TDD duty cycle become part
   of the sensing problem. A receive-only sniffer can use a reduced CPCL mode;
   an authorized UE additionally enables feedback, fusion, and adaptive
   transmission.
3. **Sidelink mesh:** Cars, drones, or other mobile nodes act as both sensors
   and illuminators. The mesh can be network controlled or ad hoc, and may
   distribute sensing, computation, and communications tasks under payload,
   energy, and link-capacity constraints.

These three patterns describe **which nodes participate and their roles**; they
do not specify the coordination boundary, number of transmission/reception
points (TRPs), cell grouping, duplex mode, or OFDM numerology. Those are
complementary design dimensions and can be combined with any of the patterns
above:

| Dimension | Cases to consider | Sensing and coordination implications |
|---|---|---|
| **Coordination scope** | **Intra-gNB** or **inter-gNB** | Intra-gNB operation can coordinate multiple distributed RUs/TRPs under one gNB; it is not necessarily collocated or monostatic. Inter-gNB operation crosses gNB control domains and therefore needs explicit timing/resource coordination, measurement association, and an exchange path for measurements or estimates. The required phase coherence depends on whether processing is coherent across links or fuses per-link measurements. |
| **Tx/Rx topology** | Single-TRP or **mTRP** (multiple TRPs) | mTRP can provide multiple illuminators, receivers, or both, and can be intra-gNB or inter-gNB. It does not by itself imply coherent joint transmission: the design must state which TRPs transmit/receive, how their links are scheduled, and whether fusion is waveform-, measurement-, or track-level. |
| **Cell grouping** | Per-cell operation or a coordinated **“combined cell”** view | A combined-cell view can describe TRPs/cells treated as one sensing domain for coordination and fusion. Use it as a logical deployment description here, not as an assumption that the cells share one physical site, cell identity, gNB, or standardized combined-cell procedure. State the actual gNB/TRP and coordination boundaries. |
| **Duplexing** | TDD, FDD, or mixed FDD/TDD across participating links or cells | TDD and FDD change when and on which frequencies each endpoint can transmit or receive. A mixed deployment must account for band separation, RF isolation, UL/DL availability, scheduling, and Tx/Rx synchronization; do not assume that links can directly share the same reference or resources. |
| **Numerology** | Common or mixed numerologies | Numerology (for example, subcarrier spacing, cyclic prefix, and the resulting symbol/slot timing) is distinct from duplexing. Across numerologies, align each link to its own resource grid and account for timing and inter-numerology interference when scheduling or combining measurements. |

In particular, **FDD+TDD is a mixed-duplexing case, not by itself an
inter-numerology case**. A network may have mixed duplex modes with a common
numerology, mixed numerologies within one duplex mode, or both. For CPCL or
other coherent processing, identify the actual transmitted waveform and
reference per link; otherwise, combine calibrated per-link measurements or
tracks at the appropriate coordination layer rather than assuming network-wide
waveform/phase coherence.

```mermaid
flowchart LR
    T[Illuminating\nradio node] -->|communication\nwaveform| X((Target))
    X -->|bistatic echo| R1[Rx / sniffer 1]
    X -->|bistatic echo| R2[Rx / sniffer 2]
    T -->|direct LOS\nreference| R1
    T -->|direct LOS\nreference| R2
    R1 --> S[Local sensing\nand\ndelay-Doppler\nestimates]
    R2 --> S
    S --> F[Site / cloud\nfusion and\ntracking]
    F --> U[Authorized\napplication\nor operator view]
    F -. CSI, schedule and\nmission feedback .-> T
```

![Source paper Fig. 5: distributed MS-ISAC architecture with monostatic and bistatic links](images/fig-05-distributed-architecture.png)

*Source: Fig. 5, cropped from Thomä et al. (2026).*

### Cooperation layers

- **SLC:** Each Tx/Rx pair synchronizes, reconstructs the reference, estimates
  paths and produces local delay/Doppler/direction measurements.
- **IntraSC:** A site or mobile cloud coordinates links and resources, aligns
  measurements in time, associates targets, and fuses local measurements.
- **InterSC:** Sites exchange target states/trajectories and create a wider
  situation picture. This layer can combine ISAC with other sensors and
  operator information; raw waveform-level coherence is not generally needed.

## 3. Propagation and measurement model

### 3.1 Link channel

For node \(n\) transmitting and node \(m\) receiving, the measured channel
frequency response can be estimated from the transmitted reference \(X_n(f)\)
and received signal \(Y_m(f)\):

\[
H_{m,n}(f) \simeq \frac{Y_m(f)}{X_n(f)}, \qquad 1\leq m,n\leq N .
\]

Stacking all links gives the distributed input/output channel matrix
\(\mathbf H(f)\in\mathbb C^{N\times N}\). Each entry includes a monostatic
diagonal or a bistatic off-diagonal link. In practice, cross-correlation or
frequency-domain system identification estimates the CIR/FRF.

### 3.2 Target, LOS, and clutter contributions

A useful bistatic decomposition is

\[
H_{m,n} = H_{\mathrm{LOS}}
 + H_{\mathrm{direct\ illumination}}H_{\mathrm{direct\ target\ response}}
 + H_{\mathrm{scattered\ illumination}}H_{\mathrm{scattered\ target\ response}}
 + H_{\mathrm{clutter}} .
\]

- **LOS** is not just interference: it supports frame/carrier synchronization,
  reference recovery, and differential eToF/eDoppler.
- **BTP components** pass through the target and carry target information.
- **Clutter** includes paths not associated with the target and can mask weak
  target echoes through sidelobes.
- A blocked direct path does not necessarily mean the target is unobservable:
  target-related multipath, spatial diversity, and known environmental
  reflections can provide alternative constraints.

![Source paper Fig. 6: bistatic line-of-sight, target, and clutter paths](images/fig-06-multipath-geometry.png)

*Source: Fig. 6, cropped from Thomä et al. (2026).*

For simulation, represent a link as a sum of \(P\) specular components:

\[
H(f,t)=\sum_{p=1}^{P}\gamma_p
  e^{-j2\pi f\tau_p}e^{-j2\pi t\alpha_p}+w(f,t),
\]

where \(\tau_p\) is path delay, \(\alpha_p\) is Doppler, \(\gamma_p\) is a
complex path weight (possibly polarization-dependent), and \(w\) represents
measurement noise/model error. The paper's finite observation aperture is
limited by bandwidth \(B\) and coherent observation duration \(T\). The
corresponding ideal Fourier resolutions are approximately \(1/B\) in delay and
\(1/T\) in Doppler; model-based estimation can resolve beyond these nominal
bin spacings under suitable SNR, model, and sampling conditions, but is not
unlimited or immune to ambiguity.

The complex target response can vary with aspect, frequency, polarization,
range, and time. For electrically large targets with many comparable
scattering contributions, the paper describes a circular-Gaussian field and
chi-square-like received-power variation; a dominant specular return instead
leads to a noncentral chi-square (squared-Rician) model. Extended targets
occupy correlated neighboring delay bins. These effects matter when generating
target amplitudes and independent-link fading in Monte Carlo experiments.

### 3.3 Bistatic geometry and state

Let \(\mathbf p\) be target position and \(\mathbf t_i,\mathbf r_i\) the
transmitter and receiver positions for link \(i\). The target-path length and
direct-path length are

\[
L_i(\mathbf p)=\|\mathbf p-\mathbf t_i\|+\|\mathbf p-\mathbf r_i\|,
\qquad
L_{0,i}=\|\mathbf t_i-\mathbf r_i\|.
\]

The absolute bistatic delay is \(L_i/c\); the CPCL excess delay is

\[
\tau_i^{\mathrm{ex}}=\frac{L_i(\mathbf p)-L_{0,i}}{c}.
\]

An absolute bistatic ToF measurement defines an ellipsoid with foci at Tx and
Rx. An eToF measurement defines the same family after adding the known direct
baseline. In 2D the surface is an ellipse; in 3D it is an ellipsoid. At least
three geometrically informative links are needed for a 3D position in the
basic range-only formulation. Additional measurements improve robustness
and expose outliers, but uniqueness and accuracy depend on geometry.

```text
                         target p
                       * (unknown)
                    .-' `-.
                 .-'       `-.        constant Tx-to-target +
      Tx  *------'             `------*  Rx path length
          \___________________________/
                   direct baseline

Each measured path-length sum defines an ellipsoid/ellipse.
The target is estimated from the intersection of several such surfaces.
Shallow intersection angles -> high GDoP and large position uncertainty.
```

For moving nodes and targets, define eDoppler through the time derivative of
the excess path length:

\[
\alpha_i^{\mathrm{ex}}
=-\frac{1}{\lambda}\frac{d}{dt}
 \left(L_i(\mathbf p(t))-L_{0,i}(t)\right),
\]

where \(\lambda=c/f_c\); the sign depends on the adopted complex-exponential
and approaching/receding convention. This derivative automatically includes
target and sensor velocities. It is a useful implementation definition because
it avoids ambiguity about unit-vector direction conventions. Each link measures
a different projection of target velocity. Three-dimensional velocity therefore
requires enough independent Doppler geometries; more links help resolve
ambiguities and reduce variance.

![Source paper Fig. 7: bistatic eToF ellipsoids and target velocity geometry](images/fig-07-bistatic-ellipsoids.png)

*Source: Fig. 7, cropped from Thomä et al. (2026).*

For noisy or redundant range observations, solve a nonlinear weighted least
squares problem over position, or linearize around the previous verified
position and use a pseudoinverse for a local update. The paper emphasizes
overdetermined estimation, outlier rejection (e.g., blocked LOS), and careful
link selection. Poorly crossing ellipses produce high GDoP even with accurate
individual delays.

### 3.4 Coverage and diversity

Monostatic free-space radar return falls approximately with the fourth power of
range; a simplified bistatic link budget is

\[
P_r \approx
\frac{P_tG_tG_r\lambda^2\sigma_b}
{(4\pi)^3 R_t^2R_r^2 L_{\mathrm{sys}}},
\]

where \(R_t,R_r\) are target distances from Tx/Rx, \(\sigma_b\) is
bistatic reflectivity/RCS, and \(L_{\mathrm{sys}}\) collects losses. This is a
simulation-level approximation, not a complete target/channel model from the
paper. It explains the Cassini-like coverage contours: performance is often
best near either radio node, moderate between them, and poor when far from
both. Distributed nodes trade site density and node placement for coverage,
diversity, and favorable GDoP.

![Source paper Fig. 8: shallow versus well-conditioned ellipse intersections](images/fig-08-gdop.png)

*Source: Fig. 8, cropped from Thomä et al. (2026).*

## 4. Radio access and resource allocation

The communication network provides the time/frequency/spatial access toolbox,
but sensing needs coordinated measurements of a common target rather than
independent user links.

| Access option | Sensing benefit | Main cost / caveat |
|---|---|---|
| Broadcast (one Tx, many Rx) | Concurrent bistatic observations; useful for dynamic targets and sniffers. | Only a subset of all Tx/Rx pairs is observed at once. |
| OFDMA / FDMA | Separates links in frequency; supports simultaneous measurements. | Per-link occupied bandwidth controls delay/range resolution. Fragmented bands cause ambiguity/grating lobes for plain FFT processing. |
| TDMA | Separates links in time; straightforward orthogonal access. | Reduces per-link slow-time sampling and can reduce unambiguous Doppler range. |
| Joint transmission (JT) | Multiple transmitters illuminate together; potentially more target energy and spatial diversity. | Extended-target returns may be noncoherent; without waveform separation, individual bistatic geometries can be lost. |
| Spatial/polarimetric access | Raises SNR, suppresses clutter, adds directional/polarimetric observables. | Beam search, calibration, hardware constraints, and additional resource cost. |

![Source paper Fig. 11: two transmitters sharing OFDM resources for different target-motion geometries](images/fig-11-orthogonal-access.png)

*Source: Fig. 11, cropped from Thomä et al. (2026).*

Resource allocation must consider below items:

- Sensing SINR/detection probability and false alarm/missed detection rates.
- Delay, Doppler, angle resolution and coherent integration duration.
- Target and sensor geometry, GDoP, target motion, and range migration.
- Frequency-time sparsity, multipath selectivity, waveform/CSI quality.
- Communication throughput, latency, reliability, and fairness.
- Transport and computation cost/latency for sensors fusion.

**TDMA details:** uniform interleaving maintains the slow-time sample rate for
each link. Sharing slots lowers that rate and hence the unambiguous Doppler
span. Nonuniform scheduling can prioritize fast targets/links, but requires
estimators that honor the actual sample times. Averaging adjacent OFDM symbols
can improve SNR; coherent processing across a longer interval improves Doppler
resolution but is limited by phase stability and target range migration.

![Source paper Fig. 12: uniform and nonuniform TDMA resource-block allocation](images/fig-12-tdma-resource-grid.png)

*Source: Fig. 12, cropped from Thomä et al. (2026).*

**FDMA details:** compact contiguous allocation yields conventional range
resolution. At fixed total bandwidth, fragmented allocation may increase
effective frequency spread but causes sidelobes/grating lobes, so the FFT of
zero-filled samples is not a reliable general estimator.

![Source paper Fig. 13: compact and fragmented FDMA resource allocation](images/fig-13-fdma-resource-grid.png)

*Source: Fig. 13, cropped from Thomä et al. (2026).*

**CRB-driven scheduling:** for model parameters
\(\boldsymbol\theta=[\tau_p,\alpha_p,\gamma_p]\), parameter variance is bounded
by

\[
\operatorname{var}(\theta_i)\geq
\operatorname{CRB}(\theta_i)=
[\mathbf F^{-1}(\boldsymbol\theta)]_{ii},
\quad
F_{ij}\propto \frac{2}{\sigma^2}
\operatorname{Re}\left\langle
\frac{\partial H}{\partial\theta_i},
\frac{\partial H}{\partial\theta_j}\right\rangle .
\]

The Fisher information is additive over observed resource elements. For delay
estimation, allocating energy near the available spectrum edges increases
RMS bandwidth and can reduce the bound. A practical heuristic from the paper
is to place links with tighter delay/Doppler resolution needs toward outer
frequency/time regions and links with looser needs toward the interior, while
interleaving links as needed. For realistic multiple-target/multipath scenarios,
compute the FIM from the actual masked resource grid and optimize resource
allocation under communication and sensing constraints; do not assume one
universal schedule.

The paper proposes a **sensing-cycle** concept: (1) broad surveillance and
acquisition, accepting coarse/ambiguous detections; (2) targeted refinement
with changed repetition rate, coherent interval, bandwidth, and beam; (3)
tracking with predictive link/resource adaptation. AI may help with this
nonconvex multi-objective scheduling problem, but is a proposed direction, not a
specified algorithm or trained model in the article.

### Example PRB policy: one downlink gNB and multiple receive-only sensors

Assume one gNB illuminates the scene in downlink and \(M\) spatially separated,
receive-only sensors listen to it. This is the paper's **broadcast** access
case: every sensor can observe the same gNB transmission at the same time.
Therefore, do **not** divide the gNB's PRBs by \(M\). The sensors produce
different bistatic measurements from their different locations; a sensor with
poor geometry or echo SNR may contribute little, but giving it a separate PRB
does not fix that by itself. This plan does not cover sensors that also transmit;
multiple active illuminators need coordinated orthogonal access or a designed
joint-transmission scheme.

Use the following as an initial scheduler policy, not as a PRB percentage
specified by the paper. Let \(B\) be the downlink PRB-symbol budget available
for PDSCH after mandatory control, reference-signal, and guard overhead. Count
resource in PRB-symbols so that frequency and time allocations are comparable.

| Sensing cycle | gNB scheduler action | Incremental downlink cost |
|---|---|---|
| Surveillance / acquisition | Schedule ordinary PDSCH for communication and let all sensors observe eligible payload REs using CPCL/reference reconstruction. Do not reserve sensing-only PRBs initially. | No sensing-only PDSCH reservation; actual sensing still requires receiver processing and measurement reporting. |
| Target refinement | If measured detection/CRB targets are missed, start with a sensing-only allocation of \(0.05B\), selected from otherwise schedulable data resources. Use known/PRS-like illumination where supported; choose frequency edges for delay precision and suitable repeated time samples for Doppler. | About 5% fewer PRB-symbols remain for PDSCH than in the no-reservation baseline, if this resource would otherwise carry data. |
| Tracking | Recompute the sensing need from the current CSI/KPI, target motion, and geometry each scheduling window. Keep the reservation at 0% when shared payload meets the requirement; otherwise adjust it in 5-point steps, initially testing no more than \(0.10B\). | Data capacity falls only for the fraction made exclusive to sensing; the 10% value is an experiment cap, not a universal optimum. |

For example, with \(B=100\) normalized schedulable PRB-symbol units, a 5-unit
refinement reservation leaves at most 95 units for PDSCH; the other 95 can
still serve both communication and sensing when their payload is usable as a
sensing reference. This is a resource-budget illustration, not a prediction
that throughput falls by exactly 5%: actual goodput depends on MCS, traffic
demand, retransmissions, and the particular NR overhead and scheduling
configuration. If the sensing-only reservation takes resources that would
otherwise be idle, its immediate user-throughput cost may be zero.

At each update, the scheduler should:

1. Form a candidate set of sensor links using geometry/GDoP, echo SNR, and
   target visibility; all selected receive-only sensors may listen to the same
   transmission.
2. First use eligible communication payload REs. Add exclusive sensing REs
   only when the required detection probability, delay/Doppler CRB, or track
   continuity is not met, while enforcing the communication rate/latency
   floor.
3. Allocate compact frequency bands when using conventional FFT processing;
   if resources are fragmented or sparse, pass the actual RE mask to a
   sparse/model-based estimator rather than treating missing REs as measured
   zeros. Prioritize frequency edges for demanding delay precision and
   adequate slow-time repetition for demanding Doppler precision.
4. Include uplink measurement reports and fusion transport in the capacity
   budget separately; these consume resources even though the receive-only
   sensors do not each need a separate downlink PRB allocation.

This operational example applies the paper's broadcast access, payload reuse,
adaptive sensing cycles, and CRB-guided scheduling principles. The paper does
not prescribe the 5% starting point or 10% test cap; those are explicit,
adjustable starting assumptions for a scheduler simulation and should be
swept against both sensing KPIs and communication goodput. See Thomä et al.,
[“Distributed multisensor ISAC”](https://doi.org/10.1038/s44459-026-00041-2),
especially the MS radio access discussion (pp. 11–13) and resource scheduling
(pp. 18–19).

```mermaid
flowchart LR
    A[Surveillance\nand\nacquisition] --> B[Coarse target\nhypotheses]
    B --> C[Choose links\nand allocate REs]
    C --> D[Refine delay,\nDoppler,\nand direction]
    D --> E[Associate and\nupdate tracks]
    E --> F[Predict target\nand\nlink geometry]
    F --> C
    E -. lost track\nor new area .-> A
    C -. balance sensing\nwith data service .-> G[Communication\nQoS]
```

## 5. CPCL reference recovery and delay-Doppler processing

### 5.1 Why the reference matters

Conventional PCL compares a surveillance signal with a clean copy of the
transmitted waveform, traditionally acquired by a separate directional receive
channel. CPCL uses the communication system's synchronization and reference
signals and reconstructs the transmitted payload from the received OFDM signal.
The direct LOS and BTP are observed in the same CIR, allowing eToF/eDoppler
differences to cancel common Tx/Rx clock and oscillator offsets. The reduced
sniffer mode can use received transmissions without active transmitter
coordination; full CPCL can additionally exploit network feedback, transmitter
adaptation, and data fusion.

The receiver must account for empty resource elements, unequal pilot/data
power, QAM symbol magnitudes, and RF-chain distortion. Inverse filtering all
carriers indiscriminately can amplify noise or introduce incorrect samples.

```mermaid
flowchart TD
    A[OFDM\nreceive\nsamples] --> B[Frame/carrier sync,\nCP removal,\nFFT]
    B --> C[Pilot CSI and\ninitial\nequalization]
    C --> D[Detect active REs\nand recover\npayload\nreference]
    D --> E[Refined equalization / Turbo-CPCL]
    E --> F[Active-RE inverse\nfiltering\nwith SINR weights]
    F --> G[Per-symbol\nCIR sequence]
    G --> H{Resource\ngrid}
    H -->|Dense and uniform| I[Frequency FFT\nthen\nslow-time FFT]
    H -->|Sparse or irregular| J[Weighted complex\npath-model\nfit]
    I --> K[LOS/BTP detection and\nrelative eToF/eDoppler]
    J --> K
    K --> L[Multilink\nassociation\nand\nstate fusion]
```

![Source paper Fig. 15: OFDM receiver processing through delay-Doppler estimation](images/fig-15-ofdm-processing.png)

*Source: Fig. 15, cropped from Thomä et al. (2026).*

### 5.2 Turbo-CPCL processing algorithm

1. **Synchronize and demodulate:** use network timing/carrier synchronization;
   remove the cyclic prefix and FFT the OFDM symbols.
2. **Estimate an initial channel:** estimate CSI at pilots and interpolate for
   initial equalization.
3. **Recover the reference:** detect active/empty resource elements (e.g.,
   DMRS information or a null-hypothesis test in reduced mode), equalize the
   received communication payload, and form an initial reconstructed reference.
4. **Refine iteratively:** use the payload estimate and the full transmitted
   power/resource structure to improve channel equalization/reference recovery.
   The refined reference captures linear and nonlinear transmitter distortion
   present over the air; avoid assuming the ideal digital constellation is the
   exact emitted waveform.
5. **Estimate per-symbol FRF:** on active carriers, use the reference-normalized
   relation
   \[
   \widehat H(f)=\frac{Y(f)X_{\mathrm{ref}}^*(f)}
   {|X_{\mathrm{ref}}(f)|^2}
   =\frac{Y(f)}{X_{\mathrm{ref}}(f)}.
   \]
   Exclude empty carriers and weight low-SNR samples less; use regularization
   rather than unstable division near zero magnitude.
6. **Form CIRs:** transform over frequency for each symbol to obtain a sequence
   of channel impulse responses. Record LOS and target-related returns.
7. **Estimate Doppler:** coherently transform the CIR sequence over slow time.
   The coherent processing interval \(T\) sets nominal Doppler resolution
   (\(\sim1/T\)).
8. **Detect and localize:** separate LOS, target, and clutter components;
   measure target delay/Doppler relative to LOS; associate link measurements
   and fuse them into position/velocity.

### 5.3 FFT versus model-based estimation

With a fully and uniformly occupied OFDM grid, a frequency FFT followed by a
slow-time FFT is efficient and produces the delay-Doppler scattering function.
The ambiguity function (AF) of a finite \(B\times T\) aperture has sinc-like
mainlobes and sidelobes; weak targets can be hidden by stronger clutter. FFT
bin maxima also quantize parameters, and long coherent intervals can blur
moving targets through range-Doppler migration.

For sparse/nonuniform resource elements, **do not substitute zero for an
unobserved channel sample and treat it as data**. This changes the AF and can
create uncontrolled sidelobes. Instead, fit the physical channel model only at
the actually observed resource elements:

\[
(\widehat{\boldsymbol\tau},\widehat{\boldsymbol\alpha},
\widehat{\boldsymbol\gamma})
=\arg\min_{\boldsymbol\tau,\boldsymbol\alpha,\boldsymbol\gamma}
\sum_{k\in\Omega} w_k
\left|H_{\mathrm{obs}}(f_k,t_k)-
\sum_{p=1}^{P}\gamma_p
e^{-j2\pi f_k\tau_p}e^{-j2\pi t_k\alpha_p}\right|^2 .
\]

Here \(\Omega\) is the observed RE set and \(w_k\) reflects sample reliability.
This is a weighted complex least-squares / maximum-likelihood formulation
under appropriate noise assumptions. It interpolates the sampled complex
response according to the propagation model and can separate overlapping
components more effectively than magnitude-domain CLEAN. Model order \(P\)
must be selected or estimated; the article does not prescribe one universal
order-selection rule. Recursive estimation is suggested for real-time operation.

![Source paper Fig. 17: delay-Doppler distortion from a sparsely occupied resource grid](images/fig-17-sparse-delay-doppler.png)

*Source: Fig. 17, cropped from Thomä et al. (2026).*

![Source paper Fig. 18: sparse active resource blocks in the frequency-time aperture](images/fig-18-sparse-resource-sampling.png)

*Source: Fig. 18, cropped from Thomä et al. (2026).*

![Source paper Fig. 19: channel samples and gaps caused by sparse resource allocation](images/fig-19-sparse-channel-sampling.png)

*Source: Fig. 19, cropped from Thomä et al. (2026).*

For improved estimates:

- jointly estimate delay and Doppler, then add angle/direction and polarization
  where hardware/calibration allows;
- use local off-grid refinement after FFT initialization if FFT is used;
- model the actual irregular RE pattern, symbol timing, and per-RE SNR;
- use a device model for frequency response, phase noise, I/Q imbalance,
  nonlinearities, and array-pattern calibration when those effects matter;
- use CRB/FIM calculations with the actual resource mask to compare schedules
  and estimate achievable variance.

## 6. Synchronization, precoding, and spatial sensing

### 6.1 Differential synchronization

Communication synchronization (PSS/SSS, DMRS/PT-RS, cyclic prefix) gives a
useful baseline for frame coordination. High-accuracy bistatic delay and
Doppler can demand finer timing and carrier-phase stability than ordinary data
decoding. CPCL mitigates this by estimating LOS and BTP from the same
reference/channel record and taking their delay/frequency differences. This
avoids requiring perfectly aligned independent Tx/Rx clocks for every
measurement, provided the reference is correctly reconstructed and the LOS
can be identified.

For higher-level fusion, use absolute timestamps and align measurements to a
common time. The paper cites approximately ±130 ns timing and 0.05 ppm
frequency synchronization as 5G infrastructure figures; these are context,
not universal design guarantees. GNSS/GPSDO, IEEE 1588 PTP/White Rabbit, and
network-based methods are discussed as possible references. Moving mesh nodes
need explicit treatment of clock drift, sensor motion, phase drift, and stale
data. Doppler compensation should follow the LOS reference, not an arbitrary
composite of LOS and multipath.

### 6.2 Multipath exploitation

- **Background subtraction:** for static sensors and static clutter, subtract a
  stored or slowly updated background so moving-target returns remain. It is
  harder when the sensors move or the environment changes.
- **Time-reversal focusing (TRF):** estimate a channel, conjugate/time-reverse
  its response, and precode using CSI feedback. The effective channel is
  autocorrelation-like and can focus energy in delay/space. It requires the
  channel to remain sufficiently stable over estimation, feedback, and
  transmission. The paper's illustrative measured-channel example used 3.75 GHz
  and 100 MHz bandwidth; a roughly four-wavelength displacement caused about
  10 dB correlation loss in that example.
- **Target-aware focusing:** remove target-independent clutter before
  precoding; otherwise end-to-end focusing may focus clutter as well as the
  target. Extended-target response may be stochastic/noncoherent, so
  increasing reflected energy is often more defensible than assuming coherent
  focusing on a single scatterer.
- **Doppler precoding:** compensate a predicted target Doppler to extend
  coherent integration, subject to motion/model error.
- **Ray-tracing / fingerprint methods:** known reflectors can act as virtual
  anchors in model-based localization; fingerprints can be learned or compared
  with a digital-twin database. These are complementary approaches, not a
  replacement for careful path association.
- **Multifrequency and polarization:** lower frequencies aid coverage and
  diffraction/penetration; higher bandwidth and arrays can improve resolution
  and SNR but increase blockage/beam-management sensitivity. Polarimetric
  Tx/Rx channels can characterize target response with a \(2\times2\) Jones
  matrix; antenna and target coordinate frames must be calibrated.

### 6.3 Beamforming and direction estimation

Beamforming increases link SNR and can filter clutter. Bidirectional Tx/Rx
beamforming can retain a clean LOS reference while selecting target returns.
It requires beam search and calibrated patterns. Hybrid/analog beamforming
often hides individual antenna outputs, limiting direct MUSIC/ESPRIT array
processing.

The paper describes an alternative using overlapping, shift-invariant beams
whose phase centers differ. Their phase difference provides high-resolution
direction information with fewer digital ports than physical antenna elements.
Multiple translated subarrays can extend this to multiple arrivals, provided
patterns and phase relationships are calibrated. For DoD, transmit waveforms
must be distinguishable, usually through orthogonal time-frequency resources.

## 7. Data fusion and tracking

At each link, local processing yields delay, Doppler, direction, confidence,
and timestamp. Fusion must address target association, non-isochronous
measurements, missing/occluded paths, link-specific uncertainties, and
network latency.

Recommended separation:

1. **Sensor link:** estimate paths and local target observables; report quality,
   timestamp, covariance/confidence, and sensor pose/velocity.
2. **Site/cloud:** align times (prefer prediction to a common epoch), select
   geometrically useful sensors, reject outliers, solve a weighted nonlinear
   state estimate, and update target tracks.
3. **Inter-site:** exchange tracks/trajectories or summarized observations;
   perform global association and situation-level fusion.

Kalman-family trackers can model 3D position, velocity, and optionally
acceleration/orientation. Multi-target data association can use methods such as
MHT or JPDA. Tracking provides smoothing, prediction through missed detections,
and a basis for predictive resource allocation. Early/raw fusion preserves
information but increases transport/computation demands; late fusion of local
estimates is more modular and can reduce shared data. Keep uncertainty and
measurement age attached to all reports.

```mermaid
flowchart LR
    A[Per-link observables<br/>delay,\nDoppler,\ndirection,\nconfidence,\ntime] --> B[SLC<br/>path and target\nmeasurements]
    B --> C[IntraSC<br/>time alignment,\nassociation,\nrobust fusion]
    C --> D[Local tracks and\nmission control]
    D --> E[InterSC<br/>track/trajectory\nfusion]
    E --> F[Global situation\npicture]
    D -. resource requests\nand predicted\ngeometry .-> B
```

## 8. Simulation recipe

The following is a practical baseline for a new simulator. It is an
implementation plan derived from the paper, not code or a parameter set
published by the authors.

### 8.1 State and configuration

Define a configuration with:

- carrier frequency \(f_c\), occupied bandwidth \(B\), OFDM numerology,
  subcarrier/symbol spacing, CP, coherent interval \(T\), and frame schedule;
- node positions, velocities, antennas, Tx power, frequencies, and clock/LO
  offsets;
- target position, velocity, optional acceleration/extent/orientation, and
  reflection model;
- link set / resource mask \(\Omega_i\), per-link power, and SNR;
- clutter paths, blockage, multipath, measurement noise, and RF impairments;
- fusion epoch, communication/transport delay, and data-dropout model.

Use SI units consistently. Keep **absolute path delay**, **excess delay**, and
**Doppler sign convention** explicit in code and result files.

### 8.2 End-to-end algorithm

```text
Input: node states, target states, channel/resource configuration, noise model

1. Generate candidate Tx/Rx links and their time-frequency resource masks.
2. For each link and coherent frame:
   a. Compute LOS, target, target-multipath, and clutter path delays/Dopplers.
   b. Generate complex path weights, including aspect/polarization/fading.
   c. Sample H(f,t) only on allocated REs; apply waveform and RF impairments.
   d. Add receiver noise; optionally model blockage, clock, and phase drift.
   e. Recover reference and estimate H on active REs (CPCL processing).
   f. Estimate paths:
      - uniform dense grid: FFT-based initialization;
      - sparse/irregular grid: weighted complex model fit.
   g. Detect target paths; measure eToF/eDoppler relative to LOS.
3. Time-align per-link measurements and transform to a common coordinate frame.
4. Associate measurements with target hypotheses; reject inconsistent/outlier links.
5. Estimate position/velocity with weighted nonlinear LS, then update tracks.
6. Compute sensing KPIs and communication/resource costs.
7. Optionally adapt selected links, RE allocation, beam, or integration time;
   repeat for the next sensing cycle.
Output: per-link estimates, fused tracks, uncertainty, detection statistics,
        resource use, latency, and communication performance.
```

### 8.3 Estimation and fusion blocks

**Reference and channel estimation:** apply inverse filtering only on active
carriers. Use a noise-aware denominator/regularizer and weighted fitting when
power varies across data carriers. Model the true RE mask rather than
interpolating missing samples as if measured.

**Delay-Doppler detection:** retain complex samples for model fitting. For
FFT-based plots, label axes in seconds and hertz and preserve the true slow-time
sampling schedule. When target motion changes delay appreciably over \(T\),
shorten coherent processing or use a joint range-Doppler/range-migration model.

**Position/velocity solve:** use an objective such as

\[
\widehat{\mathbf p}
=\arg\min_{\mathbf p}\sum_i w_i
\left(\widehat{\tau}^{\mathrm{ex}}_i-
\frac{\|\mathbf p-\mathbf t_i\|+\|\mathbf p-\mathbf r_i\|
-\|\mathbf t_i-\mathbf r_i\|}{c}\right)^2,
\]

and fit velocity by minimizing residuals between measured eDoppler and the
time derivative of the modeled excess path. For moving nodes, propagate node
states to each measurement timestamp. Use robust losses or gating for
outliers, then report covariance/CRB or empirical uncertainty.

### 8.4 Minimum experiment matrix

| Dimension | Cases to compare |
|---|---|
| Geometry | well-conditioned vs shallow ellipse intersections; varying Tx/Rx baseline; node density. |
| Waveform/grid | dense OFDM, TDMA-interleaved, compact FDMA, fragmented/sparse REs. |
| Estimator | 2D FFT with dense data; zero-filled sparse FFT as a negative control; weighted model fit. |
| Propagation | LOS and clutter; target near Tx/Rx; target multipath; blocked LOS; correlated/extended target. |
| Synchronization | ideal, fixed timing/frequency offset, drift/phase noise, CPCL differential cancellation. |
| Motion | stationary target, radial/tangential motion, sensor motion, range migration. |
| Coordination | broadcast, orthogonal links, JT, adaptive sensing cycles. |
| Precoding | none, beamforming, background subtraction, channel-matched TRF with channel aging. |

Report probability of detection versus false alarm, delay/Doppler RMSE, 3D
position and velocity RMSE, track continuity/false tracks, GDoP, CRB, SNR,
resource utilization, communication throughput/latency, and fusion delay.
Sweep SNR and repeat independent target/channel realizations; separate
algorithmic error from geometry and synchronization error. Include confidence
intervals rather than a single realization. Compare estimated covariance with
empirical error to detect model mismatch.

### 8.5 Published measurement example (reference check)

The article's demonstration is a **measurement**, not a simulation data set:

- Three stationary radio nodes (one Tx and two Rx) were mounted on cars; a
  target car moved through the scene.
- USRP X310 radios; 33 dBm transmit power; 5.2 GHz carrier; 80 MHz bandwidth.
- A periodically repeated multisine had a 3.2 \(\mu\)s period and uniform
  carrier magnitude, with phases chosen for low peak-to-average power ratio.
  The known transmitted waveform was correlated at the receivers; GPSDOs
  synchronized the nodes.
- Slow-time observation was about 50 ms. The target was barely visible in the
  power-delay profile, including after background subtraction, but appeared
  clearly in delay-Doppler maps.
- Reported target estimates: Rx1 at 137 ns eToF and 380 Hz eDoppler; Rx2 at
  98 ns eToF and 355 Hz eDoppler.
- The two eToF constraints did not uniquely/localization-robustly intersect;
  sensor placement yielded high GDoP. This is a useful negative lesson:
  detecting a target on multiple links does not ensure a well-conditioned
  position estimate.

![Source paper Fig. 24: the three-node dual-link measurement setup](images/fig-24-measurement-setup.png)

*Source: Fig. 24, cropped from Thomä et al. (2026).*

![Source paper Fig. 27: measured delay-Doppler result at receiver 1](images/fig-27-rx1-delay-doppler.png)

*Source: Fig. 27, cropped from Thomä et al. (2026).*

![Source paper Fig. 28: measured delay-Doppler result at receiver 2](images/fig-28-rx2-delay-doppler.png)

*Source: Fig. 28, cropped from Thomä et al. (2026).*

![Source paper Fig. 29: eToF ellipse intersection for the measured target](images/fig-29-measured-target-ellipses.png)

*Source: Fig. 29, cropped from Thomä et al. (2026).*

The authors state that the data set and code are not publicly available; the
example is therefore a qualitative reference point, not a directly
reproducible benchmark.

### 8.6 Source-paper metrics and KPI targets (Not Satisfied with the update, need to revisit details from paper!)

The 6G-DISAC paper frames its metrics around PHY-layer and system performance:
waveform shaping, hardware compliance, and new signal- and semantic-processing
techniques. It calls out the communication-versus-sensing resource-allocation
trade-off as essential to determining whether use-case KPIs are met. Evaluation
must extend beyond the usual single-sensing-receive-node metrics to collaborative
sensing with multiple nodes. The paper says the framework will be evaluated
through **seven** distinct Quantitative Targets (QTs), but the Metrics and KPIs
section enumerates only **six** (QT1–QT6); it gives no QT7, so none is inferred
here.

| Target | Metric and baseline/reference stated in the paper | 6G-DISAC target | Rationale or enabling approach stated in the paper |
|---|---|---|---|
| **QT1: Spectral efficiency** | Relative comparison with current 3GPP Release 18 MIMO benchmarks. | **30% relative improvement**; no frequency range is specified. | Sensing-aided communications are expected to refine MIMO channel-matrix and geometric-channel-parameter estimates, improving spectral efficiency. |
| **QT2: User positioning accuracy** | The paper says the current baseline is undefined. | Absolute accuracy of **0.01 m in FR2** and **0.1 m in FR1**. | Combine positioning and sensing, using large bandwidths and apertures. |
| **QT3: Energy efficiency** | The paper says it compares with standards outlined in **3GPP TR 38.684**. | **40% improvement in FR2**. | As for QT1, sensing-aided communication is expected to improve geometric-channel-parameter estimation and thereby energy efficiency. |
| **QT4: User orientation accuracy** | Current baseline stated as **1°–5°**. | **Less than 1° in FR2**. | UE-side antenna arrays and numerous base stations (BSs) or access points. |
| **QT5: Object positioning accuracy** | Current baseline stated as **1 m**. | **Less than 1 m in FR1** and **0.1 m in FR2**. | Large apertures in FR1; large bandwidths and arrays in FR2; numerous transmitters and receivers in both frequency ranges. |
| **QT6: Object velocity-estimation accuracy** | Current baseline stated as **1 m/s**. | **Less than 1 m/s in FR1**. | Large integration times in FR1. |

#### Comparison with the 3GPP references in the paper

These are **6G-DISAC project targets**, not values that the paper identifies as
3GPP-mandated ISAC requirements. The comparison points are mixed: QT1 uses
Release 18 MIMO benchmarks, and QT3 cites TR 38.684; neither is presented in
this section as a distributed-ISAC KPI specification. The QT3 identifier
should be verified: the paper prints “TR 38.684,” while the [3GPP portal entry
for TR 38.864](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=3987)
describes a different report, “Study on network energy savings for NR.” Do not
silently replace the paper's cited number with 38.864.

The paper also cites the 3GPP SA1 Release 19 study on ISAC [4], identified in
the [3GPP portal as TR 22.837, “Study on Integrated Sensing and
Communication”](https://portal.3gpp.org/desktopmodules/Specifications/SpecificationDetails.aspx?specificationId=4044),
but supplies no numerical KPI baseline from that study for QT1–QT6. In the
adjacent architectural discussion, it says 3GPP supports eMBB, URLLC, mMTC,
and UE positioning, while significant architectural changes are needed for
full sensing integration.

Accordingly, do not read the QT numbers as standardized 3GPP thresholds or as
directly comparable benchmark results. The source does not give enough
measurement definitions or test conditions here to normalize the targets
against the cited baselines. A fair comparison should align the KPI definition,
frequency range, deployment and node configuration, bandwidth/aperture, and
resource allocation; these factors affect whether a reported improvement is
comparable across a single-node 3GPP baseline and collaborative multisensor
DISAC.

## 9. Practical pitfalls and decision checklist

- **Do not conflate delay and range:** bistatic range is a sum of two ranges;
  CPCL commonly measures excess path length after subtracting the Tx/Rx
  baseline.
- **Do not assume the brightest path is the target:** LOS/clutter may dominate;
  use Doppler, background, geometry, and path models.
- **Do not zero-fill missing REs as true observations:** model the sparse mask.
- **Do not equate array beam selection with high-resolution DoA:** phase
  comparison/calibration is needed for sub-beam direction estimates.
- **Do not maximize link count blindly:** select for independent geometry,
  SNR, and timing; surplus links are useful for robustness if transport allows.
- **Do not extend coherent integration without checking phase coherence and
  range migration.**
- **Do not expect fully coherent joint transmission from an extended,
  statistically varying target; noncoherent illumination may be the realistic
  diversity/energy objective.**
- **Do not ignore the communication service:** sensing REs, pilots, feedback,
  and reports compete for channel, latency, and compute budgets.
- **Do not report a position without uncertainty, timestamp, geometry quality,
  and measurement provenance.**

## 10. Source outline and limits

The paper covers architecture/use cases, distributed channel and target
propagation, multistatic target geometry, radio access and OFDMA scheduling,
CPCL reference recovery, model-based delay-Doppler estimation, CRB-driven
resource allocation, time-reversal and spatial precoding, synchronization,
fusion/tracking, and the measurement example summarized above. It is a
principles/review article: detailed propagation simulation, a complete
end-to-end scheduler, and a public implementation are outside its scope.

## 11. Power constraints and optimization opportunities

This section and sections 12-13 are an additive research update dated
2026-11-10. They supplement, rather than replace, the preceding source-paper
summary. References [P1]-[S4] identify related literature; equations and
implementation policies explicitly called **proposed** are extensions for
this guide, not algorithms supplied by the source paper. No universal power
saving, sensing-reservation percentage, or deployment-ready optimizer is
claimed.

### 11.1 Which power constraints matter?

**Distributed multisensor ISAC is power constrained, but adding receivers
does not necessarily multiply illumination power.** Several receive-only
sensors can observe one broadcast transmission, as in section 4. Their
receiver, processing, synchronization, and reporting costs still increase
system energy consumption.

| Constraint or cost | What to model | Consequence for optimization |
|---|---|---|
| Per-transmitter RF power | Each gNB/TRP, UE, or illuminator has its own maximum transmit power. Include per-antenna limits where relevant. | A network-wide sum-power constraint alone can admit physically impossible allocations. |
| Shared communication/sensing waveform | Communication payload and beams may also illuminate targets. Optional sensing-only signals consume additional resources and power. | Do not automatically count the same radiated waveform twice as separate communication and sensing power. |
| Average, peak, and radiated-power limits | Average RF power, amplifier back-off/PAPR, permitted EIRP, and applicable spectral masks are distinct limits. | A feasible average-power solution can still violate hardware or emission constraints. Use deployment-specific limits, not assumed universal values. |
| Battery and duty cycle | Mobile nodes consume energy during Tx, Rx, processing, synchronization, and reporting, including switching/wakeup overhead. | Optimize energy over a scheduling horizon, not just instantaneous radiated power. |
| Active-node overhead | RF chains, ADCs, baseband processing, and transport can consume power even at low transmit power. | Reducing RF power alone need not minimize total system consumption; selective activation can matter. |
| Interference and receiver dynamic range | Direct-path illumination, clutter, other transmitters, and self/cross-link interference affect usable echo quality. | Increasing transmit power can also increase interference or saturation; model sensing SINR and receiver feasibility. |

Section 3.4 gives the simplified bistatic echo dependence
\(P_r\propto P_t/(R_t^2R_r^2)\). Favorable node placement and geometry can
therefore be more valuable than increasing power on a weak link. Increasing
all powers also cannot repair an unobservable position or distinguish
ambiguities caused by a poor resource pattern.

### 11.2 What optimization research is already available?

The following are literature findings, not claims that this repository
implements those methods:

| Research direction | Established problem setting | Relevance and boundary |
|---|---|---|
| Sensing-aware power allocation [P1] | Cell-free massive-MIMO multistatic detection; maximize sensing SINR while meeting minimum communication SINR and per-AP power constraints. | A concrete starting point for distributed infrastructure power control. Its detection objective is not automatically a localization or tracking guarantee. |
| Joint sensing/communication beamforming [P2] | Joint beamformer design in cell-free ISAC MIMO systems. | Spatial power control can improve the sensing/communication tradeoff. Transfer requires checking the paper's channel, coordination, and coherence assumptions. |
| Joint AP activation and power allocation [P3] | Select active APs together with their powers to reduce consumption in cell-free ISAC. | Supports studying node-selection savings beyond RF power adjustment. Account for receiver and transport costs appropriate to this deployment. |
| Joint resource and power allocation [S2, S4] | Optimize OFDM resources with sensing/communication constraints; dynamic TDD additionally couples resources to AP modes. | Power control should ultimately use the same resource mask and Tx/Rx decisions as the scheduler. |

Cell-free infrastructure results do not directly establish performance for
asynchronous CPCL sniffers, arbitrary mobile meshes, or extended targets with
noncoherent returns. Solver details and numerical gains must be checked in
the selected paper and reproduced under the simulator's assumptions before
being used as benchmarks.

### 11.3 Proposed geometry-aware minimum-power baseline

First fix beams, participating nodes, and the time-frequency mask. A useful
**proposed baseline**, assuming continuous operation over the fixed interval,
is

\[
\begin{aligned}
\min_{\mathbf p,\mathbf a}\quad&
\sum_i\left(\frac{p_i}{\eta_i}+a_iP_{\mathrm{fixed},i}\right)\\
\text{subject to}\quad&
0\le p_i\le a_iP_i^{\max},\qquad a_i\in\{0,1\},\\
&\mathrm{SINR}^{\mathrm{comm}}_u\ge\gamma_u,\quad \forall u,\\
&\mathbf J_{\mathrm{pos}}\succ0,\qquad
\operatorname{tr}(\mathbf J_{\mathrm{pos}}^{-1})
\le\epsilon_{\mathrm{pos}}^2 .
\end{aligned}
\]

Here \(p_i\) is total radiated power at node \(i\), \(\eta_i\) is amplifier
efficiency, and \(P_{\mathrm{fixed},i}\) is active-node operating power.
A receive-only node has \(p_i=0\) but can have nonzero operating power.
Initially hold \(\mathbf a\) fixed; subsequently optimize activation as well.
The objective is electrical power in watts; multiply by the fixed interval
to obtain energy. Variable duty cycles require the energy model in section
12.3 instead.

\(\mathbf J_{\mathrm{pos}}\) is effective position information after
accounting for unknown channel, clock, and other nuisance parameters. For an
identifiable nuisance block, this is the Schur complement
\(\mathbf J_{xx}-\mathbf J_{xz}\mathbf J_{zz}^{-1}\mathbf J_{zx}\), not
simply the position block of the full FIM. Sum independent measurement
information before eliminating shared nuisance parameters; correlated
observations require their joint covariance. Reject singular/unobservable
geometries rather than reporting a misleading finite bound using a
pseudoinverse.

The CRB is a local lower bound under the assumed statistical model, not an
achieved RMSE or detection guarantee. Evaluate actual localization error,
missed detections, outliers, and synchronization mismatch as well. If the
application is detection rather than localization, use a validated detection
constraint at a specified false-alarm rate instead of substituting a position
CRB.

With fixed masks and beams, some simplified power-control problems are
convex; general interference-coupled formulations need not be. Adding binary
activation, beams, or resource decisions generally creates a nonconvex
mixed-integer problem. Alternating optimization or successive convex
approximation are candidate implementation techniques, not guarantees of a
global optimum for this proposed model.

## 12. Scheduling constraints and joint power-resource design

### 12.1 Scheduling is more than assigning a sensing PRB percentage

The source article [S1] describes sensing-aware radio access and adaptive
sensing cycles. Related research explicitly treats limited OFDM occupancy
[S3], dynamic-TDD resource and AP-mode allocation [S2], and cooperative
resource-element/power design with ambiguity constraints [S4].

The following checklist combines those research directions with engineering
constraints for the deployments in this guide. It is not a claim that every
cited paper includes every constraint, or that a research scheduler is a
standardized NR procedure.

| Scheduling constraint | Why it matters | Required scheduler/estimator treatment |
|---|---|---|
| Communication commitments | Rate floors alone do not ensure packet deadlines, reliability, or fairness. Control, reference signals, guards, and retransmissions also occupy resources. | Protect mandatory resources and separately track goodput, queueing/deadline misses, and sensing reservations. |
| Half-duplex and TDD availability | A half-duplex node cannot illuminate and receive on the same carrier at the same instant. UL/DL directions and switching guards limit feasible links. | Jointly choose Tx/Rx roles and time windows. Full-duplex requires an explicit residual self-interference and isolation model. |
| Mixed duplexing/numerology | Nodes may not share bands, symbol boundaries, or available transmit/receive periods. | Use each link's actual waveform and timestamps; enforce band compatibility, guards, and cross-link/inter-numerology interference constraints. |
| Transmitter separation and reuse | Overlapping illuminators can be inseparable or interfere. One distinguishable broadcast waveform can nevertheless support many receivers. | Use orthogonal REs, separable waveforms, or an explicitly modeled spatial/coherent reuse scheme. Do not assign exclusive downlink REs to every listening receiver. |
| Frequency aperture and occupancy | Fragmentation changes RMS bandwidth and ambiguity sidelobes; equal PRB counts can have different sensing performance. | Evaluate the actual frequency mask, power weights, and ambiguity region, not only occupied bandwidth or CRB. |
| Slow-time sampling | Sparse/interleaved opportunities change Doppler ambiguity and sample support. | Schedule adequate repetition for the expected Doppler range and pass actual sample times to the estimator. |
| Coherent processing interval (CPI) | Longer integration improves local precision but adds latency and can violate phase stability or target-motion assumptions. | Bound CPI using coherence, range migration, acceleration, and reporting deadlines; apply motion compensation when justified. |
| Target revisit and measurement age | A target can become poorly tracked between otherwise accurate observations. | Enforce task-specific revisit/deadline requirements and predict uncertainty to the fusion/output epoch. |
| Geometry and observability | Several strong links can still provide nearly redundant position information. | Select links jointly for visibility, independent geometry, uncertainty, and robustness; do not rank by SNR alone. |
| Synchronization and calibration | Timing, frequency, phase drift, and stale calibration consume budget and bias measurements. | Reserve needed calibration/reference opportunities and include residual errors in the measurement model. Do not assume network-wide phase coherence. |
| Reporting, fronthaul, and compute | Raw samples, local parameters, and tracks have different bit rates and processing delays. | Budget report transmissions, transport queues, and processing capacity; use a fusion representation that can meet the deadline. |
| Energy and activation | Aggressive revisits or frequent wakeups can exhaust mobile nodes despite low average RF power. | Couple duty cycle, node activation, and switching energy to power allocation. |

### 12.2 Precision, ambiguity, and freshness are different constraints

For an illustrative single-link model with uniformly spaced slow-time
samples separated by \(\Delta t_s\), Doppler aliases with period
\(1/\Delta t_s\). A conventional centered unambiguous interval is

\[
|\nu|<\frac{1}{2\Delta t_s},
\qquad
\Delta\nu_{\mathrm{res}}\approx\frac{1}{T_{\mathrm{CPI}}}.
\]

These are sampling/aperture relationships, not achieved estimation-error
bounds. Bistatic Doppler-to-velocity conversion depends on Tx-target-Rx
geometry; do not apply the monostatic factor \(v=\lambda\nu/2\) universally.
For irregular schedules, neither the mean spacing nor one largest-gap rule
fully characterizes ambiguity: evaluate the actual sample pattern.

Similarly, contiguous frequency aperture \(B_{\mathrm{span}}\) gives a
nominal delay-resolution scale near \(1/B_{\mathrm{span}}\), while a uniform
frequency comb with spacing \(\Delta f_s\) has delay replicas spaced by
\(1/\Delta f_s\). These simplified relationships do not replace CP,
observation-window, multipath, and reference-recovery feasibility checks.
They concern delay, not a universal monostatic range conversion.

For a known waveform and a simplified isolated-link matched-filter model,
a useful normalized mask diagnostic is

\[
A(\delta\tau,\delta\nu)=
\frac{\left|\sum_{(n,k)\in\Omega}
e_{n,k}\exp\!\left(j2\pi[f_n\delta\tau-t_k\delta\nu]\right)\right|}
{\sum_{(n,k)\in\Omega}e_{n,k}},
\]

where \(e_{n,k}\) is transmitted RE energy. Evaluate nonempty masks and
measure sidelobes over a declared delay-Doppler region **excluding the
intended mainlobe**. The complete channel/waveform model may require a more
detailed ambiguity function. Two schedules can have favorable local CRBs
but different false-peak behavior; test both.

Resource-occupancy research [S3] studies CRB-oriented waveform design and
model-based recovery of undersampled sensing channels. Such recovery relies
on structural assumptions; reconstructed samples are not independent new
measurements and must not be counted as extra observed REs in the FIM.
Cooperative allocation research [S4] explicitly combines RE/power choices
with ambiguity-sidelobe constraints.

For tracking, add freshness requirements such as

\[
t^{\mathrm{obs}}_{q,m+1}-t^{\mathrm{obs}}_{q,m}\le G_q^{\max},
\qquad
t^{\mathrm{output}}_{q,m}-t^{\mathrm{obs}}_{q,m}\le D_q^{\max}.
\]

Here \(q\) identifies a target/task; \(G_q^{\max}\) and \(D_q^{\max}\) are
application-defined revisit and end-to-end age limits, not universal values
from the cited papers. Define the observation epoch explicitly for a CPI.
Include acquisition, processing, reporting, transport, and fusion delays in
output age. Propagate track covariance to that output epoch; a small CRB at
an old measurement time is not sufficient.

### 12.3 Proposed joint scheduling model

Use the following as a **simulation proposal**, not a reproduction of one
paper's complete optimization problem.

Let \(x_{i,n,k}\) indicate transmission by node \(i\) on subcarrier \(n\),
symbol \(k\); let \(p_{i,n,k}\) be its average RF power on that RE.
Let \(t_{i,k}\), \(r_{i,k}\), and \(a_{i,k}\) be binary Tx, Rx, and active
indicators. For half-duplex nodes on the modeled carrier, require

\[
x_{i,n,k}\le t_{i,k},\qquad
t_{i,k}+r_{i,k}\le a_{i,k},\qquad
0\le p_{i,n,k}\le x_{i,n,k}P_{i,n,k}^{\max},\qquad
\sum_n p_{i,n,k}\le P_i^{\max}.
\]

Also enforce configured TDD/band availability and switching guards. A
selected observation of transmitter \(i\) by receiver \(j\) must have both
the corresponding transmission and \(r_{j,k}=1\), plus an available reference
and sufficient echo quality. Several receivers can select that same
transmission: their observations do not each consume a new downlink RE.

If choosing network-wide orthogonal illumination as a baseline, impose
\(\sum_i x_{i,n,k}\le1\) within that coordination domain. This is a deliberate
model restriction, **not** a universal ISAC requirement. Spatial reuse or
joint transmission needs a different interference/separability model.

A horizon energy objective can be written as

\[
\min E_{\mathrm{total}}=
\sum_k\Delta t_k\sum_i
\left(\frac{\sum_n p_{i,n,k}}{\eta_i}
+P^{\mathrm{op}}_{i,k}\right)
+E_{\mathrm{switch}}+E_{\mathrm{report/transport}},
\]

where \(\Delta t_k\) is the modeled symbol duration, RF powers sum across
simultaneous subcarriers, and \(P^{\mathrm{op}}_{i,k}\) includes mode-dependent
receiver/baseband/idle power. Specify accounting boundaries so that reporting
RF energy or transport power is not counted twice.

Constrain this objective by communication QoS/deadlines, node energy
budgets, per-node power limits, feasible Tx/Rx modes, sensing accuracy and
observability, revisit/output-age limits, ambiguity sidelobes, and
reporting/compute capacity. For example, a horizon transport constraint
\(\sum_\ell b_\ell\le C_{\mathrm{transport}}H\) is only a capacity check;
individual report deadlines additionally require a queue/service model.
Here \(b_\ell\) is report size in bits, \(C_{\mathrm{transport}}\) is bit/s,
and \(H\) is horizon duration in seconds.

Compute information from the actual selected observations and powers.
Additivity requires appropriate conditional independence or a joint
noise/correlation model; separate receiver reports sharing a reference can
have correlated errors. For tracking, combine the dynamic prior and new
measurements in an appropriate filtering/posterior-information model rather
than treating a static CRB as the predicted track covariance.

### 12.4 Practical solution strategy and infeasibility handling

1. **Screen feasible links:** use band/TDD availability, reference recovery,
   visibility, calibration, receiver limits, and geometry. Predict target
   state and communication demand over a short horizon.
2. **Solve a fixed-schedule baseline:** hold beams, masks, and active nodes
   fixed; optimize power under communication and sensing requirements.
   Record solver feasibility and constraint residuals.
3. **Improve discrete choices:** exchange candidate REs or nodes using
   geometry/information gain per incremental energy, while retaining
   Doppler sampling, ambiguity, and deadline constraints. Recompute the
   continuous power solution after each accepted change.
4. **Scale deliberately:** use mixed-integer optimization on small instances
   as a reference. Consider relaxation, alternating optimization, or
   successive approximation for larger cases; verify feasibility after
   rounding and do not claim global optimality without a proof.
5. **Execute in receding horizon:** freeze mandatory/imminent allocations,
   include coordination and solver runtime, then update from actual
   detections, channel estimates, queues, and track uncertainty.

If requirements cannot be met, explicitly report which constraints are
infeasible or violated. Distinguish proven infeasibility from solver timeout
or failure to find a feasible point. Do not silently relax communication
protection or issue an apparently valid track. Any fallback, such as a
longer revisit interval, reduced coverage, or dropped low-priority sensing
task, requires an explicit application policy and degraded-status output.

### 12.5 Validation additions to the experiment matrix

These experiments extend section 8.4; they are proposed comparisons rather
than published performance claims.

| Comparison or stress case | What to hold constant or measure |
|---|---|
| Equal power/all eligible nodes vs optimized powers vs joint activation | Use the same task requirements and traffic; report total electrical energy, not only RF power. |
| Uniform/interleaved vs compact vs fragmented/adaptive masks | Compare equal occupied RE count and energy; measure delay/Doppler ambiguity, actual RMSE, false peaks, and CRB. |
| Communication-only payload reuse vs optional sensing-only REs | Track incremental reservation and actual communication goodput/deadline effects separately. |
| Fixed TDD vs optimized feasible Tx/Rx modes | Include guard time, switching cost, cross-link interference, and unavailable observations. |
| Static accuracy objective vs freshness-aware tracking | Measure output-epoch position/velocity error, revisit gaps, measurement age, and track loss under motion. |
| Raw-data vs parameter/track reporting | Measure transport bits, queue delay, processing time, information loss, and correlation handling. |
| Ideal knowledge vs stale CSI, drift, blockage, and uncertain target RCS | Test sensitivity, outages, robust margins, and truthful infeasibility/degraded-status reporting. |
| Small exact reference vs scalable heuristic | Report runtime, feasible-solution rate, constraint residuals, and objective gap where an optimum/bound is available. |

Keep detection probability at a declared false-alarm rate, localization and
velocity error, communication service, and energy on the same comparison
plots or Pareto analysis. Evaluate independent trials and confidence
intervals; no single SNR, CRB, or energy percentage establishes overall
superiority.

## 13. Research references for the power and scheduling update

The following links identify the investigated literature. Preprints are
research proposals/results, not standards or field-deployment guarantees.
The descriptions above deliberately avoid unverified numerical savings and
do not assume that one paper covers all constraints in the proposed model.

- **[P1]** Zinat Behdad et al., "Multi-Static Target Detection and Power
  Allocation for Integrated Sensing and Communication in Cell-Free Massive
  MIMO," IEEE Transactions on Wireless Communications, 2024; preprint
  initially posted in 2023:
  <https://arxiv.org/abs/2305.12523>.
- **[P2]** Umut Demirhan and Ahmed Alkhateeb, "Cell-Free ISAC MIMO Systems:
  Joint Sensing and Communication Beamforming," 2023 preprint:
  <https://arxiv.org/abs/2301.11328>.
- **[P3]** "Joint Access Point Activation and Power Allocation for Cell-Free
  Massive MIMO Aided ISAC Systems," 2025 preprint:
  <https://arxiv.org/abs/2507.09425>.
- **[S1]** Reiner Thoma et al., "Distributed multisensor ISAC," 2026:
  <https://doi.org/10.1038/s44459-026-00041-2>; preprint:
  <https://arxiv.org/abs/2511.13104>. This is the source discussed in
  sections 1-10, not a newly supplied scheduler implementation.
- **[S2]** "Resource Allocation for Dynamic TDD-Enabled Integrated Sensing
  and Communication Systems," IEEE Transactions on Vehicular Technology,
  2025: <https://ieeexplore.ieee.org/document/10884055>.
  Joint power/subcarrier allocation and AP-mode selection under sensing
  CRLB and transmit-power constraints; alternating optimization is a
  published solution direction.
- **[S3]** Silvia Mura et al., "Waveform Design for OFDM-based ISAC Systems
  Under Resource Occupancy Constraint," 2023 preprint:
  <https://arxiv.org/abs/2312.12298>. Relevant to limited occupancy,
  delay/Doppler CRB, and recovery of undersampled sensing channels.
- **[S4]** Shoushuo Zhang et al., "Cooperative OFDM-ISAC Networks: Performance
  Analysis and Resource Allocation," 2026 preprint:
  <https://arxiv.org/abs/2604.26552>. Studies cooperative fusion and joint
  RE/power allocation with per-BS budgets, communication requirements,
  and delay-Doppler ambiguity-sidelobe constraints.
