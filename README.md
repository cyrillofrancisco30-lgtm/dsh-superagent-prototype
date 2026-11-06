xa-evidence-collector-v2.3/
├── collector.py          # Camada 1 + 2 - coleta
├── manifest.py           # Camada 3 - SÓ depois de validação
├── verifier.py           # Verificação independente
├── test_collector.py      # Testes fail-closed
└── README.md


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
