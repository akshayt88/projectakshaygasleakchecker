# --- Import MicroPython module for Micro:bit ---
from microbit import *   # gives access to all pins, LED display, and timing functions

# --- CONSTANTS (values that don’t change) ---
DANGER_THRESHOLD = 300   # Adjust depending on your sensor sensitivity (0–1023)

# --- MAIN LOOP (runs forever) ---
while True:
    # 1️⃣ READ the gas sensor value from pin0 (analog signal)
    gas_level = pin0.read_analog()  # gets a number between 0 and 1023

    # 2️⃣ PRINT the reading (you can see it in the console when connected by USB)
    print("Gas level:", gas_level)

    # 3️⃣ CHECK if the reading is above the danger threshold
    if gas_level > DANGER_THRESHOLD:
        # If dangerous → turn ON buzzer and show alert
        pin1.write_digital(1)   # send power to buzzer (active buzzer)
        display.show("!")       # show "!" on LED screen
    else:
        # If safe → turn OFF buzzer and show dash
        pin1.write_digital(0)
        display.show("-")

    # 4️⃣ WAIT 2 seconds before next reading (adjust speed as needed)
    sleep(2000)
