# 🔐 RBAC & ABAC Automation Project

A complete, portfolio‑ready IAM automation project that assigns access using **RBAC** (role‑based) and **ABAC** (attribute‑based) rules pulled from HR data. Includes a CLI, policy files, unit tests, and an optional Flask API for real‑time authorization decisions.
---

## 📁 Repo Structure

```
rbac-abac-automation/
├─ README.md
├─ requirements.txt
├─ .env.example
├─ .gitignore
├─ pyproject.toml                    # optional; uses setuptools if preferred
├─ .github/
│  └─ workflows/ci.yml              # pytest + flake8
├─ data/
│  ├─ users.csv
│  └─ entitlements.csv
├─ policies/
│  ├─ roles.yaml                    # RBAC role → entitlements
│  ├─ mapping.yaml                  # job_title/department → roles
│  └─ abac_policies.yaml            # ABAC policies (conditions → allow)
├─ src/
│  ├─ __init__.py
│  ├─ models.py
│  ├─ hr_loader.py
│  ├─ rbac.py
│  ├─ abac.py
│  ├─ assign.py                     # CLI: generate RBAC/ABAC decisions & exports
│  └─ policy_eval_api.py            # Flask API for realtime decisions
├─ exporters/
│  └─ okta_export.py                # example: build group assignment payloads
└─ tests/
   ├─ test_rbac.py
   └─ test_abac.py
```

---

## 🚀 Quickstart

```bash
# 1) Create and activate a virtual env
python3 -m venv .venv && source .venv/bin/activate

# 2) Install deps
pip install -r requirements.txt

# 3) Run unit tests
pytest -q

# 4) Generate assignments & decisions from HR data
python -m src.assign --users data/users.csv \
  --roles policies/roles.yaml --mapping policies/mapping.yaml \
  --abac policies/abac_policies.yaml --out out/

# 5) Start the decision API (optional)
export FLASK_APP=src/policy_eval_api.py
flask run -p 5050
# Then POST to http://localhost:5050/decision
```

**Outputs** land in `out/`:

* `rbac_assignments.csv` – user → roles → entitlements
* `abac_decisions.csv` – user/resource decisions
* `okta_groups.json` – sample group payloads (per role) for downstream provisioning

---

## 🧩 Policies & Data Samples

### `policies/roles.yaml`

```yaml
# Role → entitlements
roles:
  Sales.Rep:
    description: Field seller role
    entitlements:
      - app:salesforce:standard
      - group:okta:sales_users
  Sales.Manager:
    description: Sales manager with approvals
    entitlements:
      - app:salesforce:manager
      - group:okta:sales_mgrs
  IT.Support:
    description: Helpdesk level 1
    entitlements:
      - app:jira:agent
      - group:okta:it_support
```

### `policies/mapping.yaml`

```yaml
# HR attributes → RBAC roles
mappings:
  - when:
      department: Sales
      job_title: ^(Account Executive|Sales Representative)$
    assign_roles:
      - Sales.Rep
  - when:
      department: Sales
      job_title: ^(Sales Manager|Regional Manager)$
    assign_roles:
      - Sales.Manager
  - when:
      department: IT
      job_title: ^(Helpdesk|Support Specialist)$
    assign_roles:
      - IT.Support
```

### `policies/abac_policies.yaml`

```yaml
# Simple ABAC rules using safe expressions on user & resource attributes
# evaluation order: first match wins; default = deny
policies:
  - name: Allow Salesforce Standard to Sales users
    effect: allow
    when: |
      resource.app == 'salesforce' and resource.permission == 'standard' and user.department == 'Sales'

  - name: Allow Salesforce Manager to Sales managers only
    effect: allow
    when: |
      resource.app == 'salesforce' and resource.permission == 'manager' and 'Manager' in user.job_title

  - name: Allow Jira Agent to IT Support in US during business hours
    effect: allow
    when: |
      resource.app == 'jira' and resource.permission == 'agent'
      and user.department == 'IT' and 'Support' in user.job_title
      and user.country == 'US' and (8 <= ctx.hour < 18)
```

### `data/users.csv`

```csv
user_id,first_name,last_name,email,department,job_title,country,employment_type
u001,Ada,Lovelace,ada@example.com,Sales,Sales Representative,US,Full-Time
u002,Alan,Turing,alan@example.com,Sales,Sales Manager,US,Full-Time
u003,Grace,Hopper,grace@example.com,IT,Support Specialist,US,Contractor
u004,Linus,Torvalds,linus@example.com,Engineering,Staff Engineer,FI,Full-Time
```

### `data/entitlements.csv` (optional reference)

```csv
entitlement_id,type,value,description
E1,app,salesforce:standard,Salesforce standard license
E2,app,salesforce:manager,Salesforce manager license
E3,app,jira:agent,Jira service desk agent
E4,group,okta:sales_users,Okta group for Sales users
E5,group,okta:sales_mgrs,Okta group for Sales managers
E6,group,okta:it_support,Okta group for IT support
```

---

## 🧱 Code

### `requirements.txt`

```txt
Flask==3.0.3
PyYAML==6.0.2
pydantic==2.8.2
python-dotenv==1.0.1
pandas==2.2.2
pytest==8.3.2
flake8==7.1.1
```

### `.env.example`

```dotenv
# used by API for demo
TZ=America/New_York
```

### `.gitignore`

```gitignore
.venv/
__pycache__/
*.pyc
out/
.env
```

### `pyproject.toml` (optional)

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[tool.pytest.ini_options]
pythonpath = ["."]
```

### `src/__init__.py`

```python
__all__ = []
```

### `src/models.py`

```python
from pydantic import BaseModel
from typing import List, Dict, Any

class User(BaseModel):
    user_id: str
    first_name: str
    last_name: str
    email: str
    department: str
    job_title: str
    country: str | None = None
    employment_type: str | None = None

class Resource(BaseModel):
    app: str
    permission: str

class DecisionContext(BaseModel):
    hour: int = 12  # 0-23, caller sets based on runtime

class Assignment(BaseModel):
    user_id: str
    roles: List[str]
    entitlements: List[str]

class PolicyDecision(BaseModel):
    user_id: str
    resource: Dict[str, Any]
    effect: str  # allow|deny
    matched_policy: str | None = None
```

### `src/hr_loader.py`

```python
import csv
from typing import List
from .models import User


def load_users_csv(path: str) -> List[User]:
    users: List[User] = []
    with open(path, newline="") as f:
        reader = csv.DictReader(f)
        for row in reader:
            users.append(User(**row))
    return users
```

### `src/rbac.py`

```python
import re
import yaml
from dataclasses import dataclass
from typing import Dict, List
from .models import User, Assignment

@dataclass
class RoleConfig:
    roles: Dict[str, Dict]
    mappings: List[Dict]


def load_roles(path_roles: str, path_mapping: str) -> RoleConfig:
    with open(path_roles) as fr:
        roles = yaml.safe_load(fr)["roles"]
    with open(path_mapping) as fm:
        mappings = yaml.safe_load(fm)["mappings"]
    return RoleConfig(roles=roles, mappings=mappings)


def match_mapping(user: User, mapping: Dict) -> bool:
    cond = mapping.get("when", {})
    for k, v in cond.items():
        uval = getattr(user, k, None) or ""
        if v.startswith("^"):
            if not re.search(v, uval):
                return False
        else:
            if uval != v:
                return False
    return True


def assign_roles(user: User, cfg: RoleConfig) -> Assignment:
    roles: List[str] = []
    ents: List[str] = []
    for m in cfg.mappings:
        if match_mapping(user, m):
            roles.extend(m.get("assign_roles", []))
    # dedupe roles
    roles = sorted(set(roles))
    for r in roles:
        ents.extend(cfg.roles.get(r, {}).get("entitlements", []))
    return Assignment(user_id=user.user_id, roles=roles, entitlements=sorted(set(ents)))
```

### `src/abac.py`

```python
import ast
import yaml
from typing import Any, Dict, List
from .models import User, Resource, DecisionContext, PolicyDecision

ALLOWED_NODES = (
    ast.Module, ast.Expr, ast.BoolOp, ast.BinOp, ast.UnaryOp, ast.Compare,
    ast.Name, ast.Load, ast.Constant, ast.And, ast.Or, ast.Not, ast.Eq, ast.NotEq,
    ast.Lt, ast.LtE, ast.Gt, ast.GtE, ast.In, ast.Add, ast.Sub, ast.Mult, ast.Div,
    ast.Attribute, ast.Call
)
ALLOWED_CALLS = {"len": len}


def safe_eval(expr: str, env: Dict[str, Any]) -> bool:
    tree = ast.parse(expr, mode="exec")
    for node in ast.walk(tree):
        if not isinstance(node, ALLOWED_NODES):
            raise ValueError(f"Disallowed syntax: {type(node).__name__}")
        if isinstance(node, ast.Call):
            if not isinstance(node.func, ast.Name) or node.func.id not in ALLOWED_CALLS:
                raise ValueError("Only whitelisted calls allowed")
    compiled = compile(tree, "<policy>", "exec")
    # execute expression and capture last Expr value
    local_env = {**ALLOWED_CALLS, **env}
    result = None
    exec(compiled, {}, local_env)
    # find the last expression value; require expression to assign to _result
    # Simpler: expect expression returns bool via eval mode
    return bool(eval(expr, {}, local_env))


def load_policies(path: str) -> List[Dict[str, Any]]:
    with open(path) as f:
        data = yaml.safe_load(f)
    return data.get("policies", [])


def decide(user: User, resource: Resource, ctx: DecisionContext, policies: List[Dict[str, Any]]) -> PolicyDecision:
    env = {
        "user": user.model_dump(),
        "resource": resource.model_dump(),
        "ctx": ctx.model_dump(),
    }
    for p in policies:
        expr = p.get("when", "")
        try:
            if safe_eval(expr, env):
                return PolicyDecision(
                    user_id=user.user_id,
                    resource=resource.model_dump(),
                    effect=p.get("effect", "deny"),
                    matched_policy=p.get("name")
                )
        except Exception:
            continue
    return PolicyDecision(user_id=user.user_id, resource=resource.model_dump(), effect="deny", matched_policy=None)
```

### `exporters/okta_export.py`

```python
import json
from pathlib import Path
from typing import Dict, List
from dataclasses import dataclass

@dataclass
class GroupMapping:
    role: str
    group: str


def build_okta_group_payloads(role_to_groups: Dict[str, List[str]]) -> Dict[str, dict]:
    payloads = {}
    for role, groups in role_to_groups.items():
        payloads[role] = {
            "role": role,
            "okta_groups": groups
        }
    return payloads


def write_payloads(payloads: Dict[str, dict], out_dir: str) -> None:
    Path(out_dir).mkdir(parents=True, exist_ok=True)
    p = Path(out_dir) / "okta_groups.json"
    with open(p, "w") as f:
        json.dump(payloads, f, indent=2)
```

### `src/assign.py`

```python
import argparse
import csv
from pathlib import Path
from typing import List, Dict
from datetime import datetime
from .models import User, Resource, DecisionContext
from .hr_loader import load_users_csv
from .rbac import load_roles, assign_roles
from .abac import load_policies, decide
from exporters.okta_export import build_okta_group_payloads, write_payloads


def write_csv(path: Path, rows: List[Dict]):
    if not rows:
        return
    path.parent.mkdir(parents=True, exist_ok=True)
    with open(path, "w", newline="") as f:
        writer = csv.DictWriter(f, fieldnames=list(rows[0].keys()))
        writer.writeheader()
        for r in rows:
            writer.writerow(r)


def main():
    ap = argparse.ArgumentParser()
    ap.add_argument("--users", required=True)
    ap.add_argument("--roles", required=True)
    ap.add_argument("--mapping", required=True)
    ap.add_argument("--abac", required=True)
    ap.add_argument("--out", default="out/")
    args = ap.parse_args()

    users: List[User] = load_users_csv(args.users)
    cfg = load_roles(args.roles, args.mapping)
    policies = load_policies(args.abac)

    rbac_rows = []
    role_to_groups: Dict[str, List[str]] = {}

    for u in users:
        asn = assign_roles(u, cfg)
        rbac_rows.append({
            "user_id": u.user_id,
            "email": u.email,
            "roles": ";".join(asn.roles),
            "entitlements": ";".join(asn.entitlements)
        })
        for role in asn.roles:
            groups = [e for e in asn.entitlements if e.startswith("group:")]
            role_to_groups.setdefault(role, sorted(set(groups)))

    write_csv(Path(args.out) / "rbac_assignments.csv", rbac_rows)
    payloads = build_okta_group_payloads(role_to_groups)
    write_payloads(payloads, args.out)

    # ABAC demo: evaluate a small resource matrix
    resources = [
        Resource(app="salesforce", permission="standard"),
        Resource(app="salesforce", permission="manager"),
        Resource(app="jira", permission="agent"),
    ]
    now = datetime.now()
    ctx = DecisionContext(hour=now.hour)
    abac_rows = []
    for u in users:
        for r in resources:
            d = decide(u, r, ctx, policies)
            abac_rows.append({
                "user_id": u.user_id,
                "email": u.email,
                "resource": f"{r.app}:{r.permission}",
                "effect": d.effect,
                "matched_policy": d.matched_policy or "<none>"
            })
    write_csv(Path(args.out) / "abac_decisions.csv", abac_rows)


if __name__ == "__main__":
    main()
```

### `src/policy_eval_api.py`

```python
from flask import Flask, request, jsonify
from datetime import datetime
from .models import User, Resource, DecisionContext
from .abac import load_policies, decide

app = Flask(__name__)
policies = None

@app.before_first_request
def _load():
    global policies
    policies = load_policies("policies/abac_policies.yaml")

@app.post("/decision")
def decision():
    data = request.get_json(force=True)
    user = User(**data["user"])  # expect user dict
    resource = Resource(**data["resource"])  # {app, permission}
    hour = data.get("ctx", {}).get("hour", datetime.now().hour)
    ctx = DecisionContext(hour=hour)
    d = decide(user, resource, ctx, policies)
    return jsonify(d.model_dump())

@app.get("/health")
def health():
    return {"status": "ok"}

if __name__ == "__main__":
    app.run(port=5050)
```

---

## ✅ Tests

### `tests/test_rbac.py`

```python
from src.rbac import load_roles, assign_roles
from src.models import User

def test_assign_sales_rep(tmp_path):
    cfg = load_roles("policies/roles.yaml", "policies/mapping.yaml")
    u = User(user_id="u001", first_name="Ada", last_name="Lovelace", email="ada@example.com",
             department="Sales", job_title="Sales Representative", country="US", employment_type="Full-Time")
    asn = assign_roles(u, cfg)
    assert "Sales.Rep" in asn.roles
    assert any(e.startswith("app:salesforce:standard") for e in asn.entitlements)
```

### `tests/test_abac.py`

```python
from src.abac import load_policies, decide
from src.models import User, Resource
from src.models import DecisionContext

def test_allow_salesforce_standard_for_sales():
    policies = load_policies("policies/abac_policies.yaml")
    user = User(user_id="u001", first_name="Ada", last_name="Lovelace", email="ada@example.com",
                department="Sales", job_title="Sales Representative", country="US", employment_type="Full-Time")
    res = Resource(app="salesforce", permission="standard")
    d = decide(user, res, DecisionContext(hour=10), policies)
    assert d.effect == "allow"
```

---

## 🧪 GitHub Actions CI

### `.github/workflows/ci.yml`

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - run: python -m pip install --upgrade pip
      - run: pip install -r requirements.txt
      - run: flake8 src
      - run: pytest -q
```

---

## 🧰 How to upload to GitHub

1. Create the repository on GitHub (name it **rbac-abac-automation**)
2. On your machine:

```bash
git init
 git add .
 git commit -m "feat: RBAC & ABAC automation initial commit"
 git branch -M main
 git remote add origin https://github.com/<your-username>/rbac-abac-automation.git
 git push -u origin main
```

---

## 📝 What you learned / Talking points

* **RBAC:** deterministic, easy to audit; uses job\_title/department mapping to assign roles → entitlements
* **ABAC:** dynamic decisions from attributes (e.g., country, time, employment\_type). Great for **context‑aware** access.
* **Zero Trust flavor:** decisions depend on context (`ctx.hour`), not just static groups.
* **Engineering rigor:** config‑driven policies (YAML), unit tests, CI, and clean exports for IdP provisioning.

---

## 🔄 Extensions you can add later

* Write to Okta via API (create groups, add users)
* Add risk score to ABAC context (device posture, geolocation)
* Add PBAC examples (permissions derived from relationships/graph)
* Replace safe\_eval with a full CEL/Rego evaluator if desired
