Working Sample Files:

extensions.conf  http.conf  pjsip.conf  rtp.conf




Analysis of ari_app_with_rtp.js

    Purpose: This script is an ARI (Asterisk REST Interface) application that manages SIP calls, bridges them with an external RTP server, records audio for 10 seconds, and plays it back to the caller. It uses Node.js with libraries for ARI interaction, logging, and RTP handling.
    Key Components:
        Modules: Uses ari-client for ARI, util.promisify for promise-based operations, winston for logging, and a custom RTPServer module for RTP.
        Logger: Configured with Winston to output logs to console and a file (app.log), with a max size of 1MB.
        ARI Connection: Connects to Asterisk at http://127.0.0.1:8088 with credentials asterisk:asterisk and app name stasis_app.
        RTP: Uses port 12000 for RTP communication.
        Event Handling: Listens for StasisStart, StasisEnd, error, and close events to manage call lifecycle.
        Audio Flow: Answers calls, creates a bridge, adds an external media channel, records audio via RTP, and plays it back after 10 seconds.
        Error Handling: Catches uncaught exceptions and logs errors at various stages.
    Execution Flow:
        Connects to ARI and starts the stasis_app.
        On StasisStart:
            For SIP channels: Answers, creates a bridge, starts RTP recording, and adds an external media channel.
            For UnicastRTP channels: Adds them to the bridge with retry logic.
        After 10 seconds, stops recording and plays back the audio.
        On StasisEnd: Cleans up bridges and stops RTP if no calls remain.

List of Functions for README
Below is a summarized list of the key functions in ari_app_with_rtp.js, formatted for inclusion in a README:
markdown

# Function Summary for `ari_app_with_rtp.js`

- **`addExtToBridge(client, channel, bridgeId, retries = 5, delay = 500)`**  
  - **Description**: Adds an external media channel (`UnicastRTP`) to an existing bridge with retry logic if the initial attempt fails.
  - **Parameters**: 
    - `client`: ARI client instance.
    - `channel`: Channel object to add to the bridge.
    - `bridgeId`: ID of the bridge to join.
    - `retries`: Number of retry attempts (default: 5).
    - `delay`: Delay in milliseconds between retries (default: 500ms).
  - **Behavior**: Attempts to fetch the bridge, adds the channel, retries on failure up to `retries` times with `delay` ms between attempts, logs success or failure.

- **Main Anonymous Async Function `(async () => {...})()`**  
  - **Description**: Initializes and runs the ARI application, connecting to Asterisk, starting the `stasis_app`, and setting up event handlers.
  - **Behavior**: 
    - Connects to ARI server at `http://127.0.0.1:8088` with credentials `asterisk:asterisk`.
    - Starts the ARI application named `stasis_app`.
    - Handles call events (`StasisStart`, `StasisEnd`), manages bridges, records audio via RTP for 10 seconds, and plays it back.
    - Logs connection status, channel events, and errors.

- **`client.on('StasisStart', async (evt, channel) => {...})`**  
  - **Description**: Event handler triggered when a channel enters the `stasis_app` (call starts).
  - **Behavior**: 
    - For `UnicastRTP` channels: Adds them to the bridge with retry logic.
    - For SIP channels: Answers the call, creates a mixing bridge, starts RTP recording, and adds an external media channel; schedules playback after 10 seconds.
    - Logs channel start, answer, and RTP server actions.

- **`client.on('StasisEnd', async (evt, channel) => {...})`**  
  - **Description**: Event handler triggered when a channel leaves the `stasis_app` (call ends).
  - **Behavior**: 
    - Removes `UnicastRTP` channels from the map.
    - Destroys bridges for SIP channels, stops RTP server if no calls remain.
    - Logs channel end and bridge destruction.

- **`client.on('error', (err) => {...})`**  
  - **Description**: Event handler for ARI client errors.
  - **Behavior**: Logs the error message when an issue occurs with the ARI client.

- **`client.on('close', () => {...})`**  
  - **Description**: Event handler for WebSocket connection closure.
  - **Behavior**: Logs when the ARI WebSocket connection closes.

- **`process.on('uncaughtException', (err) => {...})`**  
  - **Description**: Global handler for uncaught exceptions in the Node.js process.
  - **Behavior**: Logs the error with its stack trace to prevent application crashes.









Analysis of rtpServer.js

    Purpose: This script implements an RTPServer class for recording incoming RTP audio (μ-law format) into a WAV file and playing it back over RTP. It’s designed to work with an ARI application (like ari_app_with_rtp.js) to process audio streams in real-time.
    Key Components:
        Modules: Uses dgram for UDP communication, fs for file operations, and wav for WAV file creation.
        Class Structure: Defines RTPServer with methods for audio conversion, RTP header creation, recording, and playback.
        Audio Handling: Converts μ-law (from RTP) to 16-bit PCM for WAV recording and back to μ-law for playback.
        Logging: Logs key events (start, stop, packet counts) to the console with timestamps.
        RTP: Listens on a specified port (default 12000), processes incoming packets, and sends playback packets every 20ms.
    Execution Flow:
        Initialization: Sets up a UDP socket, WAV writer, and initial state via the constructor.
        Start: Binds the socket, starts recording RTP packets to a WAV file, and logs packet reception.
        Stop: Closes the socket and WAV file, logging the total packets received.
        Playback: Reads the WAV file, converts it to μ-law, and sends it back as RTP packets to the last source.
    Key Features:
        Supports μ-law encoding/decoding for compatibility with Asterisk’s PCMU codec.
        Logs the first packet and every 100 packets (configurable via logInterval).
        Handles errors gracefully (e.g., short packets, file read failures).

List of Functions for README
Below is a summarized list of the key methods in rtpServer.js, formatted for inclusion in a README:
markdown

# Function Summary for `rtpServer.js`

- **`constructor(options = {})`**  
  - **Description**: Initializes an `RTPServer` instance with configurable options for RTP handling.
  - **Parameters**: 
    - `options.port`: Port number to listen on for RTP packets.
    - `options.outFile`: Output WAV file name (default: `'call_recording.wav'`).
    - `options.logInterval`: Number of packets between summary logs (default: `100`).
  - **Behavior**: Sets up a UDP socket, initializes state variables, and prepares for RTP audio processing.

- **`muLawToLinear(mu)`**  
  - **Description**: Converts a single μ-law sample to a 16-bit linear PCM value.
  - **Parameters**: 
    - `mu`: 8-bit μ-law sample value.
  - **Returns**: 16-bit PCM sample.
  - **Behavior**: Applies the μ-law decoding formula to expand the compressed sample into linear PCM.

- **`convertBuffer(muBuffer)`**  
  - **Description**: Converts a buffer of μ-law samples to a 16-bit PCM buffer.
  - **Parameters**: 
    - `muBuffer`: Buffer containing μ-law samples.
  - **Returns**: Buffer with PCM samples (2 bytes per sample).
  - **Behavior**: Iterates over μ-law samples, converts each to PCM, and writes to a new buffer.

- **`linearToMuLaw(sample)`**  
  - **Description**: Converts a single 16-bit linear PCM sample to a μ-law value.
  - **Parameters**: 
    - `sample`: 16-bit PCM sample value.
  - **Returns**: 8-bit μ-law value.
  - **Behavior**: Applies the μ-law encoding formula, compressing the PCM sample into μ-law format.

- **`convertPCMBufferToMuLaw(pcmBuffer)`**  
  - **Description**: Converts a buffer of 16-bit PCM samples to μ-law samples.
  - **Parameters**: 
    - `pcmBuffer`: Buffer containing PCM samples (2 bytes per sample).
  - **Returns**: Buffer with μ-law samples (1 byte per sample).
  - **Behavior**: Reads PCM samples, converts each to μ-law, and stores in a new buffer.

- **`buildRTPHeader(seq, timestamp, ssrc)`**  
  - **Description**: Builds a 12-byte RTP header for playback packets.
  - **Parameters**: 
    - `seq`: 16-bit sequence number.
    - `timestamp`: 32-bit timestamp.
    - `ssrc`: 32-bit synchronization source identifier.
  - **Returns**: Buffer containing the RTP header.
  - **Behavior**: Constructs a Version 2 RTP header with payload type 0 (PCMU).

- **`start()`**  
  - **Description**: Starts the RTP server, listening for incoming packets and recording to a WAV file.
  - **Behavior**: Initializes a WAV writer, binds the UDP socket, logs packet reception (first and every 100 packets), converts μ-law to PCM, and writes to the file.

- **`stop()`**  
  - **Description**: Stops the RTP server and finalizes the WAV file.
  - **Behavior**: Logs total packets received, closes the UDP socket, ends the WAV writer, and resets the running state.

- **`playback()`**  
  - **Description**: Plays back the recorded WAV file over RTP to the last source.
  - **Behavior**: Reads the WAV file, converts PCM to μ-law, builds RTP packets, and sends them every 20ms to the last RTP source; logs playback start and completion.
