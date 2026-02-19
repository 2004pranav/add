# Analytics Dashboard — Architecture

## Overview

This is a multi-client analytics dashboard built with Vite + React + TypeScript.
Each client has their own CSV data files and a JSON config that fully controls
what their dashboard looks like and how KPIs are computed.

The core principle: **adding a new client requires zero code changes**.
You only need to:
1. Create a JSON config in `public/configs/clients/{clientId}.json`
2. Upload CSV files to `public/data/{clientId}/`
3. Add one entry to `src/clients/registry.ts` (for the sidebar)

---

## Folder Structure

```
project-root/
│
├── public/
│   ├── configs/
│   │   └── clients/
│   │       ├── pomerene.json       ← one JSON config per client
│   │       ├── drlo.json
│   │       ├── southside.json
│   │       └── ...
│   │
│   └── data/
│       ├── pomerene/               ← CSV files for each client
│       │   ├── chargebypostdate.csv
│       │   ├── paymentbypostdate.csv
│       │   ├── chargebyservicedate.csv
│       │   ├── paymentbyservicedate.csv
│       │   ├── adjustmentbyservicedate.csv
│       │   ├── denial.csv
│       │   ├── openar.csv
│       │   ├── aging.csv
│       │   └── automation.csv
│       ├── drlo/
│       │   └── ...
│       └── southside/
│           └── ...
│
└── src/
    ├── clients/
    │   └── registry.ts             ← master list of client IDs and names (sidebar)
    │
    ├── services/
    │   ├── dataLoader.ts           ← fetches JSON config + parses CSV files via PapaParse
    │   ├── kpiEngine.ts            ← computes KPI values by calling the right formula
    │   └── kpiFormulas.ts          ← all formula definitions (compute, format, downBetter)
    │
    ├── types/
    │   └── client.ts               ← all TypeScript interfaces and types
    │
    ├── configs/
    │   └── dashboardSections.ts    ← DashboardSection type definition (kept for type safety)
    │
    ├── components/
    │   ├── DashboardLayout.tsx     ← reads sections from client JSON, renders the page
    │   ├── DashboardSidebar.tsx    ← reads from src/clients/registry.ts
    │   ├── DashboardHeader.tsx
    │   ├── KpiCards.tsx            ← pure UI, receives computed KpiResult[]
    │   └── DashboardCharts.tsx     ← pure UI chart components
    │
    └── pages/
        ├── ClientDashboard.tsx     ← orchestrates: load config → fetch CSVs → compute KPIs → render
        └── Index.tsx               ← all-clients aggregated view
```

---

## Files Deleted (replaced by this architecture)

| Deleted File | Reason |
|---|---|
| `src/data/clients.ts` | All data was hardcoded here. Replaced by CSV files + service layer. |
| `src/configs/clientDashboardLayouts.ts` | Layout now lives in each client's JSON under `layout.sections`. |
| `src/configs/clientKpiLayouts.ts` | KPI list now lives in each client's JSON under `kpis`. |
| `src/configs/clientLoader.ts` | Used dynamic `import()` to load JSON from `src/`. Replaced by `dataLoader.ts` which uses `fetch()` from `public/`. |
| `src/configs/clients/*.json` | Old JSON configs with incomplete structure. Replaced by full configs in `public/configs/clients/`. |

---

## Client JSON Config

**Location:** `public/configs/clients/{clientId}.json`
**Fetched at runtime via:** `fetch("/configs/clients/pomerene.json")`

This is the single source of truth for a client. It defines:
- Which CSV files to load
- Which KPIs to show and which formula to use for each
- Which dashboard sections (charts) to render

```json
{
  "clientId": "pomerene",
  "name": "Pomerene Hospital",
  "shortName": "Pomerene",

  "dataSources": {
    "chargeByPostDate":        "chargebypostdate.csv",
    "paymentByPostDate":       "paymentbypostdate.csv",
    "chargeByServiceDate":     "chargebyservicedate.csv",
    "paymentByServiceDate":    "paymentbyservicedate.csv",
    "adjustmentByServiceDate": "adjustmentbyservicedate.csv",
    "denials":                 "denial.csv",
    "openAR":                  "openar.csv",
    "aging":                   "aging.csv",
    "automation":              "automation.csv"
  },

  "kpis": [
    { "key": "totalClaims", "label": "Total Claims", "formulaKey": "countClaims"         },
    { "key": "gcr",         "label": "GCR",          "formulaKey": "grossCollectionRate" },
    { "key": "ncr",         "label": "NCR",           "formulaKey": "netCollectionRate"  },
    { "key": "denialRate",  "label": "Denial Rate",   "formulaKey": "denialRate"         },
    { "key": "fpr",         "label": "FPR",           "formulaKey": "firstPassRate"      },
    { "key": "ccr",         "label": "CCR",           "formulaKey": "cleanClaimRate"     }
  ],

  "layout": {
    "sections": [
      "performanceTrends",
      "revenueOverview",
      "automationScore",
      "arAging",
      "claimsVolume",
      "periodComparison",
      "automationPenetration"
    ]
  }
}
```

### JSON Fields

| Field | Type | Description |
|---|---|---|
| `clientId` | string | Unique ID, must match the filename and the URL (`/client/pomerene`) |
| `name` | string | Full display name |
| `shortName` | string | Short name used in the sidebar |
| `dataSources` | object | Maps a logical key → actual CSV filename inside `public/data/{clientId}/` |
| `kpis` | array | Ordered list of KPI cards to show. Each entry has `key`, `label`, `formulaKey` |
| `kpis[].key` | string | Unique KPI identifier, maps to icon/color in the UI registry |
| `kpis[].label` | string | Text displayed on the KPI card |
| `kpis[].formulaKey` | string | Which formula to use from `kpiFormulas.ts` |
| `layout.sections` | string[] | Ordered list of dashboard sections to render. Valid values defined in `dashboardSections.ts` |

---

## KPI Formulas File

**Location:** `src/services/kpiFormulas.ts`

This file is the single place where all KPI computation logic lives.
Every formula defines three things:
- `format` — how to display the computed number (`"number"`, `"percent"`, `"currency"`)
- `downBetter` — whether a decrease is good (e.g. Denial Rate going down = green)
- `compute` — the actual function that receives the parsed CSV datasets and returns a number

```ts
export const KPI_FORMULAS: Record<string, KpiFormula> = {

  countClaims: {
    format: "number",
    downBetter: false,
    compute: (datasets) => datasets.chargeByPostDate.length
  },

  grossCollectionRate: {
    format: "percent",
    downBetter: false,
    compute: (datasets) => {
      const payments = sum(datasets.paymentByPostDate, "payment_amount");
      const charges  = sum(datasets.chargeByPostDate,  "charge_amount");
      return charges > 0 ? (payments / charges) * 100 : 0;
    }
  },

  netCollectionRate: {
    format: "percent",
    downBetter: false,
    compute: (datasets) => {
      const payments     = sum(datasets.paymentByPostDate,       "payment_amount");
      const charges      = sum(datasets.chargeByPostDate,        "charge_amount");
      const adjustments  = sum(datasets.adjustmentByServiceDate, "adjustment_amount");
      const denominator  = charges - adjustments;
      return denominator > 0 ? (payments / denominator) * 100 : 0;
    }
  },

  denialRate: {
    format: "percent",
    downBetter: true,        // ← going DOWN is good for this metric
    compute: (datasets) => {
      const denied = datasets.denials.length;
      const total  = datasets.chargeByPostDate.length;
      return total > 0 ? (denied / total) * 100 : 0;
    }
  },

  firstPassRate: {
    format: "percent",
    downBetter: false,
    compute: (datasets) => {
      const total  = datasets.chargeByPostDate.length;
      const denied = datasets.denials.length;
      return total > 0 ? ((total - denied) / total) * 100 : 0;
    }
  },

  cleanClaimRate: {
    format: "percent",
    downBetter: false,
    compute: (datasets) => {
      const total  = datasets.chargeByPostDate.length;
      const denied = datasets.denials.length;
      return total > 0 ? ((total - denied) / total) * 100 : 0;
    }
  },

  totalPayments: {
    format: "currency",
    downBetter: false,
    compute: (datasets) => sum(datasets.paymentByPostDate, "payment_amount")
  },

  totalOpenAR: {
    format: "currency",
    downBetter: true,        // ← lower open AR is better
    compute: (datasets) => sum(datasets.openAR, "ar_amount")
  }

};
```

### Why a separate formulas file?

- A client's JSON references `"formulaKey": "denialRate"` — it does not contain any math
- All business logic is in one place — easy to audit and update
- Two clients can use the same formula or different ones independently
- Adding a new formula does not touch any client config

---

## Service Layer

### `src/services/dataLoader.ts`

Responsible for:
1. Fetching the client JSON config from `public/configs/clients/{clientId}.json`
2. Fetching each CSV file listed in `dataSources` from `public/data/{clientId}/`
3. Parsing CSVs using PapaParse into arrays of row objects
4. Returning the config and parsed datasets to the caller

```
fetch /configs/clients/pomerene.json
  → parse JSON → ClientConfig

for each entry in dataSources:
  fetch /data/pomerene/chargebypostdate.csv  → PapaParse → DataRow[]
  fetch /data/pomerene/paymentbypostdate.csv → PapaParse → DataRow[]
  ...

return { config, datasets }
```

If a CSV file is missing (404), that dataset returns an empty array `[]`.
The dashboard still renders — KPIs that depend on that file will show `—`.

### `src/services/kpiEngine.ts`

Responsible for:
1. Receiving the client config and parsed datasets
2. For each KPI in `config.kpis`, looking up the formula by `formulaKey` in `KPI_FORMULAS`
3. Calling `formula.compute(datasets)` to get the raw number
4. Formatting the number according to `formula.format`
5. Returning an array of `KpiResult` objects ready for the UI

### Data flow in `ClientDashboard.tsx`

```
URL: /client/pomerene
        ↓
ClientDashboard mounts
        ↓
dataLoader.loadClientConfig("pomerene")
  → fetch /configs/clients/pomerene.json
        ↓
dataLoader.loadClientDataSets("pomerene", config.dataSources)
  → fetch + parse all CSV files in parallel
        ↓
kpiEngine.computeAllKpis(config.kpis, datasets)
  → for each KPI, call KPI_FORMULAS[formulaKey].compute(datasets)
        ↓
chartTransformers.buildChartData(datasets)
  → group CSV rows by month, bucket, reason etc.
        ↓
DashboardLayout receives:
  - config    (clientId, layout.sections, kpi metadata)
  - kpis      (computed KpiResult[])
  - chartData (performanceData, arAging, claimsVolume, etc.)
```

---

## TypeScript Types

**Location:** `src/types/client.ts`

```ts
// One parsed CSV row
type DataRow = Record<string, string>;

// All datasets for a client
type ClientDataSets = Record<string, DataRow[]>;

// One KPI definition from the JSON
interface KpiDefinition {
  key: string;
  label: string;
  formulaKey: string;
}

// The formula entry in kpiFormulas.ts
interface KpiFormula {
  format: "number" | "percent" | "currency";
  downBetter: boolean;
  compute: (datasets: ClientDataSets) => number;
}

// What the UI receives after computation
interface KpiResult {
  key: string;
  label: string;
  value: string;       // formatted: "95.8%", "$1.1M", "4,231"
  rawValue: number;
  change: string;      // "+2.5%", "-1.3%", "N/A"
  changeRaw: number;
  format: "number" | "percent" | "currency";
  downBetter: boolean;
}

// The full client config from JSON
interface ClientConfig {
  clientId: string;
  name: string;
  shortName: string;
  dataSources: Record<string, string>;
  kpis: KpiDefinition[];
  layout: {
    sections: DashboardSection[];
  };
}

// Everything the dashboard needs to render
interface ResolvedClient {
  config: ClientConfig;
  kpis: KpiResult[];
  chartData: ClientChartData;
}
```

---

## Client Registry

**Location:** `src/clients/registry.ts`

The only file updated when onboarding a new client (besides the JSON + CSVs).
Used exclusively by the sidebar to render the client list.

```ts
export const CLIENT_REGISTRY = [
  { id: "pomerene",    name: "Pomerene Hospital",  shortName: "Pomerene"    },
  { id: "drlo",        name: "Dr Lo",              shortName: "Dr Lo"       },
  { id: "southside",   name: "Southside Eye",      shortName: "Southside"   },
  // add new clients here
];
```

---

## Dashboard Sections

**Location:** `src/configs/dashboardSections.ts` — **kept, not deleted**

This file only defines the TypeScript union type for valid section names.
It is NOT config — it is a type contract used by the compiler to catch typos
in JSON configs and component code.

```ts
export const DASHBOARD_SECTIONS = [
  "performanceTrends",
  "revenueOverview",
  "automationScore",
  "automationPenetration",
  "arAging",
  "claimsVolume",
  "periodComparison",
  "topDenials",
  "clientPerformance",
] as const;

export type DashboardSection = typeof DASHBOARD_SECTIONS[number];
```

---

## KPI Display Registry

**Location:** `src/kpis/registry.ts`

Maps each KPI `key` to its display metadata: icon, colors.
This is purely visual and lives in the frontend — it has nothing to do with
data or formulas. When a new KPI key is introduced, one entry is added here.

```ts
export const KPI_REGISTRY = {
  totalClaims:  { icon: FileText,    color: "blue"   },
  totalPayments:{ icon: DollarSign,  color: "green"  },
  gcr:          { icon: TrendingUp,  color: "purple" },
  ncr:          { icon: Activity,    color: "amber"  },
  denialRate:   { icon: TrendingDown,color: "red"    },
  fpr:          { icon: Zap,         color: "blue"   },
  ccr:          { icon: ShieldCheck, color: "green"  },
  totalOpenAR:  { icon: Wallet,      color: "purple" },
};
```

---

## Adding a New Client — Step by Step

1. **Create the JSON config**
   - Copy an existing config from `public/configs/clients/`
   - Set `clientId`, `name`, `shortName`
   - Update `dataSources` with the CSV filenames for this client
   - Set `kpis` array — pick which KPIs to show and which `formulaKey` each uses
   - Set `layout.sections` — pick which chart blocks to show

2. **Upload CSV files**
   - Create folder `public/data/{clientId}/`
   - Upload all CSV files referenced in `dataSources`

3. **Add to registry**
   - Open `src/clients/registry.ts`
   - Add one line: `{ id: "{clientId}", name: "...", shortName: "..." }`

That's it. No other code changes required.

---

## CSV Column Name Defaults

The chart transformers expect these default column names in each CSV file.
If a client's CSVs use different column names, a `columnMapping` override
can be added to their JSON config.

| CSV File | Column | Default Name |
|---|---|---|
| chargeByPostDate | date | `post_date` |
| chargeByPostDate | amount | `charge_amount` |
| chargeByPostDate | claim ID | `claim_id` |
| paymentByPostDate | date | `post_date` |
| paymentByPostDate | amount | `payment_amount` |
| adjustmentByServiceDate | date | `service_date` |
| adjustmentByServiceDate | amount | `adjustment_amount` |
| denials | date | `denial_date` |
| denials | reason | `denial_reason` |
| denials | amount | `denied_amount` |
| openAR | date | `ar_date` |
| openAR | amount | `ar_amount` |
| aging | bucket label | `bucket` |
| aging | amount | `amount` |
| automation | item name | `name` |
| automation | percentage | `value` |
