# Scripting & Coding Rounds

> ⭐ Priority area — senior DevOps interviews at product companies include live coding

## Folder Structure

```
09-scripting-coding/
├── python/
│   ├── dsa/            # Data structures & algorithms practice
│   ├── automation/     # Kubernetes client, REST APIs, cloud SDKs
│   └── challenges/     # Timed problem solutions
├── bash/
│   ├── scripts/        # Production-ready script templates
│   └── challenges/     # Bash problems
└── solutions/          # Archive of solved problems
```

## What Product Companies Test

At senior level, expect:
1. **Live coding (Python):** DSA problems — arrays, hashmaps, strings, maybe trees/graphs
2. **Scripting tasks:** "Write a script that does X" (K8s client, API calls, log parsing)
3. **Bash/shell:** Automation scripts, text processing, cron-like logic
4. **Debugging:** Read broken code and fix it

## Python DSA — Priority Topics

| Topic | NeetCode Category | Priority | Status |
|---|---|---|---|
| Two Pointers | Arrays | High | 🔴 |
| Sliding Window | Arrays | High | 🔴 |
| HashMap / HashSet | Hashing | High | 🔴 |
| Stack | Stack | High | 🔴 |
| Binary Search | Binary Search | Medium | 🔴 |
| Linked List basics | Linked List | Medium | 🔴 |
| BFS / DFS (graphs) | Graphs | Medium | 🔴 |
| Heaps / Priority Queue | Heap | Low | 🔴 |

**Target:** NeetCode 75 (not full 150 — focus on patterns, not quantity)

## Python Automation — Must-Know

```python
# Kubernetes client
from kubernetes import client, config
config.load_incluster_config()  # or load_kube_config()
v1 = client.CoreV1Api()
pods = v1.list_namespaced_pod(namespace="default")

# Azure SDK
from azure.identity import DefaultAzureCredential
from azure.mgmt.containerservice import ContainerServiceClient
credential = DefaultAzureCredential()

# REST API with retry
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retry = Retry(total=3, backoff_factor=1, status_forcelist=[500, 502, 503, 504])
adapter = HTTPAdapter(max_retries=retry)
session.mount("https://", adapter)
```

## Bash Script Templates

### Deployment health check
```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:?Usage: $0 <namespace> <deployment>}"
DEPLOYMENT="${2:?}"
TIMEOUT=120

kubectl rollout status deployment/"${DEPLOYMENT}" \
  -n "${NAMESPACE}" \
  --timeout="${TIMEOUT}s"
```

### Pod log error scanner
```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:-default}"
PATTERN="${2:-ERROR|FATAL|panic}"
SINCE="${3:-1h}"

kubectl get pods -n "${NAMESPACE}" --no-headers -o custom-columns=':metadata.name' | \
  while read -r pod; do
    echo "=== ${pod} ==="
    kubectl logs "${pod}" -n "${NAMESPACE}" --since="${SINCE}" 2>/dev/null | \
      grep -E "${PATTERN}" || true
  done
```

## Coding Interview Tips

1. **Think out loud** — interviewers want to hear your thought process
2. **Clarify edge cases** before coding (empty input, single element, large input)
3. **State time/space complexity** after solving
4. **Start with brute force**, then optimise — don't jump to optimal immediately
5. **Test with examples** — walk through your code manually
6. **DevOps framing** — when asked "how would you do this in production?" add error handling, logging, retry logic

## Resources

- [NeetCode.io](https://neetcode.io/) — structured DSA with video solutions
- [DevOps Exercises — scripting section](https://github.com/bregman-arie/devops-exercises)
- [Test Your Sysadmin Skills](https://github.com/trimstray/test-your-sysadmin-skills)
- [Kubernetes Python Client](https://github.com/kubernetes-client/python)
- [Python Azure SDK](https://learn.microsoft.com/en-us/azure/developer/python/)
