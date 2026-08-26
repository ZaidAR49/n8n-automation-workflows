# 🌍 Utility Bill Carbon Calculator (CarbonFlow)

An AI-powered ESG & Carbon Accounting automation workflow built in **n8n**. It ingests utility bills, fuel receipts, and water invoices (PDFs/images), uses **Google Gemini (via Vertex AI)** to extract consumption metrics and monetary amounts, normalizes physical units, and calculates precise greenhouse gas emissions ($\text{CO}_2\text{e}$) via the **Climatiq API**.

[![n8n](https://img.shields.io/badge/n8n-Workflow-orange.svg)](https://n8n.io/)
[![Google Gemini](https://img.shields.io/badge/AI-Gemini%203.7%20Flash-4285F4.svg)](https://cloud.google.com/vertex-ai)
[![Climatiq API](https://img.shields.io/badge/API-Climatiq%20v1-00B074.svg)](https://www.climatiq.io/)
[![GHG Protocol](https://img.shields.io/badge/ESG-Scope%201%20%7C%202%20%7C%203-blue.svg)](https://ghgprotocol.org/)

---

## 📋 Table of Contents

- [Overview](#-overview)
- [GHG Protocol Scope Coverage](#-ghg-protocol-scope-coverage)
- [Workflow Architecture](#️-workflow-architecture)
- [Workflow Nodes Breakdown](#-workflow-nodes-breakdown)
- [Unit Normalization & Climatiq Factor Mapping](#-unit-normalization--climatiq-factor-mapping)
- [Webhook API Specification](#-webhook-api-specification)
  - [Input Request](#1-input-request)
  - [Output Response](#2-output-response)
- [Prerequisites & Credentials](#-prerequisites--credentials)
- [Setup & Deployment](#-setup--deployment)
- [Frontend Dashboard Integration](#-frontend-dashboard-integration)
- [Edge Cases & Error Handling](#-edge-cases--error-handling)
- [Tech Stack](#-tech-stack)

---

## 🔍 Overview

Manual carbon accounting from utility bills is slow, error-prone, and inconsistent due to varying supplier formats, mixed utilities (e.g., combined water & electricity bills like DEWA), and differing measurement units.

This workflow provides a fully automated end-to-end ingestion and calculation pipeline:
1. **Document Ingestion**: Receives invoice/receipt scans or PDFs via a REST Webhook.
2. **Multimodal AI Extraction**: Employs **Gemini 3.7 Flash** with structured output parsing to audit and extract vendor, consumption values, units, costs, and currencies.
3. **Unit Normalization**: Translates arbitrary localized units (`kWh`, `IG`, `m³`, `Liters`, `Gallons`, `Barrels`) into standardized Climatiq-compatible parameters.
4. **Carbon Calculation**: Computes verified $\text{CO}_2\text{e}$ emissions against global emission factor databases using the Climatiq API.
5. **Clean Response**: Merges calculation metrics with metadata and delivers a clean JSON payload back to the caller in real-time.

---

## 🌐 GHG Protocol Scope Coverage

| Utility Type | Target GHG Scope | Common Identifiers / Keywords | Activity & Factor Type |
|---|---|---|---|
| ⚡ **Electricity** | **Scope 2** *(Indirect Emissions)* | `Electricity`, `Consumption`, `kWh`, `MWh` | Residual grid electricity mix |
| 💧 **Water Supply** | **Scope 3** *(Value Chain / Purchased Goods)* | `Water`, `Water Consumption`, `IG`, `m3`, `Gallons` | Municipal water supply |
| ⛽ **Fuel (Gasoline / Diesel)** | **Scope 1** *(Direct Emissions)* | `Petrol`, `Super 98`, `Special 95`, `Diesel`, `Liters` | Motor gasoline / transport fuel |

> **Note on Multi-Utility Bills:** If a single document (such as a municipal DEWA bill) contains both Electricity and Water charges, the AI Agent extracts them as separate entries, calculating emissions for each stream independently.

---

## 🏗️ Workflow Architecture

```
Incoming Webhook (POST PDF/Image Bill)
                  │
                  ▼
         AI Agent (LangChain)
      ┌───────────┴───────────┐
      │                       │
[Google Vertex AI]    [Structured Output Parser]
 (Gemini 3.7 Flash)       (JSON Schema)
      │
      ├───► [On Error] ──► Respond to Webhook1 (Error Handling)
      │
      ▼ [Success]
Format for Climatiq (JS Code — Unit Normalization & Activity Mapping)
                  │
                  ▼
Climatiq API Calculation (HTTP Request — https://api.climatiq.io/data/v1/estimate)
                  │
                  ▼
Clean & Merge Data (JS Code — Merge metadata, compute rounded CO2e)
                  │
                  ▼
Respond to Webhook (Return 200 OK JSON Response)
```

---

## 🧩 Workflow Nodes Breakdown

### 1. `Webhook`
- **Node Type:** `n8n-nodes-base.webhook` (v2.1)
- **HTTP Method:** `POST`
- **Path:** `86c60ff3-b73a-4884-96ca-f5ddd7c381af`
- **Response Mode:** `responseNode` (holds the HTTP connection open until the final JSON payload is ready).
- **Function:** Accepts multipart file payloads (scans, receipts, invoices in image or PDF formats).

---

### 2. `AI Agent` & Supporting AI Nodes
- **Node Type:** `@n8n/n8n-nodes-langchain.agent` (v3.1)
- **Model:** `Google Vertex Chat Model` (`gemini-3.7-flash`)
- **Output Parser:** `Structured Output Parser`
- **Binary Pass-through:** `passthroughBinaryImages: true`, `passthroughBinaryPdfs: true`
- **System Prompt Rules:**
  - Operates as a strict **AI Data Extraction Auditor** for ESG reporting.
  - Identifies electricity (Scope 2), water (Scope 3), and fuel (Scope 1).
  - Ignores non-utility items (e.g., convenience store snacks on gas station receipts).
  - Handles **spend-based fallbacks** when consumption volume is missing by extracting total monetary amount and ISO currency.
- **Error Handling:** `onError: continueErrorOutput` routes to `Respond to Webhook1` in case of unparseable documents.

---

### 3. `Format for Climatiq`
- **Node Type:** `n8n-nodes-base.code` (JavaScript — Run Once for All Items)
- **Purpose:** Normalizes user/bill units into Climatiq's strict unit definitions and assigns official Climatiq emission factor activity IDs.

```javascript
// Example Activity Mapping
if (entry.type === "electricity") {
  activityId = "electricity-supply_grid-source_residual_mix";
  parameters = { energy: entry.value, energy_unit: normalizeEnergyUnit(entry.unit) };
} else if (entry.type === "water") {
  activityId = "water_supply-type_na";
  parameters = { volume: entry.value, volume_unit: normalizeVolumeUnit(entry.unit) };
} else if (entry.type === "fuel") {
  activityId = "fuel-type_motor_gasoline-fuel_use_na";
  parameters = { volume: entry.value, volume_unit: normalizeVolumeUnit(entry.unit) };
}
```

---

### 4. `Climatiq API Calculation`
- **Node Type:** `n8n-nodes-base.httpRequest` (v4.2)
- **Endpoint:** `POST https://api.climatiq.io/data/v1/estimate`
- **Authentication:** `httpBearerAuth` (Climatiq API Key)
- **Payload:**
  ```json
  {
    "emission_factor": {
      "activity_id": "electricity-supply_grid-source_residual_mix",
      "data_version": "^37"
    },
    "parameters": {
      "energy": 1500,
      "energy_unit": "kWh"
    }
  }
  ```

---

### 5. `Clean & Merge Data`
- **Node Type:** `n8n-nodes-base.code` (JavaScript)
- **Purpose:** Combines the original extracted bill metadata (vendor name, cost, currency, original consumption) with Climatiq's calculation results (`co2e`, `co2e_unit`, emission region).
- **Rounding:** Formats total emissions to 2 decimal places (`parseFloat(apiData.co2e.toFixed(2))`).

---

### 6. `Respond to Webhook`
- **Node Type:** `n8n-nodes-base.respondToWebhook` (v1.5)
- **Purpose:** Sends the structured JSON array back to the client application with HTTP status 200.

---

## 📐 Unit Normalization & Climatiq Factor Mapping

### Volume Normalization (`normalizeVolumeUnit`)
| Input Units on Invoices | Normalized Unit (`Climatiq`) |
|---|---|
| `l`, `liter`, `liters`, `litre`, `litres` | `l` |
| `ml`, `milliliter`, `milliliters` | `ml` |
| `m3`, `cubic meter`, `cubic meters`, `cbm` | `m3` |
| `cubic foot`, `cubic feet`, `ft3` | `cubic_foot` |
| `gallon`, `gallons`, `gal`, `us gallon` | `gallon_us` |
| `ig` *(Imperial Gallons)* | `gallon_us` *(mapped to standard fallback)* |
| `bbl`, `barrel`, `barrels` | `bbl` |

### Energy Normalization (`normalizeEnergyUnit`)
| Input Units on Invoices | Normalized Unit (`Climatiq`) |
|---|---|
| `kwh`, `kw h`, `kilowatt hour`, `kilowatt hours` | `kWh` |
| `mwh` | `MWh` |
| `wh` | `Wh` |
| `gj` | `GJ` |
| `mj` | `MJ` |
| `btu` | `Btu` |

---

## 📡 Webhook API Specification

### 1. Input Request
- **Endpoint:** `POST <N8N_INSTANCE_URL>/webhook/86c60ff3-b73a-4884-96ca-f5ddd7c381af`
- **Content-Type:** `multipart/form-data`
- **Body:** Binary file field containing the invoice/receipt image or PDF.

### 2. Output Response
- **Status:** `200 OK`
- **Content-Type:** `application/json`

```json
[
  {
    "type": "electricity",
    "company_name": "DEWA",
    "consumption_value": 1500,
    "consumption_unit": "kWh",
    "cost_amount": 450,
    "currency": "AED",
    "total_co2e": 612.45,
    "co2e_unit": "kg",
    "emission_region": "AE"
  },
  {
    "type": "water",
    "company_name": "DEWA",
    "consumption_value": 6000,
    "consumption_unit": "gallon_us",
    "cost_amount": 210,
    "currency": "AED",
    "total_co2e": 18.32,
    "co2e_unit": "kg",
    "emission_region": "AE"
  }
]
```

---

## 🔑 Prerequisites & Credentials

Configure the following credentials in n8n under **Settings → Credentials**:

| Credential Name | n8n Credential Type | Description |
|---|---|---|
| **Google Service Account** | `Google API` / Service Account | GCP Service Account with Vertex AI API enabled for Gemini models |
| **Bearer Auth account** | `HTTP Bearer Auth` | Climatiq API Key (`Bearer YOUR_CLIMATIQ_API_KEY`) from [climatiq.io](https://www.climatiq.io/) |

---

## 🚀 Setup & Deployment

1. **Import Workflow**:
   - In n8n, navigate to **Workflows → Import from File**.
   - Select [workflow.json](file:///c:/Users/ziedr/Desktop/disk/n8n-automation-workflows/Telegram%20AI%20Community%20Manager/Utility%20Bill%20Carbon%20Calculator/workflow.json).
2. **Assign Credentials**:
   - Open the **Google Vertex Chat Model** node and select your Google Cloud Service Account.
   - Open the **Climatiq API Calculation** node and select your Climatiq Bearer Auth credential.
3. **Verify GCP Vertex AI Project**:
   - Ensure the Google Vertex Chat Model is pointing to your active Google Cloud Project ID (default configured: `n8n-gemini-app-500118`).
4. **Activate**:
   - Toggle the workflow status to **Active**.
   - Copy the Production Webhook URL for client integration.

---

## 💻 Frontend Dashboard Integration

This workflow serves as the backend calculation engine for the **CarbonFlow** dashboard.

- **Frontend Repository:** [GitHub - ZaidAR49/CarbonFlow](https://github.com/ZaidAR49/CarbonFlow)
- **Features:** Direct invoice drag-and-drop, real-time ESG metrics visualization, historical trend tracking, and Scope 1/2/3 breakdown analytics.

---

## 🛡️ Edge Cases & Error Handling

| Scenario | Handling Mechanism |
|---|---|
| **Combined Utility Bill (e.g. DEWA)** | AI Agent splits Electricity and Water into distinct JSON array items; both are processed sequentially through Climatiq. |
| **Receipt with Store / Non-Fuel Items** | System prompt instructs the auditor agent to discard supermarket/convenience items and isolate fuel volume only. |
| **Consumption Volume Missing** | Falls back to financial spend extraction (`cost_amount` + `currency`) to allow spend-based emission estimations. |
| **Unparseable or Corrupted File** | AI Agent errors are captured via `continueErrorOutput` and routed to `Respond to Webhook1` to prevent hanging requests. |
| **Unit Variations & Typos** | Normalization helper functions sanitize and map common spelling differences (`cbm`, `m3`, `cubic meters`, `IG`). |

---

## 🛠️ Tech Stack

- **Workflow Engine:** [n8n](https://n8n.io/) (v1.0+)
- **LLM Engine:** [Google Gemini 3.7 Flash](https://cloud.google.com/vertex-ai) via LangChain Vertex AI
- **Carbon Accounting Engine:** [Climatiq API](https://www.climatiq.io/) (Data Version `^37`)
- **Parser & Transformer:** JavaScript (ES6) / LangChain Structured Output Parser
