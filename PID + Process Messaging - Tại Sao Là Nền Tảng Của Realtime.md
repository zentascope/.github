# PID + Process Messaging - Tại Sao Là Nền Tảng Của Realtime?

## Trước Tiên - Mental Model Đúng

```
Hầu hết lập trình viên nghĩ realtime system như sau:

  Client -> HTTP Request -> Server xử lý -> Response -> Client

Erlang nghĩ khác hoàn toàn:

  Client = 1 process
  Connection = 1 process  
  User session = 1 process
  Room/Channel = 1 process
  
  Chúng NÓI CHUYỆN với nhau qua message
  như actor trong thực tế
```

---

## 1. PID Là Gì Trong Thực Tế

```elixir
# Khi 1 WebSocket connection đến:
# Phoenix tạo ra 1 process, cấp cho nó 1 PID

# PID trông như thế này:
#<0.234.0>   <- node 0, process 234, creation 0

# Quan trọng hơn: PID là địa chỉ UNIQUE của process
# trên toàn bộ cluster, kể cả nhiều node

pid = spawn(fn -> 
  receive do
    {:message, content} -> IO.puts(content)
  end
end)
# pid = #PID<0.234.0>

# Gửi message đến đúng process đó:
send(pid, {:message, "hello"})
# Process #PID<0.234.0> nhận được, xử lý, không ai khác nhận
```

```
Tưởng tượng thực tế:

  Bệnh viện có 10,000 bệnh nhân đang theo dõi realtime
  
  Erlang:
    Mỗi bệnh nhân = 1 process với PID riêng
    Doctor gửi lệnh: send(patient_pid, {:change_medication, new_dose})
    -> Chỉ đúng process của bệnh nhân đó nhận
    -> Xử lý ngay lập tức
    -> Không ảnh hưởng 9,999 bệnh nhân còn lại

  Go:
    Không có "địa chỉ" cho từng bệnh nhân
    Phải dùng shared map + mutex, hoặc Redis
    -> Lookup -> Lock -> Update -> Unlock -> Notify
```

---

## 2. Tại Sao PID + Send Tạo Ra Realtime Thực Sự

### Scenario: 50,000 users đang xem live auction, giá thay đổi mỗi 100ms

```elixir
# ERLANG - Luồng dữ liệu trực tiếp

# Auction process - 1 process quản lý phiên đấu giá
defmodule AuctionRoom do
  use GenServer

  def handle_cast({:new_bid, amount, bidder_pid}, state) do
    new_state = %{state | current_price: amount, leader: bidder_pid}
    
    # Notify tất cả 50,000 watchers
    # Mỗi watcher là 1 PID trong state.watchers
    Enum.each(state.watchers, fn watcher_pid ->
      send(watcher_pid, {:price_update, amount})
      # Đây là in-memory message, không có network hop
      # Không có serialization
      # Không có broker ở giữa
    end)
    
    {:noreply, new_state}
  end
end

# Mỗi WebSocket connection là 1 process
defmodule WatcherProcess do
  use GenServer
  
  def handle_info({:price_update, amount}, state) do
    # Nhận message -> push ngay xuống WebSocket
    # Latency: microseconds
    Phoenix.Channel.push(state.socket, "price_update", %{price: amount})
    {:noreply, state}
  end
end
```

```
Timeline Erlang:
  Bid arrives -> AuctionRoom process -> send() to 50K PIDs
  -> Each WatcherProcess receives -> push to WebSocket
  
  Total time: 1-5ms cho 50,000 notifications
  Không có Redis, không có Kafka, không có network hop
```

```go
// GO - Cùng bài toán

type AuctionHub struct {
    clients    map[string]*Client  // userID -> client
    mu         sync.RWMutex        // phải lock khi access map
    broadcast  chan BidUpdate
}

func (h *AuctionHub) handleBroadcast() {
    for update := range h.broadcast {
        h.mu.RLock()
        for _, client := range h.clients {
            select {
            case client.send <- update:  // non-blocking send
            default:
                // Channel full -> drop message hoặc disconnect client
                // Đây là vấn đề thực tế trong Go WebSocket
                close(client.send)
                delete(h.clients, client.id)
            }
        }
        h.mu.RUnlock()
    }
}

// Vấn đề 1: RLock trên 50,000 clients = bottleneck
// Vấn đề 2: "default: drop" -> mất message
// Vấn đề 3: Hub này chỉ biết clients trên CÙNG POD
//           -> clients trên pod khác không nhận được
//           -> PHẢI có Redis Pub/Sub ở giữa
```

---

## 3. Stateful WebSocket - Điều Go + K8s Không Thể Đảm Bảo Native

### Vấn đề 1: Process Identity Across Pods

```
ERLANG:
  PID là địa chỉ global trong cluster
  
  Node1: send(pid_on_node2, message)
  -> Message đến đúng process trên Node2
  -> Không cần biết process đang ở node nào
  -> Transparent hoàn toàn

  #PID<2.234.0>  <- "2" nghĩa là node index 2
  Gửi từ node 1 đến node 2: tự động qua Erlang distribution


GO + K8s:
  Goroutine không có "địa chỉ" global
  Pod A không biết goroutine nào đang chạy trên Pod B
  
  Muốn gửi message đến specific user trên Pod B:
    1. Lookup Redis: user X đang ở Pod B
    2. HTTP call đến Pod B internal endpoint
    3. Pod B tìm goroutine của user X trong local map
    4. Gửi xuống WebSocket
  
  Mỗi targeted message = 2 network hops + Redis lookup
```

### Vấn đề 2: State Sống Ở Đâu

```
ERLANG - State sống TRONG process:

defmodule UserSession do
  use GenServer
  
  # State của user: lịch sử, context, preferences
  # Sống in-memory, zero latency access
  # Tự động GC khi process chết
  # Tự động persist nếu cần (ETS, Mnesia)
  
  def handle_info({:message, msg}, state) do
    updated = %{state |
      message_history: [msg | state.message_history],
      last_seen: DateTime.utc_now(),
      unread_count: state.unread_count + 1
    }
    {:noreply, updated}
  end
end

# Không cần Redis để giữ session state
# Không cần serialize/deserialize
# Access state: đơn giản là pattern match


GO + K8s - State PHẢI externalize:

type UserSession struct {
    UserID       string
    History      []Message   // không thể giữ trong goroutine
    LastSeen     time.Time   // vì pod có thể restart bất cứ lúc nào
    UnreadCount  int         // K8s không đảm bảo pod sống mãi
}

// Mỗi operation phải:
func (s *Server) handleMessage(userID string, msg Message) {
    // 1. Load state từ Redis
    session, err := s.redis.Get(ctx, "session:"+userID)
    
    // 2. Deserialize JSON
    var userSession UserSession
    json.Unmarshal([]byte(session), &userSession)
    
    // 3. Update
    userSession.History = append(userSession.History, msg)
    
    // 4. Serialize + Save lại Redis
    data, _ := json.Marshal(userSession)
    s.redis.Set(ctx, "session:"+userID, data, ttl)
    
    // 5. Mới xử lý được business logic
    // Mỗi message = 2 Redis ops = 1-3ms overhead
}
```

### Vấn đề 3: Pod Restart = Connection Drop

```
GO + K8s:
  
  Pod đang có 25,000 connections
      │
      ├── K8s quyết định restart pod (OOM, crash, deploy)
      │
      ▼
  25,000 WebSocket connections bị CUT đột ngột
  
  Clients phải:
    1. Detect disconnect (1-30 giây tùy timeout)
    2. Reconnect
    3. Re-authenticate
    4. Re-fetch state từ Redis
    5. Re-subscribe channels
  
  Trong y tế: 30 giây mù dữ liệu bệnh nhân
  Trong thanh toán: transaction state không rõ ràng


ERLANG:
  
  Process crash (không phải node crash):
    -> Supervisor restart process trong microseconds
    -> WebSocket connection giữ nguyên (TCP vẫn sống)
    -> Client không biết gì đã xảy ra
    -> State được khôi phục từ init callback
  
  Node crash (toàn bộ server down):
    -> libcluster phát hiện trong vài giây
    -> Clients reconnect -> load balancer -> node còn lại
    -> PubSub tự redistribute
    -> Chỉ mất connections đang trên node đó
    -> Các node khác không bị ảnh hưởng
```

### Vấn đề 4: Ordering và Consistency

```
ERLANG - Mailbox đảm bảo ordering:

  Process A gửi 3 messages đến Process B:
    send(pid_b, {:event, 1})
    send(pid_b, {:event, 2})
    send(pid_b, {:event, 3})
  
  Process B nhận ĐÚNG THỨ TỰ: 1, 2, 3
  Erlang đảm bảo per-process FIFO
  Không cần sequence number, không cần reorder buffer


GO + K8s - Không có ordering guarantee:

  Goroutine A gửi qua Redis Pub/Sub:
    redis.Publish("events", event1)
    redis.Publish("events", event2)
    redis.Publish("events", event3)
  
  Goroutine B nhận qua Redis:
    Có thể nhận: 1, 3, 2  (network reorder)
    Có thể nhận: 1, 2     (message loss nếu subscriber lag)
    Có thể nhận: 1, 1, 2, 3 (Redis retry duplicate)
  
  Phải tự implement: sequence numbers + reorder buffer
  + deduplication + acknowledgment
  -> Đây chính là reinventing Erlang message passing
```

---

## 4. Bảng So Sánh - Cái Gì Go + K8s Không Thể Đảm Bảo Native

```
┌──────────────────────────────┬─────────────────┬──────────────────────┐
│ Capability                   │ Erlang/Phoenix  │ Go + K8s             │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Targeted message đến 1 user  │ send(pid, msg)  │ Redis lookup         │
│ bất kể đang ở node nào       │ 0 network hop   │ + HTTP internal call │
│                              │                 │ = 2 hops             │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Session state                │ In-process      │ External Redis       │
│                              │ 0ms access      │ 1-3ms per access     │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Process/goroutine crash      │ Restart micro-  │ Pod restart 30-60s   │
│ recovery                     │ seconds,        │ toàn bộ connections  │
│                              │ connection live │ bị drop              │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Message ordering             │ Guaranteed FIFO │ Không guaranteed     │
│                              │ per process     │ tự implement         │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Broadcast 50K connections    │ Direct send()   │ Redis fanout         │
│                              │ in-process      │ + goroutine per sub  │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Connection survive           │ Có - process    │ Không - pod restart  │
│ server-side crash            │ restart ≠       │ = connection drop    │
│                              │ conn drop       │                      │
├──────────────────────────────┼─────────────────┼──────────────────────┤
│ Scale thêm node              │ Join cluster    │ Redis + NATS config  │
│                              │ tự động         │ thêm pod phức tạp    │
└──────────────────────────────┴─────────────────┴──────────────────────┘
```

---

## 5. Tóm Lại Bằng 1 Câu

```
Erlang PID + send() = mỗi entity trong hệ thống
(user, connection, room, transaction) là 1 actor
có địa chỉ riêng, mailbox riêng, state riêng,
tự xử lý message theo thứ tự, tự recover khi crash.

Go + K8s phải dùng Redis + NATS + service mesh
để tái tạo lại đúng những gì Erlang có sẵn từ 1986.
```
