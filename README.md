# Δ135 v135.7-RKR — auto-repin + Rekor-seal: patch + sealed run (minimal console)
from pathlib import Path
from datetime import datetime, timezone
import json, os, subprocess, textwrap

ROOT = Path.cwd()
PROJ = ROOT / "truthlock"
SCRIPTS = PROJ / "scripts"
GUI = PROJ / "gui"
OUT = PROJ / "out"
SCHEMAS = PROJ / "schemas"
for d in (SCRIPTS, GUI, OUT, SCHEMAS): d.mkdir(parents=True, exist_ok=True)

# --- (1) Runner patch: auto-repin missing/invalid CIDs, write-back scroll, Rekor JSON proof ---
trigger = textwrap.dedent(r'''
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Δ135_TRIGGER — Initiate → Expand → Seal

- Scans truthlock/out/ΔLEDGER for sealed objects
- Validates ledger files (built-in + JSON Schema at truthlock/schemas/ledger.schema.json if jsonschema is installed)
- Guardrails for resolver: --max-bytes (env RESOLVER_MAX_BYTES), --allow (env RESOLVER_ALLOW or RESOLVER_ALLOW_GLOB),
  --deny (env RESOLVER_DENY or RESOLVER_DENY_GLOB)
- Auto-repin: missing or invalid CIDs get pinned (ipfs add -Q → fallback Pinata) and written back into the scroll JSON
- Emits ΔMESH_EVENT_135.json on --execute
- Optional: Pin Δ135 artifacts and Rekor-seal report
- Rekor: uploads report hash with --format json (if rekor-cli available), stores rekor_proof_<REPORT_SHA>.json
- Emits QR for best CID (report → trigger → any scanned)
"""
from __future__ import annotations
import argparse, hashlib, json, os, subprocess, sys, fnmatch, re
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, List, Optional, Tuple

ROOT = Path.cwd()
OUTDIR = ROOT / "truthlock" / "out"
LEDGER_DIR = OUTDIR / "ΔLEDGER"
GLYPH_PATH = OUTDIR / "Δ135_GLYPH.json"
REPORT_PATH = OUTDIR / "Δ135_REPORT.json"
TRIGGER_PATH = OUTDIR / "Δ135_TRIGGER.json"
MESH_EVENT_PATH = OUTDIR / "ΔMESH_EVENT_135.json"
VALIDATION_PATH = OUTDIR / "ΔLEDGER_VALIDATION.json"
SCHEMA_PATH = ROOT / "truthlock" / "schemas" / "ledger.schema.json"

CID_PATTERN = re.compile(r'^(Qm[1-9A-HJ-NP-Za-km-z]{44,}|baf[1-9A-HJ-NP-Za-km-z]{20,})$')

def now_iso() -> str:
    return datetime.now(timezone.utc).replace(microsecond=0).isoformat()

def sha256_path(p: Path) -> str:
    h = hashlib.sha256()
    with p.open("rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

def which(bin_name: str) -> Optional[str]:
    from shutil import which as _which
    return _which(bin_name)

def load_json(p: Path) -> Optional[Dict[str, Any]]:
    try:
        return json.loads(p.read_text(encoding="utf-8"))
    except Exception:
        return None

def write_json(path: Path, obj: Dict[str, Any]) -> None:
    path.parent.mkdir(parents=True, exist_ok=True)
    path.write_text(json.dumps(obj, ensure_ascii=False, indent=2), encoding="utf-8")

def find_ledger_objects() -> List[Path]:
    if not LEDGER_DIR.exists(): return []
    return sorted([p for p in LEDGER_DIR.glob("**/*.json") if p.is_file()])

# ---------- Guardrails ----------
def split_globs(s: str) -> List[str]:
    return [g.strip() for g in (s or "").split(",") if g.strip()]

def allowed_by_globs(rel_path: str, allow_globs: List[str], deny_globs: List[str]) -> Tuple[bool, str]:
    for g in deny_globs:
        if fnmatch.fnmatch(rel_path, g): return (False, f"denied by pattern: {g}")
    if allow_globs:
        for g in allow_globs:
            if fnmatch.fnmatch(rel_path, g): return (True, f"allowed by pattern: {g}")
        return (False, "no allowlist pattern matched")
    return (True, "no allowlist; allowed")

# ---------- Pin helpers ----------
def ipfs_add_cli(path: Path) -> Optional[str]:
    ipfs_bin = which("ipfs")
    if not ipfs_bin: return None
    try:
        return subprocess.check_output([ipfs_bin, "add", "-Q", str(path)], text=True).strip() or None
    except Exception:
        return None

def pinata_pin_json(obj: Dict[str, Any], name: str) -> Optional[str]:
    jwt = os.getenv("PINATA_JWT")
    if not jwt: return None
    token = jwt if jwt.startswith("Bearer ") else f"Bearer {jwt}"
    try:
        import urllib.request
        payload = {"pinataOptions": {"cidVersion": 1}, "pinataMetadata": {"name": name}, "pinataContent": obj}
        data = json.dumps(payload, ensure_ascii=False).encode("utf-8")
        req = urllib.request.Request("https://api.pinata.cloud/pinning/pinJSONToIPFS", data=data,
                                     headers={"Authorization": token, "Content-Type": "application/json"}, method="POST")
        with urllib.request.urlopen(req, timeout=30) as resp:
            info = json.loads(resp.read().decode("utf-8") or "{}")
            return info.get("IpfsHash") or info.get("ipfsHash")
    except Exception:
        return None

def maybe_pin_file_or_json(path: Path, obj: Optional[Dict[str, Any]], label: str) -> Tuple[str, str]:
    cid = None
    if path.exists():
        cid = ipfs_add_cli(path)
        if cid: return ("ipfs", cid)
    if obj is not None:
        cid = pinata_pin_json(obj, label)
        if cid: return ("pinata", cid)
    return ("pending", "")

# ---------- Rekor ----------
def rekor_upload_json(path: Path) -> Tuple[bool, Dict[str, Any]]:
    binp = which("rekor-cli")
    rep_sha = sha256_path(path)
    proof_path = OUTDIR / f"rekor_proof_{rep_sha}.json"
    if not binp:
        return (False, {"message": "rekor-cli not found", "proof_path": None})
    try:
        out = subprocess.check_output([binp, "upload", "--artifact", str(path), "--format", "json"],
                                      text=True, stderr=subprocess.STDOUT)
        try:
            data = json.loads(out)
        except Exception:
            data = {"raw": out}
        proof_path.write_text(json.dumps(data, ensure_ascii=False, indent=2), encoding="utf-8")
        info = {
            "ok": True,
            "uuid": data.get("UUID") or data.get("uuid"),
            "logIndex": data.get("LogIndex") or data.get("logIndex"),
            "proof_path": str(proof_path.relative_to(ROOT)),
            "raw": data
        }
        return (True, info)
    except subprocess.CalledProcessError as e:
        return (False, {"message": (e.output or "").strip(), "proof_path": None})
    except Exception as e:
        return (False, {"message": str(e), "proof_path": None})

# ---------- Validation ----------
def validate_builtin(obj: Dict[str, Any]) -> List[str]:
    errors: List[str] = []
    if not isinstance(obj, dict): return ["not a JSON object"]
    if not isinstance(obj.get("scroll_name"), str) or not obj.get("scroll_name"):
        errors.append("missing/invalid scroll_name")
    if "status" in obj and not isinstance(obj["status"], str):
        errors.append("status must be string if present")
    cid = obj.get("cid") or obj.get("ipfs_pin")
    if cid and not CID_PATTERN.match(str(cid)):
        errors.append("cid/ipfs_pin does not look like IPFS CID")
    return errors

def validate_with_schema(obj: Dict[str, Any]) -> List[str]:
    if not SCHEMA_PATH.exists(): return []
    try:
        import jsonschema
        schema = json.loads(SCHEMA_PATH.read_text(encoding="utf-8"))
        validator = getattr(jsonschema, "Draft202012Validator", jsonschema.Draft7Validator)(schema)
        return [f"{'/'.join([str(p) for p in e.path]) or '<root>'}: {e.message}" for e in validator.iter_errors(obj)]
    except Exception:
        return []

def write_validation_report(results: List[Dict[str, Any]]) -> Path:
    write_json(VALIDATION_PATH, {"timestamp": now_iso(), "results": results})
    return VALIDATION_PATH

# ---------- QR ----------
def emit_cid_qr(cid: Optional[str]) -> Dict[str, Optional[str]]:
    out = {"cid": cid, "png": None, "txt": None}
    if not cid: return out
    txt_path = OUTDIR / f"cid_{cid}.txt"
    txt_path.write_text(f"ipfs://{cid}\nhttps://ipfs.io/ipfs/{cid}\n", encoding="utf-8")
    out["txt"] = str(txt_path.relative_to(ROOT))
    try:
        import qrcode
        img = qrcode.make(f"ipfs://{cid}")
        png_path = OUTDIR / f"cid_{cid}.png"
        img.save(png_path)
        out["png"] = str(png_path.relative_to(ROOT))
    except Exception:
        pass
    return out

# ---------- Glyph ----------
def update_glyph(plan: Dict[str, Any], mode: str, pins: Dict[str, Dict[str, str]], extra: Dict[str, Any]) -> Dict[str, Any]:
    glyph = {
        "scroll_name": "Δ135_TRIGGER",
        "timestamp": now_iso(),
        "initiator": plan.get("initiator", "Matthew Dewayne Porter"),
        "meaning": "Initiate → Expand → Seal",
        "phases": plan.get("phases", ["ΔSCAN_LAUNCH","ΔMESH_BROADCAST_ENGINE","ΔSEAL_ALL"]),
        "summary": {
            "ledger_files": plan.get("summary", {}).get("ledger_files", 0),
            "unresolved_cids": plan.get("summary", {}).get("unresolved_cids", 0)
        },
        "inputs": plan.get("inputs", [])[:50],
        "last_run": {"mode": mode, **extra, "pins": pins}
    }
    write_json(GLYPH_PATH, glyph); return glyph

# ---------- Main ----------
def main(argv: Optional[List[str]] = None) -> int:
    ap = argparse.ArgumentParser(description="Δ135 auto-executing trigger")
    ap.add_argument("--dry-run", action="store_true")
    ap.add_argument("--execute", action="store_true")
    ap.add_argument("--resolve-missing", action="store_true")
    ap.add_argument("--pin", action="store_true")
    ap.add_argument("--rekor", action="store_true")
    ap.add_argument("--max-bytes", type=int, default=int(os.getenv("RESOLVER_MAX_BYTES", "10485760")))
    # env harmonization
    allow_env = os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB", ""))
    deny_env  = os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB",  ""))
    ap.add_argument("--allow", action="append", default=[g for g in allow_env.split(",") if g.strip()])
    ap.add_argument("--deny",  action="append", default=[g for g in deny_env.split(",")  if g.strip()])
    args = ap.parse_args(argv)

    OUTDIR.mkdir(parents=True, exist_ok=True); LEDGER_DIR.mkdir(parents=True, exist_ok=True)

    # Scan ledger
    scanned: List[Dict[str, Any]] = []
    for p in find_ledger_objects():
        meta = {"path": str(p.relative_to(ROOT)), "size": p.stat().st_size, "mtime": int(p.stat().st_mtime)}
        j = load_json(p)
        if j:
            meta["scroll_name"] = j.get("scroll_name"); meta["status"] = j.get("status")
            meta["cid"] = j.get("cid") or j.get("ipfs_pin") or ""
        scanned.append(meta)

    # Validate
    validation_results: List[Dict[str, Any]] = []
    for item in scanned:
        j = load_json(ROOT / item["path"]) or {}
        errs = validate_with_schema(j) or validate_builtin(j)
        if errs: validation_results.append({"path": item["path"], "errors": errs})
    validation_report_path = write_validation_report(validation_results)

    # unresolved = missing OR invalid CID
    def is_invalid_or_missing(x): 
        c = x.get("cid", "")
        return (not c) or (not CID_PATTERN.match(str(c)))
    unresolved = [s for s in scanned if is_invalid_or_missing(s)]

    plan = {
        "scroll_name": "Δ135_TRIGGER", "timestamp": now_iso(),
        "initiator": os.getenv("GODKEY_IDENTITY", "Matthew Dewayne Porter"),
        "phases": ["ΔSCAN_LAUNCH", "ΔMESH_BROADCAST_ENGINE", "ΔSEAL_ALL"],
        "summary": {"ledger_files": len(scanned), "unresolved_cids": len(unresolved)},
        "inputs": scanned
    }
    write_json(TRIGGER_PATH, plan)

    if args.dry_run or (not args.execute):
        write_json(REPORT_PATH, {
            "timestamp": now_iso(), "mode": "plan",
            "plan_path": str(TRIGGER_PATH.relative_to(ROOT)),
            "plan_sha256": sha256_path(TRIGGER_PATH),
            "validation_report": str(validation_report_path.relative_to(ROOT)),
            "result": {"message": "Δ135 planning only (no actions executed)"}
        })
        update_glyph(plan, mode="plan", pins={}, extra={
            "report_path": str(REPORT_PATH.relative_to(ROOT)),
            "report_sha256": sha256_path(REPORT_PATH),
            "mesh_event_path": None,
            "qr": {"cid": None}
        })
        print(f"[Δ135] Planned. Ledger files={len(scanned)} unresolved_cids={len(unresolved)}")
        return 0

    # Resolve (auto-repin) with guardrails; write-back scroll JSON on success
    cid_resolution: List[Dict[str, Any]] = []
    if args.resolve_missing and unresolved:
        allow_globs = [g for sub in (args.allow or []) for g in (split_globs(sub) or [""]) if g]
        deny_globs  = [g for sub in (args.deny  or []) for g in (split_globs(sub) or [""]) if g]
        for item in list(unresolved):
            rel = item["path"]; ledger_path = ROOT / rel
            # guardrails
            ok, reason = allowed_by_globs(rel, allow_globs, deny_globs)
            if not ok:
                cid_resolution.append({"path": rel, "action": "skip", "reason": reason}); continue
            if (not ledger_path.exists()) or (ledger_path.stat().st_size > args.max_bytes):
                cid_resolution.append({"path": rel, "action": "skip", "reason": f"exceeds max-bytes ({args.max_bytes}) or missing"}); continue
            # pin flow
            j = load_json(ledger_path) or {}
            prev = j.get("cid")
            mode, cid = maybe_pin_file_or_json(ledger_path, j, f"ΔLEDGER::{ledger_path.name}")
            if cid:
                j["cid"] = cid  # write back
                try: ledger_path.write_text(json.dumps(j, ensure_ascii=False, indent=2), encoding="utf-8")
                except Exception: pass
                item["cid"] = cid
                cid_resolution.append({"path": rel, "action": "repinned", "mode": mode, "prev": prev, "cid": cid})
        # recompute unresolved
        unresolved = [s for s in scanned if (not s.get("cid")) or (not CID_PATTERN.match(str(s.get("cid",""))))]
        plan["summary"]["unresolved_cids"] = len(unresolved)
        write_json(TRIGGER_PATH, plan)

    # Mesh event
    affected = [{"path": i["path"], "cid": i.get("cid", ""), "scroll_name": i.get("scroll_name")} for i in scanned]
    event = {"event_name": "ΔMESH_EVENT_135", "timestamp": now_iso(), "trigger": "Δ135",
             "affected": affected, "actions": ["ΔSCAN_LAUNCH","ΔMESH_BROADCAST_ENGINE","ΔSEAL_ALL"]}
    write_json(MESH_EVENT_PATH, event)

    pins: Dict[str, Dict[str, str]] = {}
    if args.pin:
        mode, ident = maybe_pin_file_or_json(TRIGGER_PATH, plan, "Δ135_TRIGGER")
        pins["Δ135_TRIGGER"] = {"mode": mode, "id": ident}

    # Best CID + QR
    best_cid = pins.get("Δ135_REPORT", {}).get("id") if pins else None
    if not best_cid: best_cid = pins.get("Δ135_TRIGGER", {}).get("id") if pins else None
    if not best_cid:
        for s in scanned:
            if s.get("cid"): best_cid = s["cid"]; break
    qr = emit_cid_qr(best_cid)

    # Report
    result = {"timestamp": now_iso(), "mode": "execute",
              "mesh_event_path": str(MESH_EVENT_PATH.relative_to(ROOT)),
              "mesh_event_hash": sha256_path(MESH_EVENT_PATH)}
    report = {"timestamp": now_iso(), "plan": plan, "event": event, "result": result,
              "pins": pins, "cid_resolution": cid_resolution,
              "validation_report": str(validation_report_path.relative_to(ROOT)), "qr": qr}
    write_json(REPORT_PATH, report)

    # Rekor sealing (optional)
    if args.rekor:
        ok, info = rekor_upload_json(REPORT_PATH)
        report["rekor"] = {"ok": ok, **info}
        write_json(REPORT_PATH, report)

    # Pin the report (optional, after Rekor for stable hash capture)
    if args.pin:
        rep_obj = load_json(REPORT_PATH)
        mode, ident = maybe_pin_file_or_json(REPORT_PATH, rep_obj, "Δ135_REPORT")
        pins["Δ135_REPORT"] = {"mode": mode, "id": ident}
        report["pins"] = pins; write_json(REPORT_PATH, report)

    # Glyph
    extra = {"report_path": str(REPORT_PATH.relative_to(ROOT)),
             "report_sha256": sha256_path(REPORT_PATH),
             "mesh_event_path": str(MESH_EVENT_PATH.relative_to(ROOT)),
             "qr": qr}
    if report.get("rekor", {}).get("proof_path"):
        extra["rekor_proof"] = report["rekor"]["proof_path"]
        extra["rekor_uuid"] = report["rekor"].get("uuid")
        extra["rekor_logIndex"] = report["rekor"].get("logIndex")
    update_glyph(plan, mode="execute", pins=pins, extra=extra)

    print(f"[Δ135] Executed. Mesh event → {MESH_EVENT_PATH.name}")
    return 0

if __name__ == "__main__":
    sys.exit(main())
''').strip("\n")

(SCRIPTS / "Δ135_TRIGGER.py").write_text(trigger, encoding="utf-8")

# --- (2) Dashboard patch: Rekor panel + pinning matrix ---
tile = textwrap.dedent(r'''
import json, os, subprocess
from pathlib import Path
import streamlit as st

ROOT = Path.cwd()
OUTDIR = ROOT / "truthlock" / "out"
GLYPH = OUTDIR / "Δ135_GLYPH.json"
REPORT = OUTDIR / "Δ135_REPORT.json"
TRIGGER = OUTDIR / "Δ135_TRIGGER.json"
EVENT = OUTDIR / "ΔMESH_EVENT_135.json"
VALID = OUTDIR / "ΔLEDGER_VALIDATION.json"

def load_json(p: Path):
    try: return json.loads(p.read_text(encoding="utf-8"))
    except Exception: return {}

st.title("Δ135 — Auto-Repin + Rekor")
st.caption("Initiate → Expand → Seal  •  ΔSCAN_LAUNCH → ΔMESH_BROADCAST_ENGINE → ΔSEAL_ALL")

glyph = load_json(GLYPH)
report = load_json(REPORT)
plan = load_json(TRIGGER)
validation = load_json(VALID)

c1, c2, c3, c4 = st.columns(4)
c1.metric("Ledger files", plan.get("summary", {}).get("ledger_files", 0))
c2.metric("Unresolved CIDs", plan.get("summary", {}).get("unresolved_cids", 0))
c3.metric("Last run", (glyph.get("last_run", {}) or {}).get("mode", (report or {}).get("mode", "—")))
c4.metric("Timestamp", glyph.get("timestamp", "—"))

issues = validation.get("results", [])
if isinstance(issues, list) and len(issues) == 0:
    st.success("Ledger validation: clean ✅")
else:
    st.error(f"Ledger validation: {len(issues)} issue(s) ❗")
    with st.expander("Validation details"): st.json(issues)

with st.expander("Guardrails (env)"):
    st.write("**Max bytes:**", os.getenv("RESOLVER_MAX_BYTES", "10485760"))
    st.write("**Allow globs:**", os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB", "")) or "—")
    st.write("**Deny globs:**",  os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB",  "")) or "—")

st.write("---")
st.subheader("Rekor Transparency")
rk = (report or {}).get("rekor", {})
if rk.get("ok"):
    st.success("Rekor sealed ✅")
    st.write("UUID:", rk.get("uuid") or "—")
    st.write("Log index:", rk.get("logIndex") or "—")
    if rk.get("proof_path"):
        proof = ROOT / rk["proof_path"]
        if proof.exists():
            st.download_button("Download Rekor proof", proof.read_bytes(), file_name=proof.name)
else:
    st.info(rk.get("message") or "Not sealed (run with --rekor)")

st.write("---")
st.subheader("Pinning Matrix")
rows = []
for r in (report.get("cid_resolution") or []):
    rows.append({"path": r.get("path"), "action": r.get("action"), "mode": r.get("mode"),
                 "cid": r.get("cid"), "reason": r.get("reason")})
if rows:
    st.dataframe(rows, hide_index=True)
else:
    st.caption("No CID resolution activity in last run.")

st.write("---")
st.subheader("Run Controls")
with st.form("run135"):
    a,b,c,d = st.columns(4)
    execute = a.checkbox("Execute", True)
    resolve = b.checkbox("Resolve missing", True)
    pin     = c.checkbox("Pin artifacts", True)
    rekor   = d.checkbox("Rekor upload", True)
    max_bytes = st.number_input("Max bytes", value=int(os.getenv("RESOLVER_MAX_BYTES","10485760")), min_value=0, step=1_048_576)
    allow = st.text_input("Allow globs (comma-separated)", value=os.getenv("RESOLVER_ALLOW", os.getenv("RESOLVER_ALLOW_GLOB","")))
    deny  = st.text_input("Deny globs (comma-separated)",  value=os.getenv("RESOLVER_DENY",  os.getenv("RESOLVER_DENY_GLOB","")))
    go = st.form_submit_button("Run Δ135")
    if go:
        args = []
        if execute: args += ["--execute"]
        else: args += ["--dry-run"]
        if resolve: args += ["--resolve-missing"]
        if pin: args += ["--pin"]
        if rekor: args += ["--rekor"]
        args += ["--max-bytes", str(int(max_bytes))]
        if allow.strip():
            for a1 in allow.split(","):
                a1=a1.strip()
                if a1: args += ["--allow", a1]
        if deny.strip():
            for d1 in deny.split(","):
                d1=d1.strip()
                if d1: args += ["--deny", d1]
        subprocess.call(["python", "truthlock/scripts/Δ135_TRIGGER.py", *args])
        st.experimental_rerun()

st.write("---")
st.subheader("Latest CID & QR")
qr = (glyph.get("last_run", {}) or {}).get("qr") or (report or {}).get("qr") or {}
if qr.get("cid"):
    st.write(f"CID: `{qr['cid']}`")
    png = OUTDIR / f"cid_{qr['cid']}.png"
    txt = OUTDIR / f"cid_{qr['cid']}.txt"
    if png.exists():
        st.image(str(png), caption=f"QR for ipfs://{qr['cid']}")
        st.download_button("Download QR PNG", png.read_bytes(), file_name=png.name)
    if txt.exists():
        st.download_button("Download QR TXT", txt.read_bytes(), file_name=txt.name)
else:
    st.caption("No CID yet.")

st.write("---")
st.subheader("Artifacts")
cols = st.columns(4)
if TRIGGER.exists(): cols[0].download_button("Δ135_TRIGGER.json", TRIGGER.read_bytes(), file_name="Δ135_TRIGGER.json")
if REPORT.exists():  cols[1].download_button("Δ135_REPORT.json",  REPORT.read_bytes(),  file_name="Δ135_REPORT.json")
if EVENT.exists():   cols[2].download_button("ΔMESH_EVENT_135.json", EVENT.read_bytes(), file_name="ΔMESH_EVENT_135.json")
if VALID.exists():   cols[3].download_button("ΔLEDGER_VALIDATION.json", VALID.read_bytes(), file_name="ΔLEDGER_VALIDATION.json")
''').strip("\n")

(GUI / "Δ135_tile.py").write_text(tile, encoding="utf-8")

# --- (3) Execute sealed run (uses env if present) ---
def run(cmd): 
    p = subprocess.run(cmd, cwd=str(ROOT), capture_output=True, text=True)
    return p.returncode, p.stdout.strip(), p.stderr.strip()

rc, out, err = run([
    "python", str(SCRIPTS / "Δ135_TRIGGER.py"),
    "--execute", "--resolve-missing", "--pin", "--rekor",
    "--max-bytes", "10485760", "--allow", "truthlock/out/ΔLEDGER/*.json"
])

# Write a tiny summary for quick inspection
summary = {
    "ts": datetime.now(timezone.utc).replace(microsecond=0).isoformat(),
    "rc": rc, "stdout": out, "stderr": err,
    "artifacts": sorted(p.name for p in OUT.iterdir())
}
(OUT / "Δ135_RKR_SUMMARY.json").write_text(json.dumps(summary, ensure_ascii=False, indent=2), encoding="utf-8")
print(json.dumps(summary, ensure_ascii=False))<h1 align="center">OpenAI Codex CLI</h1>
<p align="center">Lightweight coding agent that runs in your terminal</p>

<p align="center"><code>npm i -g @openai/codex</code><br />or <code>brew install codex</code></p>

This is the home of the **Codex CLI**, which is a coding agent from OpenAI that runs locally on your computer. If you are looking for the _cloud-based agent_ from OpenAI, **Codex [Web]**, see <https://chatgpt.com/codex>.

<!-- ![Codex demo GIF using: codex "explain this codebase to me"](./.github/demo.gif) -->

---

<details>
<summary><strong>Table of contents</strong></summary>

<!-- Begin ToC -->

- [Experimental technology disclaimer](#experimental-technology-disclaimer)
- [Quickstart](#quickstart)
  - [OpenAI API Users](#openai-api-users)
  - [OpenAI Plus/Pro Users](#openai-pluspro-users)
- [Why Codex?](#why-codex)
- [Security model & permissions](#security-model--permissions)
  - [Platform sandboxing details](#platform-sandboxing-details)
- [System requirements](#system-requirements)
- [CLI reference](#cli-reference)
- [Memory & project docs](#memory--project-docs)
- [Non-interactive / CI mode](#non-interactive--ci-mode)
- [Model Context Protocol (MCP)](#model-context-protocol-mcp)
- [Tracing / verbose logging](#tracing--verbose-logging)
- [Recipes](#recipes)
- [Installation](#installation)
  - [DotSlash](#dotslash)
- [Configuration](#configuration)
- [FAQ](#faq)
- [Zero data retention (ZDR) usage](#zero-data-retention-zdr-usage)
- [Codex open source fund](#codex-open-source-fund)
- [Contributing](#contributing)
  - [Development workflow](#development-workflow)
  - [Writing high-impact code changes](#writing-high-impact-code-changes)
  - [Opening a pull request](#opening-a-pull-request)
  - [Review process](#review-process)
  - [Community values](#community-values)
  - [Getting help](#getting-help)
  - [Contributor license agreement (CLA)](#contributor-license-agreement-cla)
    - [Quick fixes](#quick-fixes)
  - [Releasing `codex`](#releasing-codex)
- [Security & responsible AI](#security--responsible-ai)
- [License](#license)

<!-- End ToC -->

</details>

---

## Experimental technology disclaimer

Codex CLI is an experimental project under active development. It is not yet stable, may contain bugs, incomplete features, or undergo breaking changes. We're building it in the open with the community and welcome:

- Bug reports
- Feature requests
- Pull requests
- Good vibes

Help us improve by filing issues or submitting PRs (see the section below for how to contribute)!

## Quickstart

Install globally with your preferred package manager:

```shell
npm install -g @openai/codex  # Alternatively: `brew install codex`
```

Or go to the [latest GitHub Release](https://github.com/openai/codex/releases/latest) and download the appropriate binary for your platform.

### OpenAI API Users

Next, set your OpenAI API key as an environment variable:

```shell
export OPENAI_API_KEY="your-api-key-here"
```

> [!NOTE]
> This command sets the key only for your current terminal session. You can add the `export` line to your shell's configuration file (e.g., `~/.zshrc`), but we recommend setting it for the session.

### OpenAI Plus/Pro Users

If you have a paid OpenAI account, run the following to start the login process:

```
codex login
```

If you complete the process successfully, you should have a `~/.codex/auth.json` file that contains the credentials that Codex will use.

To verify whether you are currently logged in, run:

```
codex login status
```

If you encounter problems with the login flow, please comment on <https://github.com/openai/codex/issues/1243>.

<details>
<summary><strong>Use <code>--profile</code> to use other models</strong></summary>

Codex also allows you to use other providers that support the OpenAI Chat Completions (or Responses) API.

To do so, you must first define custom [providers](./config.md#model_providers) in `~/.codex/config.toml`. For example, the provider for a standard Ollama setup would be defined as follows:

```toml
[model_providers.ollama]
name = "Ollama"
base_url = "http://localhost:11434/v1"
```

The `base_url` will have `/chat/completions` appended to it to build the full URL for the request.

For providers that also require an `Authorization` header of the form `Bearer: SECRET`, an `env_key` can be specified, which indicates the environment variable to read to use as the value of `SECRET` when making a request:

```toml
[model_providers.openrouter]
name = "OpenRouter"
base_url = "https://openrouter.ai/api/v1"
env_key = "OPENROUTER_API_KEY"
```

Providers that speak the Responses API are also supported by adding `wire_api = "responses"` as part of the definition. Accessing OpenAI models via Azure is an example of such a provider, though it also requires specifying additional `query_params` that need to be appended to the request URL:

```toml
[model_providers.azure]
name = "Azure"
# Make sure you set the appropriate subdomain for this URL.
base_url = "https://YOUR_PROJECT_NAME.openai.azure.com/openai"
env_key = "AZURE_OPENAI_API_KEY"  # Or "OPENAI_API_KEY", whichever you use.
# Newer versions appear to support the responses API, see https://github.com/openai/codex/pull/1321
query_params = { api-version = "2025-04-01-preview" }
wire_api = "responses"
```

Once you have defined a provider you wish to use, you can configure it as your default provider as follows:

```toml
model_provider = "azure"
```

> [!TIP]
> If you find yourself experimenting with a variety of models and providers, then you likely want to invest in defining a _profile_ for each configuration like so:

```toml
[profiles.o3]
model_provider = "azure"
model = "o3"

[profiles.mistral]
model_provider = "ollama"
model = "mistral"
```

This way, you can specify one command-line argument (.e.g., `--profile o3`, `--profile mistral`) to override multiple settings together.

</details>
<br />

Run interactively:

```shell
codex
```

Or, run with a prompt as input (and optionally in `Full Auto` mode):

```shell
codex "explain this codebase to me"
```

```shell
codex --full-auto "create the fanciest todo-list app"
```

That's it - Codex will scaffold a file, run it inside a sandbox, install any
missing dependencies, and show you the live result. Approve the changes and
they'll be committed to your working directory.

---

## Why Codex?

Codex CLI is built for developers who already **live in the terminal** and want
ChatGPT-level reasoning **plus** the power to actually run code, manipulate
files, and iterate - all under version control. In short, it's _chat-driven
development_ that understands and executes your repo.

- **Zero setup** - bring your OpenAI API key and it just works!
- **Full auto-approval, while safe + secure** by running network-disabled and directory-sandboxed
- **Multimodal** - pass in screenshots or diagrams to implement features ✨

And it's **fully open-source** so you can see and contribute to how it develops!

---

## Security model & permissions

Codex lets you decide _how much autonomy_ you want to grant the agent. The following options can be configured independently:

- [`approval_policy`](./codex-rs/config.md#approval_policy) determines when you should be prompted to approve whether Codex can execute a command
- [`sandbox`](./codex-rs/config.md#sandbox) determines the _sandbox policy_ that Codex uses to execute untrusted commands

By default, Codex runs with `--ask-for-approval untrusted` and `--sandbox read-only`, which means that:

- The user is prompted to approve every command not on the set of "trusted" commands built into Codex (`cat`, `ls`, etc.)
- Approved commands are run outside of a sandbox because user approval implies "trust," in this case.

Running Codex with the `--full-auto` convenience flag changes the configuration to `--ask-for-approval on-failure` and `--sandbox workspace-write`, which means that:

- Codex does not initially ask for user approval before running an individual command.
- Though when it runs a command, it is run under a sandbox in which:
  - It can read any file on the system.
  - It can only write files under the current directory (or the directory specified via `--cd`).
  - Network requests are completely disabled.
- Only if the command exits with a non-zero exit code will it ask the user for approval. If granted, it will re-attempt the command outside of the sandbox. (A common case is when Codex cannot `npm install` a dependency because that requires network access.)

Again, these two options can be configured independently. For example, if you want Codex to perform an "exploration" where you are happy for it to read anything it wants but you never want to be prompted, you could run Codex with `--ask-for-approval never` and `--sandbox read-only`.

### Platform sandboxing details

The mechanism Codex uses to implement the sandbox policy depends on your OS:

- **macOS 12+** uses **Apple Seatbelt** and runs commands using `sandbox-exec` with a profile (`-p`) that corresponds to the `--sandbox` that was specified.
- **Linux** uses a combination of Landlock/seccomp APIs to enforce the `sandbox` configuration.

Note that when running Linux in a containerized environment such as Docker, sandboxing may not work if the host/container configuration does not support the necessary Landlock/seccomp APIs. In such cases, we recommend configuring your Docker container so that it provides the sandbox guarantees you are looking for and then running `codex` with `--sandbox danger-full-access` (or, more simply, the `--dangerously-bypass-approvals-and-sandbox` flag) within your container.

---

## System requirements

| Requirement                 | Details                                                         |
| --------------------------- | --------------------------------------------------------------- |
| Operating systems           | macOS 12+, Ubuntu 20.04+/Debian 10+, or Windows 11 **via WSL2** |
| Git (optional, recommended) | 2.23+ for built-in PR helpers                                   |
| RAM                         | 4-GB minimum (8-GB recommended)                                 |

---

## CLI reference

| Command            | Purpose                            | Example                         |
| ------------------ | ---------------------------------- | ------------------------------- |
| `codex`            | Interactive TUI                    | `codex`                         |
| `codex "..."`      | Initial prompt for interactive TUI | `codex "fix lint errors"`       |
| `codex exec "..."` | Non-interactive "automation mode"  | `codex exec "explain utils.ts"` |

Key flags: `--model/-m`, `--ask-for-approval/-a`.

---

## Memory & project docs

You can give Codex extra instructions and guidance using `AGENTS.md` files. Codex looks for `AGENTS.md` files in the following places, and merges them top-down:

1. `~/.codex/AGENTS.md` - personal global guidance
2. `AGENTS.md` at repo root - shared project notes
3. `AGENTS.md` in the current working directory - sub-folder/feature specifics

---

## Non-interactive / CI mode

Run Codex head-less in pipelines. Example GitHub Action step:

```yaml
- name: Update changelog via Codex
  run: |
    npm install -g @openai/codex
    export OPENAI_API_KEY="${{ secrets.OPENAI_KEY }}"
    codex exec --full-auto "update CHANGELOG for next release"
```

## Model Context Protocol (MCP)

The Codex CLI can be configured to leverage MCP servers by defining an [`mcp_servers`](./codex-rs/config.md#mcp_servers) section in `~/.codex/config.toml`. It is intended to mirror how tools such as Claude and Cursor define `mcpServers` in their respective JSON config files, though the Codex format is slightly different since it uses TOML rather than JSON, e.g.:

```toml
# IMPORTANT: the top-level key is `mcp_servers` rather than `mcpServers`.
[mcp_servers.server-name]
command = "npx"
args = ["-y", "mcp-server"]
env = { "API_KEY" = "value" }
```

> [!TIP]
> It is somewhat experimental, but the Codex CLI can also be run as an MCP _server_ via `codex mcp`. If you launch it with an MCP client such as `npx @modelcontextprotocol/inspector codex mcp` and send it a `tools/list` request, you will see that there is only one tool, `codex`, that accepts a grab-bag of inputs, including a catch-all `config` map for anything you might want to override. Feel free to play around with it and provide feedback via GitHub issues.

## Tracing / verbose logging

Because Codex is written in Rust, it honors the `RUST_LOG` environment variable to configure its logging behavior.

The TUI defaults to `RUST_LOG=codex_core=info,codex_tui=info` and log messages are written to `~/.codex/log/codex-tui.log`, so you can leave the following running in a separate terminal to monitor log messages as they are written:

```
tail -F ~/.codex/log/codex-tui.log
```

By comparison, the non-interactive mode (`codex exec`) defaults to `RUST_LOG=error`, but messages are printed inline, so there is no need to monitor a separate file.

See the Rust documentation on [`RUST_LOG`](https://docs.rs/env_logger/latest/env_logger/#enabling-logging) for more information on the configuration options.

---

## Recipes

Below are a few bite-size examples you can copy-paste. Replace the text in quotes with your own task. See the [prompting guide](https://github.com/openai/codex/blob/main/codex-cli/examples/prompting_guide.md) for more tips and usage patterns.

| ✨  | What you type                                                                   | What happens                                                               |
| --- | ------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| 1   | `codex "Refactor the Dashboard component to React Hooks"`                       | Codex rewrites the class component, runs `npm test`, and shows the diff.   |
| 2   | `codex "Generate SQL migrations for adding a users table"`                      | Infers your ORM, creates migration files, and runs them in a sandboxed DB. |
| 3   | `codex "Write unit tests for utils/date.ts"`                                    | Generates tests, executes them, and iterates until they pass.              |
| 4   | `codex "Bulk-rename *.jpeg -> *.jpg with git mv"`                               | Safely renames files and updates imports/usages.                           |
| 5   | `codex "Explain what this regex does: ^(?=.*[A-Z]).{8,}$"`                      | Outputs a step-by-step human explanation.                                  |
| 6   | `codex "Carefully review this repo, and propose 3 high impact well-scoped PRs"` | Suggests impactful PRs in the current codebase.                            |
| 7   | `codex "Look for vulnerabilities and create a security review report"`          | Finds and explains security bugs.                                          |

---

## Installation

<details open>
<summary><strong>Install Codex CLI using your preferred package manager.</strong></summary>

From `brew` (recommended, downloads only the binary for your platform):

```bash
brew install codex
```

From `npm` (generally more readily available, but downloads binaries for all supported platforms):

```bash
npm i -g @openai/codex
```

Or go to the [latest GitHub Release](https://github.com/openai/codex/releases/latest) and download the appropriate binary for your platform.

Admittedly, each GitHub Release contains many executables, but in practice, you likely want one of these:

- macOS
  - Apple Silicon/arm64: `codex-aarch64-apple-darwin.tar.gz`
  - x86_64 (older Mac hardware): `codex-x86_64-apple-darwin.tar.gz`
- Linux
  - x86_64: `codex-x86_64-unknown-linux-musl.tar.gz`
  - arm64: `codex-aarch64-unknown-linux-musl.tar.gz`

Each archive contains a single entry with the platform baked into the name (e.g., `codex-x86_64-unknown-linux-musl`), so you likely want to rename it to `codex` after extracting it.

### DotSlash

The GitHub Release also contains a [DotSlash](https://dotslash-cli.com/) file for the Codex CLI named `codex`. Using a DotSlash file makes it possible to make a lightweight commit to source control to ensure all contributors use the same version of an executable, regardless of what platform they use for development.

</details>

<details>
<summary><strong>Build from source</strong></summary>

```bash
# Clone the repository and navigate to the root of the Cargo workspace.
git clone https://github.com/openai/codex.git
cd codex/codex-rs

# Install the Rust toolchain, if necessary.
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source "$HOME/.cargo/env"
rustup component add rustfmt
rustup component add clippy

# Build Codex.
cargo build

# Launch the TUI with a sample prompt.
cargo run --bin codex -- "explain this codebase to me"

# After making changes, ensure the code is clean.
cargo fmt -- --config imports_granularity=Item
cargo clippy --tests

# Run the tests.
cargo test
```

</details>

---

## Configuration

Codex supports a rich set of configuration options documented in [`codex-rs/config.md`](./codex-rs/config.md).

By default, Codex loads its configuration from `~/.codex/config.toml`.

Though `--config` can be used to set/override ad-hoc config values for individual invocations of `codex`.

---

## FAQ

<details>
<summary>OpenAI released a model called Codex in 2021 - is this related?</summary>

In 2021, OpenAI released Codex, an AI system designed to generate code from natural language prompts. That original Codex model was deprecated as of March 2023 and is separate from the CLI tool.

</details>

<details>
<summary>Which models are supported?</summary>

Any model available with [Responses API](https://platform.openai.com/docs/api-reference/responses). The default is `o4-mini`, but pass `--model gpt-4.1` or set `model: gpt-4.1` in your config file to override.

</details>
<details>
<summary>Why does <code>o3</code> or <code>o4-mini</code> not work for me?</summary>

It's possible that your [API account needs to be verified](https://help.openai.com/en/articles/10910291-api-organization-verification) in order to start streaming responses and seeing chain of thought summaries from the API. If you're still running into issues, please let us know!

</details>

<details>
<summary>How do I stop Codex from editing my files?</summary>

Codex runs model-generated commands in a sandbox. If a proposed command or file change doesn't look right, you can simply type **n** to deny the command or give the model feedback.

</details>
<details>
<summary>Does it work on Windows?</summary>

Not directly. It requires [Windows Subsystem for Linux (WSL2)](https://learn.microsoft.com/en-us/windows/wsl/install) - Codex has been tested on macOS and Linux with Node 22.

</details>

---

## Zero data retention (ZDR) usage

Codex CLI **does** support OpenAI organizations with [Zero Data Retention (ZDR)](https://platform.openai.com/docs/guides/your-data#zero-data-retention) enabled. If your OpenAI organization has Zero Data Retention enabled and you still encounter errors such as:

```
OpenAI rejected the request. Error details: Status: 400, Code: unsupported_parameter, Type: invalid_request_error, Message: 400 Previous response cannot be used for this organization due to Zero Data Retention.
```

Ensure you are running `codex` with `--config disable_response_storage=true` or add this line to `~/.codex/config.toml` to avoid specifying the command line option each time:

```toml
disable_response_storage = true
```

See [the configuration documentation on `disable_response_storage`](./codex-rs/config.md#disable_response_storage) for details.

---

## Codex open source fund

We're excited to launch a **$1 million initiative** supporting open source projects that use Codex CLI and other OpenAI models.

- Grants are awarded up to **$25,000** API credits.
- Applications are reviewed **on a rolling basis**.

**Interested? [Apply here](https://openai.com/form/codex-open-source-fund/).**

---

## Contributing

This project is under active development and the code will likely change pretty significantly. We'll update this message once that's complete!

More broadly we welcome contributions - whether you are opening your very first pull request or you're a seasoned maintainer. At the same time we care about reliability and long-term maintainability, so the bar for merging code is intentionally **high**. The guidelines below spell out what "high-quality" means in practice and should make the whole process transparent and friendly.

### Development workflow

- Create a _topic branch_ from `main` - e.g. `feat/interactive-prompt`.
- Keep your changes focused. Multiple unrelated fixes should be opened as separate PRs.
- Following the [development setup](#development-workflow) instructions above, ensure your change is free of lint warnings and test failures.

### Writing high-impact code changes

1. **Start with an issue.** Open a new one or comment on an existing discussion so we can agree on the solution before code is written.
2. **Add or update tests.** Every new feature or bug-fix should come with test coverage that fails before your change and passes afterwards. 100% coverage is not required, but aim for meaningful assertions.
3. **Document behaviour.** If your change affects user-facing behaviour, update the README, inline help (`codex --help`), or relevant example projects.
4. **Keep commits atomic.** Each commit should compile and the tests should pass. This makes reviews and potential rollbacks easier.

### Opening a pull request

- Fill in the PR template (or include similar information) - **What? Why? How?**
- Run **all** checks locally (`cargo test && cargo clippy --tests && cargo fmt -- --config imports_granularity=Item`). CI failures that could have been caught locally slow down the process.
- Make sure your branch is up-to-date with `main` and that you have resolved merge conflicts.
- Mark the PR as **Ready for review** only when you believe it is in a merge-able state.

### Review process

1. One maintainer will be assigned as a primary reviewer.
2. We may ask for changes - please do not take this personally. We value the work, we just also value consistency and long-term maintainability.
3. When there is consensus that the PR meets the bar, a maintainer will squash-and-merge.

### Community values

- **Be kind and inclusive.** Treat others with respect; we follow the [Contributor Covenant](https://www.contributor-covenant.org/).
- **Assume good intent.** Written communication is hard - err on the side of generosity.
- **Teach & learn.** If you spot something confusing, open an issue or PR with improvements.

### Getting help

If you run into problems setting up the project, would like feedback on an idea, or just want to say _hi_ - please open a Discussion or jump into the relevant issue. We are happy to help.

Together we can make Codex CLI an incredible tool. **Happy hacking!** :rocket:

### Contributor license agreement (CLA)

All contributors **must** accept the CLA. The process is lightweight:

1. Open your pull request.
2. Paste the following comment (or reply `recheck` if you've signed before):

   ```text
   I have read the CLA Document and I hereby sign the CLA
   ```

3. The CLA-Assistant bot records your signature in the repo and marks the status check as passed.

No special Git commands, email attachments, or commit footers required.

#### Quick fixes

| Scenario          | Command                                          |
| ----------------- | ------------------------------------------------ |
| Amend last commit | `git commit --amend -s --no-edit && git push -f` |

The **DCO check** blocks merges until every commit in the PR carries the footer (with squash this is just the one).

### Releasing `codex`

_For admins only._

Make sure you are on `main` and have no local changes. Then run:

```shell
VERSION=0.2.0  # Can also be 0.2.0-alpha.1 or any valid Rust version.
./codex-rs/scripts/create_github_release.sh "$VERSION"
```

This will make a local commit on top of `main` with `version` set to `$VERSION` in `codex-rs/Cargo.toml` (note that on `main`, we leave the version as `version = "0.0.0"`).

This will push the commit using the tag `rust-v${VERSION}`, which in turn kicks off [the release workflow](.github/workflows/rust-release.yml). This will create a new GitHub Release named `$VERSION`.

If everything looks good in the generated GitHub Release, uncheck the **pre-release** box so it is the latest release.

Create a PR to update [`Formula/c/codex.rb`](https://github.com/Homebrew/homebrew-core/blob/main/Formula/c/codex.rb) on Homebrew.

---

## Security & responsible AI

Have you discovered a vulnerability or have concerns about model output? Please e-mail **security@openai.com** and we will respond promptly.

---

## License

This repository is licensed under the [Apache-2.0 License](LICENSE).
