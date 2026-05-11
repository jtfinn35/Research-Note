# Note :  Jamming Attacks on the Random Access Channel in 5G and B5G Networks

![Status](https://img.shields.io/badge/status-complete-brightgreen.svg)

---

## 1. Vulnerability Analysis: The Contention-Based Nature of RACH
In 5G networks, RACH serves as the primary uplink interface for UE to achieve initial synchronization and network access. The initial transmission, known as **Msg1 (Preamble)**, is entirely contention-based and lacks integrity protection. 
Consequently, the gNB cannot differentiate between a preamble transmitted by a legitimate UE and one injected by a malicious actor. This research explores the critical system degradation that occurs when an attacker exploits this open interface not merely to cause collisions, but to systematically manipulate the gNB's adaptive noise detection mechanisms.

## 2. The Attacker's Playbook: Theory meets OAI Implementation
This is how the attacker breaks the system layer by layer:

* **PHY Layer:** Normally, a UE sends one preamble. By diving into OAI's `nr_prach.c`, the attacker was modified to generate and accumulate 3 different Zadoff-Chu sequences (M=3) on the same resource block. This artificially triples the collision probability (from 1.56% to 4.69%).
* **Carpet Bombing the Frequencies (MAC Layer):** In `nr_ra_procedures.c`, the attacker identifies all available Frequency Domain (FD) occasions and fires its preambles across *all* of them simultaneously. There is no safe frequency left for a normal UE.
* **Relentless Persistence (Scheduler):** A normal UE waits for a response (Msg2). The attacker is deaf by design. By commenting out the RAR handlers in `NR_IF_Module.c` and locking the scheduler in a continuous `nrRA_GENERATE_PREAMBLE` state, it blasts the gNB every single frame without pausing.

## 3. The Invisible Damage: Poisoning the Noise Threshold
When the gNB is constantly hit by these fake preambles, it thinks the environment has become incredibly noisy. To adapt, the gNB raises its "Noise Threshold" (p_th,i) to filter out the noise.

The paper mathematically quantifies this gNB adaptation process:
> p_th,i = (alpha * p_measured,i) + (beta * p_measured,i-1) + (gamma * p_th,i-1)

To truly understand the attacker's impact, we must break down how these specific parameters dictate the gNB's behavior:
* **The Weight Configurations (alpha, beta, gamma):** Different open-source gNBs handle these weights differently. For example, srsRAN relies solely on instantaneous noise (`alpha=1, beta=0, gamma=0`), making it instantly vulnerable. 
  Conversely, **OAI uses an IIR (Infinite Impulse Response) filter approach**. It ignores the current instant (`alpha=0`) and relies heavily on history (`gamma = 1 - beta`). Thus, the OAI equation simplifies to:
  `p_th,i = beta * p_measured,i-1 + (1 - beta) * p_th,i-1`
* **The Trap:** Because OAI updates this threshold recursively, a persistent attacker (firing every frame, T_a = 1) forces the threshold to grow geometrically. 
* **The Result:** The threshold converges to a steady-state value so high that a legitimate UE's signal (p_UE) can no longer cross the required margin (`p_th,i + delta`). The legitimate UE is effectively silenced. Access probability drops to 0%.

## 4. Real-World Proof & The Trade-off Dilemma
When deploying this on an actual OAI 5G standalone network (Band n78, 40MHz), the results perfectly mirrored the math. It proved that we cannot simply "configure" our way out of this attack by tweaking parameters. Every defense adjustment introduces a fatal trade-off:

1. **Periodicity (T_a) is Lethal:** A continuous attack (T_a = 1) kills the network by the 13th RACH occasion. However, if the attacker is lazy and attacks every 16 occasions (T_a = 16), the threshold decays back to normal, and normal UEs survive.
2. **Tuning the Update Factor (beta):** * *The Attempt:* If we decrease `beta` (e.g., from default 0.12 down to 0.01), we make the gNB "stubborn." It remembers the old, clean threshold longer and resists the attacker's noise.
   * *The Trade-off:* This makes the gNB painfully slow to adapt to *real* environmental noise changes. Normal UEs might get dropped simply because the weather changed or physical obstacles moved.
3. **Tuning the Detection Margin (delta):**
   * *The Attempt:* If we lower `delta` (e.g., from 6dB to 2dB), we make the gNB more forgiving, allowing legitimate UEs to connect even if the noise threshold is artificially high.
   * *The Trade-off:* This significantly increases the **False Alarm Rate**. The gNB becomes overly sensitive and starts hallucinating—interpreting random thermal noise as preambles. It will waste massive MAC layer resources scheduling "phantom" Msg2 responses.

## 5. What's Next? The Defense Challenge
The implementation successfully weaponized the OAI UE to prove the attack works. The next logical step for our lab is to build the shield.
Since tweaking simple threshold parameters fails, we need to design an **Adaptive Defense Mechanism** inside the gNB's MAC/PHY layer. The goal: create an algorithm that can identify the "fingerprint" of this multi-preamble jammer and isolate its signal before it poisons the threshold equation.
