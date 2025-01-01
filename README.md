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
// ignore_for_file: avoid_print

import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'dart:math';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:greenswitch/pages/global_data.dart';
import 'package:mqtt_client/mqtt_client.dart';
import 'package:mqtt_client/mqtt_server_client.dart';

/// Class to handle AWS IoT Core connectivity and MQTT communication.
/// Manages connection setup, certificate loading, and connection state.
class SmartHomeConnectivity {
  // Flag to track if connection attempt is in progress
  static bool _busyInConnecting = false;
  
  // Flag to ensure initialization happens only once
  static bool _onlyOnce = true;
  
  // AWS IoT endpoint configuration
  static const String _endpoint = aws_endpoint;
  static const int _port = 8883;
  
  // Client identifier for MQTT connection
  static late String _clientId;
  
  // Security context for SSL/TLS connection
  static final SecurityContext _context = SecurityContext.defaultContext;
  
  // MQTT client instance
  static late final MqttServerClient _client;
  
  // Certificate and key data
  static late final ByteData _clientAuthorities;
  static late final ByteData _certificateChain; 
  static late final ByteData _privateKey;

  /// Constructor - initializes client ID generation
  SmartHomeConnectivity() {
    _generateClientId();
  }

  /// Generates a random 20-digit client ID for MQTT connection
  void _generateClientId() {
    final Random random = Random();
    String id = '';
    for (int i = 0; i < 20; i++) {
      id += random.nextInt(12).toString();
    }
    _clientId = id;
    print('ClientId : $_clientId');
    _initializeAwsData();
  }

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

      // Load device certificate
      _certificateChain = await rootBundle.load('assets/aws_root/Device certificate.pem.crt');
      _context.useCertificateChainBytes(_certificateChain.buffer.asUint8List());

      // Load private key
      _privateKey = await rootBundle.load('assets/aws_root/private.pem.key');
      _context.usePrivateKeyBytes(_privateKey.buffer.asUint8List());

      print("AWS MQTT variables are initialized");
      connectAwsIot();
    }
  }

  /// Attempts to establish connection to AWS IoT Core.
  /// Implements retry mechanism with delay between attempts.
  static Future<void> connectAwsIot() async {
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

  /// Checks if client is currently connected to AWS IoT Core
  static bool isConnected() {
    return _client.connectionStatus!.state == MqttConnectionState.connected;
  }
}

/// Class to monitor and notify about AWS IoT Core connection status.
/// Implements ChangeNotifier to update UI when connection state changes.
class AWSConnectionNotifier extends ChangeNotifier {
  bool _isConnected = false;
  Timer? _connectionTimer;

  /// Getter for connection status
  bool get isConnected => _isConnected;

  /// Constructor - starts connection monitoring
  AWSConnectionNotifier() {
    print("Constructor called");
    _startConnectionMonitor();
  }

  /// Initializes periodic connection status monitoring
  void _startConnectionMonitor() {
    const Duration checkInterval = Duration(seconds: 2);
    _connectionTimer = Timer.periodic(checkInterval, _checkConnection);
  }

  /// Checks connection status and initiates reconnection if needed.
  /// Called periodically by timer.
  void _checkConnection(Timer timer) {
    final bool connectionState = SmartHomeConnectivity.isConnected();
    if (connectionState != _isConnected) {
      _isConnected = connectionState;
      notifyListeners();
    }

    if (!connectionState && !SmartHomeConnectivity._busyInConnecting) {
      SmartHomeConnectivity._busyInConnecting = true;
      SmartHomeConnectivity.connectAwsIot();
      print('AWS IoT reconnection attempt initiated');
    }
  }

  /// Cleanup timer on dispose
  @override
  void dispose() {
    _connectionTimer?.cancel();
    super.dispose();
  }
}

/// Class to manage smart home device states and controls.
/// Handles individual device switch states and MQTT publishing.
class Device {
  final int totalSwitches;
  final int totalFanSwitches;
  late final List<bool> switchStates;    // Current state of switches
  late final List<bool> uiStates;        // UI state of switches
  late final List<String> switchNames;   // Names of switches
  late final List<bool> progressStates;  // Loading states for switches

  /// Constructor - initializes device with specified number of switches
  Device({
    required this.totalSwitches,
    required this.totalFanSwitches,
  }) {
    _initializeStates();
  }

  /// Initializes arrays to track various states of switches
  void _initializeStates() {
    switchStates = List.filled(totalSwitches, false);
    uiStates = List.filled(totalSwitches, false);
    switchNames = List.filled(totalSwitches, '');
    progressStates = List.filled(totalSwitches, false);
  }

  /// Publishes switch state changes to MQTT topic
  Future<void> publishTopic({
    required String? topic,
    required String? onOrOff,
  }) async {
    if (topic == null || onOrOff == null) return;

    print('Starting topic publishing: $topic');
    final Map<String, String> data = {'status': onOrOff};
    final builder = MqttClientPayloadBuilder();
    builder.addString(jsonEncode(data));
    SmartHomeConnectivity._client.publishMessage(
      topic,
      MqttQos.atMostOnce,
      builder.payload!,
    );
    print('Topic published: $topic');
  }
}

/// Class to manage overall smart home system state and configuration.
/// Handles room management and persistence of configuration data.
class SmartHomeManager extends ChangeNotifier {
  double footerContainerSize = 50;
  final List<String> roomsNameList = [];  // List of configured room names
  final List<Device> devices = [];        // List of devices
  final List<dynamic> roomsData = [];     // Raw room configuration data
  bool gotoDashBoardState = true;         // Dashboard view state

  /// Constructor - initializes room data from storage
  SmartHomeManager() {
    print('SmartHomeManager Constructor called');
    _initializeData();
  }

  /// Loads room configuration data from persistent storage
  Future<void> _initializeData() async {
    try {
      final String? data = jsonEncode(box.get(k.roomsData));
      if (data != null) {
        roomsData.addAll(jsonDecode(data));
        _updateRoomsList();
      }
    } catch (e) {
      print('Error initializing data: $e');
    }
  }

  /// Updates the list of configured room names
  void _updateRoomsList() {
    roomsNameList.clear();
    for (var room in roomsData) {
      roomsNameList.add(room['roomName']);
    }
    print('Rooms list updated: $roomsNameList');
  }

  /// Adds a new room configuration and persists it
  Future<void> addNewRoom(Map<dynamic, dynamic> roomData) async {
    roomsData.add(roomData);
    await box.put(k.roomsData, roomsData);
    _updateRoomsList();
    gotoDashBoardState = false;
    notifyListeners();
  }

  /// Deletes a room configuration at specified index
  Future<void> deleteRoom(int index) async {
    if (index < 0 || index >= roomsData.length) return;

    roomsData.removeAt(index);
    await box.put(k.roomsData, roomsData);
    _updateRoomsList();
    notifyListeners();
  }

  /// Enables dashboard view mode
  void enableDashBoard() {
    gotoDashBoardState = true;
    notifyListeners();
  }

  /// Placeholder for future smart home data generation functionality
  Future<void> generateSmartHomeData() async {}
}
```



## License
This project is licensed under the MIT License - see the LICENSE file for details.

## Contact
For any queries, suggestions, or discussions, feel free to raise an issue or contact us through GitHub.

