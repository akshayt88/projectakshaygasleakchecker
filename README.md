import SwiftUI
import CoreBluetooth

// The main screen for your gas detector app
struct ContentView: View {
    
    // Connects the UI to the Bluetooth manager class (handles data)
    @StateObject private var bluetoothManager = BluetoothManager()
    
    var body: some View {
        VStack(spacing: 30) {
            
            // 🔹 HEADER
            Text("🔥 Gas Leak Detector")
                .font(.largeTitle)
                .bold()
            
            // 🔹 CURRENT GAS LEVEL
            VStack {
                Text("Current Gas Level")
                    .font(.headline)
                Text("\(bluetoothManager.gasLevel)")  // shows latest reading
                    .font(.system(size: 60, weight: .bold, design: .monospaced))
                    .foregroundColor(.primary)
            }
            .padding()
            .background(Color.secondary.opacity(0.1))
            .cornerRadius(15)
            .shadow(radius: 5)
            
            // 🔹 STATUS MESSAGE (danger or safe)
            Text(bluetoothManager.statusMessage)
                .font(.system(size: 28, weight: .semibold))
                .foregroundColor(bluetoothManager.isDanger ? .red : .green)
                .padding()
                .background(bluetoothManager.isDanger ? Color.red.opacity(0.1) : Color.green.opacity(0.1))
                .cornerRadius(10)
            
            // 🔹 RECONNECT BUTTON
            Button("Reconnect") {
                bluetoothManager.startScanning()  // try to find Micro:bit again
            }
            .buttonStyle(.borderedProminent)
            
            Spacer()
        }
        .padding()
        .onAppear {
            // When app opens, start scanning automatically
            bluetoothManager.startScanning()
        }
    }
}

// Preview for Xcode Canvas
#Preview {
    ContentView()
}

