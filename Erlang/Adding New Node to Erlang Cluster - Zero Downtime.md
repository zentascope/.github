# Adding New Node to Erlang Cluster - Zero Downtime

## Vấn Đề Cốt Lõi Khi Thêm Node

```
Rủi ro khi thêm node mới:
1. Split-brain - 2 node không nhận ra nhau
2. Cookie mismatch - node bị reject
3. PubSub rebalance gây message loss
4. Load balancer gửi traffic trước khi node ready
5. Hot code trên node cũ vs node mới không match
```

---

## 1. Checklist Trước Khi Thêm Node

```bash
# Trên node hiện tại - kiểm tra health
iex --name admin@current-node --cookie $COOKIE --remsh server1@192.168.1.1

# Check cluster hiện tại
Node.list()
# => [:"server2@192.168.1.2"]

# Check PubSub đang hoạt động
:pg.which_groups(:phoenix_pubsub)

# Check memory/process health
:erlang.memory()
:erlang.system_info(:process_count)
```

---

## 2. Cookie - Điều Kiện Tiên Quyết

Cookie phải **identical** trên tất cả nodes - đây là shared secret để authenticate:

```bash
# Tạo cookie mạnh một lần duy nhất cho toàn cluster
openssl rand -base64 48 | tr -d '\n' > /etc/erlang/.erlang.cookie
chmod 400 /etc/erlang/.erlang.cookie

# Copy y chang sang node mới
scp /etc/erlang/.erlang.cookie user@new-node:/etc/erlang/.erlang.cookie
```

```elixir
# releases.exs hoặc runtime.exs - đọc từ env
config :my_app, MyApp.Release,
  cookie: System.fetch_env!("RELEASE_COOKIE")
```

```bash
# .env trên TẤT CẢ nodes - giá trị phải giống nhau
RELEASE_COOKIE=abc123xyz_same_on_all_nodes
RELEASE_NODE=my_app@192.168.1.3   # <- chỉ IP này thay đổi
```

---

## 3. libcluster - Tự Động Discovery

Đây là layer quan trọng nhất - node mới tự tìm và join cluster:

```elixir
# config/runtime.exs
config :libcluster,
  topologies: [
    production: [
      strategy: Cluster.Strategy.Gossip,
      config: [
        port: 45892,
        if_addr: "0.0.0.0",
        multicast_addr: "255.255.255.255",
        multicast_ttl: 1,
        # Node mới broadcast UDP -> existing nodes nhận -> connect
        secret: System.fetch_env!("CLUSTER_SECRET")
      ]
    ]
  ]
```

```
Gossip flow khi node mới boot:

New Node                    Existing Node 1         Existing Node 2
    |                              |                       |
    |-- UDP broadcast "I exist" -->|                       |
    |                              |-- Node.connect() ---->|
    |<-- Node.connect() -----------|                       |
    |                              |                       |
    | [cluster formed]             |                       |
    |<===== PubSub sync ==========>|                       |
    |<===== :pg groups sync ======>|<=====================>|
```

---

## 4. Deploy Node Mới - Step by Step

### Step 1: Chuẩn bị server mới (Hetzner)

```bash
# Trên Hetzner - tạo server mới cùng network private
# QUAN TRỌNG: dùng private network, không expose Erlang port ra internet

# Erlang distribution port - chỉ mở trong private network
ufw allow from 192.168.1.0/24 to any port 4369   # epmd
ufw allow from 192.168.1.0/24 to any port 9100:9200  # distribution
ufw allow from 0.0.0.0/0 to any port 4000        # HTTP/WS public

# /etc/hosts hoặc dùng private DNS
echo "192.168.1.1 server1" >> /etc/hosts
echo "192.168.1.2 server2" >> /etc/hosts
echo "192.168.1.3 server3" >> /etc/hosts  # node mới
```

### Step 2: Deploy app lên node mới - CHƯA nhận traffic

```bash
# Build release - phải cùng OTP version với node cũ
MIX_ENV=prod mix release

# Copy release sang node mới
rsync -avz _build/prod/rel/my_app/ user@192.168.1.3:/app/

# Start node mới - nó sẽ tự join cluster qua libcluster
RELEASE_NODE=my_app@192.168.1.3 \
RELEASE_COOKIE=$RELEASE_COOKIE \
PHX_HOST=192.168.1.3 \
/app/bin/my_app start
```

### Step 3: Verify node đã join cluster trước khi nhận traffic

```bash
# Remote shell vào node mới
/app/bin/my_app remote

# Kiểm tra đã join cluster chưa
iex> Node.list()
# PHẢI thấy: [:"my_app@192.168.1.1", :"my_app@192.168.1.2"]
# Nếu empty list -> chưa join, KHÔNG thêm vào load balancer

# Kiểm tra PubSub đã sync
iex> Phoenix.PubSub.node_name(MyApp.PubSub)
# => :"my_app@192.168.1.3"

# Kiểm tra process registry sync
iex> :pg.which_groups(:phoenix_pubsub) |> length()
# Phải > 0 nếu có active subscriptions
```

### Step 4: Health check endpoint

```elixir
# router.ex - endpoint riêng cho load balancer check
scope "/internal", MyAppWeb do
  get "/health", HealthController, :check
  get "/cluster", HealthController, :cluster_status
end
```

```elixir
defmodule MyAppWeb.HealthController do
  use MyAppWeb, :controller

  def check(conn, _params) do
    cluster_nodes = Node.list()
    
    status = %{
      node: Node.self(),
      status: :ok,
      cluster_size: length(cluster_nodes) + 1,
      cluster_nodes: cluster_nodes,
      process_count: :erlang.system_info(:process_count),
      memory_mb: div(:erlang.memory(:total), 1_048_576),
      # Node chỉ ready khi đã join cluster
      cluster_ready: length(cluster_nodes) >= 1
    }

    http_status = if status.cluster_ready, do: 200, else: 503

    conn
    |> put_status(http_status)
    |> json(status)
  end
end
```

```bash
# Load balancer chỉ add node khi endpoint này trả 200
curl http://192.168.1.3:4000/internal/health
# {"node":"my_app@192.168.1.3","cluster_ready":true,"cluster_size":3,...}
```

### Step 5: Add vào load balancer - sau khi health check pass

```nginx
# nginx upstream - thêm node mới
upstream phoenix_cluster {
    # Sticky session bằng IP hash - QUAN TRỌNG cho WebSocket
    ip_hash;
    
    server 192.168.1.1:4000;
    server 192.168.1.2:4000;
    server 192.168.1.3:4000;  # <- thêm vào đây SAU KHI health check pass
}
```

---

## 5. WebSocket Sticky Session - Tránh Connection Drop

```
KHÔNG dùng round-robin cho WebSocket:

Request 1 -> Node 1 (WS connected here)
Request 2 -> Node 2 (WS state không có ở đây -> disconnect)

PHẢI dùng ip_hash hoặc cookie-based sticky:
```

```nginx
# nginx - ip_hash sticky
upstream ws_backend {
    ip_hash;
    server 192.168.1.1:4000 weight=1;
    server 192.168.1.2:4000 weight=1;
    server 192.168.1.3:4000 weight=1;  # node mới
    
    keepalive 1000;  # connection pool đến backend
}

server {
    location /socket {
        proxy_pass http://ws_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 3600s;  # long-lived WS connection
    }
}
```

```elixir
# HAProxy alternative - tốt hơn cho WebSocket
# haproxy.cfg
backend phoenix_nodes
    balance source          # sticky by source IP
    option  http-server-close
    option  forwardfor
    
    server node1 192.168.1.1:4000 check inter 5s
    server node2 192.168.1.2:4000 check inter 5s
    server node3 192.168.1.3:4000 check inter 5s  # thêm node mới
```

---

## 6. Xử Lý Existing Connections Khi Node Mới Join

```
Khi node3 join cluster:
- Connections trên node1, node2: KHÔNG bị ảnh hưởng
- PubSub tự rebalance: node3 subscribe vào các existing topics
- NEW connections từ giờ có thể đến node3
- Existing connections tiếp tục ở node1/node2 cho đến khi reconnect tự nhiên
```

```elixir
# Nếu muốn graceful migration - optional
# Node cũ thông báo clients reconnect dần dần
defmodule MyApp.GracefulMigration do
  def nudge_reconnect(percentage) do
    # Lấy x% connections hiện tại
    # Gửi signal để client reconnect
    # Client sẽ reconnect -> load balancer -> có thể đến node mới
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "system:migration",
      {:please_reconnect, %{reason: "cluster_rebalance"}}
    )
  end
end
```

---

## 7. Rolling Update - Khi Update Code

Đây là trường hợp khó hơn - update code trên cluster đang chạy:

```bash
# Strategy: update từng node một, không update tất cả cùng lúc

# 1. Remove node1 khỏi load balancer (drain traffic)
# 2. Đợi existing connections drain (có timeout)
# 3. Deploy code mới lên node1
# 4. Node1 restart, rejoin cluster
# 5. Health check pass -> add lại vào load balancer
# 6. Lặp lại cho node2, node3
```

```elixir
# Drain connections gracefully trước khi shutdown
defmodule MyApp.GracefulShutdown do
  def drain(timeout_ms \\ 30_000) do
    # Thông báo clients sắp disconnect
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "system:global",
      {:server_draining, %{reconnect_in: 5_000}}
    )

    # Đợi connections tự đóng hoặc timeout
    Process.sleep(timeout_ms)
    System.stop(0)
  end
end
```

```bash
# Script deploy rolling update
#!/bin/bash
NODES=("192.168.1.1" "192.168.1.2" "192.168.1.3")

for node in "${NODES[@]}"; do
  echo "Deploying to $node..."
  
  # Remove khỏi LB
  ./scripts/lb_remove.sh $node
  
  # Deploy
  rsync -avz _build/prod/rel/my_app/ user@$node:/app/
  ssh user@$node "/app/bin/my_app restart"
  
  # Đợi health check pass
  until curl -sf http://$node:4000/internal/health; do
    echo "Waiting for $node to be healthy..."
    sleep 3
  done
  
  # Add lại vào LB
  ./scripts/lb_add.sh $node
  echo "$node deployed successfully"
  
  # Đợi một chút trước khi deploy node tiếp theo
  sleep 10
done
```

---

## 8. Tóm Tắt Flow An Toàn

```
[NEW SERVER]
    │
    ▼
1. Setup: cookie, firewall, hosts, env vars
    │
    ▼
2. Deploy app, start node
    │
    ▼
3. libcluster tự động join cluster
    │
    ▼
4. Verify: Node.list() thấy existing nodes
    │
    ▼
5. Health endpoint trả 200 + cluster_ready: true
    │
    ▼
6. Thêm vào load balancer (nginx ip_hash)
    │
    ▼
7. Monitor 5-10 phút: connections, memory, PubSub
    │
    ▼
[DONE - zero downtime]
```

Toàn bộ quá trình từ bước 2 đến khi nhận traffic thực tế chỉ mất **2-5 phút** và **existing connections không bị ảnh hưởng** vì Erlang PubSub tự sync, không cần restart hay migrate state thủ công.
