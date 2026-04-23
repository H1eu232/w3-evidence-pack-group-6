# Evidence Pack — Group 6 Project - HexaCode

---

## 1. Cover

| Field | Details |
|---|---|
| **Group Number** | Group 6 |
| **Member Names** | Minh Tuấn · Thành Vinh · Anh Hoàng · Hoàng Nhân · Mạnh Khang · Ngọc Thắng · Hoàng Thông · Thành Tâm |
| **Database Engine** | Amazon RDS PostgreSQL |
| **Paradigm** | Relational |
| **Database Path** | RDS Postgres / relational |

---

## 2. Data Access Pattern Log

### Part A — Access Pattern Inventory *(từ Must-Have 1)*

| # | Entity / Feature | Operation | Frequency | Notes |
|---|---|---|---|---|
| 1 | `User` | `Lookup by username/email (đăng nhập)` | `High` | `Cần tốc độ phản hồi cực nhanh` |
| 2 | `Problem` | `Get problem details by ID` | `Medium` | `Truy xuất đề bài và metadata` | 
| 3 |`Submission` | `JOIN Submission + User + Problem (xem lịch sử)` | `High` | `Truy vấn quan hệ phức tạp để hiện tên user và tên bài` |
| 4 | `AI Chatbot` | `Retrieve vector embeddings (Bedrock)` | `Low` | `Kết nối dữ liệu bài tập với AI` |

---

### Part B — Engine & Paradigm Selection Reasoning
- Pattern 1(Auth):<br>
**Engine chosen:** `RDS PostgreSQL `  
**Paradigm:** `Relational`<br>
**Reasoning:**Dữ liệu của hệ thống (User, Problem, Submission) có mối quan hệ chặt chẽ và chằng chéo (Many-to-Many). Việc sử dụng Relational Paradigm cho phép chúng ta thực hiện các câu lệnh JOIN phức tạp để lấy thông tin từ nhiều bảng chỉ trong một lần truy vấn. PostgreSQL được chọn vì nó hỗ trợ tốt cả dữ liệu cấu trúc (SQL) và dữ liệu không cấu trúc (JSONB), đồng thời có khả năng mở rộng tốt cho các tính năng AI sau này thông qua pgvector


**If high-cost managed service:** 
- Estimated monthly cost: `~$250 - $300/month` based on `[db.m7i.large, 200GB SSD, Multi-AZ]`
- Cost justification: `Chi phí này là xứng đáng để có được High Availability (Multi-AZ) và khả năng tự động sao lưu. Với một hệ thống chấm bài, việc mất dữ liệu bài nộp là không thể chấp nhận được, nên việc đầu tư vào RDS là tối ưu hơn tự cài trên EC2.`

---

### Part C — Final Decision Log

**Decision date:** `[DD/MM/YYYY]`  
**Decision maker(s):** `[Names]`  
**Alternatives considered:**

| Alternative | Reason rejected |
|---|---|
| `[e.g. DynamoDB]` | `[e.g. Complex join patterns not suited for key-value]` |
| `[e.g. Self-hosted MongoDB on EC2]` | `[e.g. Operational overhead too high for team size]` |

---

## 3. Deployment Evidence

### 3.1 Database Instance Created & Running

**Screenshot:**

![RDS instance is running](./images/RDS-instance-running.png)

**Notes:**<br>
`- Chọn db.m7i.large (2 vCPU, 8GB RAM) thay vì dòng T (burstable) vì m7i cung cấp CPU performance ổn định, không bị throttle khi hết CPU credits — phù hợp với workload liên tục từ 3 Fargate services (problem, submission, identity) kết nối đồng thời.`<br>
`- Nhóm em chọn PostgreSQL thay vì MySQL hay DynamoDB vì PostgreSQL mạnh hơn về xử lý các kiểu dữ liệu phức tạp (JSONB) và có tính năng pgvector cực tốt để lưu trữ dữ liệu vector cho AI sau này và hệ thống bài tập cần tính nhất quán cao (ACID) và các câu lệnh JOIN phức tạp giữa User - Problem - Submission. NoSQL sẽ rất khó khăn và tốn kém khi thực hiện các truy vấn quan hệ như vậy.`

---

### 3.2 Encryption at Rest

**Screenshot:**

![Encryption at rest](./images/Encryption-and-MultiAZ-configuration.png)

**Notes:** 
`Encryption enabled với AWS-managed KMS key aws/rds — chọn AWS-managed thay vì customer CMK vì chưa có compliance mandate và muốn key rotation tự động, không làm giảm hiệu năng của hệ thống.`

---

### 3.3 Multi-AZ / High Availability

**Screenshot:**

![Multi-AZ](./images/Encryption-and-MultiAZ-configuration.png)

**Notes:** 
`Single-AZ rẻ hơn nhưng rủi ro cao. Chọn Multi-AZ để đảm bảo Tính sẵn sàng cao (High Availability). Nếu một trung tâm dữ liệu của AWS gặp sự cố (thiên tai, mất điện), RDS sẽ tự động chuyển hướng (failover) sang Zone dự phòng trong < 60 giây, giúp hệ thống không bị gián đoạn.`

---

### 3.4 Automated Backups

**Screenshot:**

![Automated Backups](./images/Automated-backups.png)

**Notes:** 
`Cấu hình sao lưu tự động hàng ngày với thời gian lưu trữ 7 ngày. Sao lưu tự động loại bỏ sai sót của con người. Nó cho phép Point-in-Time Recovery, nghĩa là bạn có thể khôi phục dữ liệu chính xác đến từng giây trong quá khứ nếu lỡ tay chạy lệnh DELETE nhầm.`

---

### 3.5 Parameter / Configuration Tuning *(nếu áp dụng)*

**Screenshot / CLI output:**

```
[Custom parameter group settings]
```

**Notes:**  
`[e.g. "Tăng max_connections lên 200 vì mỗi Fargate task mở tối đa 5 connection. Default 100 sẽ bị saturate ở ~20 tasks."]`

---

## 4. Working Query Evidence
  
---

### 4.1 Operation 1 — *[JOIN query]*

**Paradigm requirement:**  
- Relational → JOIN qua ≥2 related tables  
- Key-Value → Query theo partition key  
- Document → Aggregation pipeline  
- Graph → N-hop traversal  

**Query / Command:**

```sql
SELECT 
    u.username, 
    p.title AS problem_title, 
    s.verdict_code, 
    s.status_code, 
    s.created_at
FROM 
    submission.submissions s
JOIN 
    app_identity.users u ON s.user_id = u.id
JOIN 
    problem.problems p ON s.problem_id = p.id
ORDER BY 
    s.created_at DESC;
```

**Result screenshot:**

![JOIN Query](./images/JOIN-query.png)

**What this demonstrates:**  
`Truy vấn kết hợp thông tin từ 3 bảng khác nhau trong một câu lệnh duy nhất (Relation Model). Chỉ với 1 request, ứng dụng có thể lấy toàn bộ thông tin cần thiết, giảm thiểu số lượng kết nối tới DB, tối ưu hóa tốc độ tải trang.`

---

### 4.2 Operation 2 — *[Indexed lookup]*

**Paradigm requirement:**  
- Relational → Indexed lookup (EXPLAIN shows Index Scan)  
- Key-Value → GSI query (không Scan)  
- Document → Indexed-field lookup  
- Graph → Property / node lookup  

**Query / Command:**

```sql
SELECT
    id,
    username,
    status_code
FROM
    app_identity.users
WHERE
    username = 'Hoang_Admin';
```

**Result screenshot:**

![Indexed lookup](./images/Indexed-lookup.png)

**What this demonstrates:**  
`Hệ thống sử dụng Index Scan thay vì Sequential Scan khi tìm kiếm User. Nếu không có Index, DB phải quét từng dòng một (Sequential Scan). Với Index, tốc độ tìm kiếm là cực nhanh (O(log n)). Hệ thống được thiết kế để đảm bảo tính scalability, vẫn chạy mượt khi có hàng triệu người dùng.`

---

## 5. Lambda + Bedrock Evidence
**Screenshot:**

1. The user asks the AI questions in the frontend chat widget.<br>![Ask Chatbot in frontend chat widget](./images/ask-AI-from-FE.png)


2. A lambda is triggered when a request is received.<br>![Lambda CloudWatch log entry](./images/Lambda-log-entry.png)


3. Successful response from aws bedrock -> lambda in frontend.<br>![Successful Response ](./images/Response-from-bedrock.png)


---

### 5.1 Lambda Trigger — CloudWatch Logs

**Log stream:** `[/aws/lambda/<function-name>]`  
**Timestamp:** `[YYYY-MM-DD HH:MM:SS UTC]`

**CloudWatch log entry:**

```
[Paste relevant log lines here — include REQUEST ID, timestamp, and key output]
e.g.
START RequestId: abc-123 Version: $LATEST
INFO Triggering Bedrock RAG for query: "..."
END RequestId: abc-123
REPORT RequestId: abc-123 Duration: 1243.12 ms Billed: 1300 ms
```

**Notes:**  
`[e.g. "Lambda được trigger từ SQS message sau khi submission-service push event. Cold start ~800ms, warm ~120ms."]`

---

### 5.2 Bedrock Retrieve / RetrieveAndGenerate Response

**Method used:** `[ ]` RetrieveAndGenerate &nbsp;&nbsp; `[ ]` Retrieve (then generate separately)  
**Knowledge Base ID:** `[kb-XXXXXXXXXX]`  
**Model used:** `[e.g. anthropic.claude-3-sonnet-20240229-v1:0]`

**Response (from Lambda log or CLI):**

```json
{
  "output": {
    "text": "[AI response text here]"
  },
  "citations": [
    {
      "retrievedReferences": [
        {
          "content": { "text": "[Source chunk]" },
          "location": { "s3Location": { "uri": "s3://..." } }
        }
      ]
    }
  ]
}
```

**Notes:**  
`[e.g. "Vector search hit S3 vector bucket, retrieved top-3 relevant chunks, passed vào Claude claude-3-sonnet. Response latency ~2.1s end-to-end từ Lambda invocation."]`

---

## 6. VPC + Networking Evidence

### 6.1 S3 Gateway Endpoint — Route Table

**Screenshot:**

![S3 Gateway Endpoint](./images/S3-gateway-endpoint.png)

![Route Table](./images/Route-table.png)

**Notes:**<br>
`- Thiết lập đường kết nối riêng từ VPC tới S3.`<br>
`- Chọn sử dụng Gateway Endpoint thay cho internet/NAT gateway vì dữ liệu file bài nộp đi thẳng từ Server tới S3 qua mạng nội bộ AWS, không bao giờ lộ ra Internet và truy cập qua Endpoint là miễn phí, trong khi đi qua NAT Gateway bạn phải trả tiền trên mỗi GB dữ liệu. Với hệ thống nhiều file bài tập, đây là cách tiết kiệm chi phí tối ưu nhất.`

---

### 6.2 DB Security Group — Inbound Rules (App-tier SG as Source)

**Screenshot:**

![Security Group](./images/SG-inbound-rule.png)

**Notes:**<br>
`Inbound Rule chỉ cho phép duy nhất Security Group của tầng ứng dụng (App-tier) truy cập vào cổng Database. Cách thiết lập này tuân thủ nguyên tắc "Least Privilege", cô lập hoàn toàn Database khỏi các truy cập trái phép từ bên ngoài môi trường VPC.`<br>
`Dùng SG ID vì dù server tầng App có thay đổi IP liên tục (do Auto Scaling), Database vẫn tự động nhận diện và cho phép truy cập mà không cần cấu hình lại thủ công.`

---

## 7. Negative Security Test
---

### 7.1 Test Description

**What was attempted:**  
`Thử kết nối từ máy cá nhân tới Endpoint RDS và nhận lỗi Connection timed out`

**From:** `[e.g. EC2 instance i-XXXXXXXXXX, SG: sg-YYYYYYYY (not app-tier)]`  
**To:** `[e.g. RDS endpoint xxx.rds.amazonaws.com:5432]`  
**Expected result:** Connection refused / timeout  

---

### 7.2 Evidence of Denial

**Screenshot / CLI output:**

```
[e.g.
$ psql -h xxx.rds.amazonaws.com -U admin -d mydb
psql: error: connection to server at "xxx.rds.amazonaws.com" (10.0.3.45), 
port 5432 failed: Connection timed out
]
```

**Notes:**  
`[e.g. "Timeout sau 30s confirm rằng security group không cho phép inbound 5432 từ SG ngoài app-tier. VPC flow log (nếu có) cũng show REJECT action."]`

---

## 8. Bonus — Real-World Ops Scenario *(Tùy chọn)*

> Chỉ điền nếu nhóm thực hiện ít nhất 1 scenario từ Bonus section.

---

### 8.1 Scenario Name

**Scenario attempted:** `[e.g. Simulated AZ failure / Blue-green deployment / Point-in-time restore]`  
**Date & time:** `[YYYY-MM-DD HH:MM UTC]`

---

### 8.2 Pre-condition (Before)

**Screenshot / metric:**

> *(Embed ảnh hoặc paste metric/log trước khi thực hiện scenario)*

```
[Pre-state evidence]
```

---

### 8.3 Execution

**Steps performed:**

1. `[Step 1]`
2. `[Step 2]`
3. `[Step 3]`

**Timing:**

| Event | Timestamp | Elapsed |
|---|---|---|
| Scenario triggered | `HH:MM:SS` | 0s |
| Failure detected | `HH:MM:SS` | `Xs` |
| Failover complete | `HH:MM:SS` | `Xs` |
| Service restored | `HH:MM:SS` | `Xs` |

---

### 8.4 Post-condition (After)

**Screenshot / metric:**

> *(Embed ảnh hoặc paste metric/log sau khi scenario hoàn tất)*

```
[Post-state evidence]
```

---

### 8.5 Reflection

**What worked well:**  
`[e.g. "Multi-AZ failover hoàn tất trong 87s, trong ngưỡng RTO mong đợi < 2 phút."]`

**What surprised us / could be improved:**  
`[e.g. "ElastiCache không failover tự động trong thời gian RDS failover — cache cold start gây tăng latency thêm ~15s. Cần pre-warm cache sau failover."]`

---

*— End of Evidence Pack —*
