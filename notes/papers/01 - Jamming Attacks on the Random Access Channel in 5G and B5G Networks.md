# Note :  Jamming Attacks on the Random Access Channel in 5G and B5G Networks

**Status:** In progress

---

## 1. The Core Vulnerability: Why Target the RACH?
In 5G networks, before an UE can do anything, it must knock on the gNB's door using RACH. This first knock is called **Msg1 (Preamble)**. 
The problem is that Msg1 is contention-based and completely unprotected. The gNB cannot tell if a preamble comes from a legitimate user or a malicious jammer. This research asks a critical question: *What happens if an attacker exploits this open door not just by shouting, but by systematically poisoning the gNB's ability to hear?*

## 2. The Attacker's Playbook: Theory meets OAI Implementation
To prove this vulnerability, our lab didn't just run math simulations; we built a weaponized UE using OpenAirInterface (OAI) and a USRP B210. Here is how the attacker breaks the system, layer by layer:

* **TPHY Layer:** Normally, a UE sends one preamble. By diving into OAI's `nr_prach.c`, the attacker was modified to generate and accumulate 3 different Zadoff-Chu sequences (M=3) on the same resource block. This artificially triples the collision probability (from 1.56% to 4.69%).
* **Carpet Bombing the Frequencies (MAC Layer):** In `nr_ra_procedures.c`, the attacker identifies all available Frequency Domain (FD) occasions and fires its preambles across *all* of them simultaneously. There is no safe frequency left for a normal UE.
* **Relentless Persistence (Scheduler):** A normal UE waits for a response (Msg2). Our attacker is deaf by design. By commenting out the RAR handlers in `NR_IF_Module.c` and locking the scheduler in a continuous `nrRA_GENERATE_PREAMBLE` state, it blasts the gNB every single frame without pausing.

## 3. The Invisible Damage: Poisoning the Noise Threshold
When the gNB is constantly hit by these fake preambles, it thinks the environment has become incredibly noisy. To adapt, the gNB raises its "Noise Threshold" (p_th,i) to filter out the noise.

The paper mathematically quantifies this gNB adaptation process:
> `p_th,i = (alpha * p_measured,i) + (beta * p_measured,i-1) + (gamma * p_th,i-1)`

* **The Trap:** Because OAI updates this threshold recursively (using past memory), a persistent attacker (firing every frame, T_a = 1) forces the threshold to grow geometrically. 
* **The Result:** The threshold becomes so high that a legitimate UE's signal (p_UE) can no longer cross the required margin (p_th,i + delta). The legitimate UE is effectively silenced. Access probability drops to 0%.

## 4. Real-World Proof & Key Takeaways
When we deployed this on an actual OAI 5G standalone network (Band n78, 40MHz), the results perfectly mirrored the math:
1.  **Periodicity is Lethal:** A continuous attack (T_a = 1) kills the network by the 13th RACH occasion. However, if the attacker is lazy and attacks every 16 occasions (T_a = 16), the threshold decays, and normal UEs survive.
2.  **Parameter Tuning is Not a Cure:** We tried lowering the gNB's update factor (beta) or making the detection margin (delta) more forgiving. While this buys the normal UEs a little more time, it inevitably increases false alarms from actual background noise. You cannot simply "configure" your way out of this attack.

## 5. What's Next? The Defense Challenge
Richard's implementation successfully weaponized the OAI UE to prove the attack works. The next logical step for our lab is to build the shield.
Since tweaking simple threshold parameters fails, we need to design an **Adaptive Defense Mechanism** inside the gNB's MAC/PHY layer. The goal: create an algorithm that can identify the "fingerprint" of this multi-preamble jammer and isolate its signal before it poisons the threshold equation.
