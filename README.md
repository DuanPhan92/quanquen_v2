# Smart Socket - Ổ Cắm Thông Minh

Thiết bị ổ cắm thông minh IoT cho phép điều khiển từ xa qua ứng dụng mobile và giám sát tiêu thụ điện năng.

---

## 🎯 Tính Năng

- ✅ Bật/Tắt ổ cắm từ mobile app
- ✅ Giám sát tiêu thụ: Dòng, Điện áp, Công suất
- ✅ WiFi Provisioning không cần lập trình
- ✅ OTA Update từ xa
- ✅ Calibration đã được hoàn tất tại nhà máy

---

## 📱 Hướng Dẫn Người Dùng

### 1️⃣ Kết Nối WiFi (Smart Provisioning)

> ⚠️ **Lưu ý quan trọng:** Phải gửi MQTT config **trước**, cài đặt WiFi **sau cùng**. Sau khi nhận được WiFi credentials, thiết bị sẽ **tự disconnect** khỏi provisioning và kết nối vào mạng WiFi ngay lập tức.

**Bước 1: Nhấn giữ nút để vào chế độ Reset / Provisioning**
- **Trường hợp thông thường (nút bấm đang hoạt động):** Nhấn và **giữ nút** trên thiết bị trong **3 giây**.
- **Trường hợp nút bấm đã bị khóa (trong cấu hình):** Cần **ngắt điện và cắm lại thiết bị (khởi động lại)**, sau đó nhấn và **giữ nút** trong **3 giây** (phải thao tác trong vòng 5 giây đầu tiên sau khi thiết bị khởi động lại).
- **LED NET nháy** → Thiết bị vào chế độ Reset / Provisioning.

**Bước 2: App kết nối vào thiết bị**
- Thiết bị phát WiFi SoftAP tên `PROV_XXXXXX` (6 ký tự cuối MAC)
- App kết nối vào SoftAP này

**Bước 3: Gửi MQTT config** *(bắt buộc làm TRƯỚC)*
- Gọi custom endpoint `custom-data` với JSON cài đặt MQTT (xem API bên dưới)
- Thiết bị lưu cấu hình vào NVS

**Bước 4: Gửi WiFi credentials** *(làm SAU CÙNG)*
- App gửi SSID + Password cho thiết bị
- Thiết bị **ngắt kết nối provisioning** và kết nối vào WiFi ngay
- **LED NET sáng cố định** → Kết nối MQTT thành công ✅

---

### 2️⃣ Trạng Thái LED

| Màu LED | Trạng Thái |
|---------|-----------|
| 🔴 WIFI | Nhấp nháy: Chế độ Prov, OFF: Disconnected to Broker, ON: Connected to broker |
| 🔴 LOAD | OFF: Relay switch off, ON: Relay switch on |

---

### 3️⃣ Provisioning API (Custom Endpoint)

Trong quá trình provisioning, app giao tiếp với thiết bị qua **custom endpoint** tên `custom-data`.  
Format: **JSON** (không phải Protobuf).  
Sử dụng [ESP SoftAP Provisioning](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/provisioning/wifi_provisioning.html) hoặc thư viện BLE Provisioning của Espressif.

#### Lấy thông tin thiết bị

**Request:**
```json
{
  "cmd": "info"
}
```

**Response:**
```json
{
  "cmd": "info",
  "value": {
    "type": 1,
    "fw_ver": "1.0.0.0",
    "hw_ver": "1.0.0.0"
  }
}
```

#### Cài đặt MQTT *(gửi trước khi cài WiFi)*

**Request:**
```json
{
  "cmd": "mqtt",
  "value": {
    "host": "mqtt.broker.com",
    "port": 8883,
    "username": "user",
    "password": "pass",
    "client_id": "device_abc123"
  }
}
```

> `client_id` sẽ được dùng làm **Device ID** (`{deviceId}`) cho các MQTT topics: `devices/{deviceId}/request/+`, `devices/{deviceId}/response/{requestId}`, `devices/{deviceId}/telemetry`

**Response:**
```json
{
  "cmd": "mqtt",
  "status": true
}
```

> `status: false` nếu thiếu `host` hoặc `port`

#### Cài đặt Device

**Request:**
```json
{
  "cmd": "device",
  "value": {
    "button": true
  }
}
```

> `button`: cho phép bật/tắt chức năng điều khiển relay bằng nút bấm vật lý. Nếu bằng `false`, nút nhấn sẽ bị vô hiệu hóa sau 5 giây khởi động để tránh người dùng nhấn nhầm. Tuy nhiên, trong 5 giây đầu tiên sau khi thiết bị khởi động lại, nút nhấn vẫn luôn được kích hoạt tạm thời để cho phép người dùng nhấn giữ 3 giây kích hoạt chế độ Reset / Provisioning.

**Response:**
```json
{
  "cmd": "device",
  "status": true
}
```

---

### 4️⃣ Điều Khiển

## 🔌 Giao Tiếp MQTT (RPC-style với JSON)

### Thông Tin Kết Nối

```
Protocol: MQTTS (MQTT with SSL/TLS)
Port: 8883
Format: Text (JSON)
QoS: 0
```

### Cấu Trúc Topic

*   **Request (Server → Thiết bị):**
    ```
    devices/{deviceId}/request/{requestId}
    ```
    Trong đó `{requestId}` là id duy nhất cho mỗi yêu cầu dạng số (ví dụ: `1001`), được Client/Server tự sinh ra để ánh xạ phản hồi tương ứng.
*   **Response (Thiết bị → Server):**
    ```
    devices/{deviceId}/response/{requestId}
    ```
    Thiết bị sẽ phản hồi kết quả vào topic này với `{requestId}` khớp với `{requestId}` nhận được từ request.
*   **Telemetry / Event (Thiết bị chủ động đẩy lên):**
    ```
    devices/{deviceId}/telemetry
    ```
    Chứa thông tin online/offline (Last Will), cập nhật trạng thái relay, cảnh báo cắm/rút tải hoặc định kỳ báo cáo dữ liệu điện năng tiêu thụ.

---

### RPC API Endpoints

Mọi yêu cầu gửi từ Server xuống thiết bị đều tuân theo cấu trúc JSON:
```json
{
  "method": "<tên_method>",
  "params": {}
}
```

#### 1. Lấy Trạng Thái Relay
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "relay_read",
      "params": {}
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "relay_read",
          "params": {
            "value": true
          }
        }
        ```
        *(value: `true` = Bật, `false` = Tắt)*

#### 2. Điều Khiển Relay (Bật/Tắt)
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "relay_write",
      "params": {
        "value": true
      }
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "relay_write",
          "params": {
            "value": true
          }
        }
        ```

#### 3. Lấy Dữ Liệu Tiêu Thụ (Info)
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "info",
      "params": {}
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "info",
          "params": {
            "volt": 220.4,
            "current": 0.51,
            "power": 97.6,
            "energy": 12.8
          }
        }
        ```
    *   **Đơn vị:**
        *   `volt`: V (Vôn)
        *   `current`: A (Ampe, làm tròn 2 chữ số thập phân)
        *   `power`: W (Oát)
        *   `energy`: kWh (Kilôoát giờ, làm tròn 1 chữ số thập phân)

#### 4. Reset Điện Năng Cộng Dồn
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "reset_energy",
      "params": {}
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "reset_enrgy",
          "params": {
            "status": true
          }
        }
        ```
        *(Lưu ý: method phản hồi là `reset_enrgy` không có chữ `e` thứ hai để tương thích ngược với API cũ)*

#### 5. Lấy Cường Độ Tín Hiệu WiFi (RSSI)
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "rssi",
      "params": {}
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "rssi",
          "params": {
            "rssi": -58
          }
        }
        ```

#### 6. Ping / Pong (Keep Alive)
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "ping",
      "params": {}
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "pong",
          "params": {}
        }
        ```

#### 7. Cập Nhật Cấu Hình Ngưỡng Báo Cáo (Setting Set/Get)
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request Get:**
    ```json
    {
      "method": "setting_get",
      "params": {}
    }
    ```
*   **Request Set (cho phép gửi riêng rẽ hoặc gộp chung các tham số thay đổi):**
    ```json
    {
      "method": "setting_set",
      "params": {
        "voltage_report_delta_mv": 5000,
        "current_report_delta_ma": 100,
        "power_report_delta_w": 5,
        "pf_report_delta_x1000": 100,
        "energy_report_delta_wh": 10
      }
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "setting_set", // hoặc "setting_get" tương ứng
          "params": {
            "voltage_report_delta_mv": 5000,
            "current_report_delta_ma": 100,
            "power_report_delta_w": 5,
            "pf_report_delta_x1000": 100,
            "energy_report_delta_wh": 10
          }
        }
        ```

#### 8. Yêu Cầu Cập Nhật Firmware OTA
*   **Topic:** `devices/{deviceId}/request/{requestId}`
*   **Request:**
    ```json
    {
      "method": "ota",
      "params": {
        "url": "https://example.com/firmware.bin",
        "is_app": true
      }
    }
    ```
*   **Response:**
    *   **Topic:** `devices/{deviceId}/response/{requestId}`
    *   **Payload:**
        ```json
        {
          "method": "ota",
          "params": {
            "status": true
          }
        }
        ```

---

### Telemetry / Event (Chủ Động Gửi)

Thiết bị sẽ tự động đẩy tin nhắn dạng phẳng (flat JSON) lên topic `devices/{deviceId}/telemetry`.

#### 1. Sự Kiện Online (Ngay sau khi kết nối MQTT thành công)
*   **Payload:**
    ```json
    {
      "mac": "AA:BB:CC:DD:EE:FF",
      "ip": "192.168.1.100",
      "profile": "QQGD",
      "version": "1.0.0",
      "name": "",
      "online": true
    }
    ```

#### 2. Sự Kiện Offline (MQTT Last Will & Testament)
Nếu thiết bị mất kết nối đột ngột, Broker sẽ tự động gửi message Last Will đăng ký sẵn:
*   **Payload:**
    ```json
    {
      "online": false
    }
    ```

#### 3. Cập Nhật Trạng Thái Relay
Gửi lên khi relay được bật/tắt (nhấn nút cứng vật lý hoặc điều khiển từ xa thành công):
*   **Payload:**
    ```json
    {
      "relay": true
    }
    ```

#### 4. Báo Cáo Thay Đổi Điện Năng Tiêu Thụ (Chủ động gửi định kỳ / khi vượt ngưỡng delta)
*   **Payload:**
    ```json
    {
      "volt": 220.1,
      "current": 0.42,
      "power": 92.0,
      "energy": 11.5
    }
    ```

#### 5. Cảnh Báo Cắm/Rút Tải
Được kích hoạt khi công suất tải thay đổi qua các ngưỡng thiết lập:
*   **Đã cắm tải (Công suất vượt ngưỡng >10W):**
    ```json
    {
      "load": true
    }
    ```
*   **Đã rút tải (Công suất tụt dưới ngưỡng <5W):**
    ```json
    {
      "load": false
    }
    ```

