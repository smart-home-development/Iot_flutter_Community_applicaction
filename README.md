# IoT Flutter Community Application

## Project Overview
This repository is for the frontend development of an IoT-based smart home application built using Flutter. It supports web, mobile, and desktop platforms, providing seamless control and monitoring of smart home devices.

## Contribution
We are in the early stages of development and welcome contributors to help shape this project. Whether you are experienced in Flutter, IoT integrations, or UI/UX design, your input is valuable!

## Features
- Multi-platform support (Web, Mobile, Desktop)
- Responsive and user-friendly interface
- Integration with IoT devices
- Open-source collaboration

## Getting Started
### Prerequisites
- Flutter SDK (latest stable version)
- IDE (Visual Studio Code, Android Studio, etc.)
- Git

### Installation
1. Clone this repository:
   git clone https://github.com/smart-home-development/iot_flutter_community_app.git

2. Navigate to the project directory:
   cd iot_flutter_community_app

3. Install dependencies:
   flutter pub get

4. Run the application:
   flutter run

## How to Contribute
1. Fork the repository.
2. Create a new branch:
   git checkout -b feature/your-feature-name
3. Commit your changes:
   git commit -m "Add your message here"
4. Push your branch:
   git push origin feature/your-feature-name
5. Open a pull request.

Example aws mqtt class
``` dart
/// Initializes AWS IoT Core connection parameters and loads certificates.
/// This is called only once during app lifecycle.
static Future<void> _initializeAwsData() async {
  if (_onlyOnce) {
    _onlyOnce = false;
    
    // Configure MQTT client
    _client = MqttServerClient.withPort(_endpoint, _clientId, _port);
    _client.secure = true;
    _client.keepAlivePeriod = 200;
    _client.setProtocolV311();
    _client.logging(on: false);
/// Initializes AWS IoT Core connection parameters and loads certificates.
/// This is called only once during app lifecycle.
static Future<void> _initializeAwsData() async {
  if (_onlyOnce) {
    _onlyOnce = false;
    
    // Configure MQTT client
    _client = MqttServerClient.withPort(_endpoint, _clientId, _port);
    _client.secure = true;
    _client.keepAlivePeriod = 200;
    _client.setProtocolV311();
    _client.logging(on: false);

    // Load root CA certificate
    _clientAuthorities = await rootBundle.load('assets/aws_root/AmazonRootCA1.pem');
    _context.setTrustedCertificatesBytes(_clientAuthorities.buffer.asUint8List());
    // Load root CA certificate
    _clientAuthorities = await rootBundle.load('assets/aws_root/AmazonRootCA1.pem');
    _context.setTrustedCertificatesBytes(_clientAuthorities.buffer.asUint8List());

    // Load device certificate
    _certificateChain = await rootBundle.load('assets/aws_root/Device certificate.pem.crt');
    _context.useCertificateChainBytes(_certificateChain.buffer.asUint8List());
    // Load device certificate
    _certificateChain = await rootBundle.load('assets/aws_root/Device certificate.pem.crt');
    _context.useCertificateChainBytes(_certificateChain.buffer.asUint8List());

    // Load private key
    _privateKey = await rootBundle.load('assets/aws_root/private.pem.key');
    _context.usePrivateKeyBytes(_privateKey.buffer.asUint8List());
    // Load private key
    _privateKey = await rootBundle.load('assets/aws_root/private.pem.key');
    _context.usePrivateKeyBytes(_privateKey.buffer.asUint8List());

    print("AWS MQTT variables are initialized");
    
    // Connect to AWS IoT Core
    while (_client.connectionStatus!.state != MqttConnectionState.connected) {
      try {
        print('Connecting to AWS IoT core.....');
        await _client.connect();
        print('AWS IoT connected successfully');
        _busyInConnecting = false;
        await Future.delayed(const Duration(seconds: 5));
      } catch (e) {
        print("AWS IoT connection failed: $e");
        await Future.delayed(const Duration(seconds: 10));
      }
    }
  }
}

```



## License
This project is licensed under the MIT License - see the LICENSE file for details.

## Contact
For any queries, suggestions, or discussions, feel free to raise an issue or contact us through GitHub.

