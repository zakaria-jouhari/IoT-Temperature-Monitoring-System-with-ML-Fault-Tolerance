# 🌡️ IoT Temperature Monitoring System with ML Fault Tolerance

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![Arduino](https://img.shields.io/badge/Arduino-1.8.19-00979D.svg)](https://www.arduino.cc/)

An advanced IoT temperature monitoring system featuring **machine learning-based fault tolerance** for critical industrial applications. The system automatically switches to ML predictions when sensor failures occur, ensuring 100% service continuity.

![System Architecture](docs/images/architecture.png)

## ✨ Key Features

- 🤖 **ML Fault Tolerance**: Automatic failover to linear regression predictions during sensor failures
- 🔄 **Real-time Monitoring**: 1Hz data acquisition with sub-second recovery time
- 📊 **Live Dashboard**: Node-RED interface with dual-trace visualization (real vs predicted)
- ⚡ **Edge Computing**: Local ML inference on Jetson Nano (< 1ms latency)
- 🎯 **High Accuracy**: ±0.35°C MAE during normal operation, ±0.5°C during failures
- 📡 **MQTT Integration**: Seamless cloud connectivity with HiveMQ
- 🔌 **I2C Communication**: Robust protocol with automatic error detection

## 🏗️ System Architecture

```
┌─────────────┐         ┌──────────────┐         ┌─────────────┐
│   Arduino   │ I2C     │ Jetson Nano  │  MQTT   │  Node-RED   │
│   (Edge)    ├────────►│  (Gateway)   ├────────►│  (Cloud)    │
│             │ 400kHz  │              │ 1883    │             │
└─────────────┘         └──────────────┘         └─────────────┘
  Temp Sensor           ML Prediction            Visualization
  LED Indicators        LCD Display              Dashboard
```

### Three-Layer Architecture

1. **Edge Layer (Arduino)**: Temperature generation, LED feedback, I2C slave
2. **Gateway Layer (Jetson Nano)**: ML engine, data aggregation, failover logic
3. **Cloud Layer (Node-RED)**: Web dashboard, analytics, historical data

## 📋 Hardware Requirements

| Component | Model | Specifications |
|-----------|-------|----------------|
| SBC | NVIDIA Jetson Nano | 4GB RAM, 128-core GPU |
| MCU | Arduino Uno | ATmega328P, 16MHz |
| Display | LCD 16x2 I2C | PCF8574, Address 0x27 |
| LEDs | RGB 5mm | Red/Yellow/Green status |
| Resistors | 1/4W | 3x 120Ω for LED protection |

## 🛠️ Software Requirements

- **Jetson Nano**: Ubuntu 18.04 LTS, Python 3.8+
- **Arduino IDE**: 1.8.19 or later
- **Node-RED**: 3.0.2+ with dashboard plugin
- **Python Libraries**: `smbus2`, `paho-mqtt`, `numpy`, `RPLCD`

## 🚀 Quick Start

### 1. Clone Repository

```bash
git clone https://github.com/yourusername/iot-temp-monitor-ml.git
cd iot-temp-monitor-ml
```

### 2. Setup Arduino

```bash
cd arduino
# Open arduino_temp_sensor.ino in Arduino IDE
# Select Board: Arduino Uno
# Select Port: /dev/ttyACM0 (or your port)
# Upload the sketch (Ctrl+U)
```

### 3. Setup Jetson Nano

```bash
cd jetson
chmod +x setup.sh
./setup.sh  # Installs dependencies

# Verify I2C devices
sudo i2cdetect -y 1  # Should show Arduino (0x08) and LCD (0x27)

# Run the system
python3 temperature_monitor.py
```

### 4. Setup Node-RED Dashboard

```bash
cd node-red
npm install  # Install dashboard dependencies

# Import flows
node-red
# Open http://localhost:1880
# Import flows.json via Menu > Import
```

Access dashboard at: **http://localhost:1880/ui**

## 📊 Performance Metrics

| Metric | Value |
|--------|-------|
| Service Continuity | 100% |
| Normal Mode Accuracy | ±0.35°C (MAE) |
| Prediction Mode Accuracy | ±0.5°C |
| Recovery Time | < 1 second |
| ML Inference Latency | < 1ms |
| Update Frequency | 1 Hz |
| I2C Frequency | 400 kHz |

## 🧠 Machine Learning Model

The system uses **multivariate linear regression** with sliding windows:

```
T̂(n+1) = Σ(wi × T(n-i)) + b
         i=0 to 4
```

- **Features**: Last 5 temperature readings
- **Training**: Ordinary Least Squares (OLS)
- **Update Strategy**: Incremental (every 50 samples)
- **Confidence Levels**: 30% → 90% (based on sample count)

## 🔄 Operating Modes

| Mode | Condition | Data Source | LCD Indicator |
|------|-----------|-------------|---------------|
| **NORMAL** | I2C success | Arduino sensor | `T: XX.X°C` |
| **FALLBACK** | 1-2 I2C failures | Last valid value | `T: XX.X°C*` |
| **PREDICTION** | 3+ I2C failures | ML model | `ML:XX.X°C` |

## 📁 Project Structure

```
iot-temp-monitor-ml/
├── arduino/
│   ├── arduino_temp_sensor.ino    # Arduino sensor code
│   └── README.md
├── jetson/
│   ├── temperature_monitor.py      # Main system
│   ├── ml_predictor.py            # ML model
│   ├── setup.sh                   # Installation script
│   ├── requirements.txt
│   └── README.md
├── node-red/
│   ├── flows.json                 # Dashboard flows
│   ├── package.json
│   └── README.md
├── docs/
│   ├── images/
│   ├── architecture.md
│   ├── installation.md
│   ├── troubleshooting.md
│   └── api_reference.md
├── tests/
│   ├── test_i2c.py
│   ├── test_ml_model.py
│   └── test_mqtt.py
├── LICENSE
└── README.md
```

## 📡 MQTT Message Format

### Temperature Reading (Normal Mode)
```json
{
  "temperature": 12.5,
  "status": "NORMAL",
  "mode": "NORMAL",
  "confidence": 1.0,
  "model_trained": true,
  "samples": 95,
  "timestamp": "2025-01-12 14:23:45",
  "failure_count": 0
}
```

### ML Prediction Mode
```json
{
  "temperature": 14.3,
  "status": "WARM",
  "mode": "PREDICTION",
  "confidence": 0.87,
  "model_trained": true,
  "samples": 95,
  "timestamp": "2025-01-12 14:25:12",
  "failure_count": 5
}
```

## 🎮 MQTT Commands

Send commands to topic: `jetson/command`

| Command | Payload | Action |
|---------|---------|--------|
| `RESET` | - | Reset failure counter |
| `RETRAIN` | - | Force ML model retraining |
| `SAVE` | `{"filename": "model.pkl"}` | Save model |
| `GET_STATUS` | - | Return full system status |

## 🧪 Testing

```bash
# Test I2C communication
python3 tests/test_i2c.py

# Test ML model
python3 tests/test_ml_model.py

# Test MQTT connectivity
python3 tests/test_mqtt.py

# Run all tests
pytest tests/
```

## 🐛 Troubleshooting

### Arduino Not Detected
```bash
# Check I2C devices
sudo i2cdetect -y 1

# If not showing, check:
# - USB cable connection
# - Correct I2C address (0x08)
# - Power supply
```

### LCD Not Displaying
```bash
# Verify LCD address
sudo i2cdetect -y 1  # Should show 0x27

# Check wiring:
# - SDA → GPIO2 (Pin 3)
# - SCL → GPIO3 (Pin 5)
# - VCC → 5V
# - GND → GND
```

### MQTT Connection Failed
```bash
# Test broker connectivity
ping broker.hivemq.com

# Test MQTT subscription
mosquitto_sub -h broker.hivemq.com -t jetson/temperature -v
```

See [docs/troubleshooting.md](docs/troubleshooting.md) for more solutions.

## 📈 Experimental Results

24-hour continuous operation test:

- **Service Availability**: 100%
- **Simulated Failures**: 12 occurrences
- **Average Failure Duration**: 15.3 minutes
- **ML Prediction Accuracy**: ±0.35°C MAE
- **Maximum Prediction Error**: ±1.0°C
- **Recovery Success Rate**: 100%

![Performance Graph](docs/images/24h_test.png)

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

Please ensure your code follows:
- PEP 8 style guide for Python
- Arduino style guide for C/C++
- Comprehensive inline comments

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 👥 Authors

- **Zakaria Jouhari** - *Lead Developer* - zakaria.jouhari@uit.ac.ma
- **Nouhaila Souaidi** - *Co-Developer* - nouhaila.souai@uit.ac.ma
- **Widad Fahd** - *Co-Developer* - widad.fahd@uit.ac.ma

**Supervised by**: Professor Abderrahim Bajit (abderrahim.bajit@uit.ac.ma)

**Institution**: École Nationale des Sciences Appliquées (ENSA) Kénitra, Morocco

## 🙏 Acknowledgments

- NVIDIA for Jetson Nano platform
- Arduino community for extensive libraries
- Node-RED team for visualization tools
- HiveMQ for public MQTT broker

## 📚 Citation

If you use this project in your research, please cite:

```bibtex
@article{jouhari2025iot,
  title={IoT Temperature Monitoring System with ML-Based Fault Tolerance: Architecture, Implementation and Experimental Validation},
  author={Jouhari, Zakaria and Souaidi, Nouhaila and Fahd, Widad},
  journal={ENSA Kénitra Technical Report},
  year={2025}
}
```

## 📞 Contact & Support

- 📧 Email: zakaria.jouhari@uit.ac.ma
- 🐛 Issues: [GitHub Issues](https://github.com/yourusername/iot-temp-monitor-ml/issues)
- 💬 Discussions: [GitHub Discussions](https://github.com/yourusername/iot-temp-monitor-ml/discussions)

---

**⭐ If you find this project useful, please consider giving it a star!**

Made with ❤️ at ENSA Kénitra
