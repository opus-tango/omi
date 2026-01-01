1. Rewrite button functionality
    - Add triple press detection (three in 900ms)
    - Add quadtruple press detection (four in 1200ms) that enables/disables the LEDs
    - Stop sending any button click notifications over bluetooth
    - Move power off command to long press
    - Make single, double, and triple press have no functionality for now
2. Disable audio transfer over bluetooth, both streaming and file transfer
3. Make audio ALWAYS record to the SD card
4. Add a file on the SD card that records button press events
    - Must record what kind of event (single, double, triple)
    - Must record file pointer offset (i.e. what chunk of audio was recorded during the button event)
5. Disable the bluetooth connection altogether (we don't need it)