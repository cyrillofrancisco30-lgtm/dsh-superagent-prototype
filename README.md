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
