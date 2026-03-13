**Project:** Trackity Real-time Parser

**Services Affected:** Flink TaskManagers, Kafka Broker, Druid Ingestor

**Date:** 2026-02-04

**Status:** **RESOLVED**

---

## 1. Symptom Recognition (Dấu hiệu nhận biết sự cố)

### A. Tại Kafka & Druid (Nghịch lý Offset)

- **Lag = 0:** Druid Console hiển thị **Lag = 0**, nhưng thực tế DataSource không tăng thêm bất kỳ record nào.
    
- **Gap Offset:** Khi kiểm tra bằng **Conduktor**, Latest Offset (High Watermark) đã nhảy lên tới **390,733 [https://kafka-console.prod.tiki.services/console/tiki-trackity-kafka/topics/trackity.tiki.v2.events_parsed.r3?tab=partitions](https://kafka-console.prod.tiki.services/console/tiki-trackity-kafka/topics/trackity.tiki.v2.events_parsed.r3?tab=partitions)**, nhưng tại **Druid Supervisor**, Current Offset bị đứng im hoàn toàn tại **104,537**.
- **Silent Task:** Các Indexing Task của Druid vẫn ở trạng thái `RUNNING` nhưng không consume được dữ liệu, do bị kẹt bởi tại offset 104,537.
    

### B. Tại Flink (Logs & Metrics)

Sự cố bắt đầu bằng chuỗi lỗi xuất hiện dày đặc trong log của TaskManager:

- **Log Error:** Xuất hiện các dòng báo lỗi **"Unable to commit transaction..."** : Điều này xảy ra vì TaskManager không thể gửi tín hiệu xác nhận hoàn tất (COMMIT) transaction tới Kafka Broker; hoặc **"RecipientUnreachableException:..":** JobManager (trackity-parser-prod pod) hết mem nên restart, khiến task manager (trackity-parser-prod-task-manager) không thể gửi tín hiệu gì về được.
    

![Apache Flink Architecture](https://i.ytimg.com/vi/GqNZoo-qPgg/maxresdefault.jpg)

![](https://docs.tiki.com.vn/download/attachments/284362838/Screenshot%202026-02-05%20at%2016.19.37.png?version=1&modificationDate=1770283183465&api=v2 "Data Platform > INCIDENT REPORT: FLINK & KAFKA TRANSACTION BLOCK > Screenshot 2026-02-05 at 16.19.37.png")

![](https://docs.tiki.com.vn/download/attachments/284362838/Screenshot%202026-02-05%20at%2016.18.28.png?version=1&modificationDate=1770283115871&api=v2 "Data Platform > INCIDENT REPORT: FLINK & KAFKA TRANSACTION BLOCK > Screenshot 2026-02-05 at 16.18.28.png")

  

  

|   |
|---|
|log/flink--kubernetes-taskmanager-0-trackity-parser-prod-taskmanager-3-5.log:2026-02-04 03:47:18,258 ERROR org.apache.flink.connector.kafka.sink.KafkaCommitter         [] - Unable to commit transaction (org.apache.flink.streaming.runtime.operators.sink.committables.CommitRequestImpl@7776a776) because it's in an invalid state. Most likely the transaction has been aborted for some reason. Please check the Kafka logs for more details.<br>log/flink--kubernetes-taskmanager-0-trackity-parser-prod-taskmanager-3-5.log.1:org.apache.flink.runtime.rpc.exceptions.RecipientUnreachableException: Could not send message [RemoteFencedMessage(a5ccd2be1fa1a9b70b4fe256d6b54d9d, RemoteRpcInvocation(JobMasterGateway.updateTaskExecutionState(TaskExecutionState)))] from sender [Actor[akka.tcp://flink@10.140.29.249:6122/temp/jobmanager_8$r7t]] to recipient [Actor[akka://flink/user/rpc/jobmanager_8#-823803892]], because the recipient is unreachable. This can either mean that the recipient has been terminated or that the remote RpcService is currently not reachable.|

  

- **Checkpoint Failures:** Log của JobManager ghi nhận các Checkpoint liên tục ở trạng thái `FAILED` hoặc `EXPIRED`. Một số checkpoint thành công tạm thời nhưng có `duration` cực cao.
    

---

## 2. Investigation & Root Cause Analysis (RCA)

### A. Điều tra cấu hình Flink (Config Investigation)

Dựa trên file `ConfigMap`, nguyên nhân gốc rễ nằm ở việc phân bổ tài nguyên không hợp lý:

- **Slot Density quá dày:** Ép **32 slots** chạy trên 1 task manager pod có **2.0 vCPU**. Với mật độ này, CPU bị nghẽn khi 32 Kafka Producers cùng thực hiện các commit đồng thời (chỉ có 2 pod task manager quản lý 64 task).
    
- **Thiếu hụt RAM cho JVM:** Chỉ có **5GB RAM** cho 32 slots (~150MB/slot). Khi RocksDB và Kafka Client chiếm dụng memory, TaskManager bị OS **OOMKilll** do JVM Overhead bị tràn.
    

![](https://docs.tiki.com.vn/download/attachments/284362838/Screenshot%202026-02-05%20at%2016.08.10.png?version=1&modificationDate=1770282497097&api=v2 "Data Platform > INCIDENT REPORT: FLINK & KAFKA TRANSACTION BLOCK > Screenshot 2026-02-05 at 16.08.10.png")

### B. Cơ chế gây nghẽn (The Uncommitted Transaction)

- Do Flink crash liên tục ngay tại thời điểm đang thực hiện checkpoint, lệnh **COMMIT** chính thức không bao giờ được gửi tới Kafka.
    
- **LSO Barrier:** Kafka Broker giữ các transaction này ở trạng thái "Open". Vì Druid mặc định chạy ở chế độ `read_committed`, nó bị chặn lại tại **LSO (Last Stable Offset)** (tại vị trí 104,537) và từ chối đọc bất kỳ dữ liệu nào phía sau cho đến khi transaction đó được đóng lại (Commit hoặc Abort).
    

---

## 3. Resolution Steps (Quá trình xử lý)

### Step 1: Fix Flink Configuration

Chúng tôi đã điều chỉnh cấu hình để đảm bảo Flink có thể commit thành công:

- **Re-scale:** Giảm mật độ xuống **8 slots/TM** và tăng lên **8 TaskManagers** (tổng 64 slots).
    
- **Memory Tuning:** Nâng lên **8GB RAM/TM** và set `jvm-overhead.max: 4GB`.
    
- **Result:** Flink ổn định, log không còn báo lỗi commit, các Checkpoint thành công liên tục với latency thấp.
    
- ![](https://docs.tiki.com.vn/download/attachments/284362838/Screenshot%202026-02-05%20at%2016.09.59.png?version=1&modificationDate=1770282607512&api=v2 "Data Platform > INCIDENT REPORT: FLINK & KAFKA TRANSACTION BLOCK > Screenshot 2026-02-05 at 16.09.59.png")
- ![](https://docs.tiki.com.vn/download/attachments/284362838/Screenshot%202026-02-05%20at%2016.11.22.png?version=1&modificationDate=1770282690561&api=v2 "Data Platform > INCIDENT REPORT: FLINK & KAFKA TRANSACTION BLOCK > Screenshot 2026-02-05 at 16.11.22.png")

### Step 2: Unblock Druid (Quick Fix)

Để không phải chờ đợi Kafka Broker tự động timeout các transaction cũ (có thể mất 60 phút hoặc hơn), chúng tôi đã chọn giải pháp nhanh nhất:

- **Action:** Thay đổi cấu hình Druid sang **`isolation.level = read_uncommitted`**.
    
- **Lý do:** Ép Druid bỏ qua rào cản LSO để consume ngay lập tức backlog dữ liệu từ offset 104k đến 390k.
    
- **Result:** Hệ thống unblocked ngay lập tức, dữ liệu trở lại trạng thái Real-time chỉ sau vài phút.
    

---

## 4. Final Status & Lessons Learned

- **Status:** Hệ thống hoạt động ổn định, dữ liệu đồng bộ real-time.
    
- **Bài học:**
    
    - Khi log Flink báo **"Failed to commit"**, cần kiểm tra ngay tài nguyên (CPU/RAM) thay vì chỉ kiểm tra kết nối Kafka.
        
    - Không nên tin hoàn toàn vào chỉ số "Lag = 0" của Druid khi đang chạy `read_committed` nếu dữ liệu không về.
        
    - Cấu hình Slot density thấp (4-8 slots/node) giúp Flink ổn định hơn khi xử lý transaction Kafka dữ liệu lớn.