#General Project Setup and Working Principle 

Here’s a breakdown of the project’s components and key aspects, addressing potential points of confusion:
Purpose
This project creates a Raspberry Pi-based IoT gateway that secures a sensor network using a firewall (iptables) and an Intrusion Detection System (Snort). It publishes sensor data to AWS IoT Core via MQTT and logs security events to CloudWatch, demonstrating security infrastructure for IoT ecosystems.
Files
setup.sh: Bash script to configure iptables firewall rules.
mqtt_client.py: Python script to read DHT11 sensor data, publish it to AWS IoT Core, and log security events.
snort_rules/local.rules: Custom Snort rules to detect ping floods and unauthorized MQTT access.
README.md: Documentation with setup instructions and a demo placeholder.
Hardware
Raspberry Pi: E.g., Pi 4, running Raspbian OS.
DHT11 Sensor: Measures temperature and humidity, connected to GPIO 4.
Software
Raspbian OS: Operating system for the Pi (e.g., Raspberry Pi OS Lite).
iptables: Configures the firewall.
Snort: Intrusion Detection System.
Python 3: Runs mqtt_client.py with libraries paho-mqtt, Adafruit_DHT, and boto3.
AWS IoT Core: Receives sensor data and logs events to CloudWatch.
Key Features
Firewall: Restricts traffic to SSH (port 22) and MQTT (port 1883).
IDS: Snort detects anomalies like ping floods.
Sensor Data: Publishes temperature/humidity every 60 seconds.
Security Logging: Logs dropped packets and Snort alerts to CloudWatch.
Placeholders
Replace your-iot-endpoint.iot.region.amazonaws.com, path/to/your-certificate.pem.crt, path/to/your-private.pem.key, path/to/AmazonRootCA1.pem, and your-region in mqtt_client.py with your AWS credentials.
Testing Locally
To test this project and generate a demo (e.g., a GIF for demo.gif), you’ll set up the Raspberry Pi, configure the firewall and Snort, run the Python script, and simulate an attack. Here’s a detailed step-by-step process:
Prerequisites
Hardware: Raspberry Pi 4, DHT11 sensor, microSD card, USB power supply, jumper wires.
Software: Raspbian OS, Python 3, AWS account, MQTT Explorer (optional), a second device (e.g., laptop) for attack simulation.
Tools: Terminal access (local or SSH).
Step 1: Set Up the Raspberry Pi
Install Raspbian OS:
Download Raspberry Pi OS Lite from raspberrypi.org.
Flash it to a microSD card using Raspberry Pi Imager or a tool like Etcher.
Insert the SD card, connect power, and boot the Pi (default login: pi/raspberry).
Update the System:
Connect to the Pi (via monitor/keyboard or SSH).
Run: sudo apt update && sudo apt upgrade -y.
Install Dependencies:
Install required packages: sudo apt install python3-pip snort -y.
Install Python libraries: pip3 install paho-mqtt Adafruit_DHT boto3.
Step 2: Connect the DHT11 Sensor
Wiring:
VCC to Pi 3.3V (Pin 1).
GND to Pi GND (Pin 6).
Data to GPIO 4 (Pin 7), with a 10kΩ pull-up resistor between VCC and Data.
Verify:
Test later with mqtt_client.py.
Step 3: Configure the Firewall
Create setup.sh:
On the Pi, create the file: nano setup.sh.
Paste:
bash
#!/bin/bash
sudo iptables -F
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 1883 -j ACCEPT
sudo iptables -A INPUT -j LOG --log-prefix "IPTables-Dropped: " --log-level 4
sudo iptables-save > /etc/iptables/rules.v4
echo "Firewall configured: SSH and MQTT allowed, all else dropped."
Save (Ctrl+O, Enter, Ctrl+X) and make executable: chmod +x setup.sh.
Run:
Execute: sudo ./setup.sh.
Verify: sudo iptables -L -v (shows rules allowing SSH and MQTT).
Step 4: Set Up Snort
Configure Rules:
Create the rules file: sudo mkdir -p /etc/snort/rules && sudo nano /etc/snort/rules/local.rules.
Paste:
alert icmp any any -> $HOME_NET any (msg:"Possible Ping Flood"; flags:S; threshold: type threshold, track by_src, count 100, seconds 10; sid Wound0001;)
alert tcp any any -> $HOME_NET 1883 (msg:"Unauthorized MQTT Attempt"; flags:S; sid Wound0002;)
Save and exit.
Update Snort Config:
Edit: sudo nano /etc/snort/snort.conf.
Ensure include $RULE_PATH/local.rules is uncommented (near the end).
Test Snort:
Run: sudo snort -c /etc/snort/snort.conf -A console.
Leave it running in a terminal.
Step 5: Run the MQTT Client
Update mqtt_client.py:
Create: nano mqtt_client.py.
Paste:
python
import paho.mqtt.client as mqtt
import Adafruit_DHT
import time
import json
import boto3
from datetime import datetime

AWS_IOT_ENDPOINT = "your-iot-endpoint.iot.region.amazonaws.com"
CERT_PATH = "path/to/your-certificate.pem.crt"
KEY_PATH = "path/to/your-private.pem.key"
CA_PATH = "path/to/AmazonRootCA1.pem"
TOPIC = "iot/security_gateway/data"
cloudwatch = boto3.client('logs', region_name='your-region')
LOG_GROUP = 'IoTSecurityGateway'
LOG_STREAM = 'GatewayLogs'
SENSOR = Adafruit_DHT.DHT11
PIN = 4

def on_connect(client, userdata, flags, rc):
    print("Connected to AWS IoT with result code: " + str(rc))

def on_publish(client, userdata, mid):
    print("Message published")

client = mqtt.Client(client_id="SecurityGateway")
client.tls_set(CA_PATH, CERT_PATH, KEY_PATH)
client.on_connect = on_connect
client.on_publish = on_publish
client.connect(AWS_IOT_ENDPOINT, 8883, 60)

def log_to_cloudwatch(message):
    cloudwatch.put_log_events(
        logGroupName=LOG_GROUP,
        logStreamName=LOG_STREAM,
        logEvents=[{
            'timestamp': int(datetime.now().timestamp() * 1000),
            'message': message
        }]
    )

client.loop_start()

while True:
    humidity, temperature = Adafruit_DHT.read_retry(SENSOR, PIN)
    if humidity is not None and temperature is not None:
        payload = {
            "timestamp": str(datetime.now()),
            "temperature": temperature,
            "humidity": humidity
        }
        client.publish(TOPIC, json.dumps(payload), qos=1)
        print(f"Published: {payload}")
    else:
        log_to_cloudwatch("Sensor read error")
        print("Failed to read sensor")
    
    with open("/var/log/snort/alert", "r") as f:
        alerts = f.read()
        if alerts:
            log_to_cloudwatch(f"Security alert: {alerts}")
            print("Security event detected")

    time.sleep(60)
Update AWS credentials.
Run:
Execute: python3 mqtt_client.py.
Expect: Sensor data published every 60 seconds.
Step 6: Simulate an Attack
Ping Flood:
From another device on the same network (e.g., laptop), find the Pi’s IP: hostname -I.
Run: ping -f <Pi_IP>.
Check Snort terminal for “Possible Ping Flood” alerts and /var/log/snort/alert for logs.
Step 7: Verify AWS Integration
MQTT:
Use MQTT Explorer, subscribe to iot/security_gateway/data.
See sensor data every minute.

#CloudWatch:
In AWS Console, go to CloudWatch > Logs > IoTSecurityGateway/GatewayLogs.
Verify sensor errors or Snort alerts.
Step 8: Generate the Demo
Record: Use a screen recorder (e.g., Kazam) to capture MQTT Explorer data, Snort alerts, and CloudWatch logs during a ping flood.
Save: Export as demo.gif.
Upload to GitHub: Use “Add file” > “Upload files” in the repository.
General Working Principle
Here’s how the project operates:
Firewall:
setup.sh configures iptables to drop all incoming traffic except SSH (22) and MQTT (1883), logging dropped packets.
Sensor Data:
mqtt_client.py reads DHT11 data every 60 seconds, publishes it to AWS IoT Core (iot/security_gateway/data).
IDS:
Snort monitors network traffic, detects ping floods or unauthorized MQTT attempts, and logs alerts to /var/log/snort/alert.
Security Logging:
The Python script checks Snort logs and sends security events to CloudWatch (IoTSecurityGateway/GatewayLogs).
AWS Integration:
AWS IoT Core receives sensor data; CloudWatch logs security events for analysis.
Flow:
DHT11 → Python → MQTT → AWS IoT Core + CloudWatch; Network → Snort → Logs → Python → CloudWatch.

#Troubleshooting Tips
No Sensor Data: Check GPIO 4 wiring, pull-up resistor, or library installation.
Snort Fails: Verify config file and rules path.
MQTT Issues: Ensure AWS credentials and network access.
Firewall Blocks: Test SSH/MQTT access post-setup.
