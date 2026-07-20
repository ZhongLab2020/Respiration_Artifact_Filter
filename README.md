# Respiration Artifact Filter

## Scientific Principle

### Purpose

Respiration can introduce transient image displacement, deformation, and broad structural changes into time-resolved in vivo imaging data. These artifacts may affect a large fraction of the field of view and produce abrupt frame-to-frame changes unrelated to the biological process being measured.

The Respiration Artifact Filter identifies respiration-associated transitions by combining:

1. motion measured directly from consecutive frames of the original TIFF stack; and
2. spatial change information measured from the corresponding SSIM or change-map stack.

Strong, spatially coordinated motion is used to establish primary respiratory detections. Weaker evidence is then used to recover the onset and relaxation frames connected to each confirmed event, while localized peristaltic motion is prevented from extending an event through map evidence alone.

---

## Input Data and Frame Correspondence

For an original image stack

$$
I_1,I_2,\ldots,I_N,
$$

the corresponding map stack contains $N-1$ frames:

$$
S_1,S_2,\ldots,S_{N-1}.
$$

Each map frame describes the transition between two consecutive original frames:

$$
S_t\longleftrightarrow (I_t,I_{t+1}).
$$

Therefore, map frame $t$ is marked as an artifact frame when the transition from original frame $t$ to original frame $t+1$ is classified as respiration-associated.

The program requires:

- a single-channel, multi-frame 8-bit or 16-bit unsigned grayscale original TIFF;
- a single-channel, multi-frame 32-bit or 64-bit floating-point map TIFF;
- identical image width and height for the two stacks; and
- exactly one fewer frame in the map stack than in the original stack.

---

## Supported Map Conventions

The program automatically distinguishes between two map conventions.

### Positive-change maps

Higher values indicate stronger frame-to-frame change. This category includes:

- noise-normalized Energy maps;
- noise-normalized Excess maps;
- $1-\mathrm{SSIM}$ maps; and
- other non-negative change maps.

### Similarity-oriented maps

Higher values indicate greater similarity, while respiratory disturbance produces lower values. Conventional SSIM maps belong to this category.

The direction is inferred from the relationship between map statistics and motion-rich transitions. When motion evidence is insufficient for a reliable comparison, the program uses the overall map range and baseline as a fallback.

Map values are compressed with an inverse-hyperbolic-sine transform before histogram percentile estimation:

$$
u=\mathrm{asinh}(S).
$$

This transformation remains approximately linear near zero while accommodating normalized Energy values above 1. It avoids clipping positive-change maps to the conventional SSIM interval of $[-1,1]$.

---

## Global Motion Estimation

For every consecutive original-frame pair, the program estimates global translational displacement by phase correlation.

After mean removal, variance normalization, and Hann-window weighting, the normalized cross-power spectrum is

$$
P_t=
\frac{
\mathcal{F}(I_t)\,\overline{\mathcal{F}(I_{t+1})}
}{
\lvert \mathcal{F}(I_t)\,\overline{\mathcal{F}(I_{t+1})} \rvert+\varepsilon
}.
$$

Here, the overline denotes the complex conjugate. The phase-correlation surface is

$$
C_t=\mathcal{F}^{-1}(P_t).
$$

The location of its maximum provides the horizontal and vertical displacement:

$$
\Delta x_t,\qquad \Delta y_t.
$$

A local quadratic interpolation around the maximum provides subpixel precision. The total displacement magnitude is

$$
d_t=\sqrt{\Delta x_t^2+\Delta y_t^2}.
$$

The correlation-peak confidence is estimated relative to the median absolute background of the phase-correlation surface. Weak or ambiguous peaks therefore contribute less evidence than distinct global shifts.

---

## Spatial Motion Coherence

Respiration usually affects multiple image regions in a coordinated manner. Each original frame is divided into a $4\times4$ grid, and a local displacement vector is estimated for every sufficiently structured block.

For the retained local vectors, directional coherence is

$$
c_t=
\frac{
\lVert \sum_j\mathbf{v}_{t,j} \rVert
}{
\sum_j \lVert \mathbf{v}_{t,j} \rVert
}.
$$

The moving-block fraction is

$$
f_t=
\frac{
N_{\mathrm{moving},t}
}{
N_{\mathrm{valid},t}
}.
$$

A broad respiratory transition is expected to show both displacement and coordinated movement across a substantial fraction of valid blocks.

### Conceptual motion-vector patterns

The local vectors can be represented schematically as an arrow grid. Each grid position corresponds to one image block:

- the arrow direction represents the estimated local displacement direction;
- repeated arrows indicate a larger relative displacement magnitude; and
- `·` indicates that no reliable motion vector exceeds the local quality and magnitude thresholds.

A respiration-associated transition typically affects most of the field of view and produces vectors with similar directions:

```text
Respiration: broad and directionally coherent motion

 ↓  ↓↓  ↓↓  ↓↓
 ↓  ↓↓  ↓↓  ↓↓
 ↓  ↓↓  ↓↓  ↓↓
 ↓   ↓  ↓↓   ↓
```

By contrast, ordinary peristalsis is usually spatially localized. Large local vectors may occur, but they are confined to a smaller number of blocks and may point in different directions:

```text
Peristalsis: localized and directionally heterogeneous motion

→→   →   ·   ·
 ↑  ↓↓   ←   ·
 ·   ↓   →→  ·
 ·   ·   ·   ·
```

These diagrams are conceptual rather than literal program outputs. The detector uses the numerical vector magnitudes, directional coherence, moving-block fraction, global displacement, and map coverage together. A local arrow pattern alone is not sufficient to classify a transition as respiration.

---

## Direction-Aware Map Descriptors

For every map frame, the program calculates the whole-image mean and two tail quantiles:

$$
\overline{S}_t=\frac{1}{M}\sum_{p=1}^{M}S_t(p),
$$

$$
Q_{10,t},\qquad Q_{90,t}.
$$

The direction-aware tail descriptor is

$$
Q_{\mathrm{tail},t}=
\begin{cases}
Q_{90,t}, & \text{positive-change map},\\
Q_{10,t}, & \text{similarity-oriented map}.
\end{cases}
$$

Stable frame pairs are selected from transitions with displacement no greater than the recording median and map means on the stable side of the recording median.

For positive-change maps, the pooled 98th percentile of stable pixels defines a high-value threshold. Let $N_{\mathrm{high},t}$ denote the number of finite pixels satisfying $S_t(p)>\tau$, and let $N_{\mathrm{finite},t}$ denote the total number of finite pixels in the frame. The affected-area fraction is

$$
a_t=\frac{N_{\mathrm{high},t}}{N_{\mathrm{finite},t}}.
$$

For similarity-oriented maps, the pooled 2nd percentile defines a low-value threshold. Let $N_{\mathrm{low},t}$ denote the number of finite pixels satisfying $S_t(p)<\tau$. The affected-area fraction is then

$$
a_t=\frac{N_{\mathrm{low},t}}{N_{\mathrm{finite},t}}.
$$

---

## Robust Baseline Estimation

For a temporal metric $x_t$, the robust center is

$$
m_x=\mathrm{median}(x_t),
$$

and the robust scale is

$$
s_x=
\max\bigl[
1.4826\,\mathrm{median}(\lvert x_t-m_x\rvert),
s_{\min}
\bigr].
$$

The directional positive deviation is

$$
z_x(t)=
\begin{cases}
\max\bigl[0,\dfrac{x_t-m_x}{s_x}\bigr], & \text{higher means stronger change},\\
\max\bigl[0,\dfrac{m_x-x_t}{s_x}\bigr], & \text{lower means stronger change}.
\end{cases}
$$

The map evidence for frame $t$ is

$$
z_{\mathrm{map}}(t)=
\max\bigl[
z_a(t),z_{\mathrm{mean}}(t),z_{\mathrm{tail}}(t)
\bigr].
$$

---

## Primary Artifact Detection

Primary respiratory detections require strong motion evidence. The decision logic includes:

- unusually large global displacement together with coherent movement across multiple image blocks;
- a displacement outlier accompanied by direction-consistent map disturbance;
- very large displacement supported by sufficiently coherent local motion; or
- spatially coherent non-rigid motion accompanied by significant map disturbance.

This seed requirement prevents isolated map fluctuations from creating standalone respiratory events.

---

## Spatially Constrained Event Completion

Respiratory motion often develops and relaxes over several adjacent transitions. The strongest central transitions define the primary event, while weaker onset and recovery frames require more conservative boundary logic.

For boundary analysis, each map frame is divided into a fixed $6\times6$ grid. Within each block, the transformed map mean is compared with the corresponding block baseline obtained from stable frames. A block is classified as abnormal when its direction-aware robust deviation exceeds the mode-specific block threshold.

The spatial map-coverage fraction is

$$
g_t=\frac{N_{\mathrm{abnormal},t}}{N_{\mathrm{valid},t}}.
$$

A potential event-body frame is evaluated using three evidence classes:

1. weak global displacement in the original images;
2. broad and directionally coherent motion across original-image blocks; and
3. broad abnormal-map coverage across the $6\times6$ grid.

At least two of the three classes must be present. Map evidence alone therefore cannot recursively extend a respiratory event through localized biological activity such as ordinary peristalsis.

After the evidence-supported event body has been established, the program may add at most one directly adjacent shoulder frame on each side. The shoulder must exceed the mode-specific map-coverage threshold and must not be stronger than the immediately adjacent confirmed body frame. A shoulder frame is never used as a new growth point.

This non-recursive design recovers a weak respiratory relaxation frame while preventing a chain of map-only frames from being absorbed into the event. Three-frame map smoothing and unsupported-frame gap bridging are not used in version 2.4.

---

## Detection Modes

### Primary detection parameters

| Mode | Displacement multiplier | Map-area multiplier | Map-value multiplier | Minimum displacement | Minimum coherence | Minimum moving fraction |
|---|---:|---:|---:|---:|---:|---:|
| Conservative | 5.0 | 5.0 | 4.5 | 0.65 px | 0.75 | 0.30 |
| Standard | 3.5 | 3.5 | 3.5 | 0.45 px | 0.65 | 0.20 |
| Sensitive | 2.5 | 2.5 | 2.5 | 0.30 px | 0.55 | 0.12 |

### Event-boundary parameters

| Mode | Boundary displacement multiplier | Boundary coherence | Boundary moving fraction | Block z threshold | Body map coverage | Shoulder map coverage |
|---|---:|---:|---:|---:|---:|---:|
| Conservative | 1.8 | 0.58 | 0.16 | 5.5 | 0.40 | 0.45 |
| Standard | 1.25 | 0.50 | 0.12 | 5.0 | 0.30 | 0.35 |
| Sensitive | 0.9 | 0.43 | 0.08 | 4.5 | 0.25 | 0.30 |

The interpretation is:

- **Conservative:** prioritizes specificity and retains more frames.
- **Standard:** balances sensitivity and specificity for routine analysis.
- **Sensitive:** includes weaker respiratory boundaries but may remove more genuine biological transitions.

The selected mode should be kept consistent when experimental groups are compared.

---

## Output Maps

The program produces only TIFF outputs. It does not create NPZ, CSV, or SVG sidecar files.

### Deleted-AF Map

The **Generate Deleted-AF Map** command removes all detected artifact frames.

```text
<original map filename>_Deleted-AF.tif
```

The resulting stack contains

$$
N_{\mathrm{output}}=N_{\mathrm{map}}-N_{\mathrm{AF}}
$$

frames.

### Replaced-AF Map

The **Generate Replaced-AF Map** command preserves the complete map sequence but replaces each detected artifact frame with a full-frame NaN image.

```text
<original map filename>_Replaced-AF.tif
```

The frame count remains unchanged:

$$
N_{\mathrm{output}}=N_{\mathrm{map}}.
$$

Both outputs are written as 32-bit floating-point, single-series, ImageJ-compatible TIFF stacks.

---

## Appropriate Use and Interpretation

The method is most suitable when:

- respiration produces broad, coordinated frame-to-frame motion;
- the original image stack and matched change-map stack are available;
- the map stack contains one frame for each consecutive original-frame pair;
- respiration artifacts are temporally sparse relative to the recording; and
- the acquisition geometry remains stable.

The detector identifies transitions consistent with respiration-associated motion. It does not directly measure breathing rate, respiratory phase, tissue velocity, or physiological severity.

Other broad coordinated disturbances, including animal movement, stage vibration, focus jumps, probe displacement, or mechanical shocks, may also be detected. Very weak respiration, motion confined to a small region, or intensity modulation without detectable displacement may remain difficult to classify.
