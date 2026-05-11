# Note: Jamming Attacks on the Random Access Channel in 5G and B5G Networks

**Topic:** Cybersecurity in 5G/B5G Physical Layer (RACH Jamming)  
**Source:** Jamming Attacks on the Random Access Channel in 5G and B5G Networks
**Authors:** Wilfrid Azariah, Yi-Quan Chen, Zhong-Xin You, Ray-Guang Cheng, Shiann-Tsong Sheu, Binbin Chen  
**Key Technology:** OpenAirInterface (OAI), USRP B210

---

## 1. Short Summary
This paper develops an analytical recursive model to predict how Msg1 jamming attacks evolve the noise threshold at the gNB, subsequently validated through over-the-air experiments using an OAI-based 5G testbed. The study demonstrates that periodic, protocol-aware jamming can effectively deny service to legitimate UEs by manipulating the gNB's adaptive detection threshold.

---

## 2. The Five Cs
* **Category:** Experimental Security Analysis / Physical Layer Jamming.
* **Context:** Focuses on the Random Access Channel (RACH), a critical uplink for initial network access and synchronization.
* **Correctness:** Validated by comparing a mathematical model against real-world measurements on an OAI 5G standalone testbed.
* **Contributions:** * A recursive analytical model for noise threshold evolution.
    * Implementation of a stealthy, protocol-aware jammer using USRP and OAI.
    * Quantitative analysis of the impact of attacker periodicity ($T_a$) on UE access success.
* **Clarity:** The paper provides clear mapping between mathematical variables ($\alpha, \beta, \gamma$) and actual protocol implementations like OAI and srsRAN.

---

## 3. Key Theoretical & Experimental Content

### A. Analytical Model (Threshold Evolution)
The gNB updates its noise threshold ($p_{th,i}$) based on a recursive weighting of current and past power measurements:
* **Recursive Formula:** $p_{th,i} = \alpha \cdot p_{measured,i} + \beta \cdot p_{measured,i-1} + \gamma \cdot p_{th,i-1}$.
* **OAI Implementation:** Specifically uses recursive averaging where $\alpha=0$ and $\gamma=1-\beta$.
* **Access Condition:** A UE succeeds only if its received power ($p_{UE}$) exceeds $(p_{th,i} + \delta)$, where $\delta$ is the detection margin.

### B. Experimental Setup (Fig. 4)
* **Hardware:** Intel NUC mini-PCs paired with USRP B210 devices.
* **Environment:** 5G NR FR1 (Band n78, 3619.20 MHz, 30 kHz SCS).
* **Branch/Version:** OpenAirInterface develop branch (commit `82fb9fcc`).

### C. Major Results
* **Attacker Periodicity ($T_a$):** A continuous flooding jammer ($T_a=1$) causes the threshold to rise rapidly, blocking access around the 13th RO (Fig. 5 & 6). Moderate ($T_a=2$) or stealthy ($T_a=16$) attacks result in slower or minimal threshold increase.
* **Noise Update Factor ($\beta$):** Smaller $\beta$ values slow down threshold updates, potentially allowing UEs to succeed longer, but hinder adaptation to real environment noise.
* **Detection Margin ($\delta$):** Lowering $\delta$ increases success probability but significantly raises the risk of false alarms.

---

## 4. Critical and Creative Analysis

### Critical Reflections
* **Attacker Capability:** The model assumes the attacker has synchronized with the cell and read SIB1 to identify RO allocations. 
* **Protocol Vulnerability:** The vulnerability stems from the contention-based and unprotected nature of the Msg1 preamble.
* **Realism:** While UEs could use power ramping, they are ultimately limited by maximum PRACH transmit power, leaving them vulnerable to high-power jammers.

### Creative Extensions & Future Work
* **Advanced Jamming:** Implementing attackers that can generate multiple preambles across multiple ROs in the frequency domain simultaneously.
* **Adaptive Defense:** Designing gNB mechanisms that can distinguish between high-power noise and legitimate preambles to prevent threshold poisoning.
* **Multi-UE Impact:** Evaluating the model in high-density scenarios where collisions and jamming interact.

---

## 5. Technical Parameters Reference (Table I & II)
* **Background Noise ($P_{noise}$):** 17.4 dB.
* **Attacker Power ($P_{attacker}$):** 51 dB.
* **UE Signal Power ($P_{UE}$):** 56.4 dB.

---

## 6. Reference
* [Jamming Attacks on the Random Access Channel in 5G and B5G Networks](https://arxiv.org/abs/2602.06634).
