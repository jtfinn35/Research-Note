# Technical Review & Implementation Report: RACH Jamming in 5G Networks

**Status:** Completed
**Reviewer:** J.T. Finn
**Research Area:** 5G/B5G Physical Layer Security
**Lab:** Broadband Mobile Wireless Lab (BMW Lab), NTUST

---

## 1. Paper Metadata & Implementation Scope
* **Reference Paper:** Jamming Attacks on the Random Access Channel in 5G and B5G Networks
* **Implementation Target:** OAI-based Msg1 Protocol-Aware Jammer (Richard's `multi_preamble3` branch).
* **Keywords:** RACH Jamming, OpenAirInterface (OAI), Noise Threshold Evolution, USRP B210.
* **Objective:** To document the theoretical foundation of Msg1 jamming and detail the specific source-code modifications required to implement a multi-preamble, multi-occasion attacker in OAI.

---

## 2. Motivation and Problem Definition
* **Background:** The Random Access Channel (RACH) is a contention-based uplink channel. Because Msg1 (Preamble) lacks integrity protection, it is highly vulnerable to Denial-of-Service (DoS) attacks.
* **Core Contribution:** The research provides a recursive analytical model predicting how the gNB's adaptive noise threshold evolves under attack. The accompanying implementation transforms a standard OAI UE into a persistent jammer capable of exhausting gNB resources.

---

## 3. Theoretical Framework: Recursive Threshold Model
The gNB updates its noise threshold (p_th,i) using a recursive weighted average of current and past power measurements:

p_th,i = (alpha * p_measured,i) + (beta * p_measured,i-1) + (gamma * p_th,i-1)

* **OAI Implementation Strategy:** Uses recursive averaging where alpha = 0 and gamma = 1 - beta.
* **Impact:** Under continuous attack, the threshold converges to a high steady-state value. This causes legitimate UEs' access attempts to fail because their signal power (p_UE) can no longer exceed the poisoned threshold plus the detection margin (p_th,i + delta).

---

## 4. Technical Implementation: The OAI Attacker
To validate the vulnerability, the standard OAI UE was modified to act as a malicious jammer.
* **Base Environment:** Ubuntu 24.04 LTS, UHD 4.8.0.0, USRP B210.
* **OAI Version:** develop branch (commit `4da30019be`, `multi_preamble3` branch).

### Core Source Code Modifications
1. **Multi-Preamble Generation (PHY Layer - `nr_prach.c`):**
   Modified the signal generation loop to accumulate 3 different Zadoff-Chu sequences (M=3) simultaneously on the same time-frequency resource.
2. **Multi-FD-Occasions (MAC Layer - `nr_ra_procedures.c`):**
   Stored all available Frequency Domain (FD) occasions and configured the UE to transmit across all of them simultaneously.
3. **Continuous Transmission (MAC Scheduler - `nr_ue_scheduler.c`):**
   Forced `is_prach_frame()` to always return true, keeping the UE permanently in the `nrRA_GENERATE_PREAMBLE` state.
4. **Msg2 Blocking (PHY-MAC Interface - `NR_IF_Module.c`):**
   Commented out the RAR and DLSCH handlers, ensuring the attacker ignores any gNB responses and continues jamming.

---

## 5. Collision Probability Analysis
By modifying the UE to transmit M=3 preambles simultaneously, the collision probability is artificially inflated:
* **Single Preamble (Standard):** 1 / 64 = 1.56%
* **Multi-Preamble (Attacker, M=3):** 3 / 64 = 4.69% (A 3x improvement in attack efficiency).
* **System Impact:** In a cell with 20 legitimate UEs, the presence of this attacker drives the collision probability up to approximately 61.98%, effectively paralyzing the uplink.

---

## 6. Strategic Insights and Future Directions
1. **Implementation Success:** Richard's modifications successfully map the theoretical multi-preamble attack into a functional OAI testbed.
2. **Defense Development (Next Steps):** Since simple parameter tuning (adjusting beta or delta) cannot fully mitigate the attack without increasing false alarms, future research must focus on the gNB side. We need to develop an **Adaptive Defense Mechanism** in the OAI gNB MAC/PHY layer capable of identifying and filtering out these accumulated, high-frequency malicious preambles.
