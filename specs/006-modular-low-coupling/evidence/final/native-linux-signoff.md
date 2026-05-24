# Native Linux Signoff

`backend/tests/contract/test_native_linux_deployment_contract.py` passed as part of the focused US3 suite.

`infra/systemd/triton-server.service` uses a native `ExecStart` path and does not depend on Docker. `docker-compose.dev.yml` is documented as development/test topology only.
