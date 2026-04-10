# Scripting & Coding — Interview Scenarios

> 80 real scenarios across Python DSA, Python automation, Bash scripting, and live coding rounds.
> Format: Scenario → What's being tested → Your approach → Key code

---

## Part 1: Python — Data Structures & Algorithms (DSA)

---

### S1. Two Sum
**Scenario:** Given an array of integers and a target, return indices of two numbers that add to the target.
**Tests:** Hash map usage, O(n) vs O(n²) trade-off.
```python
def two_sum(nums: list[int], target: int) -> list[int]:
    seen = {}
    for i, n in enumerate(nums):
        diff = target - n
        if diff in seen:
            return [seen[diff], i]
        seen[n] = i
    return []
```
**Follow-up:** What if the array is sorted? (Two-pointer, O(1) space)

---

### S2. Longest Substring Without Repeating Characters
**Scenario:** Find the length of the longest substring without repeating characters.
**Tests:** Sliding window pattern.
```python
def length_of_longest_substring(s: str) -> int:
    char_index = {}
    left = max_len = 0
    for right, ch in enumerate(s):
        if ch in char_index and char_index[ch] >= left:
            left = char_index[ch] + 1
        char_index[ch] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

---

### S3. Valid Parentheses
**Scenario:** Given a string of brackets, determine if it's valid.
**Tests:** Stack usage.
```python
def is_valid(s: str) -> bool:
    stack = []
    mapping = {')': '(', '}': '{', ']': '['}
    for ch in s:
        if ch in mapping:
            top = stack.pop() if stack else '#'
            if mapping[ch] != top:
                return False
        else:
            stack.append(ch)
    return not stack
```

---

### S4. Merge Intervals
**Scenario:** Given a list of intervals, merge all overlapping intervals.
**Tests:** Sorting, greedy.
```python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    intervals.sort(key=lambda x: x[0])
    merged = [intervals[0]]
    for start, end in intervals[1:]:
        if start <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], end)
        else:
            merged.append([start, end])
    return merged
```

---

### S5. Top K Frequent Elements
**Scenario:** Return the k most frequent elements from a list.
**Tests:** Heap / bucket sort.
```python
import heapq
from collections import Counter

def top_k_frequent(nums: list[int], k: int) -> list[int]:
    count = Counter(nums)
    return heapq.nlargest(k, count.keys(), key=count.get)
```
**Follow-up:** O(n) with bucket sort — explain trade-off.

---

### S6. Binary Search
**Scenario:** Implement binary search on a sorted array.
**Tests:** Off-by-one handling, loop invariants.
```python
def binary_search(nums: list[int], target: int) -> int:
    left, right = 0, len(nums) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

---

### S7. Find First and Last Position in Sorted Array
**Scenario:** Find start and end index of a target in a sorted array. O(log n).
**Tests:** Modified binary search, boundary cases.
```python
def search_range(nums: list[int], target: int) -> list[int]:
    def find(is_first: bool) -> int:
        lo, hi, result = 0, len(nums) - 1, -1
        while lo <= hi:
            mid = (lo + hi) // 2
            if nums[mid] == target:
                result = mid
                if is_first: hi = mid - 1
                else: lo = mid + 1
            elif nums[mid] < target:
                lo = mid + 1
            else:
                hi = mid - 1
        return result
    return [find(True), find(False)]
```

---

### S8. Container With Most Water
**Scenario:** Find two lines that form a container holding the most water.
**Tests:** Two pointers, greedy reasoning.
```python
def max_area(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    max_water = 0
    while left < right:
        water = (right - left) * min(height[left], height[right])
        max_water = max(max_water, water)
        if height[left] < height[right]:
            left += 1
        else:
            right -= 1
    return max_water
```

---

### S9. 3Sum
**Scenario:** Find all unique triplets that sum to zero.
**Tests:** Sorting + two pointers, duplicate handling.
```python
def three_sum(nums: list[int]) -> list[list[int]]:
    nums.sort()
    result = []
    for i, n in enumerate(nums):
        if i > 0 and nums[i] == nums[i-1]:
            continue
        left, right = i + 1, len(nums) - 1
        while left < right:
            total = n + nums[left] + nums[right]
            if total == 0:
                result.append([n, nums[left], nums[right]])
                while left < right and nums[left] == nums[left+1]: left += 1
                while left < right and nums[right] == nums[right-1]: right -= 1
                left += 1; right -= 1
            elif total < 0:
                left += 1
            else:
                right -= 1
    return result
```

---

### S10. Product of Array Except Self
**Scenario:** Return array where each element is the product of all others. No division, O(n).
**Tests:** Prefix/suffix product arrays.
```python
def product_except_self(nums: list[int]) -> list[int]:
    n = len(nums)
    result = [1] * n
    prefix = 1
    for i in range(n):
        result[i] = prefix
        prefix *= nums[i]
    suffix = 1
    for i in range(n - 1, -1, -1):
        result[i] *= suffix
        suffix *= nums[i]
    return result
```

---

### S11. Climbing Stairs
**Scenario:** Each time you can climb 1 or 2 steps. How many ways to reach step n?
**Tests:** DP, Fibonacci pattern.
```python
def climb_stairs(n: int) -> int:
    if n <= 2:
        return n
    prev, curr = 1, 2
    for _ in range(3, n + 1):
        prev, curr = curr, prev + curr
    return curr
```

---

### S12. Coin Change
**Scenario:** Given coin denominations, find minimum coins to make amount.
**Tests:** DP, bottom-up vs top-down.
```python
def coin_change(coins: list[int], amount: int) -> int:
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], dp[i - coin] + 1)
    return dp[amount] if dp[amount] != float('inf') else -1
```

---

### S13. Longest Common Subsequence
**Scenario:** Find length of longest common subsequence of two strings.
**Tests:** 2D DP.
```python
def lcs(text1: str, text2: str) -> int:
    m, n = len(text1), len(text2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if text1[i-1] == text2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])
    return dp[m][n]
```

---

### S14. Number of Islands
**Scenario:** Given a 2D grid of '1's (land) and '0's (water), count islands.
**Tests:** BFS/DFS on grid, visited tracking.
```python
def num_islands(grid: list[list[str]]) -> int:
    if not grid:
        return 0
    rows, cols = len(grid), len(grid[0])
    count = 0

    def dfs(r, c):
        if r < 0 or r >= rows or c < 0 or c >= cols or grid[r][c] != '1':
            return
        grid[r][c] = '0'  # mark visited
        for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)]:
            dfs(r + dr, c + dc)

    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == '1':
                dfs(r, c)
                count += 1
    return count
```

---

### S15. LRU Cache
**Scenario:** Implement an LRU Cache with O(1) get and put.
**Tests:** OrderedDict or doubly linked list + hashmap.
```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = OrderedDict()

    def get(self, key: int) -> int:
        if key not in self.cache:
            return -1
        self.cache.move_to_end(key)
        return self.cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value
        if len(self.cache) > self.capacity:
            self.cache.popitem(last=False)
```

---

### S16. Find Median from Data Stream
**Scenario:** Implement a MedianFinder that supports addNum and findMedian.
**Tests:** Two heaps (max-heap + min-heap).
```python
import heapq

class MedianFinder:
    def __init__(self):
        self.small = []  # max-heap (negated)
        self.large = []  # min-heap

    def addNum(self, num: int) -> None:
        heapq.heappush(self.small, -num)
        if self.small and self.large and (-self.small[0]) > self.large[0]:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        if len(self.small) > len(self.large) + 1:
            heapq.heappush(self.large, -heapq.heappop(self.small))
        if len(self.large) > len(self.small):
            heapq.heappush(self.small, -heapq.heappop(self.large))

    def findMedian(self) -> float:
        if len(self.small) > len(self.large):
            return -self.small[0]
        return (-self.small[0] + self.large[0]) / 2.0
```

---

### S17. Word Search in Grid
**Scenario:** Find if a word exists in a 2D grid (adjacent cells, no reuse).
**Tests:** DFS with backtracking.
```python
def exist(board: list[list[str]], word: str) -> bool:
    rows, cols = len(board), len(board[0])

    def dfs(r, c, idx):
        if idx == len(word):
            return True
        if r < 0 or r >= rows or c < 0 or c >= cols or board[r][c] != word[idx]:
            return False
        temp, board[r][c] = board[r][c], '#'
        found = any(dfs(r+dr, c+dc, idx+1) for dr, dc in [(0,1),(0,-1),(1,0),(-1,0)])
        board[r][c] = temp
        return found

    return any(dfs(r, c, 0) for r in range(rows) for c in range(cols))
```

---

### S18. Spiral Matrix
**Scenario:** Return elements of a matrix in spiral order.
**Tests:** Boundary shrinking, off-by-one.
```python
def spiral_order(matrix: list[list[int]]) -> list[int]:
    result = []
    top, bottom, left, right = 0, len(matrix)-1, 0, len(matrix[0])-1
    while top <= bottom and left <= right:
        for c in range(left, right+1): result.append(matrix[top][c])
        top += 1
        for r in range(top, bottom+1): result.append(matrix[r][right])
        right -= 1
        if top <= bottom:
            for c in range(right, left-1, -1): result.append(matrix[bottom][c])
            bottom -= 1
        if left <= right:
            for r in range(bottom, top-1, -1): result.append(matrix[r][left])
            left += 1
    return result
```

---

### S19. Group Anagrams
**Scenario:** Group strings that are anagrams of each other.
**Tests:** Hashing with sorted key or character count.
```python
from collections import defaultdict

def group_anagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        groups[tuple(sorted(s))].append(s)
    return list(groups.values())
```

---

### S20. Trapping Rain Water
**Scenario:** Given heights, compute how much rain water can be trapped.
**Tests:** Two pointers or prefix/suffix max arrays.
```python
def trap(height: list[int]) -> int:
    left, right = 0, len(height) - 1
    left_max = right_max = water = 0
    while left < right:
        if height[left] <= height[right]:
            if height[left] >= left_max: left_max = height[left]
            else: water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max: right_max = height[right]
            else: water += right_max - height[right]
            right -= 1
    return water
```

---

### S21. Course Schedule (Cycle Detection)
**Scenario:** Determine if you can finish all courses given prerequisites (cycle detection in directed graph).
**Tests:** Topological sort / DFS cycle detection.
```python
def can_finish(num_courses: int, prerequisites: list[list[int]]) -> bool:
    graph = [[] for _ in range(num_courses)]
    for a, b in prerequisites:
        graph[b].append(a)
    # 0=unvisited, 1=visiting, 2=done
    state = [0] * num_courses

    def dfs(node):
        if state[node] == 1: return False  # cycle
        if state[node] == 2: return True
        state[node] = 1
        if not all(dfs(nei) for nei in graph[node]):
            return False
        state[node] = 2
        return True

    return all(dfs(i) for i in range(num_courses))
```

---

### S22. Reverse Linked List
**Scenario:** Reverse a singly linked list in-place.
**Tests:** Pointer reversal, iterative vs recursive.
```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val; self.next = next

def reverse_list(head: ListNode) -> ListNode:
    prev = None
    curr = head
    while curr:
        next_node = curr.next
        curr.next = prev
        prev = curr
        curr = next_node
    return prev
```

---

### S23. Detect Cycle in Linked List
**Scenario:** Determine if a linked list has a cycle.
**Tests:** Floyd's cycle detection (fast/slow pointers).
```python
def has_cycle(head: ListNode) -> bool:
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

---

### S24. Maximum Subarray (Kadane's Algorithm)
**Scenario:** Find the contiguous subarray with the maximum sum.
**Tests:** Kadane's, handling all-negative arrays.
```python
def max_subarray(nums: list[int]) -> int:
    max_sum = curr_sum = nums[0]
    for n in nums[1:]:
        curr_sum = max(n, curr_sum + n)
        max_sum = max(max_sum, curr_sum)
    return max_sum
```

---

### S25. Jump Game
**Scenario:** Determine if you can reach the last index given jump lengths.
**Tests:** Greedy, max-reach tracking.
```python
def can_jump(nums: list[int]) -> bool:
    max_reach = 0
    for i, n in enumerate(nums):
        if i > max_reach:
            return False
        max_reach = max(max_reach, i + n)
    return True
```

---

### S26. Meeting Rooms II (Min Heap)
**Scenario:** Find the minimum number of conference rooms required.
**Tests:** Sorting + min-heap.
```python
import heapq

def min_meeting_rooms(intervals: list[list[int]]) -> int:
    if not intervals:
        return 0
    intervals.sort(key=lambda x: x[0])
    heap = []  # stores end times
    for start, end in intervals:
        if heap and heap[0] <= start:
            heapq.heapreplace(heap, end)
        else:
            heapq.heappush(heap, end)
    return len(heap)
```

---

### S27. Validate Binary Search Tree
**Scenario:** Check if a binary tree is a valid BST.
**Tests:** Min/max boundary propagation (not just left < root < right).
```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val; self.left = left; self.right = right

def is_valid_bst(root: TreeNode) -> bool:
    def validate(node, min_val, max_val):
        if not node:
            return True
        if not (min_val < node.val < max_val):
            return False
        return (validate(node.left, min_val, node.val) and
                validate(node.right, node.val, max_val))
    return validate(root, float('-inf'), float('inf'))
```

---

### S28. Lowest Common Ancestor of BST
**Scenario:** Find LCA of two nodes in a BST.
**Tests:** BST property exploitation.
```python
def lca_bst(root: TreeNode, p: TreeNode, q: TreeNode) -> TreeNode:
    while root:
        if p.val < root.val and q.val < root.val:
            root = root.left
        elif p.val > root.val and q.val > root.val:
            root = root.right
        else:
            return root
```

---

### S29. Level Order Traversal (BFS)
**Scenario:** Return level-by-level values of a binary tree.
**Tests:** BFS with queue, level size tracking.
```python
from collections import deque

def level_order(root: TreeNode) -> list[list[int]]:
    if not root:
        return []
    result, queue = [], deque([root])
    while queue:
        level = []
        for _ in range(len(queue)):
            node = queue.popleft()
            level.append(node.val)
            if node.left: queue.append(node.left)
            if node.right: queue.append(node.right)
        result.append(level)
    return result
```

---

### S30. Serialize and Deserialize Binary Tree
**Scenario:** Implement serialize/deserialize for a binary tree.
**Tests:** BFS or preorder encoding, delimiter handling.
```python
from collections import deque

class Codec:
    def serialize(self, root: TreeNode) -> str:
        if not root:
            return ''
        result, queue = [], deque([root])
        while queue:
            node = queue.popleft()
            if node:
                result.append(str(node.val))
                queue.append(node.left)
                queue.append(node.right)
            else:
                result.append('null')
        return ','.join(result)

    def deserialize(self, data: str) -> TreeNode:
        if not data:
            return None
        values = deque(data.split(','))
        root = TreeNode(int(values.popleft()))
        queue = deque([root])
        while queue:
            node = queue.popleft()
            left_val = values.popleft()
            right_val = values.popleft()
            if left_val != 'null':
                node.left = TreeNode(int(left_val))
                queue.append(node.left)
            if right_val != 'null':
                node.right = TreeNode(int(right_val))
                queue.append(node.right)
        return root
```

---

## Part 2: Python Automation (DevOps-Specific)

---

### S31. List all pods not in Running state across all namespaces
**Tests:** Kubernetes Python client, label filtering.
```python
#!/usr/bin/env python3
import argparse
from kubernetes import client, config

def list_non_running_pods(context: str | None = None):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config(context=context)

    v1 = client.CoreV1Api()
    pods = v1.list_pod_for_all_namespaces(watch=False)
    non_running = [
        (p.metadata.namespace, p.metadata.name, p.status.phase)
        for p in pods.items
        if p.status.phase not in ('Running', 'Succeeded')
    ]
    for ns, name, phase in non_running:
        print(f"{ns}/{name}: {phase}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--context', default=None)
    args = parser.parse_args()
    list_non_running_pods(args.context)
```

---

### S32. Scale deployment and wait for rollout
```python
#!/usr/bin/env python3
import argparse, time
from kubernetes import client, config

def scale_and_wait(namespace: str, deployment: str, replicas: int, timeout: int = 120):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    apps_v1 = client.AppsV1Api()
    body = {'spec': {'replicas': replicas}}
    apps_v1.patch_namespaced_deployment_scale(deployment, namespace, body)
    print(f"Scaled {deployment} to {replicas} replicas. Waiting...")

    deadline = time.time() + timeout
    while time.time() < deadline:
        dep = apps_v1.read_namespaced_deployment(deployment, namespace)
        if dep.status.ready_replicas == replicas:
            print("Rollout complete.")
            return
        time.sleep(5)
    raise TimeoutError(f"Deployment {deployment} did not scale in {timeout}s")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--namespace', required=True)
    parser.add_argument('--deployment', required=True)
    parser.add_argument('--replicas', type=int, required=True)
    args = parser.parse_args()
    scale_and_wait(args.namespace, args.deployment, args.replicas)
```

---

### S33. Find pods with no resource requests/limits set
```python
#!/usr/bin/env python3
from kubernetes import client, config

def find_pods_without_limits():
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    v1 = client.CoreV1Api()
    pods = v1.list_pod_for_all_namespaces(watch=False)
    for pod in pods.items:
        for container in pod.spec.containers:
            r = container.resources
            missing = []
            if not r.requests:
                missing.append('requests')
            if not r.limits:
                missing.append('limits')
            if missing:
                print(f"{pod.metadata.namespace}/{pod.metadata.name}/{container.name}: missing {', '.join(missing)}")
```

---

### S34. Detect deployments with single replica (no HA)
```python
#!/usr/bin/env python3
from kubernetes import client, config

def find_single_replica_deployments(namespace: str = None):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    apps_v1 = client.AppsV1Api()
    if namespace:
        deps = apps_v1.list_namespaced_deployment(namespace)
    else:
        deps = apps_v1.list_deployment_for_all_namespaces()

    for d in deps.items:
        if (d.spec.replicas or 0) < 2:
            print(f"{d.metadata.namespace}/{d.metadata.name}: {d.spec.replicas} replica(s)")
```

---

### S35. Restart a deployment (rollout restart)
```python
#!/usr/bin/env python3
import argparse
from datetime import datetime, timezone
from kubernetes import client, config

def rollout_restart(namespace: str, deployment: str):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    apps_v1 = client.AppsV1Api()
    now = datetime.now(timezone.utc).isoformat()
    patch = {
        'spec': {
            'template': {
                'metadata': {
                    'annotations': {'kubectl.kubernetes.io/restartedAt': now}
                }
            }
        }
    }
    apps_v1.patch_namespaced_deployment(deployment, namespace, patch)
    print(f"Triggered restart of {namespace}/{deployment} at {now}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--namespace', required=True)
    parser.add_argument('--deployment', required=True)
    args = parser.parse_args()
    rollout_restart(args.namespace, args.deployment)
```

---

### S36. List secrets that haven't been rotated in 90+ days
```python
#!/usr/bin/env python3
from datetime import datetime, timezone, timedelta
from kubernetes import client, config

def find_stale_secrets(days: int = 90):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    v1 = client.CoreV1Api()
    secrets = v1.list_secret_for_all_namespaces(watch=False)
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    for s in secrets.items:
        created = s.metadata.creation_timestamp
        if created and created < cutoff:
            age = (datetime.now(timezone.utc) - created).days
            print(f"{s.metadata.namespace}/{s.metadata.name}: {age} days old")
```

---

### S37. Emit a Prometheus custom metric via Pushgateway
```python
#!/usr/bin/env python3
import argparse
from prometheus_client import CollectorRegistry, Gauge, push_to_gateway

def push_metric(job: str, metric: str, value: float, gateway: str, labels: dict):
    registry = CollectorRegistry()
    g = Gauge(metric, f'Custom metric {metric}', list(labels.keys()), registry=registry)
    g.labels(**labels).set(value)
    push_to_gateway(gateway, job=job, registry=registry)
    print(f"Pushed {metric}={value} to {gateway}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--job', required=True)
    parser.add_argument('--metric', required=True)
    parser.add_argument('--value', type=float, required=True)
    parser.add_argument('--gateway', default='http://pushgateway:9091')
    args = parser.parse_args()
    push_metric(args.job, args.metric, args.value, args.gateway, {})
```

---

### S38. Query Azure Cost Management API and report top 5 expensive resource groups
```python
#!/usr/bin/env python3
import argparse
from azure.identity import DefaultAzureCredential
from azure.mgmt.costmanagement import CostManagementClient

def top_costly_rgs(subscription_id: str, top_n: int = 5):
    credential = DefaultAzureCredential()
    client = CostManagementClient(credential)
    scope = f"/subscriptions/{subscription_id}"
    query = {
        "type": "ActualCost",
        "timeframe": "MonthToDate",
        "dataset": {
            "granularity": "None",
            "aggregation": {"totalCost": {"name": "Cost", "function": "Sum"}},
            "grouping": [{"type": "Dimension", "name": "ResourceGroupName"}]
        }
    }
    result = client.query.usage(scope, query)
    rows = sorted(result.rows, key=lambda r: r[0], reverse=True)[:top_n]
    print(f"{'Resource Group':<40} {'Cost (USD)':>12}")
    for row in rows:
        print(f"{row[1]:<40} {row[0]:>12.2f}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--subscription-id', required=True)
    parser.add_argument('--top', type=int, default=5)
    args = parser.parse_args()
    top_costly_rgs(args.subscription_id, args.top)
```

---

### S39. Parse JSON logs and extract ERROR entries with timestamps
```python
#!/usr/bin/env python3
import argparse, json, sys
from datetime import datetime

def extract_errors(log_file: str, since: str | None = None):
    since_dt = datetime.fromisoformat(since) if since else None
    with open(log_file) as f:
        for line in f:
            line = line.strip()
            if not line:
                continue
            try:
                entry = json.loads(line)
            except json.JSONDecodeError:
                continue
            level = entry.get('level', entry.get('severity', '')).upper()
            if level not in ('ERROR', 'FATAL', 'CRITICAL'):
                continue
            ts_str = entry.get('timestamp', entry.get('time', ''))
            if since_dt and ts_str:
                try:
                    ts = datetime.fromisoformat(ts_str.replace('Z', '+00:00'))
                    if ts < since_dt:
                        continue
                except ValueError:
                    pass
            msg = entry.get('message', entry.get('msg', json.dumps(entry)))
            print(f"[{ts_str}] {level}: {msg}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('log_file')
    parser.add_argument('--since', help='ISO datetime filter')
    args = parser.parse_args()
    extract_errors(args.log_file, args.since)
```

---

### S40. Watch Kubernetes events in real-time and alert on Warning events
```python
#!/usr/bin/env python3
import argparse
from kubernetes import client, config, watch

def watch_warnings(namespace: str = None):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    v1 = client.CoreV1Api()
    w = watch.Watch()
    kwargs = {'namespace': namespace} if namespace else {}
    print(f"Watching events in {'namespace: ' + namespace if namespace else 'all namespaces'}...")
    for event in w.stream(v1.list_namespaced_event if namespace else v1.list_event_for_all_namespaces, **kwargs):
        obj = event['object']
        if obj.type == 'Warning':
            print(f"WARNING | {obj.involved_object.namespace}/{obj.involved_object.name} | "
                  f"{obj.reason}: {obj.message}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--namespace', default=None)
    args = parser.parse_args()
    watch_warnings(args.namespace)
```

---

### S41. Verify all Helm releases are deployed successfully
```python
#!/usr/bin/env python3
import subprocess, json, sys

def check_helm_releases():
    result = subprocess.run(
        ['helm', 'list', '--all-namespaces', '--output', 'json'],
        capture_output=True, text=True, check=True
    )
    releases = json.loads(result.stdout)
    failed = [r for r in releases if r.get('status') != 'deployed']
    if failed:
        print("FAILED RELEASES:")
        for r in failed:
            print(f"  {r['namespace']}/{r['name']}: {r['status']}")
        sys.exit(1)
    print(f"All {len(releases)} Helm releases are deployed.")

if __name__ == '__main__':
    check_helm_releases()
```

---

### S42. Check SSL certificate expiry for a list of domains
```python
#!/usr/bin/env python3
import argparse, ssl, socket
from datetime import datetime, timezone

def check_cert_expiry(domains: list[str], warn_days: int = 30):
    ctx = ssl.create_default_context()
    for domain in domains:
        try:
            with ctx.wrap_socket(socket.socket(), server_hostname=domain) as s:
                s.settimeout(5)
                s.connect((domain, 443))
                cert = s.getpeercert()
            expiry = datetime.strptime(cert['notAfter'], '%b %d %H:%M:%S %Y %Z').replace(tzinfo=timezone.utc)
            days_left = (expiry - datetime.now(timezone.utc)).days
            status = 'CRITICAL' if days_left < 14 else 'WARNING' if days_left < warn_days else 'OK'
            print(f"{status:8} {domain}: expires in {days_left} days ({expiry.date()})")
        except Exception as e:
            print(f"ERROR    {domain}: {e}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('domains', nargs='+')
    parser.add_argument('--warn-days', type=int, default=30)
    args = parser.parse_args()
    check_cert_expiry(args.domains, args.warn_days)
```

---

### S43. Delete completed/failed Jobs older than N days
```python
#!/usr/bin/env python3
import argparse
from datetime import datetime, timezone, timedelta
from kubernetes import client, config

def cleanup_old_jobs(namespace: str, days: int = 1, dry_run: bool = False):
    try:
        config.load_incluster_config()
    except config.ConfigException:
        config.load_kube_config()

    batch_v1 = client.BatchV1Api()
    jobs = batch_v1.list_namespaced_job(namespace)
    cutoff = datetime.now(timezone.utc) - timedelta(days=days)
    for job in jobs.items:
        completed = job.status.completion_time
        failed = job.status.failed
        if completed and completed < cutoff:
            action = 'DRY-RUN delete' if dry_run else 'Deleting'
            print(f"{action}: {namespace}/{job.metadata.name}")
            if not dry_run:
                batch_v1.delete_namespaced_job(
                    job.metadata.name, namespace,
                    propagation_policy='Background'
                )

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--namespace', required=True)
    parser.add_argument('--days', type=int, default=1)
    parser.add_argument('--dry-run', action='store_true')
    args = parser.parse_args()
    cleanup_old_jobs(args.namespace, args.days, args.dry_run)
```

---

### S44. Retry decorator with exponential backoff
```python
import time, functools, logging

logger = logging.getLogger(__name__)

def retry(max_attempts: int = 3, backoff: float = 2.0, exceptions: tuple = (Exception,)):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            delay = 1.0
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    logger.warning(f"Attempt {attempt}/{max_attempts} failed: {e}. Retrying in {delay:.1f}s...")
                    time.sleep(delay)
                    delay *= backoff
        return wrapper
    return decorator

# Usage
@retry(max_attempts=5, backoff=2.0, exceptions=(ConnectionError, TimeoutError))
def call_api(url: str):
    import requests
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

---

### S45. Context manager for temporary Kubernetes port-forward
```python
import subprocess, time, contextlib, socket

@contextlib.contextmanager
def k8s_port_forward(namespace: str, pod: str, local_port: int, remote_port: int):
    proc = subprocess.Popen(
        ['kubectl', 'port-forward', f'pod/{pod}', f'{local_port}:{remote_port}', '-n', namespace],
        stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL
    )
    try:
        # Wait for port to open
        for _ in range(20):
            try:
                with socket.create_connection(('localhost', local_port), timeout=1):
                    break
            except OSError:
                time.sleep(0.5)
        yield local_port
    finally:
        proc.terminate()

# Usage
with k8s_port_forward('default', 'myapp-pod-xyz', 8080, 8080) as port:
    import requests
    resp = requests.get(f'http://localhost:{port}/health')
    print(resp.json())
```

---

### S46. Parse Terraform state to find all Azure resources missing required tags
```python
#!/usr/bin/env python3
import argparse, json, sys

REQUIRED_TAGS = {'environment', 'owner', 'cost-center'}

def check_tags(state_file: str):
    with open(state_file) as f:
        state = json.load(f)

    violations = []
    for resource in state.get('resources', []):
        if not resource.get('type', '').startswith('azurerm_'):
            continue
        for instance in resource.get('instances', []):
            attrs = instance.get('attributes', {})
            tags = attrs.get('tags') or {}
            missing = REQUIRED_TAGS - set(tags.keys())
            if missing:
                violations.append((resource['type'], attrs.get('name', '?'), missing))

    if violations:
        print(f"{'Resource':<40} {'Name':<30} {'Missing Tags'}")
        for rtype, name, missing in violations:
            print(f"{rtype:<40} {name:<30} {', '.join(sorted(missing))}")
        sys.exit(1)
    print("All resources have required tags.")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('state_file')
    args = parser.parse_args()
    check_tags(args.state_file)
```

---

### S47. Concurrent health checks across multiple endpoints
```python
#!/usr/bin/env python3
import argparse, concurrent.futures, requests

def check_endpoint(url: str, timeout: int = 5) -> tuple[str, int | None, str]:
    try:
        resp = requests.get(url, timeout=timeout)
        return url, resp.status_code, 'OK' if resp.ok else 'FAIL'
    except requests.RequestException as e:
        return url, None, str(e)

def check_all(urls: list[str], workers: int = 10):
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
        futures = {executor.submit(check_endpoint, url): url for url in urls}
        for future in concurrent.futures.as_completed(futures):
            url, status, result = future.result()
            print(f"{'OK' if result == 'OK' else 'FAIL':4} {status or '---'} {url}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('urls', nargs='+')
    args = parser.parse_args()
    check_all(args.urls)
```

---

### S48. Generate a Kubernetes deployment YAML from a template
```python
#!/usr/bin/env python3
import argparse, yaml

def generate_deployment(name: str, image: str, replicas: int, namespace: str, port: int) -> dict:
    return {
        'apiVersion': 'apps/v1',
        'kind': 'Deployment',
        'metadata': {'name': name, 'namespace': namespace},
        'spec': {
            'replicas': replicas,
            'selector': {'matchLabels': {'app': name}},
            'template': {
                'metadata': {'labels': {'app': name}},
                'spec': {
                    'containers': [{
                        'name': name,
                        'image': image,
                        'ports': [{'containerPort': port}],
                        'resources': {
                            'requests': {'cpu': '100m', 'memory': '128Mi'},
                            'limits': {'cpu': '500m', 'memory': '512Mi'}
                        },
                        'securityContext': {
                            'runAsNonRoot': True,
                            'readOnlyRootFilesystem': True,
                            'allowPrivilegeEscalation': False
                        }
                    }],
                    'securityContext': {'runAsUser': 1000}
                }
            }
        }
    }

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--name', required=True)
    parser.add_argument('--image', required=True)
    parser.add_argument('--replicas', type=int, default=2)
    parser.add_argument('--namespace', default='default')
    parser.add_argument('--port', type=int, default=8080)
    args = parser.parse_args()
    print(yaml.dump(generate_deployment(args.name, args.image, args.replicas, args.namespace, args.port)))
```

---

### S49. Rolling log file analysis — find top 10 slowest API endpoints
```python
#!/usr/bin/env python3
import argparse, json, re
from collections import defaultdict

def analyze_access_log(log_file: str, top_n: int = 10):
    # Works with nginx JSON logs or combined log format
    endpoint_times = defaultdict(list)
    json_pattern = re.compile(r'^\{')
    with open(log_file) as f:
        for line in f:
            line = line.strip()
            if json_pattern.match(line):
                try:
                    entry = json.loads(line)
                    path = entry.get('request_uri', entry.get('path', '?'))
                    duration = float(entry.get('request_time', entry.get('duration', 0)))
                    endpoint_times[path].append(duration)
                except (json.JSONDecodeError, ValueError):
                    continue

    stats = []
    for path, times in endpoint_times.items():
        stats.append((path, len(times), sum(times)/len(times), max(times)))

    stats.sort(key=lambda x: x[2], reverse=True)
    print(f"{'Endpoint':<60} {'Count':>6} {'Avg(s)':>8} {'Max(s)':>8}")
    for path, count, avg, mx in stats[:top_n]:
        print(f"{path:<60} {count:>6} {avg:>8.3f} {mx:>8.3f}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('log_file')
    parser.add_argument('--top', type=int, default=10)
    args = parser.parse_args()
    analyze_access_log(args.log_file, args.top)
```

---

### S50. Send Slack alert with Prometheus query result
```python
#!/usr/bin/env python3
import argparse, requests, os

def query_prometheus(prometheus_url: str, query: str) -> float:
    resp = requests.get(f"{prometheus_url}/api/v1/query", params={'query': query}, timeout=10)
    resp.raise_for_status()
    result = resp.json()
    vectors = result.get('data', {}).get('result', [])
    if not vectors:
        raise ValueError(f"No results for query: {query}")
    return float(vectors[0]['value'][1])

def send_slack_alert(webhook_url: str, message: str):
    resp = requests.post(webhook_url, json={'text': message}, timeout=10)
    resp.raise_for_status()

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('--prometheus', default='http://prometheus:9090')
    parser.add_argument('--query', required=True)
    parser.add_argument('--threshold', type=float, required=True)
    parser.add_argument('--alert-name', required=True)
    args = parser.parse_args()

    webhook = os.environ.get('SLACK_WEBHOOK_URL')
    if not webhook:
        raise EnvironmentError("SLACK_WEBHOOK_URL not set")

    value = query_prometheus(args.prometheus, args.query)
    if value > args.threshold:
        msg = f":fire: *{args.alert_name}* | Value: `{value:.2f}` exceeds threshold `{args.threshold}`"
        send_slack_alert(webhook, msg)
        print(f"Alert sent: {msg}")
    else:
        print(f"OK: {value:.2f} <= {args.threshold}")
```

---

## Part 3: Bash Scripting

---

### S51. Disk usage alert — alert when disk is above threshold
```bash
#!/usr/bin/env bash
set -euo pipefail

THRESHOLD="${1:-85}"
HOSTNAME="$(hostname)"

while IFS= read -r line; do
    usage=$(echo "$line" | awk '{print $5}' | tr -d '%')
    mount=$(echo "$line" | awk '{print $6}')
    if [[ "$usage" -ge "$THRESHOLD" ]]; then
        echo "ALERT: ${HOSTNAME}:${mount} is at ${usage}% (threshold: ${THRESHOLD}%)"
        # In production: pipe to PagerDuty / Slack webhook
    fi
done < <(df -h | tail -n +2)
```

---

### S52. Deployment health check post-deploy
```bash
#!/usr/bin/env bash
set -euo pipefail

NAMESPACE="${1:?Usage: $0 <namespace> <deployment> [timeout]}"
DEPLOYMENT="${2:?}"
TIMEOUT="${3:-120}"

echo "Waiting for rollout: ${NAMESPACE}/${DEPLOYMENT} (timeout: ${TIMEOUT}s)"
if kubectl rollout status deployment/"${DEPLOYMENT}" \
    -n "${NAMESPACE}" \
    --timeout="${TIMEOUT}s"; then
    echo "Rollout complete."
else
    echo "ERROR: Rollout did not complete in ${TIMEOUT}s" >&2
    kubectl describe deployment "${DEPLOYMENT}" -n "${NAMESPACE}"
    kubectl get pods -n "${NAMESPACE}" -l "app=${DEPLOYMENT}"
    exit 1
fi
```

---

### S53. Log rotation script (compress logs older than N days)
```bash
#!/usr/bin/env bash
set -euo pipefail

LOG_DIR="${1:?Usage: $0 <log_dir> [days]}"
DAYS="${2:-7}"
COMPRESSED=0

find "${LOG_DIR}" -name "*.log" -mtime "+${DAYS}" | while read -r logfile; do
    if gzip -9 "${logfile}"; then
        echo "Compressed: ${logfile}"
        (( COMPRESSED++ )) || true
    else
        echo "WARN: Failed to compress ${logfile}" >&2
    fi
done

echo "Done. Compressed ${COMPRESSED} file(s)."
```

---

### S54. Check if all required environment variables are set
```bash
#!/usr/bin/env bash
set -euo pipefail

required_vars=(
    "DATABASE_URL"
    "REDIS_URL"
    "JWT_SECRET"
    "AZURE_CLIENT_ID"
    "AZURE_TENANT_ID"
)

missing=()
for var in "${required_vars[@]}"; do
    if [[ -z "${!var:-}" ]]; then
        missing+=("$var")
    fi
done

if [[ ${#missing[@]} -gt 0 ]]; then
    echo "ERROR: Missing required environment variables:" >&2
    printf '  - %s\n' "${missing[@]}" >&2
    exit 1
fi

echo "All required environment variables are set."
```

---

### S55. Automated namespace cleanup — delete namespaces stuck in Terminating
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Scanning for namespaces stuck in Terminating..."
kubectl get namespaces --no-headers | grep Terminating | awk '{print $1}' | \
while read -r ns; do
    echo "Force-finalizing namespace: ${ns}"
    kubectl get namespace "${ns}" -o json | \
        python3 -c "import sys, json; d=json.load(sys.stdin); d['spec']['finalizers']=[]; print(json.dumps(d))" | \
        kubectl replace --raw "/api/v1/namespaces/${ns}/finalize" -f -
done
```

---

### S56. Generate a summary of recent CI/CD pipeline failures from GitLab API
```bash
#!/usr/bin/env bash
set -euo pipefail

GITLAB_URL="${GITLAB_URL:?GITLAB_URL not set}"
TOKEN="${GITLAB_TOKEN:?GITLAB_TOKEN not set}"
PROJECT_ID="${1:?Usage: $0 <project_id>}"

curl -sf --header "PRIVATE-TOKEN: ${TOKEN}" \
    "${GITLAB_URL}/api/v4/projects/${PROJECT_ID}/pipelines?status=failed&per_page=10" | \
    python3 -c "
import sys, json
for p in json.load(sys.stdin):
    print(f\"ID: {p['id']:8} | Branch: {p.get('ref','?'):30} | Created: {p['created_at']}\")
"
```

---

### S57. Wait for a service to become healthy before continuing deployment
```bash
#!/usr/bin/env bash
set -euo pipefail

URL="${1:?Usage: $0 <health_check_url> [max_retries] [interval]}"
MAX_RETRIES="${2:-30}"
INTERVAL="${3:-5}"

for (( i=1; i<=MAX_RETRIES; i++ )); do
    if curl -sf --max-time 5 "${URL}" > /dev/null 2>&1; then
        echo "Service healthy at attempt ${i}."
        exit 0
    fi
    echo "Attempt ${i}/${MAX_RETRIES}: not healthy yet. Waiting ${INTERVAL}s..."
    sleep "${INTERVAL}"
done

echo "ERROR: Service did not become healthy after ${MAX_RETRIES} attempts." >&2
exit 1
```

---

### S58. Find and report Docker images not pulled recently (cleanup candidates)
```bash
#!/usr/bin/env bash
set -euo pipefail

DAYS="${1:-30}"
echo "Images not used in the last ${DAYS} days:"
docker image ls --format '{{.Repository}}:{{.Tag}}\t{{.CreatedAt}}\t{{.ID}}' | \
    while IFS=$'\t' read -r image created id; do
        created_ts=$(date -d "${created}" +%s 2>/dev/null || date -j -f "%Y-%m-%d %T" "${created%% *} ${created#* }" +%s 2>/dev/null || echo 0)
        cutoff_ts=$(date -d "${DAYS} days ago" +%s 2>/dev/null || date -v-"${DAYS}"d +%s 2>/dev/null || echo 0)
        if [[ "$created_ts" -lt "$cutoff_ts" ]]; then
            echo "${image} (${id}) created: ${created}"
        fi
    done
```

---

### S59. Rotate Kubernetes secret from Azure Key Vault
```bash
#!/usr/bin/env bash
set -euo pipefail

KEYVAULT="${1:?Usage: $0 <keyvault> <secret-name> <k8s-namespace> <k8s-secret>}"
SECRET_NAME="${2:?}"
K8S_NAMESPACE="${3:?}"
K8S_SECRET="${4:?}"

echo "Fetching ${SECRET_NAME} from Key Vault ${KEYVAULT}..."
NEW_VALUE=$(az keyvault secret show \
    --vault-name "${KEYVAULT}" \
    --name "${SECRET_NAME}" \
    --query "value" -o tsv)

echo "Updating Kubernetes secret ${K8S_NAMESPACE}/${K8S_SECRET}..."
kubectl create secret generic "${K8S_SECRET}" \
    --from-literal="${SECRET_NAME}=${NEW_VALUE}" \
    --namespace="${K8S_NAMESPACE}" \
    --dry-run=client -o yaml | kubectl apply -f -

echo "Done: ${K8S_NAMESPACE}/${K8S_SECRET} updated from Key Vault."
```

---

### S60. Parallel SSH health check across a list of hosts
```bash
#!/usr/bin/env bash
set -euo pipefail

HOSTS_FILE="${1:?Usage: $0 <hosts_file>}"
SSH_KEY="${SSH_KEY:-~/.ssh/id_rsa}"
TIMEOUT="${2:-5}"

check_host() {
    local host="$1"
    if ssh -i "${SSH_KEY}" \
        -o StrictHostKeyChecking=no \
        -o ConnectTimeout="${TIMEOUT}" \
        -o BatchMode=yes \
        "${host}" "echo OK" 2>/dev/null; then
        echo "OK:   ${host}"
    else
        echo "FAIL: ${host}"
    fi
}
export -f check_host
export SSH_KEY TIMEOUT

parallel -j 20 check_host < "${HOSTS_FILE}"
```

---

### S61. Extract pod restart counts above threshold
```bash
#!/usr/bin/env bash
set -euo pipefail

THRESHOLD="${1:-5}"
NAMESPACE="${2:---all-namespaces}"

if [[ "${NAMESPACE}" == "--all-namespaces" ]]; then
    FLAGS="--all-namespaces"
else
    FLAGS="-n ${NAMESPACE}"
fi

echo "Pods with restart count >= ${THRESHOLD}:"
kubectl get pods ${FLAGS} -o json | python3 -c "
import sys, json
data = json.load(sys.stdin)
threshold = int('${THRESHOLD}')
for pod in data['items']:
    ns = pod['metadata']['namespace']
    name = pod['metadata']['name']
    for cs in pod.get('status', {}).get('containerStatuses', []):
        restarts = cs.get('restartCount', 0)
        if restarts >= threshold:
            print(f'{ns}/{name}/{cs[\"name\"]}: {restarts} restarts')
"
```

---

### S62. Canary deployment validation — compare error rates before/after
```bash
#!/usr/bin/env bash
set -euo pipefail

PROMETHEUS="${1:-http://prometheus:9090}"
NAMESPACE="${2:?}"
DEPLOYMENT="${3:?}"
THRESHOLD="${4:-1.0}"  # max allowed error rate %

query="sum(rate(http_requests_total{namespace=\"${NAMESPACE}\",deployment=\"${DEPLOYMENT}\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{namespace=\"${NAMESPACE}\",deployment=\"${DEPLOYMENT}\"}[5m])) * 100"

error_rate=$(curl -sf "${PROMETHEUS}/api/v1/query" \
    --data-urlencode "query=${query}" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); r=d['data']['result']; print(r[0]['value'][1] if r else '0')")

echo "Current error rate: ${error_rate}%"

# Use python for float comparison
python3 -c "
er = float('${error_rate}')
th = float('${THRESHOLD}')
if er > th:
    print(f'FAIL: Error rate {er:.2f}% exceeds threshold {th}%')
    exit(1)
else:
    print(f'OK: Error rate {er:.2f}% within threshold {th}%')
"
```

---

### S63. Backup etcd snapshot to Azure Blob Storage
```bash
#!/usr/bin/env bash
set -euo pipefail

SNAPSHOT_DIR="/tmp/etcd-snapshots"
ACCOUNT="${AZURE_STORAGE_ACCOUNT:?}"
CONTAINER="${AZURE_STORAGE_CONTAINER:?}"
TIMESTAMP=$(date +%Y%m%d-%H%M%S)
SNAPSHOT_FILE="${SNAPSHOT_DIR}/etcd-${TIMESTAMP}.db"

mkdir -p "${SNAPSHOT_DIR}"

ETCDCTL_API=3 etcdctl snapshot save "${SNAPSHOT_FILE}" \
    --endpoints="${ETCD_ENDPOINTS:-https://127.0.0.1:2379}" \
    --cacert="${ETCD_CA_CERT:-/etc/kubernetes/pki/etcd/ca.crt}" \
    --cert="${ETCD_CERT:-/etc/kubernetes/pki/etcd/healthcheck-client.crt}" \
    --key="${ETCD_KEY:-/etc/kubernetes/pki/etcd/healthcheck-client.key}"

az storage blob upload \
    --account-name "${ACCOUNT}" \
    --container-name "${CONTAINER}" \
    --name "etcd-${TIMESTAMP}.db" \
    --file "${SNAPSHOT_FILE}" \
    --auth-mode login

echo "Snapshot uploaded: etcd-${TIMESTAMP}.db"
rm -f "${SNAPSHOT_FILE}"
```

---

### S64. Monitor and auto-restart a crashed process
```bash
#!/usr/bin/env bash
set -euo pipefail

PROCESS_NAME="${1:?Usage: $0 <process_name> <restart_command>}"
RESTART_CMD="${2:?}"
MAX_RESTARTS="${3:-5}"
CHECK_INTERVAL="${4:-30}"
RESTART_COUNT=0

echo "Monitoring process: ${PROCESS_NAME}"
while true; do
    if ! pgrep -x "${PROCESS_NAME}" > /dev/null 2>&1; then
        if (( RESTART_COUNT >= MAX_RESTARTS )); then
            echo "FATAL: ${PROCESS_NAME} restarted ${MAX_RESTARTS} times. Giving up." >&2
            exit 1
        fi
        echo "WARN: ${PROCESS_NAME} not running. Restarting (attempt $((RESTART_COUNT+1))/${MAX_RESTARTS})..."
        eval "${RESTART_CMD}"
        (( RESTART_COUNT++ )) || true
    else
        RESTART_COUNT=0
    fi
    sleep "${CHECK_INTERVAL}"
done
```

---

### S65. Generate compliance report — list all K8s pods running as root
```bash
#!/usr/bin/env bash
set -euo pipefail

echo "Pods running as root (UID 0) or without runAsNonRoot:"
kubectl get pods --all-namespaces -o json | python3 -c "
import sys, json
data = json.load(sys.stdin)
for pod in data['items']:
    ns = pod['metadata']['namespace']
    name = pod['metadata']['name']
    pod_sc = pod.get('spec', {}).get('securityContext', {})
    pod_user = pod_sc.get('runAsUser')
    pod_non_root = pod_sc.get('runAsNonRoot', False)
    for c in pod.get('spec', {}).get('containers', []):
        c_sc = c.get('securityContext', {})
        user = c_sc.get('runAsUser', pod_user)
        non_root = c_sc.get('runAsNonRoot', pod_non_root)
        if user == 0 or (user is None and not non_root):
            print(f'{ns}/{name}/{c[\"name\"]}: runAsUser={user}, runAsNonRoot={non_root}')
"
```

---

### S66. Infrastructure drift check — compare desired vs actual replica counts
```bash
#!/usr/bin/env bash
set -euo pipefail

# Expected: file with lines like "namespace/deployment=replicas"
CONFIG_FILE="${1:?Usage: $0 <config_file>}"

DRIFT_FOUND=false
while IFS='=' read -r resource expected; do
    namespace=$(echo "$resource" | cut -d'/' -f1)
    deployment=$(echo "$resource" | cut -d'/' -f2)
    actual=$(kubectl get deployment "${deployment}" -n "${namespace}" \
        -o jsonpath='{.spec.replicas}' 2>/dev/null || echo "MISSING")
    if [[ "${actual}" != "${expected}" ]]; then
        echo "DRIFT: ${namespace}/${deployment} expected=${expected} actual=${actual}"
        DRIFT_FOUND=true
    else
        echo "OK:    ${namespace}/${deployment} = ${actual}"
    fi
done < "${CONFIG_FILE}"

${DRIFT_FOUND} && exit 1 || exit 0
```

---

### S67. Multi-cluster context switcher with validation
```bash
#!/usr/bin/env bash
set -euo pipefail

TARGET_CONTEXT="${1:?Usage: $0 <context_name>}"

available=$(kubectl config get-contexts -o name 2>/dev/null)
if ! echo "${available}" | grep -qx "${TARGET_CONTEXT}"; then
    echo "ERROR: Context '${TARGET_CONTEXT}' not found." >&2
    echo "Available contexts:" >&2
    echo "${available}" >&2
    exit 1
fi

kubectl config use-context "${TARGET_CONTEXT}"
echo "Switched to context: ${TARGET_CONTEXT}"

# Validate connectivity
if kubectl cluster-info --request-timeout=5s > /dev/null 2>&1; then
    echo "Cluster is reachable."
else
    echo "WARNING: Could not reach cluster API server." >&2
    exit 1
fi
```

---

### S68. Automated image tag updater in Helm values
```bash
#!/usr/bin/env bash
set -euo pipefail

VALUES_FILE="${1:?Usage: $0 <values.yaml> <new_tag>}"
NEW_TAG="${2:?}"

if [[ ! -f "${VALUES_FILE}" ]]; then
    echo "ERROR: ${VALUES_FILE} not found" >&2
    exit 1
fi

# Create backup
cp "${VALUES_FILE}" "${VALUES_FILE}.bak"

# Update image tag using sed (assumes format: tag: "old_tag")
sed -i "s/^\(\s*tag:\s*\)['\"].*['\"]/\1'${NEW_TAG}'/" "${VALUES_FILE}"

echo "Updated image tag to '${NEW_TAG}' in ${VALUES_FILE}"
diff "${VALUES_FILE}.bak" "${VALUES_FILE}" || true
```

---

### S69. Report on nodes near resource pressure
```bash
#!/usr/bin/env bash
set -euo pipefail

CPU_THRESHOLD="${1:-80}"    # percent
MEM_THRESHOLD="${2:-85}"    # percent

echo "Nodes with high resource utilization:"
kubectl top nodes --no-headers | awk -v cpu="${CPU_THRESHOLD}" -v mem="${MEM_THRESHOLD}" '
{
    cpu_pct = substr($3, 1, length($3)-1) + 0
    mem_pct = substr($5, 1, length($5)-1) + 0
    status = ""
    if (cpu_pct >= cpu) status = status "CPU:" cpu_pct "% "
    if (mem_pct >= mem) status = status "MEM:" mem_pct "%"
    if (status != "") print $1 " -> " status
}'
```

---

### S70. Lock file pattern to prevent concurrent script execution
```bash
#!/usr/bin/env bash
set -euo pipefail

LOCK_FILE="/var/lock/$(basename "$0").lock"

acquire_lock() {
    exec 9>"${LOCK_FILE}"
    if ! flock -n 9; then
        echo "ERROR: Another instance is running (lock: ${LOCK_FILE})" >&2
        exit 1
    fi
}

release_lock() {
    flock -u 9
    rm -f "${LOCK_FILE}"
}

trap release_lock EXIT
acquire_lock

echo "Running with exclusive lock..."
# Your script logic here
sleep 10
echo "Done."
```

---

## Part 4: Live Coding Round Scenarios (Timed)

---

### S71. Write a function that parses a log line and extracts fields
**Scenario (10 min):** Parse nginx access log lines into structured dicts.
```python
import re
from typing import Optional

LOG_PATTERN = re.compile(
    r'(?P<ip>\S+) \S+ \S+ \[(?P<time>[^\]]+)\] '
    r'"(?P<method>\S+) (?P<path>\S+) \S+" '
    r'(?P<status>\d+) (?P<bytes>\d+)'
)

def parse_nginx_log(line: str) -> Optional[dict]:
    m = LOG_PATTERN.match(line)
    if not m:
        return None
    return {
        'ip': m.group('ip'),
        'time': m.group('time'),
        'method': m.group('method'),
        'path': m.group('path'),
        'status': int(m.group('status')),
        'bytes': int(m.group('bytes'))
    }
```

---

### S72. Implement a simple rate limiter (token bucket)
```python
import time, threading

class TokenBucket:
    def __init__(self, rate: float, capacity: int):
        self.rate = rate          # tokens per second
        self.capacity = capacity
        self.tokens = float(capacity)
        self.last_refill = time.monotonic()
        self._lock = threading.Lock()

    def _refill(self):
        now = time.monotonic()
        elapsed = now - self.last_refill
        self.tokens = min(self.capacity, self.tokens + elapsed * self.rate)
        self.last_refill = now

    def acquire(self, tokens: int = 1) -> bool:
        with self._lock:
            self._refill()
            if self.tokens >= tokens:
                self.tokens -= tokens
                return True
            return False
```

---

### S73. Write a function to deep-merge two dicts (Helm values pattern)
```python
def deep_merge(base: dict, override: dict) -> dict:
    result = base.copy()
    for key, value in override.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

# Test
base = {'image': {'tag': 'v1', 'pullPolicy': 'IfNotPresent'}, 'replicas': 2}
override = {'image': {'tag': 'v2'}, 'resources': {'limits': {'cpu': '500m'}}}
print(deep_merge(base, override))
# {'image': {'tag': 'v2', 'pullPolicy': 'IfNotPresent'}, 'replicas': 2, 'resources': {...}}
```

---

### S74. Implement a circuit breaker
```python
import time
from enum import Enum

class State(Enum):
    CLOSED = 'closed'
    OPEN = 'open'
    HALF_OPEN = 'half_open'

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: float = 30.0):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self._state = State.CLOSED
        self._failure_count = 0
        self._opened_at: float = 0.0

    def call(self, func, *args, **kwargs):
        if self._state == State.OPEN:
            if time.monotonic() - self._opened_at >= self.recovery_timeout:
                self._state = State.HALF_OPEN
            else:
                raise RuntimeError("Circuit breaker is OPEN")
        try:
            result = func(*args, **kwargs)
            self._on_success()
            return result
        except Exception:
            self._on_failure()
            raise

    def _on_success(self):
        self._failure_count = 0
        self._state = State.CLOSED

    def _on_failure(self):
        self._failure_count += 1
        if self._failure_count >= self.failure_threshold:
            self._state = State.OPEN
            self._opened_at = time.monotonic()
```

---

### S75. Read a YAML file, modify a field, write it back
```python
#!/usr/bin/env python3
import argparse, yaml

def update_yaml_field(file_path: str, key_path: str, new_value):
    """key_path like 'spec.replicas' or 'image.tag'"""
    with open(file_path) as f:
        data = yaml.safe_load(f)

    keys = key_path.split('.')
    target = data
    for key in keys[:-1]:
        target = target.setdefault(key, {})
    target[keys[-1]] = new_value

    with open(file_path, 'w') as f:
        yaml.dump(data, f, default_flow_style=False)

    print(f"Set {key_path}={new_value} in {file_path}")

if __name__ == '__main__':
    parser = argparse.ArgumentParser()
    parser.add_argument('file')
    parser.add_argument('key_path')
    parser.add_argument('value')
    args = parser.parse_args()
    update_yaml_field(args.file, args.key_path, args.value)
```

---

### S76. Write a producer-consumer pipeline with threading
```python
import threading, queue, time, random

def producer(q: queue.Queue, n: int):
    for i in range(n):
        item = f"item-{i}"
        q.put(item)
        print(f"[PRODUCER] queued {item}")
        time.sleep(random.uniform(0.05, 0.2))
    q.put(None)  # sentinel

def consumer(q: queue.Queue, worker_id: int):
    while True:
        item = q.get()
        if item is None:
            q.put(None)  # pass sentinel along
            break
        print(f"[CONSUMER-{worker_id}] processing {item}")
        time.sleep(random.uniform(0.1, 0.3))
        q.task_done()

q = queue.Queue(maxsize=10)
t1 = threading.Thread(target=producer, args=(q, 20))
workers = [threading.Thread(target=consumer, args=(q, i)) for i in range(3)]

t1.start()
for w in workers: w.start()
t1.join()
for w in workers: w.join()
```

---

### S77. Detect anagram groups in a word list (frequency count approach)
```python
from collections import defaultdict

def find_anagram_groups(words: list[str]) -> list[list[str]]:
    """O(n * k) where k is max word length — faster than sorting approach."""
    groups = defaultdict(list)
    for word in words:
        key = tuple(sorted(word))  # or use Counter as frozenset
        groups[key].append(word)
    return [g for g in groups.values() if len(g) > 1]
```

---

### S78. Implement a simple pub/sub event bus
```python
from collections import defaultdict
from typing import Callable

class EventBus:
    def __init__(self):
        self._subscribers: dict[str, list[Callable]] = defaultdict(list)

    def subscribe(self, event: str, handler: Callable):
        self._subscribers[event].append(handler)

    def publish(self, event: str, data=None):
        for handler in self._subscribers.get(event, []):
            try:
                handler(data)
            except Exception as e:
                print(f"Handler {handler.__name__} failed for event '{event}': {e}")

# Usage
bus = EventBus()
bus.subscribe('deploy.success', lambda d: print(f"Notifying Slack: {d}"))
bus.subscribe('deploy.success', lambda d: print(f"Updating metrics: {d}"))
bus.publish('deploy.success', {'service': 'api', 'version': 'v1.2.3'})
```

---

### S79. Write a function that chunks a list for parallel processing
```python
from typing import Generator

def chunk_list(lst: list, size: int) -> Generator[list, None, None]:
    for i in range(0, len(lst), size):
        yield lst[i:i + size]

import concurrent.futures

def process_in_parallel(items: list, func, chunk_size: int = 100, workers: int = 10) -> list:
    results = []
    with concurrent.futures.ThreadPoolExecutor(max_workers=workers) as executor:
        futures = [
            executor.submit(lambda chunk: [func(x) for x in chunk], chunk)
            for chunk in chunk_list(items, chunk_size)
        ]
        for future in concurrent.futures.as_completed(futures):
            results.extend(future.result())
    return results
```

---

### S80. Write a config loader with environment variable override (12-factor app pattern)
```python
import os, yaml
from dataclasses import dataclass, field
from typing import Any

@dataclass
class AppConfig:
    database_url: str = ''
    redis_url: str = 'redis://localhost:6379'
    log_level: str = 'INFO'
    port: int = 8080
    debug: bool = False

    @classmethod
    def from_yaml(cls, path: str) -> 'AppConfig':
        config = {}
        if os.path.exists(path):
            with open(path) as f:
                config = yaml.safe_load(f) or {}

        # Environment variables override file config
        env_map = {
            'DATABASE_URL': ('database_url', str),
            'REDIS_URL': ('redis_url', str),
            'LOG_LEVEL': ('log_level', str),
            'PORT': ('port', int),
            'DEBUG': ('debug', lambda v: v.lower() in ('true', '1', 'yes')),
        }
        for env_key, (field_name, converter) in env_map.items():
            if env_key in os.environ:
                config[field_name] = converter(os.environ[env_key])

        return cls(**{k: v for k, v in config.items() if hasattr(cls, k)})
```
