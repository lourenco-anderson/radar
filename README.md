# Passive Radar Signal Processing Lab

A progressive Python project exploring how a reference signal can be used to estimate the delay, amplitude, phase and Doppler of simulated echoes.

The project builds a simplified reference–surveillance receiver, starting with a single ideal echo and progressing through complex noise, multiple targets, interference cancellation and delay–Doppler processing. The emphasis is on deriving the processing steps, implementing them and checking their behavior through controlled experiments.

**Current status:** notebook-based simulations using a synthetic binary reference. Delay–Doppler estimation has been implemented for one echo; experiments with two echoes sharing the same delay and having different Dopplers are in progress. An OFDM illuminator is planned.

## Repository guide

| Notebook | Topic |
| --- | --- |
| `1-Basics.ipynb` | Single-echo model, cross-correlation and estimation of delay, amplitude and phase. |
| `2-Noise.ipynb` | Complex Gaussian noise, SNR control and Monte Carlo evaluation of delay estimation. |
| `3-Two_targets.ipynb` | Two delayed echoes, local peak selection, sidelobe interference and successive cancellation. |
| `4-Doppler.ipynb` | Time-varying phase, Doppler compensation, frequency search and delay–Doppler maps. |

Follow the numbered notebooks in order for the main learning path. A fifth notebook on two echoes with equal delay and distinct Dopplers will be added once its experiments are complete.

## Signal model

Let $R(t)$ denote the reference-channel signal. A single surveillance-channel echo is modeled as

$$
V(t)=\alpha e^{i\phi}R(t-e),
$$

where $\alpha\geq0$ is the amplitude ratio, $\phi$ is the relative phase and $e$ is the delay relative to the reference.

The cross-correlation is defined using the convention

$$
C(\tau)=\int_{-\infty}^{+\infty}V(t)R^*(t-\tau)\,dt.
$$

For one complete, noiseless echo, at the correct alignment,

$$
C(e)=\alpha e^{i\phi}E_R.
$$

In the discrete implementation, the reference energy and estimators are

$$
E_R=\sum_k|R[k]|^2,
\qquad
\hat\alpha=\frac{|C[\hat d]|}{E_R},
\qquad
\hat\phi=\arg C[\hat d],
$$

where $C[\hat d]$ denotes the correlation value at the estimated lag. The discrete delay $\hat d$ is selected by maximizing the correlation magnitude, and the delay in seconds is $\hat e=\hat d/f_s$.

With `np.correlate(V, R, mode="full")`, the array starts at lag `-(len(R) - 1)`. Therefore, the peak's array index must be converted to a lag:

```python
C = np.correlate(V, R, mode="full")
peak_idx = np.argmax(np.abs(C))
delay_samples = peak_idx - (len(R) - 1)
```

The initial reference is a finite pseudo-random sequence with samples in $\{-1,+1\}$. Delayed echoes are constructed with zero padding rather than circular wrapping, preserving the complete waveform.

## Experiments implemented so far

### Single echo

The initial experiments recover a known delay, amplitude and phase. They also check that inverting the echo does not change the delay estimate when the correlation magnitude is used.

A representative noiseless configuration uses $N=128$, $d=20$ samples, $\alpha=0.2$ and $\phi=\pi/3$. For unit-magnitude reference samples, the expected correlation peak magnitude is $\alpha N$.

### Noise and statistical evaluation

Complex Gaussian noise is added to the surveillance channel while the reference remains ideal:

$$
V_{\mathrm{noisy}}[k]=V[k]+w[k],
\qquad
w[k]=\sigma(x[k]+iy[k]),
$$

where $x$ and $y$ are independent standard Gaussian sequences. For a desired noise power $P_w$,

$$
\sigma=\sqrt{P_w/2},
\qquad
P_w=\frac{P_{\mathrm{signal}}}{10^{\mathrm{SNR}_{\mathrm{dB}}/10}}.
$$

The current experiments define signal power over the full padded observation vector, including its initial zeros. This convention should be retained when interpreting or comparing SNR values.

Repeated independent noise realizations are used to measure:

- Exact delay error rate: the fraction of trials with $\hat d\neq d$.
- Mean absolute error: $\frac{1}{M}\sum_{j=1}^{M}|\hat d_j-d|$, in samples.
- Histograms of absolute delay errors.

The experiments illustrate occasional selection of distant correlation peaks at low SNR and the benefit of coherent integration with longer references. A finite experiment with zero observed errors does not establish a zero error probability. Exact percentages depend on the reference and noise realizations.

### Two echoes and successive cancellation

Two stationary echoes are modeled as

$$
V(t)=\alpha_1e^{i\phi_1}R(t-e_1)
+\alpha_2e^{i\phi_2}R(t-e_2).
$$

Their correlation patterns add, including sidelobes. This allows a strong echo to mask a weaker one and contaminates amplitude/phase estimates even without noise.

The current processing identifies local maxima above a manually selected magnitude threshold. It then reconstructs the dominant echo from estimated parameters, subtracts it and correlates the residual to reveal the weaker echo. Reestimating the first echo after subtracting the second improves the estimates in the tested scenario.

The threshold is exploratory: it has not been calibrated to a false-alarm probability. The cancellation experiment is currently noiseless, and imperfect reconstruction leaves residual interference.

### Doppler and delay–Doppler processing

A constant Doppler is represented by a linear phase evolution:

$$
\phi(t)=\phi_0+2\pi f_Dt,
\qquad
V(t)=\alpha e^{i(\phi_0+2\pi f_Dt)}R(t-e).
$$

For each candidate frequency $f$, the surveillance signal is derotated and correlated with the reference:

$$
\mathcal C(\tau,f)
=\int V(t)e^{-i2\pi ft}R^*(t-\tau)\,dt.
$$

The largest magnitude supplies a joint delay–Doppler estimate. Storing the complete complex correlation for every candidate produces a matrix with Doppler candidates as rows and delay lags as columns.

Experiments include Doppler compensation, 2D magnitude maps and frequency cuts through the target delay. Increasing the reference from 256 to 2000 samples at $f_s=1\,\mathrm{MHz}$ increases the active observation duration from $256\,\mu\mathrm{s}$ to $2\,\mathrm{ms}$ and narrows the Doppler response.

## Geometry and resolution

For fixed transmitter and receiver positions, let $L_0$ be their direct separation, $L_1$ the transmitter–target distance and $L_2$ the target–receiver distance. The relative delay satisfies

$$
ce=L_1+L_2-L_0.
$$

Thus, delay measures **excess path length**. In a plane, a positive fixed excess path defines an ellipse with transmitter and receiver as its foci; it does not provide a unique target position.

| Quantity | Approximate scale | Interpretation |
| --- | --- | --- |
| Delay grid spacing | $1/f_s$ | Spacing between tested integer delays. |
| Excess-path grid spacing | $c/f_s$ | Path-length increment corresponding to one sample. |
| Delay resolution | $1/B$ | Typical scale for separating echoes in delay. |
| Excess-path resolution | $c/B$ | Resolution in the sum-path excess, rather than target position. |
| Doppler resolution | $1/T$ | Typical scale for separating nearby Dopplers. |

Refining a search grid or interpolating does not add observation information. For the current constant-magnitude reference with a rectangular active interval, the frequency cut at the true delay has its first nulls at approximately $f_D\pm1/T$: its null-to-null main-lobe width is $2/T$.

## Running the notebooks

The examples use Python, NumPy, Matplotlib and Jupyter. From the repository root, a minimal environment can be created with:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m jupyter lab
```

Open a numbered notebook and run its cells from the beginning. Restarting the kernel and running all cells helps avoid stale variables from previous experiments. Random experiments can produce different outputs on successive runs; dependency pinning and consistent random-seed handling are planned.

## Current scope and limitations

- Synthetic baseband simulation using a binary reference; no OFDM, DVB-T or DAB waveform has been implemented yet.
- Integer-sample delays, ideal reference reception and coherent channels.
- Doppler modeled as a phase rotation with fixed envelope delay during observation; no range migration or full moving-target propagation model.
- No real RF recordings, hardware impairments or calibrated radar link budget.
- Local peak thresholds and cancellation demonstrations do not yet establish calibrated detection or false-alarm performance.
- Two echoes with identical delay and Doppler reduce to one equivalent complex coefficient in this model.

## Roadmap

- [ ] Complete and validate two-echo experiments with equal delay and distinct Dopplers.
- [ ] Compare resolvable and closely spaced Dopplers across observation durations, amplitudes and relative phases.
- [ ] Evaluate joint delay–Doppler estimates under noise, including Dopplers between grid points.
- [ ] Replace the binary reference with a configurable OFDM illuminator and examine its ambiguity response.
- [ ] Add strong direct-path leakage and static clutter; evaluate cancellation and its effect on weak echoes.
- [ ] Introduce detection thresholds with measured false-alarm and detection probabilities; explore CFAR.
- [ ] Add fractional delays and clarify the relationship between waveform bandwidth, sampling and resolution.
- [ ] Refactor reusable simulation/estimation functions, fix random seeds where appropriate and add dependency specifications and regression checks.
- [ ] Export representative figures and consolidate the technical results for portfolio presentation.

Possible later extensions include standards-specific broadcast illuminators, recorded SDR data, array processing and tracking. These are outside the currently implemented scope.
