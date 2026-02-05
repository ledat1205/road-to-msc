## Flink for Recommendation
Flink excels here because recommendations in e-commerce must react instantly to user behavior (clicks, views, adds-to-cart, purchases) while handling massive scale (millions of events/sec during peaks, as seen at Alibaba, Xiaomi, Zalando, and similar platforms).

### 1. Real-Time Session-Based Recommendations (Short-Term User Interest)
**Use Case**: Capture a user's current browsing session (e.g., last 5–30 minutes) to recommend "more like what you're viewing now" — ideal for fashion, electronics, or flash sales where intent changes quickly.

**How Flink Fits**:
- Key by `userId`.
- Use `KeyedProcessFunction` or `RichFlatMapFunction` + `ValueState` / `ListState` to maintain recent items in session windows.
- Apply simple heuristics (e.g., co-views in session) or lightweight similarity (cosine on item embeddings).
- Output fresh recs every few seconds/minutes to frontend via Kafka sink.

**Example Pattern** (PyFlink or Java — pseudocode snippet):
```python
# PyFlink example (simplified)
class SessionRecommender(KeyedProcessFunction):
    def open(self, runtime_context):
        self.recent_items = runtime_context.get_state(ListStateDescriptor("recent", Types.STRING()))

    def process_element(self, event, ctx):
        user_id = event['userId']
        product_id = event['productId']
        
        # Append to session state (with TTL for auto-expiry)
        self.recent_items.add(product_id)
        
        # Get top similar (e.g., pre-loaded item sim matrix or simple co-occurrence)
        similar = get_similar_products(product_id, self.recent_items.get())  # custom logic
        
        # Emit recommendation event
        yield {'userId': user_id, 'recommendations': similar[:10], 'ts': event['ts']}
```

**Production Twist**: Combine with TTL state (Flink 1.12+) to auto-clean old sessions.

### 2. Real-Time Personalized Recommendations via Model Inference (Collaborative Filtering / Content-Based)
**Use Case**: Score items for each user using a pre-trained model (e.g., ALS, LightFM, two-tower neural net) updated incrementally or queried remotely.

**How Flink Fits**:
- Remote inference pattern: Flink enriches user context → calls model server (e.g., TensorFlow Serving, TorchServe, or OpenAI for semantic) → ranks items.
- Or incremental updates: Use Flink state to maintain user vectors and periodically sync with offline model.
- Alibaba & Xiaomi use similar patterns at trillion-event scale for search + recs.

**Example Pattern** (Java — remote inference with HTTP async):
```java
// Async I/O for low-latency model calls
DataStream<UserEvent> events = ...;

AsyncDataStream.unorderedWait(
    events.keyBy(UserEvent::getUserId),
    new AsyncHttpModelCaller("http://model-service/predict"),  // returns top items
    500, TimeUnit.MILLISECONDS,
    10000  // capacity
).addSink(new KafkaSink<>("recs-topic"));
```

**Advanced**: Flink 2.2+ has `ML_PREDICT` and `VECTOR_SEARCH` for in-stream LLM inference or vector similarity — great for semantic item matching (e.g., "similar to this dress but cheaper").

### 3. Real-Time Trending / Viral Item Detection + Boosting
**Use Case**: Boost "hot right now" items in recs (e.g., trending sneakers during a sale) while personalizing.

**How Flink Fits**:
- Global aggregation over sliding/tumbling windows (e.g., views/adds-to-cart last 10 min).
- Use `KeyedProcessFunction` + `MapState` for approximate top-K (Count-Min Sketch or Lossy Counting).
- Join with per-user stream for hybrid recs (personal + trending boost).

**Example Pattern**:
- One job computes global trending (broadcast state).
- Another job joins per-user stream with trending via `BroadcastConnectedStream`.

### 4. Cart Abandonment & Next-Best-Offer Recommendations
**Use Case**: If user adds items but doesn't purchase in X minutes → trigger personalized "complete your purchase" offers or reminders.

**How Flink Fits**:
- CEP (Complex Event Processing): Pattern like `(addToCart → !purchase within 15 min)`.
- Or use timers in `KeyedProcessFunction`:
  - On `addToCart`: register timer.
  - On timer fire (no purchase): compute recs based on cart contents.

**Example**:
```java
public class AbandonmentDetector extends KeyedProcessFunction<String, Event, Recommendation> {
    @Override
    public void processElement(Event event, Context ctx, Collector<Recommendation> out) {
        if (event.type.equals("ADD_TO_CART")) {
            ctx.timerService().registerEventTimeTimer(ctx.timestamp() + 15 * 60 * 1000);
            // Store cart state
        } else if (event.type.equals("PURCHASE")) {
            ctx.timerService().deleteEventTimeTimer(...);
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx, Collector<Recommendation> out) {
        // Cart abandonment detected → generate recs (e.g., similar + discount)
        out.collect(buildAbandonmentRec(ctx.getCurrentKey()));
    }
}
```

### 5. Real-Time Customer 360 for Unified Recommendations
**Use Case**: Merge multi-channel behavior (app, web, in-store via beacons, social signals) into one view for consistent recs across touchpoints.

**How Flink Fits**:
- Event-time sessionization + enrichment from multiple Kafka topics.
- Use `IntervalJoin` or `CoProcessFunction` to build enriched user profiles in real time.
- Sink to feature store (Redis, DynamoDB) for serving layer or direct to recs engine.

### Quick Comparison Table

| Use Case                        | Key Flink Feature          | Latency Target | Personalization Level | Example Companies/Tools |
|---------------------------------|----------------------------|----------------|-----------------------|-------------------------|
| Session-based recs              | Keyed state + windows      | < 1 sec        | Short-term            | Zalando, general e-comm |
| Model inference scoring         | Async I/O or ML_PREDICT    | 50–500 ms      | High (collaborative)  | Alibaba, Xiaomi         |
| Trending / viral boosting       | Global aggregation + broadcast | < 5 sec     | Hybrid                | Flash sales platforms   |
| Cart abandonment triggers       | CEP or timers              | Minutes        | Medium                | Retail/e-commerce       |
| Multi-channel Customer 360      | Joins + enrichment         | Seconds        | Full                  | Unified commerce setups |

## Apache Flink CEP

Here’s a practical **Apache Flink CEP (Complex Event Processing)** example tailored to an **e-commerce recommendation system**: **High-Value Cart Abandonment Detection with Timeout**.

This is one of the most common and business-valuable CEP patterns in e-commerce:
- User adds high-value## items to cart (e.g., total > $200).
- No purchase occurs within a timeout (e.g., 15–30 minutes).
- Trigger: Send personalized recommendation/reminder (e.g., "Complete your purchase – here are similar items / 10% off code").

Flink CEP excels here because it declaratively matches temporal patterns like "add-to-cart sequence → no purchase within window".

### Prerequisites
- Flink 1.15+ (CEP library included in `flink-streaming-java`)
- Kafka input topic with JSON events like:
  ```json
  {"userId": "u123", "eventType": "ADD_TO_CART", "productId": "p456", "price": 150.0, "quantity": 1, "timestamp": 1738760000000}
  {"userId": "u123", "eventType": "PURCHASE", "cartTotal": 450.0, "timestamp": ...}
  ```
- Add dependencies (Maven):
  ```xml
  <dependency>
      <groupId>org.apache.flink</groupId>
      <artifactId>flink-cep</artifactId>
      <version>1.18.0</version> <!-- match your Flink version -->
  </dependency>
  ```

### Full Flink CEP Job (Java) – CartAbandonmentWithRecommendations.java

```java
import org.apache.flink.api.common.eventtime.WatermarkStrategy;
import org.apache.flink.api.common.serialization.SimpleStringSchema;
import org.apache.flink.connector.kafka.source.KafkaSource;
import org.apache.flink.connector.kafka.source.enumerator.initializer.OffsetsInitializer;
import org.apache.flink.cep.CEP;
import org.apache.flink.cep.PatternSelectFunction;
import org.apache.flink.cep.PatternStream;
import org.apache.flink.cep.pattern.Pattern;
import org.apache.flink.cep.pattern.conditions.SimpleCondition;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.time.Duration;
import java.util.List;
import java.util.Map;

public class CartAbandonmentWithRecommendations {

    public static class UserEvent {
        public String userId;
        public String eventType;
        public String productId;
        public double price;
        public int quantity;
        public long timestamp;

        // Constructor, getters/setters omitted for brevity
        public static UserEvent fromJson(String json) {
            // Parse logic here (use ObjectMapper)
            ObjectMapper mapper = new ObjectMapper();
            try {
                JsonNode node = mapper.readTree(json);
                UserEvent e = new UserEvent();
                e.userId = node.get("userId").asText();
                e.eventType = node.get("eventType").asText();
                e.productId = node.path("productId").asText(null);
                e.price = node.path("price").asDouble(0.0);
                e.quantity = node.path("quantity").asInt(1);
                e.timestamp = node.get("timestamp").asLong();
                return e;
            } catch (Exception ex) {
                return null;
            }
        }
    }

    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.getConfig().setAutoWatermarkInterval(200);

        // Kafka source (events topic)
        KafkaSource<String> source = KafkaSource.<String>builder()
                .setBootstrapServers("localhost:9092")
                .setTopics("user-events")
                .setGroupId("flink-cep-abandonment")
                .setStartingOffsets(OffsetsInitializer.earliest())
                .setValueOnlyDeserializer(new SimpleStringSchema())
                .build();

        DataStream<UserEvent> events = env.fromSource(source, WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(30)), "Kafka Events")
                .map(UserEvent::fromJson)
                .filter(e -> e != null);

        // Key by userId (important for per-user patterns)
        DataStream<UserEvent> keyedEvents = events.keyBy(e -> e.userId);

        // CEP Pattern: Sequence of ADD_TO_CART (high value) followed by no PURCHASE within 15 min
        Pattern<UserEvent, ?> abandonmentPattern = Pattern
                .<UserEvent>begin("addHighValue")
                    .where(new SimpleCondition<UserEvent>() {
                        @Override
                        public boolean filter(UserEvent event) {
                            return "ADD_TO_CART".equals(event.eventType) &&
                                   (event.price * event.quantity) >= 100.0; // threshold per item or adjust to cart total
                        }
                    })
                .followedBy("moreAdds") // optional: more adds
                    .where(new SimpleCondition<UserEvent>() {
                        @Override
                        public boolean filter(UserEvent event) {
                            return "ADD_TO_CART".equals(event.eventType);
                        }
                    }).oneOrMore().optional()
                .notFollowedBy("purchase") // no purchase after
                    .where(new SimpleCondition<UserEvent>() {
                        @Override
                        public boolean filter(UserEvent event) {
                            return "PURCHASE".equals(event.eventType);
                        }
                    })
                .within(Time.minutes(15)); // timeout window

        PatternStream<UserEvent> patternStream = CEP.pattern(keyedEvents, abandonmentPattern);

        // When pattern matches (timeout triggers), select and generate recommendation
        DataStream<String> recommendations = patternStream.select(new PatternSelectFunction<UserEvent, String>() {
            @Override
            public void select(Map<String, List<UserEvent>> pattern, Collector<String> out) {
                List<UserEvent> adds = pattern.get("addHighValue");
                // In real system: combine with moreAdds, look up user history, call rec model
                String userId = adds.get(0).userId;
                String abandonedProducts = adds.stream()
                        .map(e -> e.productId)
                        .reduce((a, b) -> a + ", " + b).orElse("unknown");

                String recMessage = String.format(
                    "{\"userId\":\"%s\", \"type\":\"CART_ABANDONMENT\", \"abandoned\":\"%s\", \"recommendations\":[\"similar-p1\",\"discount-10%%\"], \"timestamp\":%d}",
                    userId, abandonedProducts, System.currentTimeMillis()
                );

                out.collect(recMessage);
            }
        });

        // Sink: send to Kafka "recommendations" topic or print
        recommendations.print(); // For demo → in prod: add KafkaSink

        env.execute("E-Commerce Cart Abandonment CEP → Recommendations");
    }
}
```

### How It Works
- **Pattern explained**:
  - Starts with at least one high-value `ADD_TO_CART`.
  - Allows zero or more additional adds (`.oneOrMore().optional()`).
  - Requires **no** `PURCHASE` event within 15 minutes (`.notFollowedBy(...) .within(...)` → timeout triggers match).
- **Event time** → use proper `WatermarkStrategy` + timestamps from events (add `TimestampAssigner` if needed).
- **Output** → JSON ready for downstream (e.g., push notification service, email, or in-app rec banner).

### Production Enhancements
- Use `.followedByAny()` or stricter sequences if order matters.
- Enrich with cart total: maintain partial state with `KeyedProcessFunction` + CEP.
- Add strict contiguity (`.next()` vs `.followedBy()`) or relaxed (`.followedByAny()`).
- Persist abandoned cart state → query for cross-sell/upsell recs.
- Scale: Flink handles millions of users with keyed state + RocksDB backend.

This pattern is production-proven (inspired by Ververica/Booking.com-style demos and Alibaba/DoorDash real-time abandonment flows).

