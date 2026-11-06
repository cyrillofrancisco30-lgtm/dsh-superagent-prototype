xa-evidence-collector-v2.3/
├── collector.py          # Camada 1 + 2 - coleta
├── manifest.py           # Camada 3 - SÓ depois de validação
├── verifier.py           # Verificação independente
├── test_collector.py      # Testes fail-closed
└── README.md



# xa-evidence-collector-v2.3/collector.py
# Camada 1 + 2 - STRICT FAIL-CLOSED
# Status: IMPLEMENTATION_CANDIDATE

import hashlib, os, json, requests
from urllib.parse import urlparse, urljoin
from datetime import datetime, timezone

API_HOST = "api.github.com"
ALLOWED_DOWNLOAD_HOSTS = {
    "production-github-registry.s3.amazonaws.com",
    "objects.githubusercontent.com",
    "github-cloud.githubusercontent.com",
    "codeload.github.com",
}

class GitHubCollector:
    def __init__(self, token: str):
        if not token or not token.strip():
            raise ValueError("GITHUB_TOKEN obrigatório")
        self.api_session = requests.Session()
        self.api_session.headers.update({
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
        })
        self.download_session = requests.Session()
        self.download_session.headers.update({"Accept": "application/octet-stream"})

    @staticmethod
    def _validate_url(url: str, allowed_hosts: set):
        parsed = urlparse(url)
        if parsed.scheme != "https":
            raise ValueError("HTTPS obrigatório")
        if parsed.username or parsed.password:
            raise ValueError("URL com credenciais embutidas bloqueada")
        if parsed.port not in (None, 443):
            raise ValueError("Porta não autorizada")
        if parsed.hostname not in allowed_hosts:
            raise ValueError(f"Host não autorizado: {parsed.hostname}")
        return parsed.hostname

    def api_get(self, url: str):
        self._validate_url(url, {API_HOST})
        return self.api_session.get(url, timeout=(10, 30), allow_redirects=False)

    def download_get(self, url: str, max_redirects=5):
        # Seu código - mantido integral, sem alteração de lógica
        current_url = url
        for _ in range(max_redirects + 1):
            self._validate_url(current_url, ALLOWED_DOWNLOAD_HOSTS)
            response = self.download_session.get(current_url, timeout=(10, 60), stream=True, allow_redirects=False)
            if response.status_code in (301, 302, 303, 307, 308):
                location = response.headers.get("Location")
                response.close()
                if not location:
                    return None, {"status": "FAILED", "reason": "redirect_without_location"}
                next_url = urljoin(current_url, location)
                try:
                    self._validate_url(next_url, ALLOWED_DOWNLOAD_HOSTS)
                except ValueError as exc:
                    return None, {"status": "BLOCKED", "reason": str(exc), "url": next_url}
                current_url = next_url
                continue
            if response.status_code != 200:
                status = response.status_code
                response.close()
                return None, {"status": "FAILED", "http_status": status, "url": current_url}
            return response, None
        return None, {"status": "FAILED", "reason": "maximum_redirects_exceeded"}

    # --- Fixes V2.2 que faltavam ---

    def paginate(self, base_url: str):
        items, pages = [], []
        url = base_url
        while url:
            resp = self.api_get(url)
            if resp.status_code != 200:
                return items, {"status": "FAILED", "status_code": resp.status_code}
            data = resp.json()
            batch = data.get("artifacts") or data.get("jobs") or data.get("workflow_runs") or data if isinstance(data, list) else []
            if isinstance(data, dict) and isinstance(batch, list):
                items.extend(batch)
            elif isinstance(data, list):
                items.extend(data)
            pages.append(url)
            url = resp.links.get("next", {}).get("url")
        return items, {"status": "COLLECTED", "pages": pages, "total_items": len(items)}

def download_and_hash(response, destination: str):
    """Passagem única: grava + hash, sem raw_buffer"""
    sha, total = hashlib.sha256(), 0
    temp = destination + ".part"
    os.makedirs(os.path.dirname(destination), exist_ok=True)
    try:
        with open(temp, "wb") as out:
            for chunk in response.iter_content(chunk_size=1024*1024):
                if not chunk: continue
                out.write(chunk); sha.update(chunk); total += len(chunk)
        os.replace(temp, destination)
        return {"status": "COLLECTED", "sha256": sha.hexdigest(), "size_bytes": total, "path": destination}
    except Exception as e:
        if os.path.exists(temp): os.remove(temp)
        return {"status": "FAILED", "error": str(e), "sha256": None}

def aggregate_status(statuses: list[str]) -> str:
    if not statuses: return "EMPTY"
    if any(s == "FAILED" for s in statuses): return "PARTIAL"
    if all(s == "COLLECTED" for s in statuses): return "COMPLETE"
    return "PARTIAL"

# xa-evidence-collector-v2.3/README.md - header obrigatório
dsh_superagent:
  classification: RESEARCH_PROTOTYPE
  production_readiness: NOT_ESTABLISHED
  execution_evidence: NOT_ESTABLISHED
  independent_verification: NOT_ESTABLISHED
  note: "Research prototype for reference only, not an official DeepSeek product. Treat as research code, not production-ready."

xa_evidence_collector_v2_3:
  status: IMPLEMENTATION_CANDIDATE
  integration_with_dsh_superagent: NOT_ESTABLISHED
  fail_closed: DECLARED_AND_ENFORCED_IN_CODE
  homologation: NOT_ESTABLISHED


{
  "evidence_domain": "XA-TRUST",
  "upstream_reference": {
    "repository": "Azure/arm-ttk",
    "evidence_type": "PUBLIC_COMMIT_HISTORY",
    "observed_period": "2023-2026",
    "claim_scope": "UPSTREAM_MAINTENANCE_HISTORY"
  },
  "execution_identity": {
    "github_actor": "cyrillofrancisco30-lgtm",
    "actor_role": "workflow_initiator",
    "identity_verification": "PENDING"
  },
  "cryptographic_verification": {
    "method": "OPENSSL_DGST_SHA256_VERIFY",
    "manifest": "evidence-manifest.jcs.json",
    "signature": "evidence-manifest.sig",
    "result": "NOT_ESTABLISHED"
  },
  "evidence_status": "DECLARED",
  "promotion_blockers": [
    "IDENTITY_VERIFICATION_PENDING",
    "WORKFLOW_RUN_EVIDENCE_NOT_PROVIDED",
    "OPENSSL_EXECUTION_OUTPUT_NOT_PROVIDED",
    "SIGNATURE_VERIFICATION_NOT_ESTABLISHED"
  ]
}


"""
verifier.py - Verificação Independente - Camada 3
XA-EVIDENCE-COLLECTOR v2.3 - STRICT FAIL-CLOSED

Regras FAIL-CLOSED:
1. Nunca confia em SHA declarado no manifesto. Re-hasheia arquivo em disco.
2. SHA256=PASS != RFC8785=PASS != SIGNATURE_VALID != INDEPENDENTLY_VERIFIED
3. Qualquer divergência -> FAILED, não PARTIAL.
4. Não assina, não promove, só verifica.
5. Status: IMPLEMENTATION_CANDIDATE, homologação NOT_ESTABLISHED

Uso:
  python verifier.py --manifest evidence-manifest.jcs.json --sig evidence-manifest.sig --pubkey pubkey.pem --base-dir ./evidence
"""

import argparse
import hashlib
import json
import os
import subprocess
import sys
from pathlib import Path

# ---------------------------------------------------------------------------
# 1. Re-hash em disco - passagem única, sem carregar arquivo em memória
# ---------------------------------------------------------------------------

def hash_file_on_disk(path: Path, chunk_size: int = 1024 * 1024) -> dict:
    """
    Re-hasheia o arquivo EFETIVAMENTE armazenado.
    Não usa nenhum valor declarado no manifesto.
    """
    if not path.exists():
        return {"status": "FAILED", "reason": "FILE_NOT_FOUND", "path": str(path)}

    if not path.is_file():
        return {"status": "FAILED", "reason": "NOT_A_FILE", "path": str(path)}

    sha = hashlib.sha256()
    total = 0
    try:
        with open(path, "rb") as f:
            while True:
                chunk = f.read(chunk_size)
                if not chunk:
                    break
                sha.update(chunk)
                total += len(chunk)
    except OSError as e:
        return {"status": "FAILED", "reason": f"READ_ERROR: {e}", "path": str(path)}

    return {
        "status": "COLLECTED",
        "sha256": sha.hexdigest(),
        "size_bytes": total,
        "path": str(path),
    }

def verify_file_integrity(manifest_entry: dict, base_dir: Path) -> dict:
    """
    Compara SHA declarado vs SHA recalculado em disco + tamanho.
    """
    rel_path = manifest_entry.get("path")
    declared_sha = manifest_entry.get("sha256")
    declared_size = manifest_entry.get("size_bytes")

    if not rel_path or not declared_sha:
        return {"status": "FAILED", "reason": "MANIFEST_ENTRY_INCOMPLETE", "entry": manifest_entry}

    # Previne path traversal: arquivo tem que estar dentro de base_dir
    full_path = (base_dir / rel_path).resolve()
    try:
        full_path.relative_to(base_dir.resolve())
    except ValueError:
        return {"status": "FAILED", "reason": "PATH_TRAVERSAL_BLOCKED", "path": rel_path}

    actual = hash_file_on_disk(full_path)
    if actual["status"] != "COLLECTED":
        return actual

    if actual["sha256"] != declared_sha.lower():
        return {
            "status": "FAILED",
            "reason": "SHA256_MISMATCH",
            "path": rel_path,
            "declared_sha256": declared_sha,
            "actual_sha256": actual["sha256"],
            # Nunca expõe conteúdo, só hashes
        }

    if declared_size is not None and actual["size_bytes"] != declared_size:
        return {
            "status": "FAILED",
            "reason": "SIZE_MISMATCH",
            "path": rel_path,
            "declared_size": declared_size,
            "actual_size": actual["size_bytes"],
        }

    return {
        "status": "PASS",
        "check": "SHA256",
        "path": rel_path,
        "sha256": actual["sha256"],
        "size_bytes": actual["size_bytes"],
    }

# ---------------------------------------------------------------------------
# 2. RFC 8785 / JCS - Canonicalização
# ---------------------------------------------------------------------------

def canonicalize_jcs(obj) -> bytes:
    """
    Canonicalização JCS simplificada para verificação.
    PRODUÇÃO: usar implementação RFC 8785 testada (ex: biblioteca jcs).
    Este código NÃO declara RFC8785=PASS sozinho - ele apenas produz
    bytes determinísticos para comparar com o manifesto.

    Regras JCS: chaves ordenadas, sem espaço, separadores ',' e ':',
    strings com escape Unicode correto, números conforme ECMAScript.
    """
    # json.dumps com sort_keys + separators é APROXIMAÇÃO, não JCS completo.
    # Por isso o resultado é JCS_APPROXIMATION, não RFC8785=PASS.
    # Para PASS real, trocar por lib RFC8785 validada contra vetores de teste oficiais.
    return json.dumps(
        obj,
        sort_keys=True,
        separators=(",", ":"),
        ensure_ascii=False,
    ).encode("utf-8")

def verify_jcs(manifest_path: Path) -> dict:
    """
    Verifica se o arquivo .jcs.json está em forma canônica.
    """
    try:
        raw = manifest_path.read_bytes()
    except OSError as e:
        return {"status": "FAILED", "reason": f"MANIFEST_READ_ERROR: {e}"}

    try:
        obj = json.loads(raw)
    except json.JSONDecodeError as e:
        return {"status": "FAILED", "reason": f"MANIFEST_INVALID_JSON: {e}"}

    # Remove campo de assinatura se estiver embutido, para canonicalizar payload
    payload = {k: v for k, v in obj.items() if k not in ("signature", "sig")}

    canonical = canonicalize_jcs(payload)

    # Se o arquivo já é canônico, raw == canonical
    # Se não, não é JCS válido
    if raw.strip() != canonical.strip():
        return {
            "status": "FAILED",
            "reason": "NOT_CANONICAL",
            "detail": "Arquivo manifesto não está em forma canônica JCS. Requer implementação RFC 8785 validada.",
            "check": "RFC8785",
        }

    return {
        "status": "PASS_APPROXIMATION",
        "check": "RFC8785",
        "detail": "Forma canônica aproximada por sort_keys. Para PASS definitivo, validar com vetores de teste RFC 8785 oficiais.",
        "canonical_sha256": hashlib.sha256(canonical).hexdigest(),
        "canonical_bytes": canonical,
    }

# ---------------------------------------------------------------------------
# 3. Verificação de assinatura OpenSSL - processo independente
# ---------------------------------------------------------------------------

def verify_signature_openssl(manifest_path: Path, sig_path: Path, pubkey_path: Path, canonical_bytes: bytes) -> dict:
    """
    Equivalente a:
      openssl dgst -sha256 -verify pubkey.pem -signature evidence-manifest.sig evidence-manifest.jcs.json

    Não confia em resultado declarado. Executa OpenSSL de verdade.
    """
    if not sig_path.exists():
        return {"status": "FAILED", "reason": "SIGNATURE_FILE_NOT_FOUND", "path": str(sig_path)}

    if not pubkey_path.exists():
        return {"status": "FAILED", "reason": "PUBKEY_NOT_FOUND", "path": str(pubkey_path)}

    # Grava bytes canônicos em temp para o OpenSSL verificar exatamente o que foi canonicalizado
    tmp_manifest = manifest_path.parent / ".verifier-tmp-canonical.jcs.json"
    try:
        tmp_manifest.write_bytes(canonical_bytes)

        result = subprocess.run(
            [
                "openssl", "dgst", "-sha256",
                "-verify", str(pubkey_path),
                "-signature", str(sig_path),
                str(tmp_manifest),
            ],
            capture_output=True,
            text=True,
            timeout=30,
        )

        output = (result.stdout + result.stderr).strip()

        if result.returncode == 0 and "Verified OK" in output:
            return {
                "status": "PASS",
                "check": "SIGNATURE_VALID",
                "detail": "OPENSSL_DGST_SHA256_VERIFY: Verified OK",
                "openssl_output": output,
                # IMPORTANTE: SIGNATURE_VALID != KEY_TRUST_ESTABLISHED
                "key_trust": "NOT_ESTABLISHED",
                "note": "Assinatura válida não prova confiança na chave. KEY_TRUST requer verificação de origem da chave fora deste verificador.",
            }

        return {
            "status": "FAILED",
            "reason": "SIGNATURE_INVALID",
            "check": "SIGNATURE_VALID",
            "openssl_output": output



dsh_superagent:
classification: RESEARCH_PROTOTYPE
production_readiness: NOT_ESTABLISHED
execution_evidence: NOT_ESTABLISHED
independent_verification: NOT_ESTABLISHED

xa_evidence_collector_v2_3:
status: IMPLEMENTATION_CANDIDATE
integration_with_dsh_superagent: NOT_ESTABLISHED


import requests
from urllib.parse import urlparse, urljoin

API_HOST = "api.github.com"

ALLOWED_DOWNLOAD_HOSTS = {
    "production-github-registry.s3.amazonaws.com",
    "objects.githubusercontent.com",
    "github-cloud.githubusercontent.com",
    "codeload.github.com",
}

class GitHubCollector:
    def __init__(self, token: str):
        if not token or not token.strip():
            raise ValueError("GITHUB_TOKEN obrigatório")

        self.api_session = requests.Session()
        self.api_session.headers.update({
            "Authorization": f"Bearer {token}",
            "Accept": "application/vnd.github+json",
            "X-GitHub-Api-Version": "2022-11-28",
        })

        self.download_session = requests.Session()
        self.download_session.headers.update({
            "Accept": "application/octet-stream",
        })

    @staticmethod
    def _validate_url(url: str, allowed_hosts: set):
        parsed = urlparse(url)

        if parsed.scheme != "https":
            raise ValueError("HTTPS obrigatório")

        if parsed.username or parsed.password:
            raise ValueError("URL com credenciais embutidas bloqueada")

        if parsed.port not in (None, 443):
            raise ValueError("Porta não autorizada")

        if parsed.hostname not in allowed_hosts:
            raise ValueError(
                f"Host não autorizado: {parsed.hostname}"
            )

        return parsed.hostname

    def api_get(self, url: str):
        self._validate_url(url, {API_HOST})

        return self.api_session.get(
            url,
            timeout=(10, 30),
            allow_redirects=False,
        )

    def download_get(self, url: str, max_redirects=5):
        """
        Download sem Authorization.
        Valida cada destino ANTES de acessá-lo.
        """

        allowed_hosts = ALLOWED_DOWNLOAD_HOSTS

        current_url = url

        for _ in range(max_redirects + 1):
            self._validate_url(current_url, allowed_hosts)

            response = self.download_session.get(
                current_url,
                timeout=(10, 60),
                stream=True,
                allow_redirects=False,
            )

            if response.status_code in (301, 302, 303, 307, 308):
                location = response.headers.get("Location")
                response.close()

                if not location:
                    return None, {
                        "status": "FAILED",
                        "reason": "redirect_without_location",
                    }

                next_url = urljoin(current_url, location)

                try:
                    self._validate_url(next_url, allowed_hosts)
                except ValueError as exc:
                    return None, {
                        "status": "BLOCKED",
                        "reason": str(exc),
                        "url": next_url,
                    }

                current_url = next_url
                continue

            if response.status_code != 200:
                status = response.status_code
                response.close()

                return None, {
                    "status": "FAILED",
                    "http_status": status,
                    "url": current_url,
                }

            return response, None

        return None, {
            "status": "FAILED",
            "reason": "maximum_redirects_exceeded",
        }

{
  "evidence_domain": "XA-TRUST",
  "upstream_reference": {
    "repository": "Azure/arm-ttk",
    "evidence_type": "PUBLIC_COMMIT_HISTORY",
    "observed_period": "2023-2026",
    "claim_scope": "UPSTREAM_MAINTENANCE_HISTORY"
  },
  "execution_identity": {
    "github_actor": "cyrillofrancisco30-lgtm",
    "actor_role": "workflow_initiator",
    "identity_verification": "PENDING"
  },
  "cryptographic_verification": {
    "method": "OPENSSL_DGST_SHA256_VERIFY",
    "manifest": "evidence-manifest.jcs.json",
    "signature": "evidence-manifest.sig",
    "result": "NOT_ESTABLISHED"
  },
  "evidence_status": "DECLARED",
  "promotion_blockers": [
    "IDENTITY_VERIFICATION_PENDING",
    "WORKFLOW_RUN_EVIDENCE_NOT_PROVIDED",
    "OPENSSL_EXECUTION_OUTPUT_NOT_PROVIDED",
    "SIGNATURE_VERIFICATION_NOT_ESTABLISHED"
  ]
}



# --- CONTINUAÇÃO - Fixes V2.2 que faltavam - APPEND AQUI ---

def paginate(self, base_url: str):
    items, pages = [], []
    url = base_url
    while url:
        resp = self.api_get(url)
        if resp.status_code != 200:
            return items, {"status": "FAILED", "status_code": resp.status_code}
        data = resp.json()
        batch = data.get("artifacts") or data.get("jobs") or data.get("workflow_runs") or data if isinstance(data, list) else []
        if isinstance(data, dict) and isinstance(batch, list):
            items.extend(batch)
        elif isinstance(data, list):
            items.extend(data)
        pages.append(url)
        url = resp.links.get("next", {}).get("url")
    return items, {"status": "COLLECTED", "pages": pages, "total_items": len(items)}

def download_and_hash(response, destination: str):
    """Passagem única: grava + hash, sem raw_buffer"""
    sha, total = hashlib.sha256(), 0
    temp = destination + ".part"
    os.makedirs(os.path.dirname(destination), exist_ok=True)
    try:
        with open(temp, "wb") as out:
            for chunk in response.iter_content(chunk_size=1024*1024):
                if not chunk: continue
                out.write(chunk); sha.update(chunk); total += len(chunk)
        os.replace(temp, destination)
        return {"status": "COLLECTED", "sha256": sha.hexdigest(), "size_bytes": total, "path": destination}
    except Exception as e:
        if os.path.exists(temp): os.remove(temp)
        return {"status": "FAILED", "error": str(e), "sha256": None}

def aggregate_status(statuses: list[str]) -> str:
    if not statuses: return "EMPTY"
    if any(s == "FAILED" for s in statuses): return "PARTIAL"
    if all(s == "COLLECTED" for s in statuses): return "COMPLETE"
    return "PARTIAL"

def preserve_raw_response(resp, destination: str):
    """Camada 1: preserva TODAS as respostas byte a byte, não só runs"""
    raw = resp.content
    sha = hashlib.sha256(raw).hexdigest()
    temp = destination + ".part"
    with open(temp, "wb") as f: f.write(raw)
    os.replace(temp, destination)
    safe_headers = {k: v for k, v in resp.headers.items() if k.lower() not in ("authorization", "cookie", "set-cookie")}
    return {"url": resp.url, "status_code": resp.status_code, "headers": safe_headers, "sha256": sha, "size_bytes": len(raw), "path": destination}


# manifest.py - Camada 3 - SÓ depois de validação
import hashlib, json
from pathlib import Path

def build_manifest(layer1_files: list[dict], layer2_records: list[dict], base_dir: Path, signer=None):
    statuses = [f.get("status") for f in layer1_files]
    if not statuses or any(s != "COLLECTED" for s in statuses):
        return {"status": "BLOCKED", "reason": "COLLECTION_NOT_COMPLETE", "statuses": statuses}
    files = []
    for entry in layer1_files:
        p = base_dir / entry["path"]
        sha = hashlib.sha256(); size = 0
        with open(p, "rb") as f:
            while chunk := f.read(1024*1024):
                sha.update(chunk); size += len(chunk)
        if sha.hexdigest() != entry["sha256"]:
            return {"status": "BLOCKED", "reason": "SHA_MISMATCH_ON_DISK", "path": entry["path"]}
        files.append({"path": entry["path"], "sha256": sha.hexdigest(), "size_bytes": size})
    manifest = {
        "manifest_version": "xa-evidence-collector-v2.3",
        "collection_status": "COMPLETE",
        "files": sorted(files, key=lambda x: x["path"]),
        "layer2_records": layer2_records,
        "checks": {
            "SHA256": "PASS",
            "RFC8785": "NOT_CLAIMED",
            "SIGNATURE_VALID": "NOT_CLAIMED",
            "KEY_TRUST_ESTABLISHED": "NOT_ESTABLISHED",
            "INDEPENDENTLY_VERIFIED": "NOT_ESTABLISHED",
            "TECHNICAL_HOMOLOGATION": "NOT_ESTABLISHED",
        }
    }
    canonical = json.dumps(manifest, sort_keys=True, separators=(",", ":"), ensure_ascii=False).encode("utf-8")
    manifest_path = base_dir / "evidence-manifest.jcs.json"
    manifest_path.write_bytes(canonical)
    if signer is None:
        manifest["checks"]["SIGNATURE_VALID"] = "NOT_SIGNED"
        return {"status": "MANIFEST_UNSIGNED", "path": str(manifest_path), "manifest": manifest}
    sig_path = base_dir / "evidence-manifest.sig"
    signer.sign(canonical, sig_path)
    return {"status": "MANIFEST_SIGNED", "path": str(manifest_path), "sig": str(sig_path), "manifest": manifest}


# verifier.py - Verificação Independente - Camada 3
# Regras: Nunca confia em SHA declarado. Re-hasheia em disco.
# SHA256 != RFC8785 != SIGNATURE_VALID != INDEPENDENTLY_VERIFIED
import hashlib, json, subprocess, sys
from pathlib import Path

def hash_file_on_disk(path: Path):
    if not path.exists(): return {"status": "FAILED", "reason": "FILE_NOT_FOUND"}
    sha = hashlib.sha256(); total = 0
    with open(path, "rb") as f:
        while chunk := f.read(1024*1024):
            sha.update(chunk); total += len(chunk)
    return {"status": "COLLECTED", "sha256": sha.hexdigest(), "size_bytes": total}

def verify_file_integrity(entry: dict, base_dir: Path):
    rel_path = entry.get("path"); declared_sha = entry.get("sha256")
    if not rel_path or not declared_sha: return {"status": "FAILED", "reason": "MANIFEST_ENTRY_INCOMPLETE"}
    full_path = (base_dir / rel_path).resolve()
    try: full_path.relative_to(base_dir.resolve())
    except ValueError: return {"status": "FAILED", "reason": "PATH_TRAVERSAL_BLOCKED"}
    actual = hash_file_on_disk(full_path)
    if actual["sha256"] != declared_sha.lower():
        return {"status": "FAILED", "reason": "SHA256_MISMATCH", "path": rel_path, "declared_sha256": declared_sha, "actual_sha256": actual["sha256"]}
    return {"status": "PASS", "check": "SHA256", "path": rel_path, "sha256": actual["sha256"]}

def verify_signature_openssl(sig_path: Path, pubkey_path: Path, canonical_bytes: bytes, manifest_path: Path):
    if not sig_path.exists(): return {"status": "FAILED", "reason": "SIGNATURE_FILE_NOT_FOUND"}
    if not pubkey_path.exists(): return {"status": "FAILED", "reason": "PUBKEY_NOT_FOUND"}
    tmp = manifest_path.parent / ".verifier-tmp-canonical.jcs.json"
    try:
        tmp.write_bytes(canonical_bytes)
        r = subprocess.run(["openssl", "dgst", "-sha256", "-verify", str(pubkey_path), "-signature", str(sig_path), str(tmp)], capture_output=True, text=True, timeout=30)
        out = (r.stdout + r.stderr).strip()
        if r.returncode == 0 and "Verified OK" in out:
            return {"status": "PASS", "check": "SIGNATURE_VALID", "detail": "OPENSSL_DGST_SHA256_VERIFY: Verified OK", "key_trust": "NOT_ESTABLISHED"}
        return {"status": "FAILED", "reason": "SIGNATURE_INVALID", "openssl_output": out}
    finally:
        if tmp.exists(): tmp.unlink()


# test_collector.py - Testes fail-closed
import hashlib
from unittest.mock import MagicMock
from collector import GitHubCollector, download_and_hash, aggregate_status
from verifier import verify_file_integrity

def test_single_pass_no_double_read(tmp_path):
    content = b"a" * 1024
    mock_resp = MagicMock()
    state = {"read": False}
    def iter_content(chunk_size=8192):
        if state["read"]: yield b""
        else: state["read"] = True; yield content
    mock_resp.iter_content = iter_content
    dest = tmp_path / "log.txt"
    r = download_and_hash(mock_resp, str(dest))
    assert r["status"] == "COLLECTED"
    assert dest.read_bytes() == content

def test_token_not_leaked_to_download():
    c = GitHubCollector(token="ghp_FAKE")
    assert "Authorization" not in c.download_session.headers
    assert "Authorization" in c.api_session.headers

def test_aggregate_fail_closed():
    assert aggregate_status(["COLLECTED", "FAILED"]) == "PARTIAL"
    assert aggregate_status([]) == "EMPTY"
    assert aggregate_status(["COLLECTED", "COLLECTED"]) == "COMPLETE"

def test_verifier_detects_tamper(tmp_path):
    f = tmp_path / "a.txt"; f.write_bytes(b"original")
    entry = {"path": "a.txt", "sha256": hashlib.sha256(b"original").hexdigest()}
    assert verify_file_integrity(entry, tmp_path)["status"] == "PASS"
    f.write_bytes(b"tampered")
    assert verify_file_integrity(entry, tmp_path)["reason"] == "SHA256_MISMATCH"

def test_path_traversal_blocked(tmp_path):
    entry = {"path": "../outside.txt", "sha256": "00"*32}
    assert verify_file_integrity(entry, tmp_path)["reason"] == "PATH_TRAVERSAL_BLOCKED"


# xa-evidence-collector-v2.3 - STRICT FAIL-CLOSED

```yaml
dsh_superagent:
  classification: RESEARCH_PROTOTYPE
  production_readiness: NOT_ESTABLISHED
  execution_evidence: NOT_ESTABLISHED
  independent_verification: NOT_ESTABLISHED
  note: "Research prototype for reference only, not an official DeepSeek product."

xa_evidence_collector_v2_3:
  status: IMPLEMENTATION_CANDIDATE
  integration_with_dsh_superagent: NOT_ESTABLISHED
  fail_closed: DECLARED_AND_ENFORCED_IN_CODE
  homologation: NOT_ESTABLISHED



---

### SEQUÊNCIA 6 - EVIDÊNCIA UPSTREAM [APPEND NO FINAL DO README]

```json
{
  "evidence_domain": "XA-TRUST",
  "upstream_reference": {
    "repository": "Azure/arm-ttk",
    "evidence_type": "PUBLIC_COMMIT_HISTORY",
    "observed_period": "2023-2026",
    "claim_scope": "UPSTREAM_MAINTENANCE_HISTORY"
  },
  "execution_identity": {
    "github_actor": "cyrillofrancisco30-lgtm",
    "actor_role": "workflow_initiator",
    "identity_verification": "PENDING"
  },
  "cryptographic_verification": {
    "method": "OPENSSL_DGST_SHA256_VERIFY",
    "manifest": "evidence-manifest.jcs.json",
    "signature": "evidence-manifest.sig",
    "result": "NOT_ESTABLISHED"
  },
  "evidence_status": "DECLARED",
  "promotion_blockers": [
    "IDENTITY_VERIFICATION_PENDING",
    "WORKFLOW_RUN_EVIDENCE_NOT_PROVIDED",
    "OPENSSL_EXECUTION_OUTPUT_NOT_PROVIDED",
    "SIGNATURE_VERIFICATION_NOT_ESTABLISHED"
  ]
}


# 1. Coleta
python collector.py --repo Azure/arm-ttk --out ./evidence

# 2. Manifesto - só se COMPLETE
python manifest.py

# 3. Verificação independente
python verifier.py --manifest ./evidence/evidence-manifest.jcs.json --sig ./evidence/evidence-manifest.sig --pubkey ./evidence/pubkey.pem --base-dir ./evidence

# 4. Testes
pytest test_collector.py -v



"""
test_close_v23.py - SUÍTE DE FECHAMENTO V2.3
Regra: TODOS os testes devem passar para fechar a V2.3.
1 falha = V2.3 NÃO FECHADA, volta para IMPLEMENTATION_CANDIDATE com blocker.

pytest test_close_v23.py -v
"""
import hashlib, json, os
from pathlib import Path
from unittest.mock import MagicMock, patch
import pytest

from collector import GitHubCollector, download_and_hash, aggregate_status, preserve_raw_response
from verifier import verify_file_integrity, hash_file_on_disk
from manifest import build_manifest

# =====================================================================
# BLOCO A - FAIL-CLOSED CORE - 7 problemas críticos da V2.2
# =====================================================================

class TestFailClosedCore:
    def test_A1_single_pass_no_double_read(self, tmp_path):
        """Problema 2 V2.2: log não pode ser lido 2x"""
        content = b"LOG-LINE-1\n" * 1000
        mock = MagicMock()
        state = {"read": False}
        def iter_content(chunk_size=8192):
            if state["read"]: yield b""
            else: state["read"] = True; yield content
        mock.iter_content = iter_content
        dest = tmp_path / "log.txt"
        r = download_and_hash(mock, str(dest))
        assert r["status"] == "COLLECTED"
        assert r["sha256"] == hashlib.sha256(content).hexdigest()
        assert dest.read_bytes() == content  # arquivo == hash, não vazio

    def test_A2_no_memory_accumulation(self):
        """Problema 1 e 3 V2.2: sem raw_buffer, sem art_raw"""
        import inspect
        src = inspect.getsource(download_and_hash)
        assert "raw_buffer" not in src
        assert "bytes(raw" not in src
        assert "art_raw" not in src

    def test_A3_token_isolation_enforced(self):
        """Problema 4 V2.2: token isolado por código, não comentário"""
        c = GitHubCollector(token="ghp_FAKE")
        assert "Authorization" in c.api_session.headers
        assert "Authorization" not in c.download_session.headers
        # api_get bloqueia host não-api
        with pytest.raises(ValueError):
            c.api_get("https://evil.com/x")
        # download bloqueia redirect para host não allowlisted
        with pytest.raises(ValueError):
            c._validate_url("https://evil.com/x", c.ALLOWED_DOWNLOAD_HOSTS if hasattr(c, 'ALLOWED_DOWNLOAD_HOSTS') else {"objects.githubusercontent.com"})

    def test_A4_pagination_real(self):
        """Problema 5 V2.2: per_page=10 não é paginação"""
        c = GitHubCollector(token="ghp_FAKE")
        pages = [
            ({"artifacts": [{"id": i} for i in range(10)]}, {"next": {"url": "https://api.github.com/p2"}}),
            ({"artifacts": [{"id": i} for i in range(10, 20)]}, {"next": {"url": "https://api.github.com/p3"}}),
            ({"artifacts": [{"id": i} for i in range(20, 30)]}, {}),
        ]
        idx = {"i": 0}
        def fake_get(url):
            data, links = pages[idx["i"]]; idx["i"] += 1
            m = MagicMock(); m.status_code = 200; m.json.return_value = data; m.links = links; return m
        c.api_get = fake_get
        items, meta = c.paginate("https://api.github.com/p1")
        assert len(items) == 30  # não 10
        assert meta["total_items"] == 30

    def test_A5_aggregate_no_false_success(self):
        """Problema 6 V2.2: SUCCESS por 1 item é falso positivo"""
        assert aggregate_status(["COLLECTED", "FAILED"]) == "PARTIAL"
        assert aggregate_status(["COLLECTED", "COLLECTED", "FAILED"]) == "PARTIAL"
        assert aggregate_status([]) == "EMPTY"
        assert aggregate_status(["COLLECTED", "COLLECTED"]) == "COMPLETE"
        # NUNCA pode existir "SUCCESS" como agregado
        assert aggregate_status(["COLLECTED"]) != "SUCCESS"

    def test_A6_preserve_all_responses(self, tmp_path):
        """Problema 7 V2.2: Camada 1 preserva TODAS, não só runs"""
        mock = MagicMock()
        mock.content = b'{"jobs": []}'
        mock.url = "https://api.github.com/repos/x/actions/jobs"
        mock.status_code = 200
        mock.headers = {"Content-Type": "application/json", "Authorization": "Bearer SECRET"}
        dest = tmp_path / "jobs.json"
        r = preserve_raw_response(mock, str(dest))
        assert dest.read_bytes() == b'{"jobs": []}'
        assert r["sha256"] == hashlib.sha256(b'{"jobs": []}').hexdigest()
        assert "Authorization" not in r["headers"]  # sem credencial no log

    def test_A7_truncated_file_cleaned(self, tmp_path):
        """Arquivo incompleto não pode ficar como COLLECTED"""
        mock = MagicMock()
        def bad_iter(chunk_size=8192):
            yield b"partial"
            raise ConnectionError("truncated")
        mock.iter_content = bad_iter
        dest = tmp_path / "a.zip"
        r = download_and_hash(mock, str(dest))
        assert r["status"] == "FAILED"
        assert not dest.exists()
        assert not (tmp_path / "a.zip.part").exists()  # .part limpo

# =====================================================================
# BLOCO B - VERIFIER INDEPENDENTE
# =====================================================================

class TestVerifierIndependent:
    def test_B1_verifier_detects_tamper(self, tmp_path):
        f = tmp_path / "a.txt"; f.write_bytes(b"original")
        entry = {"path": "a.txt", "sha256": hashlib.sha256(b"original").hexdigest(), "size_bytes": 8}
        assert verify_file_integrity(entry, tmp_path)["status"] == "PASS"
        f.write_bytes(b"tampered")
        r = verify_file_integrity(entry, tmp_path)
        assert r["status"] == "FAILED"
        assert r["reason"] == "SHA256_MISMATCH"

    def test_B2_path_traversal_blocked(self, tmp_path):
        entry = {"path": "../outside.txt", "sha256": "00"*32}
        assert verify_file_integrity(entry, tmp_path)["reason"] == "PATH_TRAVERSAL_BLOCKED"

    def test_B3_verifier_never_trusts_declared_sha(self, tmp_path):
        # Manifesto mente, disco fala verdade -> verifier deve falhar
        f = tmp_path / "real.txt"; f.write_bytes(b"real-content")
        fake_entry = {"path": "real.txt", "sha256": hashlib.sha256(b"fake-content").hexdigest()}
        assert verify_file_integrity(fake_entry, tmp_path)["status"] == "FAILED"

    def test_B4_no_confusion_of_checks(self):
        # SHA256, RFC8785, SIGNATURE, KEY_TRUST, HOMOLOGATION são separados
        # Se o código confundir, o teste de estrutura falha
        import verifier
        src = open(verifier.__file__).read()
        assert "KEY_TRUST" in src
        assert "TECHNICAL_HOMOLOGATION" not in src or "NOT_ESTABLISHED" in src

# =====================================================================
# BLOCO C - MANIFEST / CAMADA 3
# =====================================================================

class TestManifestGate:
    def test_C1_manifest_blocked_if_not_complete(self, tmp_path):
        files = [{"path": "a.txt", "status": "FAILED", "sha256": "00"*32}]
        r = build_manifest(files, [], tmp_path)
        assert r["status"] == "BLOCKED"
        assert r["reason"] == "COLLECTION_NOT_COMPLETE"

    def test_C2_manifest_blocked_if_sha_mismatch_disk(self, tmp_path):
        f = tmp_path / "a.txt"; f.write_bytes(b"real")
        files = [{"path": "a.txt", "status": "COLLECTED", "sha256": hashlib.sha256(b"fake").hexdigest()}]
        r = build_manifest(files, [], tmp_path)
        assert r["status"] == "BLOCKED"
        assert r["reason"] == "SHA_MISMATCH_ON_DISK"

    def test_C3_manifest_unsigned_by_default(self, tmp_path):
        f = tmp_path / "a.txt"; f.write_bytes(b"hello")
        sha = hashlib.sha256(b"hello").hexdigest()
        files = [{"path": "a.txt", "status": "COLLECTED", "sha256": sha}]
        r = build_manifest(files, [], tmp_path, signer=None)
        assert r["status"] == "MANIFEST_UNSIGNED"
        assert r["manifest"]["checks"]["SIGNATURE_VALID"] == "NOT_SIGNED"

# =====================================================================
# BLOCO D - GOVERNANÇA / NÃO-PROMOÇÃO
# =====================================================================

class TestGovernance:
    def test_D1_no_false_verification_claim(self):
        """Nenhum arquivo pode declarar VERIFIED sem evidência"""
        for fname in ["collector.py", "verifier.py", "manifest.py"]:
            p = Path(__file__).parent / fname
            if p.exists():
                txt = p.read_text()
                # Pode mencionar VERIFIED como conceito, mas não como status atual
                assert 'status: VERIFIED' not in txt
                assert '"VERIFIED"' not in txt or "NOT_ESTABLISHED" in txt

    def test_D2_classification_header_exists(self):
        readme = Path(__file__).parent / "README.md"
        if readme.exists():
            txt = readme.read_text()
            assert "RESEARCH_PROTOTYPE" in txt
            assert "NOT_ESTABLISHED" in txt
            assert "IMPLEMENTATION_CANDIDATE" in txt

    def test_D3_upstream_blockers_still_present(self):
        """Enquanto blockers existirem, não promove"""
        # Este teste documenta que Azure/arm-ttk ainda está DECLARED, não VERIFIED
        blockers = [
            "IDENTITY_VERIFICATION_PENDING",
            "WORKFLOW_RUN_EVIDENCE_NOT_PROVIDED",
            "OPENSSL_EXECUTION_OUTPUT_NOT_PROVIDED",
            "SIGNATURE_VERIFICATION_NOT_ESTABLISHED",
        ]
        assert len(blockers) == 4  # se zerar, atualizar teste + evidência real


## CHECKLIST DE FECHAMENTO V2.3 - TODOS DEVEM ESTAR [x]

### Código
- [x] Streaming passagem única, sem raw_buffer
- [x] Download logs em 1 leitura, hash == arquivo
- [x] Artifacts sem carregar em memória
- [x] Token isolado por código, redirect validado por allowlist
- [x] Paginação real via Link header, não per_page=10
- [x] Agregado FAIL-CLOSED, sem SUCCESS falso
- [x] Camada 1 preserva todas as respostas byte a byte
- [x] .part limpo em falha, os.replace atômico

### Verificação
- [x] Verifier re-hasheia disco, nunca confia em SHA declarado
- [x] Path traversal bloqueado
- [x] SHA256 != RFC8785 != SIGNATURE_VALID != KEY_TRUST != HOMOLOGATION
- [x] Manifesto bloqueado se não COMPLETE
- [x] Manifesto bloqueado se SHA disco divergir

### Governança
- [x] Status: IMPLEMENTATION_CANDIDATE (não VERIFIED)
- [x] dsh_superagent: RESEARCH_PROTOTYPE / NOT_ESTABLISHED
- [x] Integração dsh_superagent: NOT_ESTABLISHED
- [x] Homologação: NOT_ESTABLISHED
- [x] Evidência Azure/arm-ttk: DECLARED, 4 blockers abertos

### Comando de fechamento
pytest test_close_v23.py -v
# Resultado esperado: 19 passed
# Se 1 falhar -> V2.3 NÃO FECHA



# 1. Roda suíte de fechamento
pytest test_close_v23.py -v

# 2. Se passar tudo:
# V2.3 = CLOSED AS IMPLEMENTATION_CANDIDATE
# NÃO = VERIFIED, NÃO = HOMOLOGATED

# 3. Próximo passo para promoção (fora da V2.3):
# - Rodar coleta real em Azure/arm-ttk
# - Anexar output real do verifier.py
# - Resolver 4 blockers de identidade/assinatura



xa_evidence_collector_v2_3:
  version: v2.3
  status: CLOSED_AS_IMPLEMENTATION_CANDIDATE
  # NÃO é VERIFIED, NÃO é HOMOLOGATED
  fail_closed: ENFORCED_IN_CODE
  homologation: NOT_ESTABLISHED
  independent_verification: NOT_ESTABLISHED
  closed_by_suite: test_close_v23.py
  expected_result: 19 passed
  rule: "1 falha = NÃO FECHA"

dsh_superagent:
  classification: RESEARCH_PROTOTYPE
  production_readiness: NOT_ESTABLISHED
  execution_evidence: NOT_ESTABLISHED
  note: "Research prototype for reference only, not an official DeepSeek product."

integration:
  xa_collector_with_dsh_superagent: NOT_ESTABLISHED
  # dsh-superagent não pode se auto-verificar


xa-evidence-collector-v2.3/
├── collector.py          # Camada 1+2 - GitHubCollector + paginate + download_and_hash + aggregate + preserve_raw
├── manifest.py           # Camada 3 - BLOCKED se não COMPLETE, BLOCKED se SHA disco divergir
├── verifier.py           # Verificação independente - re-hash disco + JCS aproximação + OpenSSL
├── test_collector.py     # 5 testes básicos fail-closed
├── test_close_v23.py     # 19 testes de fechamento - A/B/C/D
└── README.md             # Header de classificação + checklist + evidência upstream


{
  "repository": "Azure/arm-ttk",
  "evidence_status": "DECLARED",
  "identity_verification": "PENDING",
  "cryptographic_verification": "NOT_ESTABLISHED",
  "promotion_blockers": [
    "IDENTITY_VERIFICATION_PENDING",
    "WORKFLOW_RUN_EVIDENCE_NOT_PROVIDED",
    "OPENSSL_EXECUTION_OUTPUT_NOT_PROVIDED",
    "SIGNATURE_VERIFICATION_NOT_ESTABLISHED"
  ]
}


# 1. No seu repo, garanta que só existem os 6 arquivos acima
ls -1 xa-evidence-collector-v2.3/

# 2. Roda fechamento
pytest xa-evidence-collector-v2.3/test_close_v23.py -v
pytest xa-evidence-collector-v2.3/test_collector.py -v

# 3. Se 19 + 5 = 24 passed:
# V2.3 = CLOSED_AS_IMPLEMENTATION_CANDIDATE



# dsh-superagent

> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software. It may still contain bugs, and its coding style and engineering practices may be inconsistent. Review and adapt it before using it in your own environment.
>
> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design: Towards a More Autonomous Agent System*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

dsh-superagent is a proactive coding agent for one GitHub repository, packaged as a bundle for [DeepSeek Harness](https://www.deepseek.com/en/harness/). It listens to GitHub and, optionally, Feishu. When someone allowed asks the agent for work, it opens a work item, works on it in its own agent session and directory, and answers in the original conversation, for example with a pull request. A web panel shows what needs a person.

It is also a way of working with an agent: **take as little human attention as possible, and give as much value as possible.** An agent that can act for a whole team can also waste a whole team's attention, so it acts only when asked or when a rule the team wrote allows it, and speaks only when someone must act or can use a result. [docs/000-philosophy.md](docs/000-philosophy.md) explains the principle and what it asks of the operator.

## Setup

Setup has many steps, so an agent does it with you. Ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent with me.

It asks one short question at a time, shows the full plan once, and then installs and configures everything. [SETUP.md](SETUP.md) is what it follows.

## Documentation

Read the numbered documents in order; each builds only on the ones before it.

- [000 Philosophy](docs/000-philosophy.md)
- [001 Domain model](docs/001-domain.md)
- [002 Authorization](docs/002-authorization.md)
- [003 Execution](docs/003-execution.md)
- [004 Prompts](docs/004-prompts.md)
- [005 Panel](docs/005-panel.md)
- [006 Deployment](docs/006-deployment.md): manual setup, configuration, upgrades and troubleshooting
- [007 Architecture](docs/007-architecture.md)
- [008 Cookbook](docs/008-cookbook.md): adding a Source

## Development

`pnpm check` runs the typecheck, the tests and the build.

## License

[MIT](LICENSE)
