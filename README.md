 if history.isEmpty {
                    Text("No history recorded yet. Readings are saved every 5 minutes.")
                        .foregroundColor(.secondary)
                        .padding(.horizontal)
                } else {
                    // Use LazyVStack for efficiency when embedding a list in a ScrollView
                    LazyVStack(spacing: 0) {
                        ForEach(history.reversed(), id: \.self) { reading in
                            HStack {
                                Image(systemName: reading > dangerThreshold ?
                                      "exclamationmark.triangle.fill" :
                                      "checkmark.circle.fill")
                                    .foregroundColor(reading > dangerThreshold ? .red : .green)
                                
                                Text("Gas Level: \(reading)")
                                    .foregroundColor(reading > dangerThreshold ? .red : .primary)
                                    .fontWeight(reading > dangerThreshold ? .bold : .regular)
                                
                                Spacer()
                            }
                            .padding(.vertical, 8)
                            .padding(.horizontal)
                            
                            // Optional divider between items
                            Divider()
                        }
                    }
                    .background(Color(.systemGray6)) // Light background for the list area
                    .cornerRadius(10)
                    .padding(.horizontal)
                }
            }
            .padding()
