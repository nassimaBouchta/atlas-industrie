# Decision note - architecture of the Atlas Industrie campus

**Author:** [First Last]
**Repository:** atlas-industrie
**Submission tag:** `tp1-rendu`
**Horizon:** 5 years. All amounts in MAD, excluding tax. Teaching FX reference: 1 EUR = 10.8 MAD (used only for cross-reference; all figures are kept in MAD).
**Scope:** design only. No device configuration, no lab topology, no performance measurement.

---

## 1. Context and requirements

Atlas Industrie is a fictitious two-building campus used as the running case study of the "Advanced Network Architectures" course:

- **Building A** — head office and IT.
- **Building B** — production.

The campus hosts six trust zones: **Infrastructure**, **IT** (office A, office B, servers), **Voice**, **Guests**, **OT** (automates, supervision) and **DMZ**. Dual-stack IPv4/IPv6 is required; the unique local prefix drawn in session 1 is **`fd43:5ad7:b34e::/48`** and is kept for the eleven following sessions.

The stated requirements, numbered because section 5 refers to them:

- **R1 — Function.** Ten access switches (six in A, four in B) serve all zones. Inter-VLAN routing is provided. The DMZ is reachable from the internet only through the firewall pair.
- **R2 — Security.** Trust zones IT / Voice / Guests / OT / DMZ / Infrastructure are separated. A default-deny policy sits between IT and OT. VLAN 99 is the parking VLAN for unused, shut-down ports and carries no address.
- **R3 — Resilience.** **Production in Building B must continue if Building A is isolated.** This is the requirement that decides between the two collapsed-core candidates.
- **R4 — Addressability.** Dual-stack IPv4/IPv6, ULA `fd43:5ad7:b34e::/48`, no overlap, every VLAN inside its zone block.
- **R5 — Operability.** The IP plan is validated by `scripts/check_vlans.py`, run in CI or by a versioned pre-commit hook. The repository is cumulative across the eleven sessions.

**Out of scope:** device configuration, lab topology, ping tests (session 2).

---

## 2. Technical targets retained

The requirements above are turned into targets that section 5 can mark *satisfied / not satisfied*:

| Ref | Target | Verification |
|---|---|---|
| T1 | 10 access switches, 6 in A / 4 in B | `inventory/hosts.yml` |
| T2 | Firewall pair on the A↔B path; DMZ behind the pair | logical diagram + this note |
| T3 | **Production continues if Building A is isolated** | topology of S2 vs S3 |
| T4 | Dual-stack, ULA `fd43:5ad7:b34e::/48`, zero overlap | `check_vlans.py` exit 0 |
| T5 | IP plan automated and gated | CI green / hook refuses |
| T6 | Five-year TCO computed from stated assumptions | section 4 |
| T7 | Every common cause named | section 6 |

99.99 % availability is treated as an **aspiration**, not a target. It is **not** demonstrated by the parallel model — see section 6.

---

## 3. The two architectures compared

Three scenarios were costed; the **decision** is between the two collapsed-core options, **S2 vs S3**, and **S1 is the reference** against which both are measured.

- **S1 — Three tiers.** Two modular L3 cores, four L3 distribution switches, ten access switches. Full hierarchy; the distribution tier carries per-VLAN policy and summarisation.
- **S2 — Collapsed core, distributed between A and B.** One fixed 10/25G L3 core per building, ten access switches. The distribution tier disappears; the cores take over inter-VLAN routing and firewall hand-off. Two fibres on distinct routes between CC-A and CC-B.
- **S3 — Collapsed core, single site.** Both cores in Building A, ten access switches, single-site fibre. The cheapest option.

### 3.1 Scenario summary

| | S1 | S2 | S3 |
|---|---|---|---|
| Devices (incl. both firewalls) | **18** | **14** | **14** |
| CAPEX (MAD) | 1,155,600 | 766,800 | 745,200 |
| Annual support — 10 % of CAPEX (MAD) | 115,560 | 76,680 | 74,520 |
| Annual operations — N × 1.5 d × 5,400 MAD (MAD) | 145,800 | 113,400 | 113,400 |
| Annual recurring (MAD) | 261,360 | 190,080 | 187,920 |
| **TCO over 5 years (MAD)** | **2,462,400** | **1,717,200** | **1,684,800** |

### 3.2 The two gaps that carry the decision

- **S2 against S1.** 2,462,400 − 1,717,200 = **745,200 MAD saved over 5 years** (≈ 30.3 % of S1's TCO). S2 removes the distribution tier and replaces modular cores with fixed cores. What is lost is not resilience but **scalability**: the fixed cores cap future port density, and the collapsed core concentrates routing and firewall hand-off in one place per building.
- **S2 against S3.** 1,717,200 − 1,684,800 = **32,400 MAD over 5 years**. This is the *price of requirement R3/T3*, converted to time in section 5.

### 3.3 Resilience read across the three scenarios

| Failure | S1 | S2 | S3 |
|---|---|---|---|
| Loss of one core | survived (pair) | survived (pair) | survived (pair) |
| Loss of one firewall | survived (pair) | survived (pair) | survived (pair) |
| **Loss of Building A** | survived | **survived** (one core in B) | **not survived** (both cores in A) |

---

## 4. Five-year costing

### 4.1 Formulas

```
annual support    = 0.10 × CAPEX
annual operations = N_devices × 1.5 days × 5,400 MAD/day
TCO(5 years)      = CAPEX + 5 × (annual support + annual operations)
```

`N_devices` **includes both firewalls**, and it is **not the same** in the three scenarios (18 / 14 / 14).

### 4.2 Price assumptions

| Item | Cost (MAD, excl. tax) |
|---|---|
| 48-port PoE+ access switch | 27,000 each |
| High-end modular L3 core | 162,000 each |
| Fixed 10/25G L3 core | 86,400 each |
| L3 distribution switch | 54,000 each |
| Firewall pair | 216,000 (constant in all three scenarios) |
| Optics & fibre — three tiers | 129,600 |
| Optics & fibre — distributed core (second fibre on a distinct route) | 108,000 |
| Optics & fibre — single site | 86,400 |
| Vendor support | 10 % of CAPEX per year |
| Operations | 1.5 days per device per year at 5,400 MAD per day |
| Horizon | 5 years |
| Excluded | power, cooling, civil works (see section 7) |

### 4.3 S1 — Three tiers (18 devices)

| Line | Calculation | MAD |
|---|---|---|
| Cores | 2 × 162,000 | 324,000 |
| Distribution | 4 × 54,000 | 216,000 |
| Access | 10 × 27,000 | 270,000 |
| Firewall pair | 1 × 216,000 | 216,000 |
| Optics & fibre | three tiers | 129,600 |
| **CAPEX** | 324,000 + 216,000 + 270,000 + 216,000 + 129,600 | **1,155,600** |
| Annual support | 0.10 × 1,155,600 | 115,560 |
| Annual operations | 18 × 1.5 × 5,400 | 145,800 |
| Annual recurring | 115,560 + 145,800 | 261,360 |
| Five-year recurring | 5 × 261,360 | 1,306,800 |
| **TCO over 5 years** | 1,155,600 + 1,306,800 | **2,462,400** |

### 4.4 S2 — Collapsed core distributed A/B (14 devices)

| Line | Calculation | MAD |
|---|---|---|
| Cores | 2 × 86,400 | 172,800 |
| Access | 10 × 27,000 | 270,000 |
| Firewall pair | 1 × 216,000 | 216,000 |
| Optics & fibre | distributed core, second fibre on a distinct route | 108,000 |
| **CAPEX** | 172,800 + 270,000 + 216,000 + 108,000 | **766,800** |
| Annual support | 0.10 × 766,800 | 76,680 |
| Annual operations | 14 × 1.5 × 5,400 | 113,400 |
| Annual recurring | 76,680 + 113,400 | 190,080 |
| Five-year recurring | 5 × 190,080 | 950,400 |
| **TCO over 5 years** | 766,800 + 950,400 | **1,717,200** |

### 4.5 S3 — Collapsed core single site (14 devices)

| Line | Calculation | MAD |
|---|---|---|
| Cores | 2 × 86,400 | 172,800 |
| Access | 10 × 27,000 | 270,000 |
| Firewall pair | 1 × 216,000 | 216,000 |
| Optics & fibre | single site | 86,400 |
| **CAPEX** | 172,800 + 270,000 + 216,000 + 86,400 | **745,200** |
| Annual support | 0.10 × 745,200 | 74,520 |
| Annual operations | 14 × 1.5 × 5,400 | 113,400 |
| Annual recurring | 74,520 + 113,400 | 187,920 |
| Five-year recurring | 5 × 187,920 | 939,600 |
| **TCO over 5 years** | 745,200 + 939,600 | **1,684,800** |

### 4.6 Reproducibility

Every figure above is recomputable from the formulas of 4.1 and the price assumptions of 4.2 alone. No number depends on a quotation that is not stated.

---

## 5. Decision and justification

### 5.1 Recommendation

**Retain S2 — collapsed core distributed between Buildings A and B.**

### 5.2 Satisfaction against the targets of section 2

| Target | S1 | S2 | S3 |
|---|---|---|---|
| T1 — 10 access, 6 A / 4 B | satisfied | satisfied | satisfied |
| T2 — firewall pair and DMZ behind it | satisfied | satisfied | satisfied |
| **T3 — production survives isolation of A** | satisfied | **satisfied** | **not satisfied** |
| T4 — dual-stack, ULA, no overlap | satisfied | satisfied | satisfied |
| T5 — automated check | satisfied | satisfied | satisfied |
| T6 — TCO computed | satisfied | satisfied | satisfied |
| T7 — common causes named | satisfied | satisfied | satisfied |

### 5.3 Why S3 is not retained

S3 is the cheapest on paper — 1,684,800 MAD, i.e. **32,400 MAD less than S2 over five years**. It is not retained because it **fails T3**: both cores sit in Building A, so an isolation of Building A takes both cores down and production stops.

**S3 is therefore not an equivalent cheaper solution; it is a different solution.** It serves to **price the requirement**.

### 5.4 The gap converted to time

Assumption stated in section 7: **108,000 MAD per hour of production outage**.

```
32,400 MAD / 108,000 MAD per hour = 0.30 h = 18 minutes
```

**S2 costs 32,400 MAD more than S3 over five years — the equivalent of 18 minutes of avoided production downtime.** Any single isolation of Building A exceeding 18 minutes over the five-year horizon makes S2 the cheaper option in expected value. No production site can credibly guarantee that Building A will never be isolated for more than 18 minutes cumulatively over five years, so the requirement is retained.

### 5.5 Why S2 over S1

S2 saves **745,200 MAD over five years** relative to S1 — roughly 30 % of S1's TCO. The distribution tier is removed and its routing function moves into the fixed cores. This is a **scalability** trade-off, not a resilience trade-off: the fixed cores cap future port density. This limitation is stated here rather than hidden.

### 5.6 What would change the decision

The recommendation is conditional. It would be reversed if:

- the business formally drops requirement R3 / target T3 — then **S3** becomes justified as the lower-cost design; or
- the port-density projection in the cores exceeds the fixed-core capacity within five years — then **S1** becomes justified as the scalable design.

Naming the reversal conditions is what makes this note defensible.

---

## 6. Risks and points of failure

### 6.1 The arithmetic ceiling of the parallel model

Three devices in series at 99.9 % each:

```
0.999 × 0.999 × 0.999 = 0.997002999  ≈ 99.70 %
```

Three devices in series at 99.9 % give **≈ 99.7 %**, not 99.99 %. Doubling the core and the firewall **does not** raise the ceiling, because the **non-redundant access switch** sets it. The result stays **below** the 99.99 % aspiration.

To approach the target, what would have to be analysed beyond the parallel model: **power supplies** (dual PSU, separate UPS), **fibre routes** (modelled in the physical view), and **failover mechanisms** (protocol convergence, firewall HA state synchronisation).

### 6.2 Common causes (the parallel model assumes independent failures — these are not)

- **Same room.** Two cores in Building A (S3) share one fire, one flood, one HVAC failure.
- **Same software version.** Both cores and both firewalls on the same firmware: one bug takes the pair.
- **Same fibre route.** Even in S2, the two routes meet at the **cable chamber under the courtyard** (see `docs/schemas/campus-physique.mmd`). One digging incident can cut both.

### 6.3 Residual risk named in one sentence

> Both fibre routes share the cable chamber under the courtyard, so a single digging incident can isolate Building B regardless of the dual-core design; the S2 resilience claim is therefore conditional on protecting that chamber — separate duct, physical separation, or a third route.

### 6.4 Operational risks

- Single-vendor lock-in on cores and firewalls.
- The collapsed core concentrates routing and firewall hand-off: a misconfiguration in S2 has a wider blast radius than in S1.
- The fixed cores cap future port growth (see 5.5).

---

## 7. Assumptions and limits

### 7.1 Assumptions

- **Topology.** S2 places one core per building; S3 places both cores in Building A.
- **Constant firewall pair.** The firewall pair (216,000 MAD) is identical in all three scenarios and therefore does not differentiate the choice.
- **Recurring costs.** Vendor support at 10 % of CAPEX per year for 5 years; operations at 1.5 days per device per year at 5,400 MAD per day.
- **Horizon.** 5 years.
- **Outage cost.** 108,000 MAD per hour of production downtime (used only in 5.4).
- **Teaching FX.** 1 EUR = 10.8 MAD, used only for cross-reference; all amounts are kept in MAD.
- **Physical view.** The two fibre routes ("north technical gallery" and "south outer facade") and the junction ("cable chamber under the courtyard") are **assumed** — no real cabling plan was available. An assumed route declared as such is usable; one presented as fact is not.

### 7.2 Excluded (not priced anywhere in this note)

Power, cooling, civil works, network cards, in-building cabling, racks, installation labour, training, software licences, VAT.

### 7.3 What this note does not claim

It does **not** demonstrate 99.99 % availability. It prices a requirement (R3 / T3) and shows the arithmetic ceiling of the parallel model (6.1), which remains below the target.

### 7.4 Out of scope

Device configuration, lab topology, performance measurement.