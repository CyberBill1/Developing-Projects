# IoT Security Gateway with Firewall and IDS

A Raspberry Pi-based IoT gateway securing a sensor network with `iptables` (firewall) and Snort (IDS), publishing data to AWS IoT Core via MQTT and logging events to CloudWatch.

## Prerequisites
- Raspberry Pi (e.g., 4) with Raspbian OS
- DHT11 sensor (GPIO 4)
- AWS IoT Core setup (certificates, endpoint)
- Python 3, `paho-mqtt`, `Adafruit_DHT`, `boto3`
- `iptables`, Snort

## Setup
1. **Hardware**: Connect DHT11 to Pi (VCC: 3.3V, GND: GND, Data: GPIO 4).
2. **System**: `sudo apt update && sudo apt install python3-pip snort -y; pip3 install paho-mqtt Adafruit_DHT boto3`.
3. **Firewall**: `chmod +x setup.sh && sudo ./setup.sh`.
4. **Snort**: Copy `snort_rules/local.rules` to `/etc/snort/rules/local.rules`, then `sudo snort -c /etc/snort/snort.conf -A console`.
5. **AWS**: Update `mqtt_client.py` with your AWS IoT Core endpoint, certs, and region.
6. **Run**: `python3 mqtt_client.py`.

## Demo
- Publishes temperature/humidity every 60s to AWS IoT Core.
- Detects ping floods (e.g., `ping -f <Pi_IP>`) and logs to CloudWatch.

![Demo](demo.gif) <!-- Add a GIF or screenshot -->
