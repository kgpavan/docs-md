# Operational Guide: Futures Accounting & Reconciliation (OTE vs. VM Models)

## 📌 Executive Summary
This document summarizes the technical and operational mechanics of **Open Trade Equity (OTE)** and **Variation Margin (VM)** accounting models within a Futures Commission Merchant (FCM) back-office architecture, focusing on the differences between **CME** and **Eurex** networks and processing via the **FIS GMI** ledger platform.

---

## 🔎 Core Framework: OTE vs. VM Accounting

| Operational Element | 🇺🇸 Open Trade Equity (OTE) Model | 🇪🇺 Variation Margin (VM) Model |
| :--- | :--- | :--- |
| **Primary Exchanges** | **CME Group** (CME, CBOT, NYMEX, COMEX) | **Eurex**, ICE Futures Europe |
| **Regulatory Jurisdiction** | CFTC / US Customer Segregation Rules | European / Asian Exchange Frameworks |
| **Accounting Classification** | Floating / Unrealized Equity | Realized Daily Cash Settlement |
| **Position Cost Basis** | Stays at **Original Execution Price** until closed | **Resets Nightly** to the closing settlement price |
| **Daily Sub-Ledger Effect** | Impacts Net Liquidating Value (NLV), cash is static | Physically debits/credits the Cash Ledger Balance |

---

## ➡️ Exchange Mechanics: CME vs. Eurex

### 🇺🇸 CME Group (OTE Model)
* **Clearing House Level:** CME Clearing executes daily Mark-to-Market (MTM) calculations and moves cash physically ("P&L sweeps") between the clearing banks of member FCMs. 
* **FCM Sub-Ledger Level:** CFTC regulations (**17 CFR § 1.32 & § 1.34**) legally bar FCMs from mixing floating customer profits directly into the settled cash ledger. Profits remain classified as OTE.
* **The Operational Challenge:** The clearing house has moved physical cash at the bank level, but the individual customer ledger cash balance (`LEDBAL`) remains static until trade liquidation.

### 🇪🇺 Eurex (VM Model)
* **Clearing House Level:** Eurex physically moves cash daily across clearing accounts based on end-of-session settlement prices.
* **FCM Sub-Ledger Level:** Instead of maintaining floating equity, the daily profit or loss is directly integrated into the client's cash ledger balance. The position is systematically closed out and re-opened at the new settlement price each morning.

---

## 📊 Back-Office Processing: The FIS GMI Infrastructure

FIS GMI utilizes distinct automated modules and table arrays depending on the exchange model flag:

### 1. The CME Point Balance (`PTBAL`) Function
To bridge the regulatory gap between banking cash sweeps and static customer customer cash accounts, GMI runs the **Point Balance (PTBAL)** valuation routine during the End-of-Day (EOD) cycle:
$$	ext{Point Balance} = (	ext{Settlement Price} - 	ext{Original Trade Price}) 	imes 	ext{Multiplier} 	imes 	ext{Quantity}$$
* GMI updates the **`PTBAL`** data bucket on the customer master record.
* It explicitly **does not** post a ledger card entry (`LG`) to the cash database.
* **Reconciliation Formula:** 
$$	ext{Total Bank Cash} = 	ext{Total Customer Ledger Cash (LEDBAL)} \pm 	ext{Total Point Balance (PTBAL)}$$

### 2. The Eurex VM Processing Logic
* GMI calculates the daily variation margin using the Eurex settlement price.
* The position file updates, **wiping out the historical entry price** and resetting the cost basis to the current settlement price.
* GMI generates an automated cash ledger entry—typically a **Non-Cash/Variation Margin ledger entry (`NC`)** or a direct hit to **`LEDBAL`**. 
* The **`PTBAL`** field for the Eurex position is reset to **zero**. Cash reconciliation is achieved directly via cash-to-cash balancing.

---

## ⚠️ Key Risks and Breaks in Cash Reconciliation

Operational teams using GMI must track three primary structural risk vectors:
1. **Commission Discrepancies:** Conflicts between round-turn commission schedules applied in GMI vs. daily exchange netting structures.
2. **FX Conversion Timing:** Variances between GMI's internal exchange rate tables applied during the EOD processing cycle and the actual spot rates locked during the CME clearing house sweep.
3. **Margin Data Integrity:** Layout mismatches between locally updated risk arrays/spread profiles and newly imported CME SPAN parameters.
