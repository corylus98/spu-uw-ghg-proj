# Frontend API Guide

This guide explains how to use the EcoMetrics Backend API, with detailed examples for column mappings and all endpoints.

**Base URL**: `http://localhost:8000/api`

---

## Table of Contents

1. [Quick Start Workflow](#quick-start-workflow)
2. [API Endpoints](#api-endpoints)
3. [Column Mapping Rules (Detailed)](#column-mapping-rules-detailed)
4. [Filter Configuration](#filter-configuration)
5. [EF_ID Lookup Configuration](#efid-lookup-configuration)
6. [Complete Request Examples](#complete-request-examples)
7. [Error Handling](#error-handling)

---

## Quick Start Workflow

```
1. GET /api/sources                              → List available source files
2. GET /api/sources/{sourceId}/preview           → Preview columns and data
3. POST /api/sessions                            → Create a new session
4. POST /api/sessions/{id}/sources/{id}/config   → Save column mapping config
5. POST /api/sessions/{id}/calculate             → Run emissions calculation
6. GET /api/sessions/{id}/analytics/summary      → Get results summary
7. POST /api/sessions/{id}/analytics/chart-data  → Get chart-ready data
```

---

## API Endpoints

### 1. Health Check

```http
GET /api/health
```

**Response:**
```json
{
  "service": "ecometrics-backend",
  "status": "healthy",
  "version": "1.0.0"
}
```

---

### 2. List Source Files

```http
GET /api/sources
```

**Response:**
```json
{
  "success": true,
  "sources": [
    {
      "sourceId": "fleetfuel_2019_2022",
      "name": "FleetFuel_2019-2022",
      "filePath": "data/raw/CONSUMPTION/FleetFuel_2019-2022.xlsx",
      "fileSize": 1850000,
      "lastModified": "2026-01-15T10:30:00Z",
      "sheets": ["Sheet1", "Data"]
    }
  ]
}
```

---

### 3. Preview Source Data

```http
GET /api/sources/{sourceId}/preview?sheet=Sheet1
```

**Query Parameters:**
| Parameter | Required | Description |
|-----------|----------|-------------|
| `sheet` | No | Excel sheet name (for .xlsx files) |

**Response:**
```json
{
  "success": true,
  "sourceId": "fleetfuel_2019_2022",
  "fileName": "FleetFuel_2019-2022.xlsx",
  "sheet": "Sheet1",
  "totalRows": 15420,
  "columns": [
    {
      "name": "EQ_EQUIP_NO",
      "type": "string",
      "sampleValues": ["1011", "4386", "5504"],
      "nullCount": 0,
      "uniqueCount": 342
    },
    {
      "name": "QTY_FUEL",
      "type": "float",
      "sampleValues": [25.5, 45.2, 12.8],
      "nullCount": 5,
      "min": 0.5,
      "max": 450.2,
      "mean": 35.6
    }
  ],
  "preview": [
    {"EQ_EQUIP_NO": "1011", "QTY_FUEL": 25.5, "FUEL_TYPE": "Gasoline", ...}
  ]
}
```

---

### 4. Create Session

```http
POST /api/sessions
Content-Type: application/json
```

**Request Body:**
```json
{
  "name": "Q4 2022 Fleet Analysis",
  "description": "Fleet emissions for Q4 2022"
}
```

**Response:**
```json
{
  "success": true,
  "sessionId": "sess_a1b2c3",
  "name": "Q4 2022 Fleet Analysis",
  "createdAt": "2026-01-25T10:00:00Z"
}
```

---

### 5. List Sessions

```http
GET /api/sessions
```

**Response:**
```json
{
  "success": true,
  "sessions": [
    {
      "sessionId": "sess_a1b2c3",
      "name": "Q4 2022 Fleet Analysis",
      "createdAt": "2026-01-25T10:00:00Z",
      "lastModified": "2026-01-25T14:30:00Z",
      "status": "active",
      "configuredSources": 2,
      "hasCalculations": true,
      "totalMtCO2e": 4523.847
    }
  ]
}
```

---

### 6. Get Session Details

```http
GET /api/sessions/{sessionId}
```

---

### 7. Delete Session

```http
DELETE /api/sessions/{sessionId}
```

**Response:**
```json
{
  "success": true,
  "sessionId": "sess_a1b2c3",
  "deletedFiles": [
    "data/sessions/sess_a1b2c3/session_meta.json",
    "data/calculated/sess_a1b2c3/fleet_emissions.parquet"
  ]
}
```

---

### 8. Save Column Mapping Configuration

```http
POST /api/sessions/{sessionId}/sources/{sourceId}/config
Content-Type: application/json
```

**See [Column Mapping Rules](#column-mapping-rules-detailed) for detailed request body structure.**

**Response:**
```json
{
  "success": true,
  "sessionId": "sess_a1b2c3",
  "sourceId": "fleetfuel_2019_2022",
  "configVersion": 1,
  "validation": {
    "status": "valid",
    "warnings": ["Column 'Vehicle Class' has 5% null values"],
    "errors": []
  },
  "savedAt": "2026-01-25T10:15:00Z"
}
```

---

### 9. Get Saved Configuration

```http
GET /api/sessions/{sessionId}/sources/{sourceId}/config
```

---

### 10. Run Emissions Calculation

```http
POST /api/sessions/{sessionId}/calculate
Content-Type: application/json
```

**Request Body:**
```json
{
  "sources": ["fleetfuel_2019_2022", "pse_2022"],
  "gwpVersion": "AR5"
}
```

| Field | Required | Default | Description |
|-------|----------|---------|-------------|
| `sources` | No | All configured | Array of source IDs to calculate |
| `gwpVersion` | No | `"AR5"` | GWP version: `"AR5"` or `"AR4"` |

**Response:**
```json
{
  "success": true,
  "sessionId": "sess_a1b2c3",
  "calculationId": "calc_a1b2c3_d4e5f6",
  "status": "completed",
  "startedAt": "2026-01-25T10:30:00Z",
  "completedAt": "2026-01-25T10:30:45Z",
  "results": {
    "fleetfuel_2019_2022": {
      "inputRows": 12850,
      "outputRows": 38550,
      "totalMtCO2e": 4523.847,
      "filePath": "data/calculated/sess_a1b2c3/fleetfuel_2019_2022_emissions.parquet"
    }
  },
  "aggregated": {
    "totalMtCO2e": 4523.847,
    "totalSources": 1,
    "yearRange": {"min": 2019, "max": 2022}
  }
}
```

---

### 11. Get Analytics Summary

```http
GET /api/sessions/{sessionId}/analytics/summary
```

**Response:**
```json
{
  "success": true,
  "sessionId": "sess_a1b2c3",
  "summary": {
    "totalMtCO2e": 7822.303,
    "totalSources": 2,
    "yearRange": {"min": 2019, "max": 2022},
    "bySector": {"Fleet": 4523.847, "Facilities": 3298.456},
    "byYear": {"2019": 1823.45, "2020": 1756.89, "2021": 1892.34, "2022": 2349.62},
    "bySubtype": {"Diesel": 1842.34, "Gasoline": 1523.67, "B20": 456.78},
    "byGHG": {"CO2": 7500.0, "CH4": 200.0, "N2O": 122.303}
  }
}
```

---

### 12. Get Chart Data

```http
POST /api/sessions/{sessionId}/analytics/chart-data
Content-Type: application/json
```

**Request Body:**
```json
{
  "chartType": "pie",
  "metric": "mtCO2e_calc",
  "groupBy": "Subtype",
  "filters": [
    {"column": "Year", "operator": "=", "value": 2022}
  ]
}
```

| Field | Required | Default | Options |
|-------|----------|---------|---------|
| `chartType` | No | `"pie"` | `"pie"`, `"bar"`, `"line"`, `"stacked_bar"` |
| `metric` | No | `"mtCO2e_calc"` | Any numeric column |
| `groupBy` | No | `"Subtype"` | Any column name |
| `filters` | No | `[]` | Array of filter objects |
| `stackBy` | No | `"GHG"` | For `stacked_bar` only |

**Pie Chart Response:**
```json
{
  "success": true,
  "chartType": "pie",
  "title": "mtCO2e_calc by Subtype (Year=2022)",
  "data": [
    {"label": "Diesel", "value": 1842.34, "percentage": 45.2, "color": "#FF6384"},
    {"label": "Gasoline", "value": 1523.67, "percentage": 37.4, "color": "#36A2EB"}
  ],
  "total": 4075.24
}
```

**Bar/Line Chart Response:**
```json
{
  "success": true,
  "chartType": "bar",
  "title": "mtCO2e_calc by Year",
  "labels": ["2019", "2020", "2021", "2022"],
  "datasets": [
    {"label": "mtCO2e_calc", "data": [1823.45, 1756.89, 1892.34, 2349.62], "backgroundColor": "#FF6384"}
  ]
}
```

---

## Column Mapping Rules (Detailed)

The `columnMappings` object defines how to transform source columns to the standard schema. Each key is the **target column name**, and the value is a **mapping rule object**.

### Required Target Columns

| Column | Type | Description |
|--------|------|-------------|
| `ACCT_ID` | string | Account/Asset identifier |
| `Year` | integer | Reporting year |
| `Consumption` | float | Activity quantity |
| `Unit` | string | Unit of consumption |
| `Subtype` | string | Fuel/utility type |
| `GHG` | string | Greenhouse gas (use GHG expansion) |

### Mapping Rule Types

---

#### 1. Direct Column Mapping (`sourceColumn`)

Copy values directly from a source column.

```json
{
  "columnMappings": {
    "ACCT_ID": {
      "sourceColumn": "EQ_EQUIP_NO"
    },
    "Consumption": {
      "sourceColumn": "QTY_FUEL"
    },
    "Subtype": {
      "sourceColumn": "FUEL_TYPE"
    },
    "LowOrg": {
      "sourceColumn": "DEPT_DEPT_CODE"
    },
    "Vehicle Location": {
      "sourceColumn": "LOC_STATION_LOC"
    }
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `sourceColumn` | Yes | Name of column in source data |

---

#### 2. Static Value (`staticValue`)

Assign a constant value to all rows.

```json
{
  "columnMappings": {
    "Unit": {
      "staticValue": "gal"
    },
    "Sector": {
      "staticValue": "Fleet"
    },
    "SourceNotes": {
      "staticValue": "Fleet Fuel Data 2019-2022"
    }
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `staticValue` | Yes | Constant value (string, number, or boolean) |

**Common Use Cases:**
- `Unit`: `"gal"`, `"KWH"`, `"kg"`, `"therms"`
- `Sector`: `"Fleet"`, `"Facilities"`, `"Landfill"`

---

#### 3. Derived Value (`derivedFrom` + `extractType`)

Extract a component from a date/datetime column.

```json
{
  "columnMappings": {
    "Year": {
      "derivedFrom": "FTK_DATE",
      "extractType": "year"
    },
    "Month": {
      "derivedFrom": "FTK_DATE",
      "extractType": "month"
    },
    "Quarter": {
      "derivedFrom": "FTK_DATE",
      "extractType": "quarter"
    }
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `derivedFrom` | Yes | Source column containing date/datetime |
| `extractType` | Yes | What to extract (see table below) |

**Supported `extractType` Values:**

| extractType | Output | Example (from "2021-10-15") |
|-------------|--------|------------------------------|
| `year` | Integer year | `2021` |
| `month` | Integer month (1-12) | `10` |
| `day` | Integer day of month | `15` |
| `quarter` | Integer quarter (1-4) | `4` |
| `dayofweek` | Integer day of week (Monday=0) | `4` (Friday) |

---

#### 4. Reference Table Join (`referenceTable`)

Look up a value from an external reference table.

```json
{
  "columnMappings": {
    "LOB": {
      "referenceTable": "data/raw/REFERENCE/LOB_LowOrgList.xlsx",
      "refSheet": "Sheet1",
      "refKey": "LowOrg",
      "sourceKey": "DEPT_DEPT_CODE",
      "columnJoined": "LOB"
    },
    "Division": {
      "referenceTable": "data/raw/REFERENCE/LOB_LowOrgList.xlsx",
      "refKey": "LowOrg",
      "sourceKey": "DEPT_DEPT_CODE",
      "columnJoined": "Division"
    }
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `referenceTable` | Yes | Path to reference file (relative to backend root) |
| `refSheet` | No | Sheet name for Excel files |
| `refKey` | Yes | Key column in reference table |
| `sourceKey` | Yes | Key column in source data |
| `columnJoined` | Yes | Column to retrieve from reference table |

**How it works:**
1. For each row in source data, get the value of `sourceKey`
2. Find matching row in reference table where `refKey` equals that value
3. Return the value from `columnJoined`

---

#### 5. GHG Expansion (`ghgType`)

Expand each row into multiple rows, one per greenhouse gas.

```json
{
  "columnMappings": {
    "GHG": {
      "ghgType": ["CO2", "CH4", "N2O"]
    }
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| `ghgType` | Yes | Array of GHG names to expand |

**How it works:**
- Input: 1 row with `Consumption = 25.5`
- Output: 3 rows, each with `Consumption = 25.5` and `GHG` = `"CO2"`, `"CH4"`, `"N2O"`

**Common GHG Lists:**
- Combustion fuels: `["CO2", "CH4", "N2O"]`
- Electricity: `["CO2", "CH4", "N2O"]`
- Refrigerants: `["R-134A"]` or `["R-410A"]`

---

## Filter Configuration

Filters are applied to raw data **before** column mapping.

```json
{
  "filters": [
    {
      "column": "FTK_DATE",
      "operator": ">=",
      "value": "2019-01-01"
    },
    {
      "column": "QTY_FUEL",
      "operator": ">",
      "value": 0
    },
    {
      "column": "FUEL_TYPE",
      "operator": "in",
      "value": ["Gasoline", "Diesel", "B20"]
    }
  ]
}
```

### Supported Operators

| Operator | Description | Value Type | Example |
|----------|-------------|------------|---------|
| `=` | Equals | any | `{"column": "Year", "operator": "=", "value": 2022}` |
| `!=` | Not equals | any | `{"column": "Status", "operator": "!=", "value": "Inactive"}` |
| `>` | Greater than | number/date | `{"column": "QTY_FUEL", "operator": ">", "value": 0}` |
| `>=` | Greater than or equal | number/date | `{"column": "FTK_DATE", "operator": ">=", "value": "2019-01-01"}` |
| `<` | Less than | number/date | `{"column": "Year", "operator": "<", "value": 2023}` |
| `<=` | Less than or equal | number/date | `{"column": "QTY_FUEL", "operator": "<=", "value": 500}` |
| `in` | In list | array | `{"column": "FUEL_TYPE", "operator": "in", "value": ["Gasoline", "Diesel"]}` |
| `not_in` | Not in list | array | `{"column": "Status", "operator": "not_in", "value": ["Deleted", "Archived"]}` |
| `is_null` | Is null/empty | (none) | `{"column": "Notes", "operator": "is_null"}` |
| `not_null` | Is not null | (none) | `{"column": "QTY_FUEL", "operator": "not_null"}` |

---

## EF_ID Lookup Configuration

Controls how emission factor IDs are constructed and looked up.

```json
{
  "efidLookup": {
    "strategy": "auto",
    "pattern": "Fuel_{Subtype}_20XX_{GHG}",
    "sectorFilter": "Fleet"
  }
}
```

| Property | Required | Default | Description |
|----------|----------|---------|-------------|
| `strategy` | No | `"auto"` | Lookup strategy |
| `pattern` | No | `"Fuel_{Subtype}_20XX_{GHG}"` | Pattern to construct EF_ID |
| `sectorFilter` | No | (none) | Filter EFID table by sector |

### Pattern Placeholders

| Placeholder | Replaced With | Example |
|-------------|---------------|---------|
| `{Subtype}` | Value from `Subtype` column | `Gasoline`, `Diesel`, `PSE` |
| `{GHG}` | Value from `GHG` column | `CO2`, `CH4`, `N2O` |
| `{Year}` | Value from `Year` column | `2022` |
| `{Sector}` | Value from `Sector` column | `Fleet`, `Facilities` |

### Common Patterns

| Data Type | Pattern | Example Output |
|-----------|---------|----------------|
| Fleet Fuel | `Fuel_{Subtype}_20XX_{GHG}` | `Fuel_Gasoline_20XX_CO2` |
| Electricity (PSE) | `Elec_PSE_{Year}_{GHG}` | `Elec_PSE_2022_CO2` |
| Electricity (SCL) | `Elec_SCL_{Year}_{GHG}` | `Elec_SCL_2022_CO2` |
| Natural Gas | `NG_{Subtype}_{GHG}` | `NG_Pipeline_CO2` |
| Refrigerants | `FacAC_{Subtype}_20XX` | `FacAC_R-134A_20XX` |

---

## Complete Request Examples

### Example 1: Fleet Fuel Data

```json
POST /api/sessions/sess_a1b2c3/sources/fleetfuel_2019_2022/config

{
  "sourceSheet": "Sheet1",

  "columnMappings": {
    "ACCT_ID": {
      "sourceColumn": "EQ_EQUIP_NO"
    },
    "Consumption": {
      "sourceColumn": "QTY_FUEL"
    },
    "Subtype": {
      "sourceColumn": "FUEL_TYPE"
    },
    "Year": {
      "derivedFrom": "FTK_DATE",
      "extractType": "year"
    },
    "Month": {
      "derivedFrom": "FTK_DATE",
      "extractType": "month"
    },
    "Unit": {
      "staticValue": "gal"
    },
    "Sector": {
      "staticValue": "Fleet"
    },
    "GHG": {
      "ghgType": ["CO2", "CH4", "N2O"]
    },
    "LowOrg": {
      "sourceColumn": "DEPT_DEPT_CODE"
    },
    "LOB": {
      "referenceTable": "data/raw/REFERENCE/LOB_LowOrgList.xlsx",
      "refKey": "LowOrg",
      "sourceKey": "DEPT_DEPT_CODE",
      "columnJoined": "LOB"
    },
    "Vehicle Location": {
      "sourceColumn": "LOC_STATION_LOC"
    },
    "Vehicle Class": {
      "sourceColumn": "Vehicle Class"
    }
  },

  "filters": [
    {
      "column": "FTK_DATE",
      "operator": ">=",
      "value": "2019-01-01"
    },
    {
      "column": "QTY_FUEL",
      "operator": ">",
      "value": 0
    }
  ],

  "efidLookup": {
    "pattern": "Fuel_{Subtype}_20XX_{GHG}",
    "sectorFilter": "Fleet"
  }
}
```

### Example 2: PSE Electricity Data

```json
POST /api/sessions/sess_a1b2c3/sources/pse_2022/config

{
  "sourceSheet": "Data",

  "columnMappings": {
    "ACCT_ID": {
      "sourceColumn": "Account_Number"
    },
    "Consumption": {
      "sourceColumn": "KWH_Usage"
    },
    "Subtype": {
      "staticValue": "PSE"
    },
    "Year": {
      "sourceColumn": "Bill_Year"
    },
    "Month": {
      "sourceColumn": "Bill_Month"
    },
    "Unit": {
      "staticValue": "KWH"
    },
    "Sector": {
      "staticValue": "Facilities"
    },
    "GHG": {
      "ghgType": ["CO2", "CH4", "N2O"]
    },
    "LowOrg": {
      "sourceColumn": "Dept_Code"
    },
    "LOB": {
      "referenceTable": "data/raw/REFERENCE/LOB_LowOrgList.xlsx",
      "refKey": "LowOrg",
      "sourceKey": "Dept_Code",
      "columnJoined": "LOB"
    }
  },

  "filters": [
    {
      "column": "KWH_Usage",
      "operator": ">",
      "value": 0
    }
  ],

  "efidLookup": {
    "pattern": "Elec_PSE_{Year}_{GHG}",
    "sectorFilter": "Facilities"
  }
}
```

### Example 3: Refrigerant Data (Single GHG)

```json
POST /api/sessions/sess_a1b2c3/sources/ac_facilities/config

{
  "columnMappings": {
    "ACCT_ID": {
      "sourceColumn": "Equipment_ID"
    },
    "Consumption": {
      "sourceColumn": "Refrigerant_kg"
    },
    "Subtype": {
      "sourceColumn": "Refrigerant_Type"
    },
    "Year": {
      "sourceColumn": "Service_Year"
    },
    "Unit": {
      "staticValue": "kg"
    },
    "Sector": {
      "staticValue": "Facilities"
    },
    "GHG": {
      "ghgType": ["R-134A"]
    }
  },

  "efidLookup": {
    "pattern": "FacAC_{Subtype}_20XX",
    "sectorFilter": "Facilities"
  }
}
```

---

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable error message",
    "details": {
      "errors": ["Specific error 1", "Specific error 2"]
    },
    "timestamp": "2026-01-25T10:30:00Z"
  }
}
```

### Error Codes

| Code | HTTP | Description |
|------|------|-------------|
| `MISSING_PARAMETER` | 400 | Required field missing in request |
| `VALIDATION_ERROR` | 400 | Config validation failed (see `details.errors`) |
| `INVALID_PARAMETER` | 400 | Invalid parameter value |
| `SOURCE_NOT_FOUND` | 404 | Source file not found |
| `SESSION_NOT_FOUND` | 404 | Session does not exist |
| `CONFIG_NOT_FOUND` | 404 | No config saved for source |
| `NO_DATA` | 404 | No calculated data available |
| `MISSING_EMISSION_FACTOR` | 422 | EF_ID not found in EFID table |
| `MISSING_GWP` | 422 | GWP value not found for GHG |
| `CALCULATION_ERROR` | 422/500 | Error during calculation |
| `STORAGE_ERROR` | 500 | File system error |

### Validation Error Example

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Configuration validation failed",
    "details": {
      "errors": [
        "Missing required column mapping: 'ACCT_ID'",
        "Source column 'INVALID_COLUMN' does not exist in raw data",
        "Reference table not found: data/raw/REFERENCE/Missing.xlsx"
      ]
    },
    "timestamp": "2026-01-25T10:30:00Z"
  }
}
```

---

## JavaScript/TypeScript Examples

### Fetch Wrapper

```typescript
const API_BASE = 'http://localhost:8000/api';

async function apiCall<T>(
  endpoint: string,
  options?: RequestInit
): Promise<T> {
  const response = await fetch(`${API_BASE}${endpoint}`, {
    headers: { 'Content-Type': 'application/json' },
    ...options,
  });

  const data = await response.json();

  if (!data.success) {
    throw new Error(data.error.message);
  }

  return data;
}
```

### Create Session and Configure Source

```typescript
// 1. Create session
const session = await apiCall<{sessionId: string}>('/sessions', {
  method: 'POST',
  body: JSON.stringify({
    name: 'Q4 2022 Analysis',
    description: 'Fleet emissions analysis'
  })
});

// 2. Save configuration
const config = await apiCall(`/sessions/${session.sessionId}/sources/fleet_fuel/config`, {
  method: 'POST',
  body: JSON.stringify({
    columnMappings: {
      ACCT_ID: { sourceColumn: 'EQ_EQUIP_NO' },
      Consumption: { sourceColumn: 'QTY_FUEL' },
      Subtype: { sourceColumn: 'FUEL_TYPE' },
      Year: { derivedFrom: 'FTK_DATE', extractType: 'year' },
      Unit: { staticValue: 'gal' },
      Sector: { staticValue: 'Fleet' },
      GHG: { ghgType: ['CO2', 'CH4', 'N2O'] }
    },
    filters: [
      { column: 'QTY_FUEL', operator: '>', value: 0 }
    ],
    efidLookup: {
      pattern: 'Fuel_{Subtype}_20XX_{GHG}',
      sectorFilter: 'Fleet'
    }
  })
});

// 3. Run calculation
const result = await apiCall(`/sessions/${session.sessionId}/calculate`, {
  method: 'POST',
  body: JSON.stringify({
    sources: ['fleet_fuel'],
    gwpVersion: 'AR5'
  })
});

console.log(`Total emissions: ${result.aggregated.totalMtCO2e} mtCO2e`);
```
