# Scale Hàng Trăm Tỷ Process - Phân Tích Thực Tế

## Trước Tiên - Đặt Lại Con Số

```
Hàng trăm tỷ Erlang processes = số không tưởng.

Hãy calibrate lại:

  Dân số thế giới:          ~8 tỷ người
  Internet users:           ~5.4 tỷ
  Concurrent users cao nhất
  (tất cả app, mọi thời điểm): ~500M - 1B

  Nếu mỗi user = 10 processes:
  1B users * 10 = 10 tỷ processes concurrent

  "Hàng trăm tỷ" = vượt xa toàn bộ internet hiện tại.

Câu hỏi thực tế hơn:
  - Hàng tỷ processes concurrent? Có thể plan được.
  - Hàng chục tỷ? Hyperscale thực sự - chỉ vài công ty.
  - Hàng trăm tỷ? Cần xem lại định nghĩa "process".
```

---

## 1. Tại Sao > 100M Thì Complexity Tương Đương

### Vấn Đề Vật Lý Không Thể Tránh

```
Dù dùng Erlang hay Go, ở scale này bạn đều đối mặt
với những vấn đề mà KHÔNG CÓ ngôn ngữ nào giải được:

┌─────────────────────────────────────────────────────┐
│ 1. SPEED OF LIGHT PROBLEM                           │
│                                                     │
│ Hà Nội -> New York: ~180ms latency (vật lý)         │
│ Erlang distribution không thể vượt vật lý           │
│ -> PHẢI có regional clusters                        │
│ -> PHẢI có data replication cross-region            │
│ -> Đây là distributed systems problem, không phải  │
│    ngôn ngữ problem                                 │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ 2. CAP THEOREM                                      │
│                                                     │
│ Consistency + Availability + Partition tolerance    │
│ Chỉ được 2 trong 3                                  │
│                                                     │
│ Erlang không bypass CAP theorem                     │
│ Ở scale global, mọi techstack đều phải chọn        │
│ và đánh đổi như nhau                                │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ 3. COORDINATION OVERHEAD                            │
│                                                     │
│ 1000 nodes cần coordinate với nhau                  │
│ :global registry sync = O(N²) messages              │
│ Erlang full-mesh = không scale quá ~200 nodes       │
│                                                     │
│ -> Phải dùng sharding + routing layer               │
│ -> Routing layer này: Erlang hay Go đều phức tạp   │
│    như nhau vì đây là distributed systems design    │
└─────────────────────────────────────────────────────┘
```

### Erlang Cluster Topology Giới Hạn

```
Full mesh (default Erlang):

  10 nodes:   45 connections    - OK
  50 nodes:   1,225 connections - bắt đầu nặng
  100 nodes:  4,950 connections - giới hạn thực tế
  200 nodes:  19,900 connections - không khuyến nghị

Ở > 100M connections cần > 100 nodes
-> Full mesh không còn hoạt động tốt
-> Phải dùng hierarchical topology
-> Lúc này design tương tự Go + K8s sharding
```

---

## 2. Lợi Thế Erlang Vẫn Còn Ở Scale Lớn

```
Dù complexity tương đương ở hyperscale,
Erlang vẫn có lợi thế QUAN TRỌNG:

┌─────────────────────────────────────────────────────┐
│ LỢI THẾ 1: Density - Nhiều hơn trên cùng hardware  │
│                                                     │
│ 1 server vật lý 128GB RAM:                          │
│   Go:     ~1M connections (100KB/conn)              │
│   Erlang: ~20M connections (3-5KB/conn)             │
│                                                     │
│ -> Erlang cần ít servers hơn 20x                    │
│ -> Ở scale 1B connections:                          │
│    Go:     ~1,000 servers                           │
│    Erlang: ~50 servers                              │
│ -> Chi phí infra khác nhau rất lớn                  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ LỢI THẾ 2: Fault isolation vẫn tốt hơn ở mọi scale │
│                                                     │
│ 1 process crash = chỉ 1 user bị ảnh hưởng          │
│ Dù cluster có 1B processes hay 1M processes         │
│ Isolation model không thay đổi                      │
│                                                     │
│ Go: Pod crash = 25K-100K users bị ảnh hưởng        │
│ Dù scale nhỏ hay lớn, đây vẫn là vấn đề            │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ LỢI THẾ 3: Ít moving parts hơn trong mỗi shard     │
│                                                     │
│ Mỗi shard Erlang: tự chứa, không cần Redis/NATS    │
│ Mỗi shard Go: cần Redis + NATS + service mesh       │
│                                                     │
│ 100 shards Erlang: 100 clusters                     │
│ 100 shards Go: 100 clusters + 300 Redis             │
│                + 300 NATS + monitoring tất cả       │
└─────────────────────────────────────────────────────┘
```

---

## 3. Chiến Lược Triển Khai Cho Scale Tỷ Processes

### Tầng 1 - Regional Cluster Architecture

```
GLOBAL SCALE ARCHITECTURE

         ┌─────────────────────────────────────┐
         │          GLOBAL ROUTING LAYER        │
         │      (DNS GeoDNS + Anycast IP)        │
         │   Không phải Erlang - đây là infra   │
         └──────┬──────────┬──────────┬─────────┘
                │          │          │
         ┌──────▼──┐  ┌────▼────┐  ┌──▼──────┐
         │ Region  │  │ Region  │  │ Region  │
         │  APAC   │  │   EU    │  │   US    │
         │         │  │         │  │         │
         │Erlang   │  │Erlang   │  │Erlang   │
         │Cluster  │  │Cluster  │  │Cluster  │
         │         │  │         │  │         │
         │Shard A  │  │Shard D  │  │Shard G  │
         │Shard B  │  │Shard E  │  │Shard H  │
         │Shard C  │  │Shard F  │  │Shard I  │
         └─────────┘  └─────────┘  └─────────┘
              │              │            │
              └──────────────┴────────────┘
                    Cross-region sync
                (chỉ sync metadata, không sync
                 toàn bộ process state)
```

### Tầng 2 - Shard Design Bên Trong

```elixir
# Mỗi shard là 1 Erlang cluster độc lập
# Xử lý 10-50M connections
# Nodes trong shard: 5-20 nodes

defmodule MyApp.ShardRouter do
  @num_shards 256  # consistent hashing ring

  def shard_for(entity_id) do
    :erlang.phash2(entity_id, @num_shards)
  end

  def route(entity_id, message) do
    shard = shard_for(entity_id)
    region = region_for_shard(shard)

    cond do
      region == current_region() ->
        # Local region - gửi trong Erlang cluster
        local_route(shard, entity_id, message)

      true ->
        # Cross-region - HTTP/gRPC đến region khác
        # Latency cao hơn nhưng chấp nhận được
        # vì đây là cross-continent anyway
        RemoteRegion.send(region, shard, entity_id, message)
    end
  end
end
```

### Tầng 3 - Process Hierarchy Trong Mỗi Shard

```
Đây là nơi Erlang thực sự tỏa sáng.
Thay vì flat list processes, dùng hierarchy:

SHARD INTERNAL STRUCTURE:

  ShardSupervisor
       │
       ├── RegionSupervisor (theo geographic area)
       │        │
       │        ├── CitySupervisor
       │        │        │
       │        │        ├── UserBucket_0 (GenServer, 10K users)
       │        │        ├── UserBucket_1
       │        │        └── UserBucket_N
       │        │
       │        └── CitySupervisor_2...
       │
       ├── ChannelSupervisor
       │        │
       │        ├── Channel_room_1 (process per room)
       │        ├── Channel_room_2
       │        └── Channel_room_N
       │
       └── SystemSupervisor
                │
                ├── MetricsCollector
                ├── HealthMonitor
                └── ShardCoordinator
```

```elixir
# UserBucket - 1 process quản lý N users
# Thay vì 1 process per user ở hyperscale
# -> Giảm process count, tăng density

defmodule MyApp.UserBucket do
  use GenServer

  # Bucket quản lý 10,000 users
  # State là map: user_id -> user_state
  def init(bucket_id) do
    {:ok, %{
      bucket_id: bucket_id,
      users: %{},      # user_id -> %UserState{}
      connections: %{} # user_id -> websocket_pid
    }}
  end

  def handle_cast({:user_message, user_id, msg}, state) do
    case Map.get(state.connections, user_id) do
      nil -> {:noreply, state}
      ws_pid ->
        send(ws_pid, {:push, msg})
        {:noreply, state}
    end
  end

  # Broadcast đến tất cả users trong bucket
  def handle_cast({:broadcast, msg}, state) do
    Enum.each(state.connections, fn {_uid, ws_pid} ->
      send(ws_pid, {:push, msg})
    end)
    {:noreply, state}
  end
end
```

### Tầng 4 - Horde Cho Dynamic Process Distribution

```elixir
# Horde là library Elixir cho distributed process registry
# và distributed supervisor - không cần full mesh

# mix.exs
{:horde, "~> 0.9"}

# Horde dùng CRDT (Conflict-free Replicated Data Type)
# thay vì :global - scale tốt hơn nhiều

defmodule MyApp.ProcessRegistry do
  use Horde.Registry

  def start_link(_) do
    Horde.Registry.start_link(
      __MODULE__,
      [keys: :unique],
      name: __MODULE__
    )
  end
end

defmodule MyApp.DistributedSupervisor do
  use Horde.DynamicSupervisor

  def start_link(_) do
    Horde.DynamicSupervisor.start_link(
      __MODULE__,
      [strategy: :one_for_one],
      name: __MODULE__
    )
  end

  # Horde tự quyết định node nào start process
  # Khi node fail, Horde restart process trên node khác
  # Không cần biết PID cụ thể ở đâu
  def start_user_process(user_id) do
    Horde.DynamicSupervisor.start_child(
      __MODULE__,
      {MyApp.UserProcess, user_id}
    )
  end
end
```

```
Tại sao Horde tốt hơn :global ở scale lớn?

:global (default Erlang):
  - Full sync mọi node khi có thay đổi
  - Lock toàn cluster khi register name mới
  - O(N) nodes bị ảnh hưởng mỗi operation

Horde (CRDT-based):
  - Eventual consistency - không cần lock
  - Mỗi node chỉ cần sync với neighbors
  - Scale đến hàng trăm nodes trong 1 cluster
  - Node join/leave không gây global lock
```

---

## 4. Số Liệu Thực Tế - Ai Đã Làm Được

```
WhatsApp (Erlang):
  - 2 tỷ users, ~100M concurrent connections
  - ~50 engineers maintain toàn bộ backend
  - Kiến trúc: nhiều Erlang clusters, sharded by user
  - Không public chi tiết nhưng confirmed Erlang core

Discord (Elixir):
  - 19M concurrent users peak
  - Chuyển từ Go sang Elixir vì memory và latency
  - Dùng Manifold library cho distributed messaging
  - 1 Guild (server) = 1 Erlang process

Riot Games (Erlang):
  - League of Legends chat service
  - 7.5M concurrent users
  - Erlang cluster, không public kiến trúc chi tiết
```

---

## 5. Honest Assessment - Hàng Trăm Tỷ Process

```
Nếu bạn thực sự cần hàng trăm tỷ processes
CONCURRENT cùng lúc:

Không có techstack nào làm được điều này
trong 1 hệ thống unified.

Bạn cần tái định nghĩa:

Option A: Hàng trăm tỷ processes TỔNG CỘNG
          (không phải concurrent)
          -> Erlang làm được, processes tạo và chết
          -> 1M processes/giây * 100K giây = 100 tỷ total
          -> Hoàn toàn feasible

Option B: Hàng tỷ concurrent, tỷ lệ active thấp
          -> Erlang hibernation - process ngủ dùng ~200 bytes
          -> 100 tỷ hibernated = 20TB RAM - không thực tế
          -> Cần tiered storage: hot/warm/cold processes

Option C: Hàng tỷ "entities" nhưng không phải
          tất cả là processes cùng lúc
          -> Đây là kiến trúc đúng đắn
          -> Virtual actors pattern
```

```elixir
# Virtual Actor Pattern - Scale đến hàng tỷ entities
# Chỉ tạo process khi entity đang active

defmodule MyApp.VirtualActor do
  # Thay vì 1 process per entity luôn luôn sống
  # Chỉ activate khi có message đến

  def send_to(entity_id, message) do
    case find_active_process(entity_id) do
      {:ok, pid} ->
        # Process đang active, gửi trực tiếp
        send(pid, message)

      :not_found ->
        # Activate process từ persistent state
        {:ok, pid} = activate(entity_id)
        send(pid, message)
    end
  end

  defp activate(entity_id) do
    # Load state từ DB/ETS
    state = MyApp.StateStore.load(entity_id)
    # Start process với state đã load
    DynamicSupervisor.start_child(
      MyApp.ActorSupervisor,
      {MyApp.EntityProcess, {entity_id, state}}
    )
  end
end

# Process tự hibernate sau X phút không có activity
defmodule MyApp.EntityProcess do
  use GenServer

  @idle_timeout 5 * 60 * 1000  # 5 phút

  def handle_info(:timeout, state) do
    # Persist state xuống DB
    MyApp.StateStore.save(state.id, state)
    # Process tự chết, giải phóng memory
    {:stop, :normal, state}
  end

  def handle_info(msg, state) do
    new_state = process(msg, state)
    # Reset timeout mỗi khi có activity
    {:noreply, new_state, @idle_timeout}
  end
end
```

---

## 6. Roadmap Thực Tế Cho Sản Phẩm Global Scale

```
GIAI ĐOẠN 1 - 0 đến 1M users:
  1 Erlang cluster, 3-5 nodes
  1 region
  Không cần sharding
  Focus: product-market fit

GIAI ĐOẠN 2 - 1M đến 50M users:
  1-3 Erlang clusters per region
  2-3 regions
  Bắt đầu consistent hashing
  Horde thay :global
  Focus: reliability, latency

GIAI ĐOẠN 3 - 50M đến 500M users:
  Nhiều shards per region (10-50 shards)
  5+ regions
  Virtual actor pattern
  Cross-region replication
  Focus: cost optimization, global latency

GIAI ĐOẠN 4 - 500M+ users:
  Tất cả techstack đều phức tạp như nhau
  Erlang vẫn tiết kiệm hardware hơn 5-10x
  Team engineering lớn hơn không thể tránh
  Focus: custom optimization per use case
```

---

## Tóm Lại

```
> 100M complexity tương đương vì:
  Distributed systems problems không phụ thuộc ngôn ngữ
  Physics, CAP theorem, network topology là universal

Erlang vẫn có lợi thế dù ở scale nào:
  20x density cao hơn -> ít server hơn -> ít tiền hơn
  Fault isolation tốt hơn ở mọi scale
  Ít external dependencies trong mỗi shard

Hàng trăm tỷ processes:
  Cần redefinition - không ai chạy concurrent số này
  Virtual actor pattern là hướng đúng
  WhatsApp/Discord là benchmarks thực tế nhất

Lời khuyên thực tế:
  Build for 10M trước, design cho 100M
  Đừng over-engineer cho "hàng trăm tỷ" từ đầu
  Erlang/Elixir sẽ scale cùng bạn tốt hơn
  bất kỳ techstack nào khác ở mỗi giai đoạn
```
