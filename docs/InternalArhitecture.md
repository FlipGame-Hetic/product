## Our Technical Decisions
### Why we use an ESP32-S3
The ESP32-S3 is compact while still delivering solid performance, and it benefits from a very large community. Its libraries are mature, well documented, and actively maintained. It also provides native Wi-Fi and Bluetooth connectivity, which makes integrations such as MQTT communication straightforward.

### Why we use MQTT
MQTT is the most reliable protocol for our use case. It is lightweight and designed for real-time communication, which keeps latency predictable. It scales easily, and it natively handles reconnections, which is critical for hardware devices that may temporarily lose connectivity.

### Why we use Google Cloud Platform
Google Cloud Platform offers a simpler interface than AWS while being significantly more reliable than Railway for real-time workloads. It scales beyond simple replicas and gives far more operational flexibility. The ecosystem and community are larger than Railway’s, and over the long term it is generally cheaper than AWS for this type of infrastructure. It is also technically more valuable to work with a real cloud environment rather than staying purely local.

Louis Dondey - Arnaud Fischer - v1.0.0 - 20/02/26
