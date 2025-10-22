import Foundation
import CoreBluetooth

// 🔹 Data structure to store each gas reading
struct GasEntry: Identifiable {
    let id = UUID()      // Unique ID for SwiftUI's List
    let level: Int       // Gas level reading
    let time: Date       // Timestamp of reading
}

// 🔹 Bluetooth manager handles connection to Micro:bit and data updates
class BluetoothManager: NSObject, ObservableObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    
    // MARK: - Published properties (automatically update UI)
    @Published var gasLevel: Int = 0               // Latest reading
    @Published var statusMessage = "Searching..."  // Status message for connection / danger
    @Published var isDanger = false                // Is gas level above danger threshold?
    
    // 🔹 Historical log of gas readings
    @Published var gasLog: [GasEntry] = []
    let maxLogEntries = 20                         // Keep only last 20 readings
    
    // MARK: - Bluetooth properties
    private var centralManager: CBCentralManager!
    private var microbitPeripheral: CBPeripheral?
    private var uartCharacteristic: CBCharacteristic?
    
    // UUIDs for Micro:bit UART service
    private let uartServiceUUID = CBUUID(string: "6E400001-B5A3-F393-E0A9-E50E24DCCA9E")
    private let uartTXUUID = CBUUID(string: "6E400003-B5A3-F393-E0A9-E50E24DCCA9E")
    
    // Threshold above which gas is considered dangerous
    let dangerThreshold = 300
    
    // MARK: - Initialization
    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
    }
    
    // MARK: - Start scanning for Micro:bit
    func startScanning() {
        statusMessage = "🔍 Scanning for Micro:bit..."
        centralManager.scanForPeripherals(withServices: [uartServiceUUID])
    }
    
    // MARK: - CBCentralManagerDelegate
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            startScanning()
        } else {
            statusMessage = "⚠️ Bluetooth not available"
        }
    }
    
    // Called when a peripheral is discovered
    func centralManager(_ central: CBCentralManager,
                        didDiscover peripheral: CBPeripheral,
                        advertisementData: [String : Any],
                        rssi RSSI: NSNumber) {
        statusMessage = "✅ Found Micro:bit!"
        microbitPeripheral = peripheral
        microbitPeripheral?.delegate = self
        centralManager.stopScan()
        centralManager.connect(peripheral)
    }
    
    // Called when connection is successful
    func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
        statusMessage = "🔗 Connected!"
        peripheral.discoverServices([uartServiceUUID])
    }
    
    // MARK: - CBPeripheralDelegate
    func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
        guard let services = peripheral.services else { return }
        for service in services where service.uuid == uartServiceUUID {
            peripheral.discoverCharacteristics([uartTXUUID], for: service)
        }
    }
    
    func peripheral(_ peripheral: CBPeripheral, didDiscoverCharacteristicsFor service: CBService, error: Error?) {
        for characteristic in service.characteristics ?? [] where characteristic.uuid == uartTXUUID {
            uartCharacteristic = characteristic
            peripheral.setNotifyValue(true, for: characteristic)
            statusMessage = "📡 Receiving data..."
        }
    }
    
    // Called when new data is received from Micro:bit
    func peripheral(_ peripheral: CBPeripheral, didUpdateValueFor characteristic: CBCharacteristic, error: Error?) {
        guard let value = characteristic.value,
              let text = String(data: value, encoding: .utf8),
              let number = Int(text.trimmingCharacters(in: .whitespacesAndNewlines)) else { return }
        
        DispatchQueue.main.async {
            // Update current gas level
            self.gasLevel = number
            
            // Update danger status
            if number > self.dangerThreshold {
                self.statusMessage = "⚠️ DANGER DETECTED!"
                self.isDanger = true
            } else {
                self.statusMessage = "✅ Safe"
                self.isDanger = false
            }
            
            // Append new reading to log
            let entry = GasEntry(level: number, time: Date())
            self.gasLog.append(entry)
            
            // Keep only last N entries to save memory
            if self.gasLog.count > self.maxLogEntries {
                self.gasLog.removeFirst(self.gasLog.count - self.maxLogEntries)
            }
        }
    }
}

