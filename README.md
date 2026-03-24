# Fixed Width File (FWF) Validator

A Python-based validation tool for **fixed-width export files** commonly used in the banking and insurance domain. It validates file structure, field-level data types, mandatory fields, and TRXN file headers against a configurable Excel-based field definition schema.

---

## Overview

Export files in banking/insurance systems (Customer/Party, Account, Transaction, Relationship, etc.) are often delivered as fixed-width text files. This tool automates quality checks on those files by reading a field definition Excel sheet and validating each record column-by-column.

### What It Validates

- **Record Length** — Every row must match the total expected width defined in the schema.
- **String/Text Fields** — Left-aligned; checks for unexpected leading whitespace and mandatory value presence. Fields marked as "NOT USED" are verified to be blank.
- **Date / DateTime Fields** — Right-aligned with leading spaces. Always stored as `YYYYMMDDHHmmSS` (14 chars) regardless of whether the schema labels it `DATE` or `DATETIME`. Validates format, mandatory presence, and correct whitespace padding.
- **Amount Fields** — Right-aligned with leading spaces. Stored as integers with implied 2 decimal places (e.g., `25.22` → `2522`). Validates integer parsability, minimum 3-digit length, mandatory presence, and whitespace padding.
- **Numeric / Integer Fields** — Validates integer parsability and mandatory presence.
- **TRXN Header** — Transaction files include a header row in the format `000000000HEADER<5-digit run number><10-digit record count>`. The tool validates each segment including cross-checking run number against the filename and record count against actual data rows.

---

## Prerequisites

- **OS:** Windows
- **Python:** 3.7+

### Python Dependencies

```
pandas
openpyxl
```

Install with:

```bash
pip install pandas openpyxl
```

---

## Input

### 1. Field Definition Excel File

An `.xlsx` workbook (Sheet1) that defines the schema of the fixed-width file. Both the Excel file and the FWF files to validate **must be in the same directory**.

| Column | Header       | Description                                      | Example        |
|--------|--------------|--------------------------------------------------|----------------|
| A      | Sr.No.       | Field sequence number                            | 1              |
| B      | Field_Name   | Name of the field                                | Account ID     |
| C      | Datatype     | One of: `String`, `Text`, `Date`, `Datetime`, `Amount`, `Numeric`, `Number`, `Int` | Date |
| D      | Field_Length  | Fixed width allocated to this field              | 20             |
| E      | Mandatory    | `Y` or `N`                                       | Y              |

> **Note:** Columns F and G are auto-calculated at runtime to derive each field's start and end (slice) positions. Do not manually populate them.

**Example:**

| Sr.No. | Field_Name | Datatype | Field_Length | Mandatory |
|--------|------------|----------|-------------|-----------|
| 1      | Account ID | String   | 10          | Y         |
| 2      | Name       | String   | 20          | Y         |
| 3      | Date       | Date     | 20          | Y         |
| 4      | Amount     | Integer  | 10          | N         |

### 2. Fixed-Width Text Files

One or more `.txt` files in the same directory. Each row is a fixed-width record whose total length equals the sum of all `Field_Length` values in the schema.

**Sample record** (total width = 60):

```
0552052544Michael Scott               2023122700000000001026716
|----10---||--------20----------||--------20--------||---10--|
Account ID       Name                   Date          Amount
```

**Field alignment convention:**
- `String` / `Text` → **Left-aligned**, padded with trailing spaces
- `Date` / `Datetime` → **Right-aligned**, padded with leading spaces (always 14-char `YYYYMMDDHHmmSS`)
- `Amount` / `Numeric` → **Right-aligned**, padded with leading spaces

---

## Usage

```bash
python fwf_validator.py
```

You will be prompted for:

```
Input Excel workbook name with extension: schema.xlsx
Input file's path: C:\exports\monthly
```

The tool will automatically pick up all `.txt` files in the given path and validate each one against the schema.

---

## Output

Results are written to a `Validated_Files` subfolder created inside the input path.

```
C:\exports\monthly\
├── schema.xlsx
├── CUST_EXPORT_20231227.txt
├── TRXN_EXPORT_00001.txt
└── Validated_Files\
    ├── CUST_EXPORT_20231227_Validated.txt       ← Clean file
    └── Error_Files\
        └── TRXN_EXPORT_00001_Error.txt          ← File with errors
```

- If **no errors** are found → result stays in `Validated_Files/` with a `_Validated.txt` suffix.
- If **errors** are found → result is moved to `Validated_Files/Error_Files/` with an `_Error.txt` suffix.

### Sample Validation Output

```
1. No of columns as per excel sheet: 4
2. Expected Record length as per excel sheet: 60
3. All record's length is correct as: 60

4. Record count in Export File is: 100
```

---

## TRXN File Format

Transaction files are identified by having `TRXN` in the filename. They contain a **header row** (row 1) followed by data rows.

**Header format** (30 characters):

```
000000000HEADER000010000001000
|---9---||--6--||--5-||---10--|
  Zeros   HEADER  Run#  RecCount
```

| Segment          | Position | Length | Description                                                     |
|------------------|----------|--------|-----------------------------------------------------------------|
| Leading Zeros    | 1–9      | 9      | Must be `000000000`                                             |
| Header Literal   | 10–15    | 6      | Must be `HEADER`                                                |
| Run Number       | 16–20    | 5      | Must match the first 5 characters of the filename's last 9 characters |
| Record Count     | 21–30    | 10     | Zero-padded count of data rows (excluding header)               |

---

## How It Works

1. **Schema Loading** — Reads the field definition Excel and computes cumulative start/end slice positions directly in columns F and G using pure Python.
2. **Record Length Check** — Validates that every row in the text file matches the expected total width.
3. **Field-Level Validation** — Iterates through each record and each column, slicing the fixed-width string and applying type-specific validation rules.
4. **TRXN Header Validation** — If the filename contains `TRXN`, validates the header row structure separately.
5. **Result Classification** — Writes validation logs and moves error files to a separate subfolder.

---

## Supported Export File Types

This tool is designed for banking/insurance domain export files including but not limited to:

- **Customer / Party** — Client demographic and identity data
- **Account** — Account details and attributes
- **Transaction (TRXN)** — Transactional records with header validation
- **Relationship** — Party-to-account or inter-party relationship mappings

---

## Limitations

- **Windows only** — Uses Windows-style file paths.
- **Single sheet** — Reads only `Sheet1` from the schema workbook.
- **Encoding** — Attempts UTF-8 first, falls back to UTF-16 for the input text files.

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

## Disclaimer

This is a personal project built as a generic utility tool. It does not contain any proprietary logic, client data, or sensitive information from any organization.
