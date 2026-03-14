# s10: Team Protocols

`s01 > s02 > s03 > s04 > s05 > s06 | s07 > s08 > s09 > [ s10 ] s11 > s12`

> *"Các đồng đội (Teammates) cần các quy tắc giao tiếp chung"* -- một dạng thức yêu cầu-phản hồi (request-response pattern) thúc đẩy (drives) mọi sự đàm phán (negotiation).

## Vấn đề

Trong s09, các teammate có làm việc và giao tiếp nhưng thiếu đi sự phối hợp (coordination) có cấu trúc:

**Shutdown**: Việc chấm dứt (killing) một thread để lại những file chỉ được viết một nửa (half-written) và config.json thì cũ kỹ (stale). Bạn cần một cái bắt tay (handshake): bên lead (lead) đưa ra một yêu cầu (requests), bên teammate phê duyệt (approve) cho việc kết thúc (finish and exit) hoặc từ chối (reject) để tiếp tục làm việc.

**Plan approval** (Phê duyệt kế hoạch): Khi lead nói "refactor the auth module," teammate ngay lập tức bắt đầu. Đối với những thay đổi rủi ro cao (high-risk), trước tiên lead nên xem xét cái plan đó.

Bên nào cũng có chung một cấu trúc: một bên gửi yêu cầu (request) đính kèm một ID định danh duy nhất (unique ID), bên còn lại đưa ra phản hồi (responds) tham chiếu tới chính ID đó.

## Giải pháp

```
Shutdown Protocol            Plan Approval Protocol
==================           ======================

Lead             Teammate    Teammate           Lead
  |                 |           |                 |
  |--shutdown_req-->|           |--plan_req------>|
  | {req_id:"abc"}  |           | {req_id:"xyz"}  |
  |                 |           |                 |
  |<--shutdown_resp-|           |<--plan_resp-----|
  | {req_id:"abc",  |           | {req_id:"xyz",  |
  |  approve:true}  |           |  approve:true}  |

Shared FSM:
  [pending] --approve--> [approved]
  [pending] --reject---> [rejected]

Trackers:
  shutdown_requests = {req_id: {target, status}}
  plan_requests     = {req_id: {from, plan, status}}
```

## Cách thức hoạt động

1. Lead khởi tạo quá trình tắt (shutdown) thông qua việc sinh ra một request_id và gửi qua inbox.

```python
shutdown_requests = {}

def handle_shutdown_request(teammate: str) -> str:
    req_id = str(uuid.uuid4())[:8]
    shutdown_requests[req_id] = {"target": teammate, "status": "pending"}
    BUS.send("lead", teammate, "Please shut down gracefully.",
             "shutdown_request", {"request_id": req_id})
    return f"Shutdown request {req_id} sent (status: pending)"
```

2. Teammate nhận request và phản hồi với kết quả approve/reject.

```python
if tool_name == "shutdown_response":
    req_id = args["request_id"]
    approve = args["approve"]
    shutdown_requests[req_id]["status"] = "approved" if approve else "rejected"
    BUS.send(sender, "lead", args.get("reason", ""),
             "shutdown_response",
             {"request_id": req_id, "approve": approve})
```

3. Giao thức phê duyệt kế hoạch (Plan approval) áp dụng một quy luật (pattern) giống hệt. Teammate nộp (submits) một plan (sinh ra một request_id), lead xem xét duyệt qua (tham chiếu cùng request_id đó).

```python
plan_requests = {}

def handle_plan_review(request_id, approve, feedback=""):
    req = plan_requests[request_id]
    req["status"] = "approved" if approve else "rejected"
    BUS.send("lead", req["from"], feedback,
             "plan_approval_response",
             {"request_id": request_id, "approve": approve})
```

Một đối tượng Máy trạng thái hữu hạn (FSM), áp dụng cho hai ứng dụng (two applications). Cùng một cỗ máy trạng thái (state machine) `pending -> approved | rejected` sẽ xử lý bất kỳ request-response protocol nào.

## Những gì đã thay đổi từ s09

| Thành phần      | Trước đây (s09)  | Sau này (s10)                  |
|----------------|------------------|------------------------------|
| Tools          | 9                | 12 (+shutdown_req/resp +plan)|
| Shutdown       | Chỉ thoát theo cách tự nhiên (Natural exit) | Request-response handshake   |
| Chặn kế hoạch (Plan gating)    | Không có             | Submit/review cùng với phê duyệt (approval)  |
| Sự tương quan (Correlation)    | Không có             | request_id trên mỗi request       |
| FSM            | Không có             | pending -> approved/rejected |

## Dùng thử

```sh
cd learn-claude-code
python agents/s10_team_protocols.py
```

1. `Spawn alice as a coder. Then request her shutdown.`
2. `List teammates to see alice's status after shutdown approval`
3. `Spawn bob with a risky refactoring task. Review and reject his plan.`
4. `Spawn charlie, have him submit a plan, then approve it.`
5. Type `/team` to monitor statuses
