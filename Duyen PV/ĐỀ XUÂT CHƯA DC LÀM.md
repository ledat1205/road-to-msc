# Tổng quan

![2x2 of online vs. offline environments, and candidate retrieval vs. ranking.](https://eugeneyan.com/assets/discovery-2x2.webp)

[https://eugeneyan.com/writing/system-design-for-discovery/](https://eugeneyan.com/writing/system-design-for-discovery/)

![Advances in Session-Based and Session-Aware Recommendation, Dissertation zur Erlangung des Grades eines DoktorsderNaturwissenschaften der Technischen Universität Dortmund an der Fakultät für Informatik von Malte Ludewig, 2020](https://zoe.lundegaard.ai/static/84db07c9c9ee1f33e7a4d3872f2a8ccc/40601/advances-in-session-based-and-session-aware-recommendation.png)

[https://zoe.lundegaard.ai/blog/real-time-customer-behavior-via-a-session-based-approach-part-1/](https://zoe.lundegaard.ai/blog/real-time-customer-behavior-via-a-session-based-approach-part-1/)

# Các key component của một recommendation system

  

|**Giai Đoạn**|**Mô Tả**|
|---|---|
|**1. Event Ingestion**|Thu thập hành vi người dùng (clicks, views, add-to-cart, purchases, impressions) theo thời gian thực. <br><br>Thu thập thay đổi sản phẩm/ ngành hàng.<br><br>Thu thập thay đổi về người dùng.|
|**2. Stream Processing**|Xử lý sự kiện thời gian thực (số lần xem, click theo category/sản phẩm).<br><br>Tuỳ theo loại feature store mà update real time/ batch.|
|**3. Feature Store**|Lưu trữ và quản lý features (chi tiết: <br><br>- **User features**: thông tin về giới tính, tuổi; lịch sử xem, tìm kiếm, mua, đánh giá sản phẩm.<br><br>- **Item features**: category, brand, popularity (nhiều lượt bán)<br><br>- **Context features**: campaign day; thiết bị, vị trí của user.<br><br>Features cần phải low-latency (lưu trên Redis/Feast) cho online inference.<br><br>Features cần phải thống nhất giữa online và offline vì offline training.|
|**4. Training**|- Heuristic / Business rules<br>    <br>    - Ví dụ: cộng điểm nếu same brand, same category với item đã xem gần đây; promoted items, items with active campaigns.<br>        <br>    - Ưu: đơn giản, explainable, deterministic.<br>        <br>    - Nhược: low recall nếu dùng độc lập.<br>        <br>- Popularity / Trending<br>    <br>    - Top-N theo sales/trending trong 1h/24h/7d.<br>        <br>    - Dùng làm fallback (cho non user, user mới) hoặc boost trending items tuỳ ý biz.<br>        <br>- Session / Recent-history based (real-time)<br>    <br>    - Lấy items từ user session (last 5–30 phút) — item→item co-view / co-click từ thu thập từ tất cả các session của tất cả các user gần đây.<br>        <br>    - Thích hợp cho “people who viewed X also viewed Y”. <br>        <br>- Collaborative Filtering / Item-Item co-occurrence<br>    <br>    - Co-purchase, co-view matrices (sparse), item-to-item similarity via PMI / cosine on co-visit vectors.<br>        <br>    - Ưu: good recall for similar items.<br>        <br>    - Nhược: cần lưu precomputed neighbors; không bắt trend nhanh.<br>        <br>- Embedding based ANN retrieval (vector search)<br>    <br>    - Item embeddings (content + behavioral), user embeddings or query embeddings; retrieval via ANN (Faiss, Milvus, RedisVector).<br>        <br>    - Ưu: flexible, captures semantics, high recall.<br>        <br>    - Nhược: cần infra cost rất lớn để lưu trữ và cập nhật các item + user embedding.<br>        <br>- Reranking model<br>    <br>    - Model (e.g., LightGBM / GBDT) rank sản phẩm dự trên yếu tố biz, như top deal, nhiều lượt mua, view (tận dụng feature store lookups),...<br>        <br>    - Dùng kết hợp với CF/ ANN/ GNN.<br>        <br>    - Ưu: can learn complex combos; dễ A/B test.|
|**6. Serving**|Hybrid retrieval/ recall: Kết hợp nhiều nguồn để ra được kết quả các sản phẩm đề xuất<br><br>- **Embedding based ANN retrieval** Sử dụng ANN queries bằng cách dùng session embeddings của user để lấy items tương tự. <br>- **Collaborative Filtering (CF)**: Recall K items từ user-based hoặc item-based CF.<br>- Pre-rank: lọc nhanh candidate set từ vài nghìn item còn khoảng vài trăm item, trước khi gửi qua ranking nặng hơn. (The **matching score** is computed as a simple **dot product** between the user and item vectors.)<br>- **Deep re-rank:** Mô hình LightGBM / GBDT<br>    <br>    Có thể áp dụng **business rules** như:<br>    <br>    - Ưu tiên item có CTR/CVR cao<br>        <br>    - Ưu tiên item đang giảm giá mạnh<br>        <br>    - Ưu tiên item giao hàng nhanh, tồn kho nhiều, v.v. <br>        <br>- **Popularity**: Lấy top 100 items phổ biến (từ Redis sorted sets, dựa trên views/sales).<br>- **Campaign/Ads Buckets**: Recall 100 items đang được chạy campaign/ ad.|
|**7. Observability & Experiments**|Log: (request, latency, feature freshness, metric drift).<br><br>Chạy A/B experiments với holdout buckets, đo KPIs: CTR (click-through rate), conversion, GMV.|

  

  

# Inference - Smartflow

### 1. DAG **recommendation_data_aggregation_daily_v2**

- **Hiện trạng**:
    
    - Chạy 3 lần/ngày (1h15, 5h15, 7h15).
        
    - Gom data từ nhiều bảng log (product_impression, trackity_tiki_click, core_events, nmv,...), hiện tại phải quét nhiều partition (có thể là 30 ngày → 365 ngày).
        
    - Đây là DAG gốc cấp data cho training và inference.
        

### 2. DAG **recommendation_train_title_v2 / recommendation_train_interaction_v2 / recommendation_train_user_favorite_cate_v2**

- **Hiện trạng**:
    
    - Các DAG train hiện ổn định, chưa cần thay đổi.
        
    - Train title (1h), interaction (4h30), user_fav_cate (14h monthly).
        

### 3. DAG **recommendation_infer_data_agg**

- **Hiện trạng**:
    
    - Chạy 8 lần/ngày (8h → 22h, mỗi 2h).
        
    - Lý do: bảng `dim_product_full` cập nhật nhiều field quan trọng (top deal, is_salable, is_brand_boosting, support_2h_delivery).
        
    - Mỗi lần chạy tốn ~1.5TB scan vì phải scan cả dim_prodcut_full.
        

### 4. DAG **recommendation_infer_area_home_p1_v2 / p2_v2**

- **Hiện trạng**:
    
    - P1: chạy 4 lần/ngày (10h30, 14h30, 18h30, 22h30) → nặng vì personalize.
        
    - P2: chạy 3 lần/ngày (12h30, 16h30, 20h30) → cũng nặng vì personalize.
        
    - Phụ thuộc vào `recommendation_infer_data_agg` + `recommendation_data_aggregation_daily_v2` + models.
        
- **Logic chung cho các task data_home_widget_***
    - **Nhánh 1 (Personalized)**: Kết hợp sản phẩm, tính điểm từ danh mục yêu thích, top view, và ngẫu nhiên; lọc 500 sản phẩm/user.
        
    - **Nhánh 2 (Recent Click)**: Lọc sản phẩm đã xem, tính điểm recent (1/(1 + ngày)), kết hợp sản phẩm tương tự; lọc 500 sản phẩm/user.
        
    - **Nhánh 3 (Cart-Based)**: Chưa dùng.
        
    - **Nhánh 4 (Top View Random)**: Gán ngẫu nhiên 300 sản phẩm top view của từng user.
        
    - **Nhánh 5 (Cold Start)**: Gán 120 sản phẩm top view cho cold start (tất cả các new user đều nhìn thấy 120 sản phẩm này).
        
    - **Loại bỏ đã mua và tương tự**: Dùng customer_purchased và product_similar_prediction_v2.
        
    - **Gộp và xếp hạng**: Đảm bảo tỷ lệ 3:2:1 (personalized:recent:top) bằng ntile.
        
    - **Đầu ra**: tiki-dwh.recommendation.data_home_widget_*_YYYYMMDD.
        

- **Logic chung cho các task infer_home_widget_***

- - Gộp dữ liệu từ data_home_*, loại bỏ trùng lặp root_product_id.
        
    - **Đầu ra**: tiki-dwh.recommendation.infer_home_widget_*_YYYYMMDD.
        

- **Logic chung cho task sync_infer_home_widget_:** Đồng bộ từ tiki-dwh.recommendation.infer_home_widget_*_YYYYMMDD vào ScyllaDB (ví dụ: p_reco_home_widget_import_v2_v1_202507011800).
- **So sánh:**
    
    |Widget|**You May Like**|**Top Deal**|**Hot Import**|**Repeat Purchase**|**Infinity Tab**|
    |---|---|---|---|---|---|
    |**Purpose**|General product recommendations|"Tiki Hero" deal recommendations|Imported product recommendations|Recommend products users are likely to repurchase|Personalized recommendations for infinity scrolling tab|
    |**Product Filter**|Broad, high-quality products: salable, not free gifts, excludes OEM brands, specific categories, official stores, rating > 3 or null|Only tiki_hero = 1 products, plus "You May Like" filters|Imported from specific countries, no vouchers/services, plus "You May Like" filters|Salable, not free gifts, seller_simple, in repurchase-prone categories|Uses pre-filtered data_home_good_products_20250828 (assumed high-quality products)|
    |**Recent Click CTE**|recent_click, recent_related_click|recent_click_hero, recent_related_click_hero|recent_click_import, recent_related_click_import|None (no recent views)|recent_click, recent_related_click (same as "You May Like")|
    |**Top Products**|Top 300 viewed products|Top 300 viewed tiki_hero products|Top 300 viewed imported products|None (no top products)|Top 300 viewed products (from product_suggest)|
    |**Non-User Recommendations**|120 top-viewed products|120 top-viewed tiki_hero products|120 top-viewed imported products|None (logged-in users only)|120 top-viewed products (same as "You May Like")|
    |**Additional Filters**|None|tiki_hero = 1|origin restrictions, no vouchers/services|Repurchase-prone categories, excludes sensitive categories|None (relies on pre-filtered product_raw)|
    |**Scoring Mechanism**|new_score = 500 - 2*batch + score, 3:2:1 ratio|Same as "You May Like"|Same as "You May Like"|delta_day (purchase recency)|new_score = 500 - 2*batch + score, 3:2:1 ratio|
    |**Input Data**|User views, category preferences, top products|Same, filtered for tiki_hero|Same, filtered for imports|Purchase history, repurchase categories|User views, pre-computed personalized list, top products|
    |**Output**|key, root_product_id, product_id, score|Same as "You May Like"|Same as "You May Like"|key, products (array of spid, rank)|key, root_product_id, product_id, score|
    
      
    

### 5. DAG **recommendation_infer_area_pdp_v2**

- **Hiện trạng**:
    
    - Chạy 5 lần/ngày (8h30, 10h30, 14h30, 18h30, 22h30).
        
    - Hiện tại chưa personalize.
        
- **So sánh:**
    
    |**Widget**|Top deal|**Similar products**|**Complementary Product**|**Infinity Tab**|
    |---|---|---|---|---|
    |**Purpose**|Recommend products similar to the input product based on a prediction model|Same as Top deal|Recommend products from complementary categories (not in same category) to the input product’s category|Recommend products from categories frequently viewed together, with similarity and new product boosts|
    |**Product Filter**|Salable, tiki_hero, rating > 3 or null|Salable, not free gifts, rating > 3 or null|Salable, not free gifts, rating > 3 or null, non-OEM brands, official stores|Salable, not free gifts, product_id not null, rating > 3 or null (product and seller level)|
    |**Recommendation Basis**|From product_similar_prediction (max_score_model, total_cate_score, score_title). **Note**: total_cate_score is the score reflecting the sale volume of the product within its own category.|Same as Top deal|Complementary category pairs from vw_related_product_v2|Similarity model + customer behavior-based category pairs (cate_pair_*)|
    |**Scoring Mechanism**|score_root = max_score_model + total_cate_score|Same as Top deal|score = 1 + w_not_oem + w_official_store, with boosts for matching or higher price segments (+0.2 or +0.1)|final_score = max_score_model + new_score_cate_cus + order_similar, +1000 for new products|
    |**Category Involvement**|None (relies solely on similar products)|Same as Top deal|Top 5 complementary categories per input category|Top 3 category pairs based on customer co-viewing behavior|
    |**Additional Filters**|score_title < 1, root_product_id != spid|Same as Top deal|Non-OEM brands, official stores, suggest_sale_price <= 0.3 * sale_price|score_title < 1, root_product_id != spid, category pairs with num_cus_pair_d90 >= 10, num_cus_d90 >= 100|
    |**Input Tables**|product_features, product_similar_prediction_v2_*|Same as Top deal|product_features, vw_related_product_v2|product_features, product_similar_prediction_v2_*, cate_pair_*|
    |**Output Limit**|100 products per mpid|Same as Top deal|No explicit limit (depends on category pairs)|500 products per mpid|
    |**Sorting Criteria**|score_root DESC|Same as Top deal|num_sold_product_d90 DESC, score DESC, sale_price ASC|final_score DESC, sale_price ASC|
    |**Special Features**|Simple similarity-based ranking|Same as Top deal|Boosts for non-OEM, official stores, and price segment matching; limits to 80 products per category|Boosts for new products (+1000), uses customer co-viewing data for category relevance|
    
      
    

# Serving - Smarter và Spectrum

**Smarter** là tên gọi chung các workload xoay xung quanh serving recommended data để hiển thị trên widget. Data infer từ dag là chưa đủ, mà còn những data về ad, thiên hướng biz như top selling cũng cần được serve chung trên các widget.

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2022.29.40.png?version=1&modificationDate=1757604585781&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 22.29.40.png")

  

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2021.50.39.png?version=1&modificationDate=1757602244022&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 21.50.39.png")Flow chính

## Vai trò mỗi serivce

### 1. smarter-api

**API**:

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2022.48.13.png?version=1&modificationDate=1757605700007&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 22.48.13.png")

**=> Nhiều api (một vài số trên đây vẫn chưa hiểu mục đích), nhưng do doc này chỉ tập trung vào các widget có sử dụng data từ dag inference, nên chỉ tập trung vào v1/blocks/products + v2/pdp**

[https://smarter.tiki.services/v1/blocks/products?block_code=home_top_deal_v3&trackity_id=125b50f7-d477-86f0-6739-c0cc166e13b0&customer_id=22434260](https://smarter.tiki.services/v1/blocks/products?block_code=home_top_deal_v3&trackity_id=125b50f7-d477-86f0-6739-c0cc166e13b0&customer_id=22434260)

  

**Trách nhiệm**:

- Tổng hợp đề xuất sản phẩm cho Trang Chi tiết Sản phẩm (PDP)
- Quản lý widget trang chủ
- ...

Kết hợp với các service khác để enrich/ ffilter data trước khi trả về.

## Dataflow chung của cả **v1/blocks/products + v2/pdp (12/9/2025)**

- **ML Model Results** (**Cassandra KV Store)**→  Retrieve inference data from model-serving.
- **Ad Services**: Advertisement content
- **Product Metadata for filtering** → Thanos, fallback về Pegasus 
    - **Availability filtering** (status 1 & 2)
    - **Store-specific filtering**
    - **Location-based filtering**
    - **Promotional badges**
    - **Advertisement integration**
    - **Batch processing** (20 items per batch)
- **Final Merge** → Weighted combination

### 2. smarter-exp

**Vai trò**: Quản lý thử nghiệm và cấu hình (Spectrum)

**Endpoint**: chỉ dùng cho internal

- **server_exp**: API thử nghiệm công khai (/v1/blocks, /v1/enrollment)
- **server_exp_ctl**: API quản trị (/v1/experiments, /v1/blocks/control)

**Trách nhiệm**:

- Quản lý thử nghiệm A/B
- Quản lý phiên bản (revisions) của các bảng inference trên scylla.
- Cấu hình widget sẽ chứa các block gì, mỗi block có data gì, lấy từ đâu, hiển thị vị trí nào trên widget.
- Quản lý bật tắt widget.

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2020.52.15.png?version=1&modificationDate=1757598743071&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 20.52.15.png")d

#### Serve data infer từ model

Ví dụ tên bảng:

- p_reco_home_widget_import_v2_v1_202507081100
- c_reco_cate_just_for_you_v1_202409020600
- top_reco_home_best_selling_v1_202409010600

**Thứ tự Ưu tiên**:

1. **Customer ID** (người dùng đăng nhập): Cá nhân hóa cao nhất
2. **Trackity ID** (người dùng ẩn danh): Nhắm mục tiêu hành vi
3. **Popular ("_")**: Dữ liệu dự phòng toàn cầu

Cấu hình trong bảng MySql (smarter-exp quản lý):

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2020.58.47.png?version=1&modificationDate=1757599133184&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 20.58.47.png")

#### Serve data theo rule (Quy tắc kinh doanh)

**Lưu trữ**: Nhiều dịch vụ ngoài + truy vấn cơ sở dữ liệu **Nguồn Dữ liệu**:

- **Elasticsearch**: Truy vấn tìm kiếm sản phẩm
- **BigQuery**: Quy tắc kinh doanh dựa trên phân tích
- **Druid**: Phân tích OLAP cho tab xu hướng PDP

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2020.56.23.png?version=1&modificationDate=1757598990564&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 20.56.23.png")

#### Serve data ad (Nội dung tài trợ)

Gọi tới service ad:

- **Ad Searcher**: [http://ad-searcher.dev.tiki.services](http://ad-searcher.dev.tiki.services)
- **Ad Widget**: [http://ad-widget.dev.tiki.services](http://ad-widget.dev.tiki.services) 

#### **Chiến lược chung**

- Vị trí cố định cho quảng cáo và widget
- Xen kẽ đề xuất ML và quy tắc

### 3. models-serving 

**Vai trò**: Phục vụ truy xuất data block model. Data đề xuất cho user/ product đã infer/tính toán hết sẵn, chỉ retrieve trực tiếp mà serve không phải là embed session data rồi chạy model infer để ra kết quả recommend gì như cái tên.

**Endpoint**: Chỉ gRPC nội bộ

**Trách nhiệm**:

- Truy xuất dữ liệu đề xuất từ Cassandra KV Store
- Cung cấp đề xuất:
    - Gợi ý cho toàn bộ user (lọc theo deal hot, bán chạy nhất trên cả sàn,...)
    - Danh mục hàng đầu cho mỗi người dùng
    - Sản phẩm hàng đầu cho mỗi người dùng
- Chuyển từ đề xuất cá nhân hóa sang phổ biến nhất nếu là non log-in user, user mới tạo tài khoản chưa engage gì.

# Chỉ số

[https://app.amplitude.com/analytics/tikivn/dashboard/i5300olz/edit/t1c30amd](https://app.amplitude.com/analytics/tikivn/dashboard/i5300olz/edit/t1c30amd)

Các số liệu sau được lấy từ **11/9/2024 - 11/9/2025**:

## HOME

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.07.43.png?version=1&modificationDate=1757567268173&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.07.43.png")![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.09.46.png?version=1&modificationDate=1757567392798&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.09.46.png")

  

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.11.49.png?version=1&modificationDate=1757567517553&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.11.49.png") ![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.14.09.png?version=1&modificationDate=1757567654005&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.14.09.png")

  

  

## PDP

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.02.08.png?version=1&modificationDate=1757566933409&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.02.08.png")![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2011.58.18.png?version=1&modificationDate=1757566704108&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 11.58.18.png")![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2011.59.21.png?version=1&modificationDate=1757566765572&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 11.59.21.png")![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2012.05.58.png?version=1&modificationDate=1757567163175&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 12.05.58.png")

  

# Hướng tối ưu

Hiện tại hệ thống recommendation có luồng infer chủ yếu dựa trên **warm data (historical data trên scylla sync trung bình mỗi 2 tiếng)** — tức là dữ liệu đã được xử lý, build sẵn từ lịch sử hành vi của người dùng. Tuy nhiên, luồng này có một số hạn chế vì không phản ứng nhanh được với các hành vi mới phát sinh (real-time user intent).

Ngoài ra dag sinh warm data này cũng có 1 vài vấn đề 

+ Không cập nhật liền được thay đổi các sản phẩm đang được business boost (top deal, flash sale,...), logic rerank/enrich/ filter ở tầng serving api cũng khó customize như search (thanos) vì một phần của infer dag đã lọc bỏ các sản phẩm tiềm năng với logic boosting của riêng nó.

+ Scan toàn bộ log của nhiều bảng trackity để lấy log 30, 90, 120 ngày, mỗi lần chạy lại dag daily_aggregation (chạy 3 lần 1 ngày).

### Các hướng cải thiện chính

#### 1. Cải thiện chất lượng mô hình

- Dùng **model hiện đại hơn**, có khả năng **mở rộng tập sản phẩm liên quan** cho một sản phẩm cụ thể (product-based recommendations), thay vì chỉ:
    
    - Dựa vào **similarity của product title**
        
    - Dựa vào các sản phẩm **cùng session / cùng lượt xem**
        
    - Dựa vào các sản phẩm **cùng category được user xem nhiều**
        
- Có thể huấn luyện **co-visitation graph**, **item2vec**, hoặc **sequential models (Transformer-based)** để nắm bắt ngữ cảnh khi người dùng duyệt qua nhiều sản phẩm liên tiếp.
    

#### 2. Tối ưu luồng infer dag hiện tại (warm data)

Viết lại các dag infer home và PDP, vẫn giữ nguyên logic, vẫn transform trên Big query và sync về Scylla, nhưng tốn ít tài nguyên chạy hơn.

#### 3. Cải thiện reranking các data đề xuất trang PDP - thêm real-time inference (hot data)

![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-09-11%20at%2023.29.40.png?version=1&modificationDate=1757608186002&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-09-11 at 23.29.40.png")

- Bổ sung thêm **luồng hot data** bên cạnh warm data:
    
    - Hot data là các **sự kiện click / view mới nhất**, cập nhật **liên tục** từ Kafka.
        
    - Có thể lưu tạm các click mới nhất của từng user (ví dụ trong Redis hoặc Apache Flink state) để **real-time infer** các sản phẩm tương tự / liên quan ngay sau khi user vừa click vào một PDP.
        
- Luồng inference lúc này sẽ bao gồm:
    
    - **Warm data** (lịch sử dài hạn đã build)
        
    - **Hot data** (ngữ cảnh ngắn hạn, session hiện tại)
        
- Data để display trang pdp sẽ bao gồm:
    - related product to product - hiện tại, từ dag train reco
    - related category to category - Mới (bởi vì không phải product nào cũng có related product precomputed, nên cần mở rộng)
- Điều này đặc biệt hữu ích cho:
    
    - **Reranking / Personalization theo ngữ cảnh**
        
        - Candidate pool vẫn đến từ precompute (related products, related categories), nhưng dùng hot data dùng để tập trung **rerank** theo session hiện tại.
            
        - Ví dụ: Candidate = giày dép nam (tổng quát), rerank bằng last 3 clicks = giày chạy bộ nam → top kết quả khớp ý định hơn.
            
        - Research thêm để integrate trong luồng search (gereral, category, store,...)
    - **Mitigate “cold start PDP” problem**
        
        - Sản phẩm mới có thể chưa có/ chưa đủ _related product_ được precompute.
            
        - Hot session clicks của user có thể bổ sung signal, tránh gợi ý “chung chung” khi hệ thống thiếu historical co-view.
            

  

# Planning

## Phase 1 - Tối ưu luồng dag infer hiện tại

- **Incremental log processing**
    
    - Purpose: compact user×product view events (recent window).
        
    - Core columns: `user_id, product_id, root_product_id, event_time, last_update.`
        
    - Deduplicate by `(user_id, root_product_id)` and keep latest event_time.
    - Schedule: incremental job every **15–30 min** (scan new logs only).
        
- **Product basic data enrichment**
    
    - Produce `unique_root_views` from recent `raw_view_pdp`.
        
    - Use `pid_mpid_mapping_mv` (MV from dim_product_full refresh every ~30 min) to resolve root product & primary_category.
        
    - _**Should include mpid and primary cate id in the trackity request directly to avoid resolving these 2 values!**_
- **Candidate early filter and ordering** 
    
    - Materialized view/ table named **`product_similarity`** with model output: `mpid -> ordered list of candidate root_products by biz score (get multiple fields tiki_hero, import, tiki_nơ, directly from dim_product_full)`
        
    - Refresh **15–30 min** so candidates are pre-ordered by business score.
        
    - Purpose: avoid heavy runtime sorting; return top-N candidates quickly.
        
    - _**This is still a heavy operation, the reranking should be moved to another layer with faster access to item features (and this item feature should be updated frequent too)**_
- **`reco_by_recency tables as (still in BQ)`**
    
    - Union of recent unique roots + high-quality similar products.
        
    - Minimal schema: `user_id, similar_products_to_recent_views (ordered array by num suggested), similar_categories_to_viewd_categories (ordered array by num of view), last_updated_at`.
        
    - Include items viewed in past **10–15 min** + **historical candidates**; truncate to top-K per user and save in new reco_by_recency_short.
        
    - Merge to reco_by_recency_long.
- **Sync to Scylla (serving)**
    
    - `Update reco_by_recency_short` per user in Scylla with TTL = 1 day (because client id should only be valid in a day)
        
- **Enrichment at API layer**
    
    - For API that is requesting recommended data **for one user**, fetch candidates from multiple tables in scylla (reco_by_recency_short, reco_by_recency_long), then:
        
        - Resolve best seller per root.
            
        - Enrich PDP metadata (images, price) before returning.
            
        - Use `similar_categories_to_view_categories to display ad reasonably.`
    - Favorite-cate products fetched from Thanos api **curl --location '0.0.0.0:6969/full/v2/products?category=46190&tiki_hero=1 → _double check if batch multiple Thanos request or have 1 new Thanos request to resolve multiple categories passed in better._** 
        
    - Global top-view should be fetched from tiki_nmv druid.
    - Home data: `reco_by_recency from dag inference` + favorite-cate products from ES + global top-view from druid real-time datasource.
- **Pdp data preparation**
    
    - New table category_similarity.
        
    - Have product_similarity (result table of reco-train dag), category_similarity data sync to Scylla to serve recommended data **per product.**
    
    - For API that is requesting recommended data **for one product**, fetch candidates from multiple tables in scylla (product_similarity, category_similarity, reco_by_recency_short, reco_by_recency_long, vw_related_product_v2 (if complentary widget)).
        
- **Real-time top selling/ trending product data** 
    - Should get from tiki_nmv directly, but double check the 404 error that happen quite often in trending pdp tab.
- **Incorporate with biz boosting products**
    - Best selling, ...
- **Operational notes** 
    
    - Monitor: latency of sync, Scylla TTL behavior, freshness of MVs.
        
    - Rollout: phased — seed small % users, verify metrics, then scale.
        

# Phụ lục - Shopee

## 1. Training

**Graph-based retrieval (LightSAGE)**: Shopee xây dựng item graphs từ user behaviors mạnh và collaborative filtering (CF), huấn luyện LightSAGE (GNN) để tạo item embeddings cho vector search/recall trong ads và item-to-item retrieval.

**Nguồn:** [https://arxiv.org/abs/2310.19394](https://arxiv.org/abs/2310.19394)

**Intgrate với milvus cho Vector/ANN recall**: Embeddings (image, video, item) được indexed trong vector database (Milvus) để thực hiện KNN/ANN recall với low latency, dùng nhiều cho multimedia và similar-item recall (video dedupe, video recall).

**Nguồn**: [https://zilliz.com/customers/shopee](https://zilliz.com/customers/shopee)

## 2. Pre-ranking / Two-tower / Fast rank

Sau khi recall trả về candidate set (hàng trăm đến hàng nghìn), Shopee dùng lightweight pre-ranking models (two-tower/prerank architecture) để giảm candidates trước full ranking. 

**Nguồn**: [https://www.adkdd.org/papers/ranktower%3A-a-synergistic-framework-for-enhancing-two-tower-pre-ranking-model/2024](https://www.adkdd.org/papers/ranktower%3A-a-synergistic-framework-for-enhancing-two-tower-pre-ranking-model/2024)

## 3. Final ranking (CTR/CR models + business rules)

Heavy ranking model (feature-rich, gradient boosted hoặc deep CTR models) chấm điểm top candidates, kết hợp business rules, caps, diversity logic.

**Nguồn**: [https://www.adkdd.org/papers/trigger-relevancy-and-diversity-inefficiency-with-dual-phase-synergistic-attention-in-shopee-recommendation-ads-system/2024](https://www.adkdd.org/papers/trigger-relevancy-and-diversity-inefficiency-with-dual-phase-synergistic-attention-in-shopee-recommendation-ads-system/2024)

  

# Phụ lục - Mô tả bảng đầu ra và đầu vào của từng dag hiện tại 

## 1. DAG **recommendation_data_aggregation_daily_v2**

**Đầu vào:**

|   |   |   |   |
|---|---|---|---|
|STT|Tên bảng|Mô tả|Nguồn|
|1|tiki-dwh.dwh.dim_product_full|Bảng tổng hợp thông tin đặc tính sản phẩm|DAG `dwh_etl_dim_product_full` <br><br>Bảng tổng hợp từ rất nhiều bảng BQ khác.|
|2|tiki-dwh.trackity_v2.core_events|Bảng log các action của user trên sàn||
|3|tiki-dwh.trackity_tiki.product_impressions|Bảng log impression của user với sản phẩm trên sàn|Topic [https://lenses.trackity.data.tiki.services/lenses/#/topics/trackity.tiki.v2.product_impressions](https://lenses.trackity.data.tiki.services/lenses/#/topics/trackity.tiki.v2.product_impressions) (không rõ transform)|
|4|tiki-dwh.cdp.datasources|Bảng log user, session view sản phẩm||
|5|tiki-dwh.olap.trackity_tiki_click|Bảng log user click gì||
|6|tiki-dwh.ecom.review|Bảng log rating sản phẩm của user||
|7|tiki-dwh.nmv.nmv|Bảng nmv hàng bán||
|8|tiki-analytics-dwh.personas.product_tier|Bảng segment giá sản phẩm (bảng chạy tay của commercials)||
|9|tiki-dwh.trackity.tiki_events_  <br>sessions_cross_devices|Bảng log action của user|Topic [https://lenses.trackity.data.tiki.services/lenses/#/topics/tiki_tracking.events](https://lenses.trackity.data.tiki.services/lenses/#/topics/tiki_tracking.events)|

  

**Đầu ra:**

|STT|Tên bảng|Mô tả|
|---|---|---|
|1|tiki-dwh.recommendation.root_product_YYYYMMDD|Lưu thông tin sản phẩm cơ bản (tên, mã, danh mục, seller_id) và tạo trường root_product_id để mapping sản phẩm.|
|2|tiki-dwh.recommendation.map_client_customer_YYYYMMDD|Mapping client_id với customer_id từ log trong 365 ngày, gán customer_id cho các log không có thông tin đăng nhập.|
|3|tiki-analytics-dwh.personas.event_staging|Lưu log 'view_pdp', 'add_to_cart', 'complete_purchase' từ core_events và 'true_impression' từ product_impressions, dùng cho luồng tính danh mục yêu thích.|
|4|tiki-dwh.recommendation.sale_orders_YYYYMMDD|Lưu thông tin sản phẩm được xem trong cùng phiên (session), trường product_id thực chất là root_product_id.|
|5|tiki-dwh.recommendation.pdp_view_raw_YYYYMMDD|Lưu log view PDP từ trackity_tiki_click và core_events (90 ngày), map với root_product và map_client_customer để bổ sung thông tin.|
|6|tiki-dwh.recommendation.product_features_pdp_view_YYYYMMDD|Tổng hợp dữ liệu view PDP theo level root/product_id (1, 7, 30 ngày), dùng cho bảng product_features.|
|7|tiki-dwh.recommendation.product_features_rating_YYYYMMDD|Tổng hợp thông tin đánh giá sản phẩm (rating trung bình, số rating, comment) theo level root/product_id.|
|8|tiki-dwh.recommendation.product_impression_raw_YYYYMMDD|Lưu thông tin impression và click từ product_impressions (30 ngày), map với root_product và map_client_customer.|
|9|tiki-dwh.recommendation.product_impression_ctr_YYYYMMDD|Tính CTR của sản phẩm từ impression và click (1, 30 ngày), dùng cho bảng product_features.|
|10|tiki-dwh.recommendation.product_sold_raw_YYYYMMDD|Lưu thông tin đơn hàng từ bảng nmv (365 ngày), tính shipping day, dùng cho product_features và customer_features.|
|11|tiki-dwh.recommendation.product_features_sold_YYYYMMDD|Tổng hợp dữ liệu bán hàng (số lượng, khách, đơn, giá, thời gian ship) theo level root/product_id (30, 90 ngày).|
|12|tiki-dwh.recommendation.product_features_v2_YYYYMMDD|Tổng hợp tất cả đặc tính sản phẩm từ các bảng root_product, product_features_pdp_view, product_features_rating, product_features_sold, product_impression_ctr và product_tier.|
|13|tiki-dwh.recommendation.cate_features_v2_YYYYMMDD|Tổng hợp đặc tính danh mục (cate 1-6) từ product_features_v2, bao gồm số sản phẩm, view, rating, bán hàng, v.v.|
|14|tiki-dwh.recommendation.cate_pair_YYYYMMDD|Tính quan hệ giữa các primary cate dựa trên session_id từ pdp_view_raw, dùng cho luồng infinity ở PDP.|
|15|tiki-dwh.recommendation.customer_cart_estimated_YYYYMMDD|Ước lượng sản phẩm còn trong giỏ hàng của user dựa trên log add, remove, purchase (365 ngày).|
|16|tiki-dwh.recommendation.customer_purchased_YYYYMMDD|Lưu thông tin sản phẩm user đã mua (365 ngày), dùng để lọc sản phẩm đã mua trong luồng personalize.|
|17|tiki-dwh.recommendation.customer_pdp_view_YYYYMMDD|Tổng hợp lượt view PDP của user theo customer_id, product_id, root_product_id (7, 30, 90 ngày).|
|18|tiki-dwh.recommendation.customer_cate_pdp_view_YYYYMMDD|Tổng hợp lượt view PDP theo primary cate của user (7, 30, 90 ngày), chưa sử dụng.|
|19|tiki-dwh.recommendation.client_pdp_view_YYYYMMDD|Tổng hợp lượt view PDP theo client_id cho user không đăng nhập (7, 30, 90 ngày), hiện chưa sử dụng.|
|20|tiki-dwh.recommendation.client_cate_pdp_view_YYYYMMDD|Tổng hợp lượt view PDP theo primary cate của client_id (7, 30, 90 ngày), chưa sử dụng.|
|21|tiki-dwh.recommendation.customer_impression_ctr_YYYYMMDD|Tổng hợp impression và click của user theo customer_id (30 ngày), chưa sử dụng.|
|22|tiki-dwh.recommendation.customer_cate_impression_ctr_YYYYMMDD|Tổng hợp impression và click theo primary cate của customer_id (30 ngày), chưa sử dụng.|
|23|tiki-dwh.recommendation.client_impression_ctr_YYYYMMDD|Tổng hợp impression và click theo client_id (30 ngày), chưa sử dụng.|
|24|tiki-dwh.recommendation.client_cate_impression_ctr_YYYYMMDD|Tổng hợp impression và click theo primary cate của client_id (30 ngày), chưa sử dụng.|

  

## 2. DAG **recommendation_train_title_v2 / recommendation_train_interaction_v2 / recommendation_train_user_favorite_cate_v2**

|**DAG**|**Bảng đầu vào**|**Bảng đầu ra**|**Mô tả đầu ra**|
|---|---|---|---|
|**recommendation_train_title_v2**|`tiki-dwh.recommendation.product_features_v2_YYYYMMDD`|`tiki-dwh.recommendation.prediction_title_similar_YYYYMMDD`|Bảng dự đoán quan hệ sản phẩm–sản phẩm theo **tên** (level `product_id`). Được gộp vào bảng quan hệ sản phẩm–sản phẩm để triển khai.|
|**recommendation_train_interaction_v2**|1. `tiki-dwh.recommendation.root_product_YYYYMMDD`  <br>2. `tiki-dwh.recommendation.sale_orders_YYYYMMDD`  <br>3. `tiki-dwh.recommendation.product_features_v2_YYYYMMDD`  <br>4. `tiki-dwh.recommendation.prediction_title_similar_YYYYMMDD`|1. `tiki-dwh.recommendation.input_data_train_interaction_YYYYMMDD`<br><br>2. `tiki-dwh.recommendation.prediction_interaction_similar_YYYYMMDD`<br><br>3. `tiki-dwh.recommendation.product_similar_prediction_v2_YYYYMMDD`|1. Input cho model `train_interaction`: đếm số cate và sản phẩm trong session (1–6 cate, 2–15 sản phẩm).<br><br>2. Quan hệ sản phẩm–sản phẩm dự đoán theo session view (level `root_product_id`)<br><br>. 3. Bảng tổng hợp quan hệ sản phẩm–sản phẩm (`score_interaction`, `score_title`, `max_score_model`, `score_cate`) phục vụ các widget reco.|
|**recommendation_train_user_favorite_cate_v2**|1. `tiki-dwh.dwh.dim_product_full`  <br>2. `tiki-dwh.nmv.nmv`  <br>3. `tiki-analytics-dwh.personas.event_staging`|1. `tiki-analytics-dwh.personas.purchase_feature`<br><br>2. `tiki-analytics-dwh.personas.view_feature`<br><br>3. `tiki-analytics-dwh.personas.purchase_label`<br><br>4. `tiki-analytics-dwh.personas.view_label`<br><br>5. `tiki-analytics-dwh.personas.data_train`<br><br>6. `tiki-analytics-dwh.personas.propensity_cate_YYYYMMDD`<br><br>7. `tiki-analytics-dwh.personas.combine_propensity`|1. Feature: số lần mua theo `primary_cate_id` (30, 90 ngày).<br><br>2. Feature: tỷ lệ CTR theo `primary_cate_id` (30, 90 ngày).<br><br>3. Label: User mua hàng theo `primary_cate_id` trong tháng N-1.<br><br>4. Label: User có hành vi add-to-cart, view PDP, impression theo `primary_cate_id` trong tháng N-1.<br><br>5. Feature + Label tổng hợp cho model train.<br><br>6. Điểm **propensity** (khả năng mua) theo `primary_cate_id` từ model classification. 7. Tổng hợp điểm propensity (6 tháng, trọng số theo tháng) → xác định cate yêu thích để **personalize Home**.|

  

## 3. DAG **recommendation_infer_data_agg**

**Đầu vào:**

|   |   |   |   |
|---|---|---|---|
|**Stt**|**Tên bảng**|**Nguồn**|**Mô tả**|
|1|tiki-dwh.dwh.dim_product_full|N/A|Bảng tổng hợp thông tin đặc tính sản phẩm|
|2|tiki-dwh.recommendation.product_features_v2_YYYYMMDD|recommendation_data_aggregation_daily_v2|Bảng features sản phẩm tổng hợp từ luồng data ở trên|

  

**Đầu ra:**

|STT|Tên bảng|Mô tả|
|---|---|---|
|1|tiki-dwh.recommendation.product_weight_YYYYMMDD|Định nghĩa trọng số boosting (tikinow, next day, brand, v.v.) để join với product_features.|
|2|tiki-dwh.recommendation.product_features_v2_with_weights_YYYYMMDD|Tổng hợp product_features_v2, dim_product_full, product_weight, cập nhật trường và thêm trọng số.|
|3|tiki-dwh.recommendation.top_view_d7_product_weight_YYYYMMDD|Lọc sản phẩm có view trong 7 ngày, tính trọng số view/tổng view cate, dùng infer widget home.|

## 4. DAG **recommendation_infer_area_home_p1_v2 / p2_v2**

|   |   |   |   |   |
|---|---|---|---|---|
|Stt|Tên bảng|Nguồn|Mô tả|Schema|
|1|`tiki-dwh.recommendation.top_view`<br><br>`_d7_product_weight_YYYYMMDD`|recommendation_infer_data_agg|Bảng sản phẩm có view 7 ngày gần nhất và trọng số trong primary cate tương ứng đã chạy ở trên|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-08-06%20at%2023.04.28.png?version=1&modificationDate=1754496275037&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-08-06 at 23.04.28.png")|
|2|`tiki-dwh.recommendation.customer`<br><br>`_pdp``_view_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng danh sách sản phẩm user đã view (lấy 7 ngày gần nhất = 7 bảng gần nhất)|![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-07-04%20at%2000.34.37.png?version=1&modificationDate=1751606491490&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.34.37.png")|
|3|`tiki-dwh.recommendation.client`<br><br>`_pdp_view_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng danh sách sản phẩm client_id (không bắt được customer_id) đã view (lấy bảng mới nhất trong 3 ngày gần nhất)<br><br>Hiện phần này code sẵn nhưng chưa chạy, vẫn note ở đây|![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-07-04%20at%2000.39.42.png?version=1&modificationDate=1751606491510&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.39.42.png")|
|4|`tiki-analytics-dwh.personas.combine_propensity`|recommendation_train_user_favorite_cate_v2|Bảng tính toán primary cate yêu thích của từng user|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-07-04%20at%2000.45.09.png?version=1&modificationDate=1751606491523&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.45.09.png")|
|5|`tiki-dwh.recommendation.product`<br><br>`_features``_v2_with_weights`<br><br>`_YYYYMMDD`|recommendation_infer_data_agg|Bảng tổng hợp product features.|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-07-04%20at%2000.40.53.png?version=1&modificationDate=1751606491536&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.40.53.png")|
|6|`tiki-dwh.recommendation.product`<br><br>`_similar``_prediction_v2_YYYYMMDD`|recommendation_train_interaction_v2|Bảng quan hệ sản phẩm - sản phẩm (level root) sau khi gộp 2 luồng title và interaction (lấy bảng mới nhất trong 3 ngày gần nhất)|![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-07-04%20at%2000.43.56.png?version=1&modificationDate=1751606491550&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.43.56.png")|
|7|`tiki-dwh.recommendation.customer`<br><br>`_cart``_estimated_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng ước lượng danh sách sản phẩm trong cart của user (lấy bảng mới nhất trong 3 ngày gần nhất)<br><br>Hiện phần này đã code sẵn trong luồng nhưng chưa chạy, vẫn note ở đây|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-07-04%20at%2000.48.09.png?version=1&modificationDate=1751606491559&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.48.09.png")|
|8|`tiki-dwh.recommendation.customer`<br><br>`_purchased``_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng danh sách hàng đã mua của user (lấy bảng mới nhất trong 3 ngày gần nhất)|![](https://docs.tiki.com.vn/download/attachments/257989905/Screenshot%202025-07-04%20at%2000.42.58.png?version=1&modificationDate=1751606491568&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.42.58.png")|
|9|`tiki-dwh.recommendation.map`<br><br>`_client_``customer_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng map client_id với customer_id (lấy bảng mới nhất trong 3 ngày gần nhất)|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-07-04%20at%2000.47.14.png?version=1&modificationDate=1751606491579&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-07-04 at 00.47.14.png")|

  

  

## 5. DAG **recommendation_infer_area_pdp_v2**

|   |   |   |   |   |
|---|---|---|---|---|
|Stt|Tên bảng|Nguồn|Mô tả|Schema|
|1|`tiki-dwh.recommendation.product_features_v2_`<br><br>`with_weights_YYYYMMDD`|recommendation_infer_data_agg|||
|2|`tiki-dwh.smarter.vw_related_product_v2`|a view from tiki-analytics-dwh.personas.related_product_v2|Bảng danh mục cate bổ sung cho nhau. Chạy tay, có thể đã outdate|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-08-29%20at%2012.17.19.png?version=1&modificationDate=1756444646155&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-08-29 at 12.17.19.png")|
|3|`tiki-dwh.recommendation.product_similar_`<br><br>`prediction_v2_YYYYMMDD`|recommendation_train_interaction_v2|||
|4|`tiki-dwh.recommendation.cate_pair_YYYYMMDD`|recommendation_data_aggregation_daily_v2|Bảng quan hệ primary cate - primary cate đã chạy ở trên|![](https://docs.tiki.com.vn/download/thumbnails/257989905/Screenshot%202025-08-29%20at%2012.19.01.png?version=1&modificationDate=1756444745223&api=v2 "Seller Center > [Recommendation] Tổng hợp một số vấn đề/ hướng tối ưu > Screenshot 2025-08-29 at 12.19.01.png")|