# Erlang/OTP Distributed Messaging - Không Cần Redis/Kafka

Bạn đúng một phần - Erlang/OTP có **distributed primitives built-in** rất mạnh. Để rõ ràng hơn, mình sẽ phân tích từng layer.

---

## 1. Erlang Distributed Nodes - Core Concept

Erlang có **native node clustering** - các node có thể gửi message trực tiếp qua PID mà không cần broker.

```elixir
# Node A (server1@host1)
# Gửi message đến PID trên node khác - native Erlang
pid_on_other_node = {some_process, :"server2@host2"}
send(pid_on_other_node, {:hello, "from node1"})

# Hoặc dùng Node.spawn
Node.spawn(:"server2@host2", fn ->
  IO.puts("Running on node 2!")
end)
```

```bash
# Khởi động node với tên và cookie (cookie = shared secret giữa các node)
iex --name server1@192.168.1.1 --cookie my_secret_cookie -S mix phx.server
iex --name server2@192.168.1.2 --cookie my_secret_cookie -S mix phx.server
```

```elixir
# Connect nodes
Node.connect(:"server2@192.168.1.2")
Node.list() # => [:"server2@192.168.1.2"]
```

---

## 2. Phoenix + libcluster - Auto Discovery

Dùng `libcluster` để tự động connect các node khi scale:

```elixir
# mix.exs
{:libcluster, "~> 3.3"},
{:phoenix, "~> 1.7"}
```

```elixir
# config/runtime.exs
config :libcluster,
  topologies: [
    my_app: [
      strategy: Cluster.Strategy.Gossip,  # UDP gossip - no Redis needed
      # Hoặc dùng Kubernetes:
      # strategy: Cluster.Strategy.Kubernetes,
      config: [
        multicast_addr: "255.255.255.255",
        port: 45892,
        if_addr: "0.0.0.0"
      ]
    ]
  ]
```

```elixir
# application.ex
def start(_type, _args) do
  topologies = Application.get_env(:libcluster, :topologies, [])

  children = [
    {Cluster.Supervisor, [topologies, [name: MyApp.ClusterSupervisor]]},
    MyAppWeb.Endpoint,
    # ...
  ]
  Supervisor.start_link(children, strategy: :one_for_one)
end
```

---

## 3. Phoenix PubSub - Built-in Pub/Sub (Thay Redis Pub/Sub)

Phoenix.PubSub dùng **pg (Erlang Process Groups)** internally - broadcast qua tất cả nodes trong cluster:

```elixir
# Broadcast từ bất kỳ node nào - tất cả nodes nhận được
Phoenix.PubSub.broadcast(MyApp.PubSub, "room:lobby", {:new_message, payload})

# Subscribe trên node bất kỳ
Phoenix.PubSub.subscribe(MyApp.PubSub, "room:lobby")

# Handle trong process
def handle_info({:new_message, payload}, state) do
  IO.inspect(payload)
  {:noreply, state}
end
```

```elixir
# config/config.exs
config :my_app, MyApp.PubSub,
  name: MyApp.PubSub,
  adapter: Phoenix.PubSub.PG2  # default - dùng Erlang :pg, no Redis
```

---

## 4. WebSocket - Phoenix Channel Cross-Node

Đây là killer feature - khi user kết nối WebSocket vào **node bất kỳ**, broadcast vẫn đến **tất cả clients** trên tất cả nodes:

```elixir
# user_socket.ex
defmodule MyAppWeb.UserSocket do
  use Phoenix.Socket

  channel "room:*", MyAppWeb.RoomChannel
  channel "user:*", MyAppWeb.UserChannel
end
```

```elixir
# room_channel.ex
defmodule MyAppWeb.RoomChannel do
  use Phoenix.Channel

  def join("room:" <> room_id, _params, socket) do
    {:ok, assign(socket, :room_id, room_id)}
  end

  def handle_in("new_message", payload, socket) do
    # Broadcast đến TẤT CẢ clients trong room - kể cả trên node khác
    broadcast!(socket, "new_message", payload)
    {:noreply, socket}
  end

  # Nhận broadcast từ server-side (ví dụ: từ GenServer trên node khác)
  def handle_info({:send_to_client, data}, socket) do
    push(socket, "server_event", data)
    {:noreply, socket}
  end
end
```

---

## 5. Gửi Message Trực Tiếp Qua PID Cross-Node

Đây là Erlang thuần - gửi thẳng đến PID của process trên node khác:

```elixir
defmodule MyApp.ProcessRegistry do
  # Dùng :global để register process name across all nodes
  def register(name) do
    :global.register_name(name, self())
  end

  def lookup(name) do
    case :global.whereis_name(name) do
      :undefined -> {:error, :not_found}
      pid -> {:ok, pid}
    end
  end

  def send_to(name, message) do
    case lookup(name) do
      {:ok, pid} -> send(pid, message)
      error -> error
    end
  end
end
```

```elixir
# Worker trên Node A - tự đăng ký
defmodule MyApp.Worker do
  use GenServer

  def start_link(id) do
    GenServer.start_link(__MODULE__, id, name: {:global, {:worker, id}})
  end

  def init(id) do
    IO.puts("Worker #{id} started on #{Node.self()}")
    {:ok, %{id: id}}
  end

  def handle_cast({:job, data}, state) do
    IO.puts("Processing job on #{Node.self()}: #{inspect(data)}")
    {:noreply, state}
  end
end
```

```elixir
# Từ Node B - gửi đến Worker trên Node A (không cần biết node nào)
GenServer.cast({:global, {:worker, "user_123"}}, {:job, %{task: "send_email"}})
```

---

## 6. Hswarm - Khi Nào Vẫn Cần Redis/Kafka?

```
Erlang Cluster thay thế được:        Vẫn nên dùng Redis/Kafka khi:
---------------------------------    ------------------------------------
✅ Redis Pub/Sub                     ⚠️  Persistent message queue
✅ Redis Session Store (dùng ETS)    ⚠️  Cross-language consumers (Python, Go...)
✅ Simple job queue (dùng GenServer) ⚠️  Replay / event sourcing at scale
✅ WebSocket broadcast               ⚠️  Analytics pipeline (Kafka)
✅ Service discovery                 ⚠️  Multi-datacenter replication
```

---

## 7. Production Setup Checklist

```elixir
# runtime.exs - full production config
config :libcluster,
  topologies: [
    k8s: [
      strategy: Cluster.Strategy.Kubernetes,
      config: [
        mode: :hostname,
        kubernetes_node_basename: "my_app",
        kubernetes_selector: "app=my-app",
        kubernetes_namespace: "production",
        polling_interval: 10_000
      ]
    ]
  ]

# Đảm bảo RELEASE_COOKIE giống nhau trên tất cả pods
# Kubernetes secret: MY_APP_COOKIE=<same_value_all_pods>
```

```dockerfile
# Dockerfile
ENV RELEASE_COOKIE=my_production_secret_cookie
ENV RELEASE_DISTRIBUTION=name
ENV RELEASE_NODE=my_app@$(hostname -f)
```

---

## Tóm Lại

| Use case | Giải pháp Erlang | Cần external? |
|---|---|---|
| WebSocket broadcast | Phoenix PubSub + PG2 | Không |
| Cross-node messaging | Native `send/2` + `:global` | Không |
| Service discovery | libcluster + Gossip/K8s | Không |
| Process registry | `:global` hoặc `Horde` | Không |
| Persistent queue | GenStage / Broadway | Có thể cần Kafka |
| Multi-lang consumers | - | Cần Kafka |

Erlang cluster với Phoenix là **self-contained** cho hầu hết real-time use cases. Redis chỉ cần nếu bạn có **non-Elixir services** cần communicate, hoặc cần **durable message storage**.
