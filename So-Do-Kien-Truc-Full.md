# Sơ Đồ Kiến Trúc Giám Sát Ceph Cluster Trên Proxmox Bằng Zabbix

Dưới đây là sơ đồ kiến trúc tổng thể (Full Architecture) mô tả cách dữ liệu giám sát đi từ hạ tầng Ceph lên tới Zabbix Server. Github hỗ trợ hiển thị trực tiếp sơ đồ này.

```mermaid
graph TD
    %% Khối Zabbix Server trung tâm
    subgraph Zabbix_Infrastructure ["🏢 Zabbix Infrastructure"]
        ZS[("🖥️ Zabbix Server / Proxy\n(10.8.10.201)")]
        ZWeb["🌐 Zabbix Web UI\n(Cấu hình Template & Macro)"]
        ZS --- ZWeb
    end

    %% Khối Cụm Proxmox VE
    subgraph Proxmox_Cluster ["☁️ Proxmox VE Cluster (5 Nodes)"]
        
        %% Node 1 (Active Ceph Manager)
        subgraph Node_1 ["🟢 Node 1 (172.17.66.121) - Active MGR"]
            ZA1["🤖 Zabbix Agent 2\n(zabbix_agent2.conf)"]
            
            subgraph Ceph_Active ["Ceph Services"]
                MGR1["👑 Ceph MGR\n(RESTful API - Port 8003)"]
                MON1["👁️ Ceph MON"]
                OSD1["💽 Ceph OSDs"]
            end
            
            Cron1["⏱️ Cronjob (mỗi phút)\nceph health > cephstatus.txt"]
            
            %% Luồng dữ liệu trong Node 1
            ZA1 -- "Zabbix Plugin Ceph\n(Gọi API HTTPS qua Port 8003)" --> MGR1
            ZA1 -. "system.run\n(Dự phòng)" .-> Cron1
        end

        %% Node 2 (Standby Ceph Manager)
        subgraph Node_2 ["🟡 Node 2 (172.17.66.122) - Standby MGR"]
            ZA2["🤖 Zabbix Agent 2"]
            MGR2["🛡️ Ceph MGR (Standby)"]
            OSD2["💽 Ceph OSDs"]
            
            ZA2 -. "Chờ kích hoạt\nnếu Node 1 down" .-> MGR2
        end

        %% Các node còn lại
        subgraph Node_N ["⚪ Các Node Khác (123, 124, 125)"]
            ZAN["🤖 Zabbix Agent 2"]
            OSDN["💽 Ceph OSDs"]
        end
    end

    %% Luồng giao tiếp mạng (Mạng ngoài)
    ZS == "Truy vấn Active/Passive\n(Port 10050/10051)" === ZA1
    ZS == "Truy vấn Active/Passive\n(Port 10050/10051)" === ZA2
    ZS == "Truy vấn Active/Passive\n(Port 10050/10051)" === ZAN

    %% Chú thích
    classDef zabbix fill:#d32f2f,stroke:#fff,stroke-width:2px,color:#fff;
    classDef cephmgr fill:#388e3c,stroke:#fff,stroke-width:2px,color:#fff;
    classDef agent fill:#1976d2,stroke:#fff,stroke-width:2px,color:#fff;
    
    class ZS,ZWeb zabbix;
    class MGR1,MGR2 cephmgr;
    class ZA1,ZA2,ZAN agent;

```

### 🔍 Giải thích luồng hoạt động (Data Flow):
1. **Zabbix Server** giao tiếp với **Zabbix Agent 2** (được cài đặt trên tất cả các node Proxmox) qua port `10050/10051`.
2. Trên **Node đang giữ quyền Active Ceph Manager** (ở ví dụ trên là Node 1), module `Ceph RESTful API` được mở ở port `8003`.
3. **Zabbix Agent 2** sử dụng Plugin tích hợp sẵn (được cấp quyền qua API Key `zabbix-monitor`) để gọi HTTP/REST trực tiếp vào port `8003` lấy toàn bộ thông số cluster (OSD, Pool, IOPS, Latency...).
4. Dữ liệu được trả về Zabbix Agent 2 và đẩy thẳng lên **Zabbix Server**.
5. Trong trường hợp API lỗi, **Cronjob** dự phòng sẽ tự động chạy lệnh `ceph health` ghi ra file `/etc/zabbix/cephstatus.txt`, và Zabbix Agent 2 sẽ đọc file này thông qua khoá `system.run` (để lấy log text cơ bản).

Khi **Node 1** chết, Ceph sẽ tự động Promote (đẩy) Manager của **Node 2** lên làm Active. Lúc này Zabbix Agent 2 trên Node 2 sẽ lập tức tiếp quản việc gọi API qua port 8003 của chính nó, giúp việc giám sát không bao giờ bị gián đoạn (High Availability).
