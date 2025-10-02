   // No NavigationView needed if you remove the NavigationLink, but keeping it
        // to hold the .navigationTitle and .onAppear modifiers
        .navigationTitle("Detector")
        .onAppear {
            // 🔹 Timer for continuous reading updates (every 2 seconds)
            Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { _ in
                
                let newGasLevel = Int.random(in: 100...500)
                
                // SAFE SYNCHRONIZATION: Audio trigger before State update
                if newGasLevel > 300 {
                    AudioServicesPlaySystemSound(1005)
                }
                
                // Update the @State property immediately after sound trigger
                self.gasLevel = newGasLevel
            }
            
            // 🔹 Timer for saving history every 5 minutes (300.0 seconds)
            Timer.scheduledTimer(withTimeInterval: 300.0, repeats: true) { _ in
                self.history.append(self.gasLevel)
            }
        }
    }
}
