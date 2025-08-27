Project files for RBAC/ABAC Role Automation
import os, textwrap, json, zipfile, io, csv

base = "/mnt/data/rbac-abac-role-automation"
os.makedirs(base, exist_ok=True)
os.makedirs(os.path.join(base, "src", "providers"), exist_ok=True)
os.makedirs(os.path.join(base, "hr"), exist_ok=True)
os.makedirs(os.path.join(base, "policies"), exist_ok=True)
os.makedirs(os.path.join(base, "reports"), exist_ok=True)

readme = """# RBAC/ABAC Role Automation (IAM Lab)

Automate role assignment and access decisions from HR data using both **RBAC** (static role mappings) and **ABAC** (attribute-based policies).  
This repo lets you **ingest HR data**, **evaluate RBAC & ABAC**, produce **audits**, and optionally **apply** the outcome to a provider (local file by default; Okta/AzureAD stubs included).

## Why this is a strong IAM-Engineer project
- Shows **identity data ingestion** (HR -> directory) and **policy engines**.
- Demonstrates **RBAC vs ABAC** with explainable outcomes and diffs.
- Adds an **apply** path you can later wire to real IdPs via REST (Okta/Azure AD).

---

## Quick Start

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Audit only (no changes), using sample HR data & policies
python src/main.py audit --hr hr/hr_sample.csv --policies policies/policies.yaml --out ./reports

# Explain ABAC & RBAC for a single user
python src/main.py explain --user-id u003 --hr hr/hr_sample.csv --policies policies/policies.yaml

# Apply desired (merged) entitlements to the local provider (a JSON file simulating a directory)
python src/main.py apply --hr hr/hr_sample.csv --policies policies/policies.yaml --provider local --state ./local_directory.json --out ./reports
