# projectakshaygasleakchecker
padding()
        .onAppear {
            // 🔹 Update gasLevel every 2 seconds
            Timer.scheduledTimer(withTimeInterval: 2.0, repeats: true) { _ in
                self.gasLevel = Int.random(in: 100...500)
                
                if self.gasLevel > 300 {
                    // 🚨 Start continuous alarm
                    if alarmTimer == nil {
                        alarmTimer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
                            AudioServicesPlaySystemSound(1005) // SMS tone
                        }
                    }
                } else {
                    // ✅ Stop alarm if safe
                    alarmTimer?.invalidate()
                    alarmTimer = nil
