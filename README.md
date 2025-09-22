# projectakshaygasleakchecker
import SwiftUI

struct ContentView: View {
    // Current gas level
    @State private var gasLevel: Int = 150
    
    // Timer that updates gas levels
    let timer = Timer.publish(every: 2.0, on: .main, in: .common).autoconnect()
    
    // Status text
    private var statusText: String {
        if gasLevel > 300 {
            return "⚠️ DANGER: Gas Leak!"
        } else {
            return "✅ SAFE"
        }
    }
    
    // Status color
    private var statusColor: Color {
        return gasLevel > 300 ? .red : .green
    }
    
    var body: some View {
        VStack(spacing: 30) {
            
            // Title
            Text("Gas Leak Detector")
                .font(.largeTitle)
                .fontWeight(.bold)
            
            // Gas Level Display
            VStack {
                Text("Current Gas Level")
                    .font(.headline)
                Text("\(gasLevel)")
                    .font(.system(size: 60, weight: .bold, design: .monospaced))
                    .foregroundColor(.primary)
            }
            .padding()
            .background(Color.secondary.opacity(0.1))
            .cornerRadius(15)
            .shadow(radius: 5)
            
            // Status (SAFE or DANGER)
            Text(statusText)
                .font(.system(size: 30, weight: .heavy, design: .rounded))
                .foregroundColor(statusColor)
                .padding()
                .background(statusColor.opacity(0.1))
                .cornerRadius(10)
        }
        .padding()
        // Update gas level every 2 seconds
        .onReceive(timer) { _ in
            self.gasLevel = Int.random(in: 100...500) // simulate live readings
        }
    }
}

