# Horde - Tại Sao Cần Và Hoạt Động Như Thế Nào 👋

<!--

**Here are some ideas to get you started:**

🙋‍♀️ A short introduction - what is your organization all about?
🌈 Contribution guidelines - how can the community get involved?
👩‍💻 Useful resources - where can the community find your docs? Is there anything else the community should know?
🍿 Fun facts - what does your team eat for breakfast?
🧙 Remember, you can do mighty things with the power of [Markdown](https://docs.github.com/github/writing-on-github/getting-started-with-writing-and-formatting-on-github/basic-writing-and-formatting-syntax)
-->


## Vấn Đề Horde Giải Quyết

```
Trước khi hiểu Horde, cần hiểu vấn đề:

Bạn có 1 Erlang cluster, 5 nodes.
User "alice" connect vào Node 1.
UserProcess của alice start trên Node 1.

Câu hỏi 1: Node 3 muốn gửi message đến alice
            -> Làm sao biết alice đang ở Node 1?

Câu hỏi 2: Node 1 crash
            -> UserProcess của alice chết
            -> Ai restart nó? Trên node nào?

Câu hỏi 3: Thêm Node 6 vào cluster
            -> Làm sao distribute processes sang node mới?

Erlang built-in không trả lời tốt 3 câu hỏi này.
Horde sinh ra để giải quyết chúng.
```

---

## 1. Erlang Built-in Có Gì - Và Thiếu Gì

### :global - Registry Mặc Định

```elixir
# Erlang built-in: :global registry
# Đăng ký tên process cho toàn cluster

:global.register_name({:user, "alice"}, self())

# Node khác lookup
pid = :global.whereis_name({:user, "alice"})
send(pid, message)
```

```
Vấn đề của :global:

Khi register_name() được gọi:

  Node1 ──broadcast──► Node2: "alice = PID<1.234.0>"
  Node1 ──broadcast──► Node3: "alice = PID<1.234.0>"
  Node1 ──broadcast──► Node4: "alice = PID<1.234.0>"
  Node1 ──broadcast──► Node5: "alice = PID<1.234.0>"

  Tất cả nodes phải ACK
  Trong thời gian chờ ACK: cluster bị LOCK
  Không ai register được name khác

  10 nodes:   chấp nhận được
  100 nodes:  lock time ~100ms - bắt đầu có vấn đề
  1000 nodes: lock time ~seconds - không dùng được
```

### Supervisor Mặc Định

```elixir
# Built-in DynamicSupervisor - chỉ biết local node

DynamicSupervisor.start_child(MyApp.Supervisor, {UserProcess, "alice"})
# -> Process chỉ start trên NODE HIỆN TẠI
# -> Không tự distribute sang node khác
# -> Không tự restart trên node khác khi node crash
```

```
Vấn đề:

  Node1 [Supervisor] -> start alice, bob, carol
  Node2 [Supervisor] -> start dave, eve, frank
  Node3 [Supervisor] -> start george, henry, ivan

  Node1 CRASH:
    alice, bob, carol -> CHẾT
    Node2, Node3 Supervisor -> không biết, không làm gì
    alice, bob, carol -> KHÔNG ĐƯỢC RESTART

  Để fix phải tự viết:
    - Monitor tất cả nodes
    - Detect node down
    - Decide node nào restart processes
    - Handle race condition khi nhiều nodes cùng detect
    -> Đây chính xác là những gì Horde làm
```

---

## 2. Horde Là Gì - Cơ Chế Bên Dưới

```
Horde = Distributed Supervisor + Distributed Registry
        được build trên CRDT

CRDT = Conflict-free Replicated Data Type
     = Cấu trúc dữ liệu có thể replicate
       mà không cần lock, không cần coordinator
```

### CRDT Hoạt Động Như Thế Nào

```
Tưởng tượng mỗi node giữ 1 bản copy của registry.
Các bản copy này có thể khác nhau tạm thời.
Nhưng chúng được thiết kế để tự merge mà không conflict.

Node1 registry:  {alice: PID<1.1>, bob: PID<1.2>}
Node2 registry:  {alice: PID<1.1>, carol: PID<2.1>}
Node3 registry:  {bob: PID<1.2>, carol: PID<2.1>}

Sau khi sync (không cần lock):
Tất cả nodes:    {alice: PID<1.1>, bob: PID<1.2>, carol: PID<2.1>}

Quy tắc merge được định nghĩa trước
-> Không bao giờ có conflict
-> Không cần global lock
-> Eventual consistency
```

```
So sánh :global vs Horde khi register name:

:global:
  Register -> Lock toàn cluster -> Chờ ACK tất cả nodes -> Unlock
  Thời gian: O(N nodes) * round-trip-time
  Blocking: CÓ

Horde:
  Register -> Update local CRDT -> Gossip đến neighbors
  Thời gian: O(1) local, async gossip sau
  Blocking: KHÔNG
  Trade-off: Eventual consistency (vài ms để sync)
```

---

## 3. Horde.Registry - Distributed Process Registry

```elixir
# Setup
defmodule MyApp.HordeRegistry do
  use Horde.Registry

  def start_link(_) do
    Horde.Registry.start_link(
      __MODULE__,
      [keys: :unique],
      name: __MODULE__
    )
  end

  def child_spec(_) do
    spec = super([])
    # Horde tự join tất cả nodes trong cluster
    %{spec | id: __MODULE__}
  end
end
```

```elixir
# Đăng ký process với tên
defmodule MyApp.UserProcess do
  use GenServer

  def start_link(user_id) do
    GenServer.start_link(
      __MODULE__,
      user_id,
      # Đăng ký vào Horde Registry thay vì :global
      name: {:via, Horde.Registry, {MyApp.HordeRegistry, user_id}}
    )
  end
end
```

```elixir
# Lookup và gửi message - từ bất kỳ node nào
defmodule MyApp.Messaging do
  def send_to_user(user_id, message) do
    case Horde.Registry.lookup(MyApp.HordeRegistry, user_id) do
      [{pid, _meta}] ->
        send(pid, message)
        :ok
      [] ->
        {:error, :user_not_found}
    end
  end
end
```

```
Điều xảy ra bên dưới khi lookup:

  Node3 gọi Horde.Registry.lookup("alice")
      │
      ▼
  Horde.Registry trên Node3 kiểm tra local CRDT copy
  (không cần network call nếu đã sync)
      │
      ▼
  Tìm thấy: alice -> PID<1.234.0> (trên Node1)
      │
      ▼
  Trả về PID
      │
      ▼
  send(pid, message) -> Erlang distribution
  tự route đến Node1
```

---

## 4. Horde.DynamicSupervisor - Distributed Supervisor

```elixir
# Setup
defmodule MyApp.HordeSupervisor do
  use Horde.DynamicSupervisor

  def start_link(_) do
    Horde.DynamicSupervisor.start_link(
      __MODULE__,
      [
        strategy: :one_for_one,
        # Horde tự chọn node để start child
        distribution_strategy: Horde.UniformDistribution
      ],
      name: __MODULE__
    )
  end

  def start_user_process(user_id) do
    Horde.DynamicSupervisor.start_child(
      __MODULE__,
      {MyApp.UserProcess, user_id}
    )
    # Horde quyết định node nào start process này
    # Developer không cần quan tâm
  end
end
```

### Distribution Strategy

```elixir
# Horde có 2 strategies built-in:

# 1. UniformDistribution - chia đều
# Node1: 33% processes
# Node2: 33% processes
# Node3: 34% processes
distribution_strategy: Horde.UniformDistribution

# 2. UniformQuorumDistribution - cần quorum
# Chỉ start process khi đủ nodes online
# Tránh split-brain
distribution_strategy: Horde.UniformQuorumDistribution

# 3. Custom strategy
defmodule MyApp.GeoDistribution do
  @behaviour Horde.DistributionStrategy

  # Start process trên node gần user nhất
  def choose_node(child_spec, members) do
    user_region = get_region(child_spec)
    members
    |> Enum.filter(&same_region?(&1, user_region))
    |> Enum.random()
  end
end
```

---

## 5. Scenario Thực Tế - Node Crash và Recovery

```
KHÔNG CÓ HORDE:

  Cluster: Node1, Node2, Node3
  Node1 đang chạy: alice, bob, carol (3000 users)

  Node1 CRASH
      │
      ▼
  Node2, Node3: phát hiện Node1 down qua heartbeat
  Node2, Node3: ... không làm gì cả
  alice, bob, carol: MẤT, không được restart
  Users: mất kết nối, phải reconnect thủ công
  State: mất nếu không persist


CÓ HORDE:

  Cluster: Node1, Node2, Node3
  Horde.DynamicSupervisor chạy trên tất cả nodes
  Node1 đang chạy: alice, bob, carol

  Node1 CRASH
      │
      ▼
  Horde detect node down (qua Erlang distribution)
      │
      ▼
  Horde tính toán: processes trên Node1 cần restart
      │
      ▼
  Horde.UniformDistribution chia:
    Node2 restart: alice, bob
    Node3 restart: carol
      │
      ▼
  Processes được restart trên nodes còn lại
  State được restore từ persistent storage
      │
      ▼
  Clients reconnect -> tìm thấy process qua Registry
  Downtime: vài giây thay vì mãi mãi
```

```elixir
# UserProcess với state persistence cho recovery

defmodule MyApp.UserProcess do
  use GenServer

  def start_link(user_id) do
    GenServer.start_link(
      __MODULE__,
      user_id,
      name: {:via, Horde.Registry, {MyApp.HordeRegistry, user_id}}
    )
  end

  def init(user_id) do
    # Khi restart sau node crash, load state từ DB
    state = case MyApp.StateStore.load(user_id) do
      {:ok, saved_state} -> saved_state      # restore
      {:error, :not_found} -> fresh_state(user_id)  # mới
    end
    {:ok, state}
  end

  def handle_info({:push, message}, state) do
    # Push xuống WebSocket nếu connected
    case state.socket_pid do
      nil -> buffer_message(state, message)
      pid -> send(pid, {:ws_push, message})
    end
    {:noreply, state}
  end

  def terminate(_reason, state) do
    # Trước khi chết: persist state
    MyApp.StateStore.save(state.user_id, state)
  end
end
```

---

## 6. Horde Với Node Mới Join Cluster

```
KHÔNG CÓ HORDE:

  Cluster ban đầu: Node1 (1000 procs), Node2 (1000 procs)
  Thêm Node3 vào cluster
  
  Node3: trống rỗng, không có process nào
  Node1, Node2: vẫn overloaded
  
  Không có rebalancing tự động
  Phải tự viết migration logic


CÓ HORDE:

  Cluster ban đầu: Node1 (1000 procs), Node2 (1000 procs)
  Thêm Node3 vào cluster
      │
      ▼
  Horde detect member mới
      │
      ▼
  UniformDistribution tính lại: mỗi node nên có ~667 procs
      │
      ▼
  Horde migrate ~333 procs từ Node1 sang Node3
  Horde migrate ~333 procs từ Node2 sang Node3
      │
      ▼
  Cluster cân bằng: Node1(667), Node2(667), Node3(666)
  
  Tự động, không cần operator can thiệp
```

---

## 7. Horde vs Các Alternatives

```
┌─────────────────┬──────────────┬────────────┬─────────────┐
│                 │ :global      │ Horde      │ Swarm       │
├─────────────────┼──────────────┼────────────┼─────────────┤
│ Registry type   │ Global lock  │ CRDT       │ Hash ring   │
│ Scale           │ ~50 nodes    │ ~200 nodes │ ~100 nodes  │
│ Consistency     │ Strong       │ Eventual   │ Eventual    │
│ Node failure    │ Manual       │ Auto       │ Auto        │
│ Rebalancing     │ Không có     │ Tự động    │ Tự động     │
│ Complexity      │ Thấp         │ Trung bình │ Trung bình  │
│ Production use  │ WhatsApp*    │ Discord    │ Ít hơn      │
└─────────────────┴──────────────┴────────────┴─────────────┘

*WhatsApp tự build distributed layer riêng
```

---

## 8. Khi Nào Dùng Horde

```
DÙNG HORDE khi:
  ✅ Cluster > 10 nodes
  ✅ Processes cần survive node failure
  ✅ Cần auto-rebalancing khi thêm/bớt node
  ✅ Số lượng dynamic processes lớn (>100K)
  ✅ Không muốn tự viết distributed supervisor

KHÔNG CẦN HORDE khi:
  ✅ Cluster nhỏ < 5 nodes
  ✅ Processes stateless, crash không sao
  ✅ Đã dùng Phoenix.PubSub cho communication
     (PubSub không cần Horde)
  ✅ Simple apps, :global là đủ
```

---

## Tóm Lại Bằng 1 Hình

```
VẤN ĐỀ          :global / built-in     HORDE
─────────────────────────────────────────────────

"Alice đang      Broadcast hỏi          CRDT local lookup
 ở node nào?"    tất cả nodes           O(1), no network

"Node crash,     Không ai xử lý         Tự detect, tự
 process chết"   developer tự làm       restart trên node khác

"Thêm node       Processes không        Tự migrate
 mới"            di chuyển              rebalance tự động

"1000 nodes      Global lock            Gossip protocol
 register        timeout, deadlock      eventual consistency
 cùng lúc"                              không block
```

Horde về bản chất là lớp **distributed systems glue** giúp Erlang cluster hoạt động như 1 hệ thống thống nhất thay vì tập hợp các nodes độc lập - đặc biệt quan trọng khi cluster scale lên và nodes liên tục join/leave.
