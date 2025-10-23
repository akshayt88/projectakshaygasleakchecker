import Foundation
import CoreBluetooth
import UserNotifications
import AudioToolbox // 🚨 Required for fast, synchronous sound alerts

// 🔹 Gas reading structure
struct GasEntry: Identifiable, Codable {
    let id = UUID()
    let level: Int
    let time: Date
}

// 🔹 Bluetooth Manager
class BluetoothManager: NSObject, ObservableObject, CBCentralManagerDelegate, CBPeripheralDelegate {
    
    // MARK: - Published properties for UI
    @Published var gasLevel: Int = 0
    @Published var statusMessage = "Searching..."
    @Published var isDanger = false
    @Published var gasLog: [GasEntry] = []
    
    // MARK: - Configuration
    let dangerThreshold = 300
    let maxLogEntries = 20
    
    // MARK: - BLE properties
    private var centralManager: CBCentralManager!
    private var microbitPeripheral: CBPeripheral?
    private var uartCharacteristic: CBCharacteristic?
    
    private let uartServiceUUID = CBUUID(string: "6E400001-B5A3-F393-E0A9-E50E24DCCA9E")
    private let uartTXUUID = CBUUID(string: "6E400003-B5A3-F393-E0A9-E50E24DCCA9E")
    
    // MARK: - Initialization
    override init() {
        super.init()
        centralManager = CBCentralManager(delegate: self, queue: nil)
        requestNotificationPermission()
        loadSavedLog()
    }
    
    // MARK: - BLE scanning
    func startScanning() {
        microbitPeripheral = nil
        statusMessage = "🔍 Scanning for Micro:bit..."
        centralManager.stopScan()
        centralManager.scanForPeripherals(withServices: [uartServiceUUID])
    }
    
    // MARK: - CBCentralManagerDelegate Methods (Connection Handlers)
    func centralManagerDidUpdateState(_ central: CBCentralManager) {
        if central.state == .poweredOn {
            startScanning()
        } else {
            statusMessage = "⚠️ Bluetooth not available"
        }
    }
    
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
    
    func centralManager(_ central: CBCentralManager, didConnect peripheral: CBPeripheral) {
        statusMessage = "🔗 Connected to Micro:bit!"
        peripheral.discoverServices([uartServiceUUID])
    }
    
    // MARK: - CBPeripheralDelegate Methods (Data Handlers)
    func peripheral(_ peripheral: CBPeripheral, didDiscoverServices error: Error?) {
        guard let services = peripheral.services else { return }
        for service in services where service.uuid == uartServiceUUID {
            peripheral.discoverCharacteristics([uartTXUUID], for: service)
        }
    }
    
    func peripheral(_ peripheral: CBPeripheral,
                    didDiscoverCharacteristicsFor service: CBService,
                    error: Error?) {
        for characteristic in service.characteristics ?? [] where characteristic.uuid == uartTXUUID {
            uartCharacteristic = characteristic
            peripheral.setNotifyValue(true, for: characteristic)
            statusMessage = "📡 Receiving data..."
        }
    }
    
    // 🚨 CRITICAL DATA RECEIVE METHOD (Synchronization Fix)
    func peripheral(_ peripheral: CBPeripheral,
                    didUpdateValueFor characteristic: CBCharacteristic,
                    error: Error?) {
        guard let value = characteristic.value,
              let text = String(data: value, encoding: .utf8),
              let number = Int(text.trimmingCharacters(in: .whitespacesAndNewlines)) else { return }
        
        // 🚨 FIX 1: Move slow operations (Log Management/Saving) OFF the main thread.
        DispatchQueue.global(qos: .utility).async { [weak self] in
            guard let self = self else { return }
            
            // --- BACKGROUND THREAD WORK (Log Management and Disk I/O) ---
            let entry = GasEntry(level: number, time: Date())
            var currentLog = self.gasLog
            currentLog.append(entry)
            
            // Keep only last N entries
            if currentLog.count > self.maxLogEntries {
                currentLog.removeFirst(currentLog.count - self.maxLogEntries)
            }
            
            self.saveLog(log: currentLog)
            
            // --- BACK TO MAIN THREAD FOR UI UPDATE ---
            DispatchQueue.main.async {
                let newIsDanger = number > self.dangerThreshold
                
                // 🚨 FIX 2: Play the fast, synchronous system sound FIRST.
                if newIsDanger {
                    AudioServicesPlaySystemSound(1005)
                }
                
                // Update all UI state variables immediately afterward.
                self.gasLevel = number
                self.isDanger = newIsDanger
                self.statusMessage = newIsDanger ? "⚠️ DANGER DETECTED!" : "✅ Safe"
                
                // Update the published log array on the main thread (fast update)
                self.gasLog = currentLog
                
                // 🚨 FIX 3: Still notify the user via UNNotification (for background operation)
                if newIsDanger {
                    self.notifyDanger(level: number)
                }
            }
        }
    }
    
    // MARK: - Notifications
    private func requestNotificationPermission() {
        UNUserNotificationCenter.current().requestAuthorization(options: [.alert, .sound]) { granted, _ in
            print("Notifications allowed: \(granted)")
        }
    }
    
    private func notifyDanger(level: Int) {
        let content = UNMutableNotificationContent()
        content.title = "⚠️ Gas Alert!"
        content.body = "Gas level is \(level) ppm!"
        content.sound = .default
        
        let request = UNNotificationRequest(identifier: UUID().uuidString,
                                            content: content,
                                            trigger: nil)
        UNUserNotificationCenter.current().add(request)
    }
    
    // MARK: - Persistent log
    private func saveLog(log: [GasEntry]) {
        if let data = try? JSONEncoder().encode(log) {
            UserDefaults.standard.set(data, forKey: "gasLog")
        }
    }
    
    private func loadSavedLog() {
        if let data = UserDefaults.standard.data(forKey: "gasLog"),
           let saved = try? JSONDecoder().decode([GasEntry].self, from: data) {
            self.gasLog = saved
        }
    }
}
