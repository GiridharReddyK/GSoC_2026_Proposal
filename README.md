# <center>**Google Summer of Code 2026**</center>

## <center>GNU Radio LoRa Satellite Ground Station Receiver with Optional TinyGS and LoRaWAN Integration</center>

### **Personal Details**

- **Name:** Karnati Giridhar Reddy
- **University**: Chaitanya Bharathi Institute of Technology (CBIT), Hyderabad
- **Degree**: B.E. in Electronics and Communication Engineering
- **Email:**
  - **Personal**: <kgiridharreddy2022@gmail.com>
  - **University**: <ugs22040_ece.giridhar@cbit.org.in>
- **Linkedin**: <http://linkedin.com/in/kgiridharreddy>
- **Github**: <https://github.com/GiridharReddyK>
- **Gitlab**: <https://gitlab.com/kgiridharreddy>
- **HAM Radio Profile**: <https://qrz.com/db/VU2TIO>
- **Country of Residence**: India
- **Timezone**: IST (GMT + 0530)
- **Primary Languages**: English; Hindi and Telugu (Native)

I am an undergraduate student pursuing a B.E. in Electronics and Communication Engineering at Chaitanya Bharathi Institute of Technology (CBIT), Hyderabad. My primary interests lie in Software Defined Radio (SDR) operations, open-source ground station technology, and satellite communications. I bring direct, hands-on experience with real-world satellite ground station deployment and operations:

- **TinyGS Operator**: I was the first TinyGS ground station operator in my state (Telangana), establishing a fully operational LoRa satellite reception station using ESP32 + SX127x hardware. This gave me good familiarity with the TinyGS MQTT architecture, satellite pass scheduling, and LoRa packet reception from LEO CubeSats including FossaSat and Norby.
- **VayuVani Ground Station**: I served as the primary contributor to the VayuVani satellite ground station project, a comprehensive open-source ground station designed for LoRa satellite and amateur radio satellite data reception using LoRa receivers.
- **SatNOGS Operator**: I was the 2nd SatNOGS ground station operator in my region, contributing observation data to the global SatNOGS network and gaining experience with automated satellite tracking, Doppler correction, and data submission pipelines.
- **Project Experience**: Through these deployments, I have built experience in GNU Radio flowgraph design, SDR hardware configuration (RTL-SDR, HackRF), antenna systems (QFH, turnstile, Yagi), Linux system administration, and Python-based automation for satellite operations.

If selected, I will dedicate 30+ hours per week to this project during the coding period. My academic semester ends in mid-May, providing uninterrupted time for GSoC work. I am deeply motivated by the opportunity to bridge the gap between SDR-based satellite reception and the LoRa satellite ecosystem, directly serving the open-source space community.

### **Introduction**

This project proposes the creation of an SDR-based ground station receiver using GNU Radio that is capable of receiving and demodulating LoRa satellite signals with standard, affordable SDR hardware. The receiver will allow conventional SDR ground stations to decode LoRa packets transmitted by Low Earth Orbit (LEO) CubeSats and optionally forward them to networks such as TinyGS or LoRaWAN infrastructure. The work will build upon existing GNU Radio LoRa implementations and produce reusable flowgraphs, optional integration modules, and comprehensive documentation.

A critical gap exists in the current open-source satellite ground station ecosystem. SatNOGS, the largest open-source ground station network with 400+ stations, deliberately refuses to support LoRa modulation citing Semtech's patent concerns. TinyGS, the primary LoRa satellite reception network with 430+ ground stations, is architecturally locked to hardware Semtech LoRa transceiver chips (SX127x/SX126x) on ESP32 microcontrollers, offering no flexibility for SDR-based reception. gr-satellites, the leading GNU Radio satellite decoder toolkit by Daniel Estevez, supports over 100 satellite decoders but does not include LoRa demodulation. No integrated open-source tool currently combines satellite tracking, Doppler compensation, and LoRa demodulation into a single SDR-based solution.

This project targets LibreCube Initiative's GSoC 2026 Project #5: "GNU Radio LoRa Receiver with Optional TinyGS / LoRaWAN Integration", listed as a 350-hour Hard-difficulty project mentored by Shayan (shayan@librecube.org), requiring C++, GNU Radio, and signal processing skills. By building on the proven gr-lora_sdr module by Joachim Tapparel (EPFL), which provides production-ready LoRa PHY layer implementation with performance within 1 dB of theoretical AWGN limits, this project will create a complete satellite reception pipeline from RF input to decoded data output with optional network forwarding.

### **Background Theory**

#### 1. LoRa Modulation: Chirp Spread Spectrum (CSS)

LoRa (Long Range) employs Chirp Spread Spectrum (CSS) modulation, where each symbol is a linear frequency chirp that sweeps across the entire allocated bandwidth. Data is encoded through cyclic time-frequency shifting of a base up-chirp: for a symbol value m (where m ranges from 0 to 2^SF - 1), the chirp's starting frequency is offset by m × BW / 2^SF. When the instantaneous frequency reaches +BW/2, it wraps cyclically to -BW/2. Each symbol therefore encodes SF bits of information since there are 2^SF possible starting positions.

The key configurable parameters of LoRa modulation are:

- **Spreading Factor (SF7–SF12)**: Controls the trade-off between data rate and sensitivity. At BW = 125 kHz with CR 4/5, data rates range from 5,469 bps (SF7) down to 293 bps (SF12). The SX1276 achieves sensitivities from -123 dBm (SF7) to -137 dBm (SF12), with each SF step improving link budget by approximately 2.5 dB. The SNR demodulation floor ranges from -7.5 dB (SF7) to -20 dB (SF12), enabling LoRa to operate well below the noise floor.
- **Bandwidth (BW)**: Configurable at 125 kHz, 250 kHz, or 500 kHz. Wider bandwidth increases data rate but reduces sensitivity. Most satellites use 125 kHz or 250 kHz.
- **Coding Rate (CR 4/5 to 4/8)**: Hamming-based forward error correction. CR 4/5 adds 25% overhead and detects 1 error; CR 4/8 adds 100% overhead and corrects 1 error with parity. The header is always encoded at CR 4/8 regardless of payload setting.
- **Sync Word**: 2 up-chirps encoding a network identifier. 0x34 for LoRaWAN public networks, 0x12 for private networks. Satellite missions typically use custom sync words.

A complete LoRa frame consists of: a preamble (configurable number of unmodulated up-chirps, typically 8 for LoRaWAN, plus 4.25 additional symbols), a sync word (2 symbols), a Start Frame Delimiter (2.25 down-chirps), an optional explicit header (8 symbols at CR 4/8 containing payload length, coding rate, and CRC flag), the payload (up to 255 bytes), and an optional 16-bit CRC using the CCITT polynomial.

#### 2. FFT-Based LoRa Demodulation Pipeline

LoRa demodulation proceeds through six well-defined stages:

1. **Dechirping**: Multiply the received signal by the conjugate base down-chirp, converting the CSS modulation into M-ary FSK where each symbol produces a single tone at a frequency proportional to the encoded symbol value.
2. **FFT Peak Detection**: Apply an N = 2^SF point FFT to each dechirped symbol period. The bin index of the maximum magnitude peak yields the estimated symbol value. This is the core demodulation step and operates efficiently in the frequency domain.
3. **Synchronization**: Use preamble chirps for coarse timing acquisition and the SFD down-chirps for fine Symbol Timing Offset (STO) and Carrier Frequency Offset (CFO) estimation. The fractional FFT bin offset directly provides CFO information.
4. **Gray Decoding**: Reverse the Gray mapping applied during modulation to recover the original bit ordering.
5. **De-interleaving**: Reverse the diagonal interleaver with block size (4/CR) × SF, which was applied during transmission to distribute burst errors across multiple codewords.
6. **Hamming Decoding and De-whitening**: Apply Hamming error correction followed by XOR de-whitening using a pseudo-random sequence generated by a 9-bit LFSR with polynomial x⁹ + x⁵ + 1.

#### 3. GNU Radio LoRa Implementations: Comparative Analysis

Three major open-source GNU Radio LoRa out-of-tree (OOT) modules exist, but only one remains viable for satellite reception work:

| Feature | gr-lora_sdr (EPFL) | gr-lora (rpp0) | gr-lora (Bastille) |
|---|---|---|---|
| **Status** | Active (2024+) | Unmaintained (~2017) | Unmaintained (~2017) |
| **GR Version** | 3.10 | 3.9 / 3.7 | 3.7 |
| **TX + RX** | Both | RX only | Both |
| **SF Range** | SF5 – SF12 | SF7 – SF12 | SF6 – SF12 |
| **CRC** | Payload + Header | None | None |
| **Implicit Header** | Yes | Partial | No |
| **Soft Decoding** | Yes | No | No |
| **Low-SNR** | Excellent (1 dB AWGN) | Poor (≥20 dB) | Unknown |
| **Satellite Use** | Plan-S, IAF paper | None | None |
| **Stars (GitHub)** | 927+ | 612 | Archived |

gr-lora_sdr (by Joachim Tapparel, EPFL Telecommunication Circuits Laboratory) is the definitive implementation and will serve as the PHY layer foundation for this project. It provides a complete transceiver chain with FFT-based dechirping, STO/CFO estimation and correction, Gray mapping, interleaving, Hamming coding, and whitening. Frame error rate performance sits within 1 dB of simulated AWGN curves in USRP experimental measurements. Critically, it has already been proven in satellite work: Plan-S (Turkish satellite company) built an Adaptive Data Rate LoRa modem for their Connecta LEO constellation on top of it, and an IAF 2023 paper demonstrated satellite communication using gr-lora_sdr with HackRF. A companion project, gr-lorawan, adds LoRaWAN MAC layer support.

#### 4. SDR Hardware for Satellite LoRa Reception

The choice of SDR hardware depends on frequency coverage, noise figure, and budget. An external Low-Noise Amplifier (LNA) is essential for satellite work with any SDR. By the Friis formula, a front-end LNA with NF < 1 dB and 20 dB gain dominates the system noise figure, making even an 8-bit RTL-SDR approach the SX1276's sensitivity. Most LoRa satellites operate at 433–437 MHz (FossaSat, Norby, FEES, SDSat), with some at 868/915 MHz.

| SDR Device | Price | ADC | NF (UHF) | Freq. Range | Notes |
|---|---|---|---|---|---|
| RTL-SDR V3/V4 | $30–40 | 8-bit | ~3.5 dB | 500 kHz–1.7 GHz | Best cost-effective; RX only |
| Airspy Mini/R2 | $99–169 | 12-bit | ~3.5 dB | 24–1700 MHz | Best sensitivity/dollar |
| PlutoSDR | $150–200 | 12-bit | ~3–5 dB | 325 MHz–3.8 GHz | No 433 MHz without hack |
| HackRF One | $300 | 8-bit | ~10–12 dB | 1 MHz–6 GHz | Poor NF; needs LNA |
| USRP B200 | $1100+ | 12-bit | ~5 dB | 70 MHz–6 GHz | Research-grade; full-duplex |

The recommended minimum ground station setup costs under $50: RTL-SDR V4 + SPF5189Z LNA + band-pass filter + QFH or turnstile antenna + Linux PC with GNU Radio 3.10 + gr-lora_sdr. This project will be developed and tested primarily with RTL-SDR V3/V4 hardware to ensure accessibility for the widest possible user base.

#### 5. Doppler Shift Considerations for LEO Satellites

LEO satellites at 550 km altitude produce maximum Doppler shifts of approximately ±10 kHz at 433 MHz and ±20 kHz at 868 MHz, with peak Doppler rates of ~245 Hz/s at mid-pass (zenith). Static Doppler is well-tolerated (at 433 MHz with BW 125 kHz, ±10 kHz represents only 8% of bandwidth). However, dynamic Doppler is the critical challenge. A 2026 empirical study measured CFO tolerance: SF7 tolerates >20 kHz (robust), SF10 tolerates ~5.5 kHz (marginal), and SF12 tolerates only ~0.5 kHz. NORBY CubeSat flight tests confirmed LoRa is immune to Doppler for SF ≤ 11 and BW > 31.25 kHz.

For this project, the Doppler correction approach combines: (1) TLE-based prediction for coarse correction using python-sgp4 or Skyfield, feeding the gr-satellites Doppler Correction block; (2) GPredict real-time tracking via the Hamlib rigctld protocol (TCP 4532) to gr-gpredict-doppler; and (3) preamble-based fine estimation using the fractional FFT bin offset within gr-lora_sdr's synchronization block for residual correction.

#### 6. TinyGS Network Architecture and Integration Challenge

TinyGS is a globally distributed open network of 430+ ground stations receiving LoRa satellite telemetry using ESP32 + Semtech SX127x/SX126x hardware. The architecture uses MQTT: ground stations connect to a centralized MQTT broker, publish received packets as JSON payloads (containing raw hex data, RSSI, SNR, frequency, SF, BW, CR, CRC status, and timestamp), and the TinyGS server aggregates and displays data at tinygs.com. An autotune system calculates satellite passes from TLE data and commands stations to retune before each pass.

Integrating a custom SDR receiver with TinyGS is not officially supported. There is no public API for submitting packets from non-TinyGS firmware. MQTT credentials are obtained through the Telegram bot @tinygs_personal_bot, and the protocol is tightly coupled to the ESP32 firmware. Three approaches are viable: (1) reverse-engineer the MQTT topic structure and JSON payload schema from the GPL-licensed firmware source (MQTT_Client.cpp); (2) connect SDR output to an ESP32 running TinyGS firmware via serial as an MQTT bridge; (3) fork the firmware to accept external data input. This integration gap is a primary motivation for this project.

#### 7. LoRaWAN Forwarding via Semtech UDP Packet Forwarder

Decoded LoRa packets can be forwarded to LoRaWAN network servers using the Semtech UDP Packet Forwarder Protocol (version 2, port 1700). The upstream JSON format embeds an rxpk array containing: time (ISO 8601), tmst (32-bit counter), freq (MHz), stat (CRC status), modu ('LORA'), datr ('SF7BW125' format), codr ('4/5'), rssi (dBm), lsnr (SNR), size (bytes), and data (base64 payload). A Python bridge script can construct these UDP datagrams from gr-lora_sdr output and send them to ChirpStack Gateway Bridge (listening on UDP 1700) or The Things Network routers.

Key challenges include: single-channel limitation (adequate for satellite work), no downlink capability with RX-only SDRs, and the fundamental difference that satellite LoRa packets typically use raw LoRa framing rather than LoRaWAN MAC headers (lacking DevAddr, FCnt, MIC). The forwarding module will handle both raw LoRa and LoRaWAN-framed packets appropriately.

### **Proposed Workflow**

The project follows a structured four-phase approach, building complexity incrementally from core reception to network integration. Each phase produces a testable, standalone deliverable.

**Phase 1: Core LoRa Satellite Receiver (Weeks 1–4)**

1. Set up GNU Radio 3.10 development environment with gr-lora_sdr, gr-osmosdr/SoapySDR source blocks for RTL-SDR/Airspy/USRP support, and verify end-to-end LoRa decoding from a local LoRa transmitter (e.g., ESP32 + SX1276 test beacon).
2. Build the core satellite receiver flowgraph: SDR Source → Band-pass Filter → Doppler Correction (Frequency Xlating FIR Filter) → gr-lora_sdr Receiver → Decoded Packet Output (ZMQ/UDP/file sink). Configure for common satellite parameters (433.0–437.0 MHz, SF7–SF12, BW 125/250 kHz, various CRs).
3. Integrate TLE-based Doppler prediction using python-sgp4 or Skyfield. Build a Python module that computes the Doppler frequency curve for a given satellite pass and feeds it to the GNU Radio flowgraph via either a pre-computed frequency file or the gr-satellites Doppler Correction block. Validate with recorded IQ data from known satellites.
4. Implement GPredict integration using the gr-gpredict-doppler block via Hamlib rigctld protocol for real-time Doppler tracking during live satellite passes.
5. Create parameterized GRC flowgraph templates supporting one-click configuration for common LoRa satellite missions (FossaSat, Norby, FEES, SDSat, SATLLA-2B).

**Phase 2: TinyGS Integration Module (Weeks 5–7)**

1. Reverse-engineer the TinyGS MQTT protocol by analyzing the open-source firmware (MQTT_Client.cpp, Radio.cpp). Document the complete topic structure, JSON payload schema, station authentication flow, and satellite configuration commands.
2. Develop a Python MQTT bridge (tinygs_bridge.py) that receives decoded packets from gr-lora_sdr (via ZMQ PUB/SUB), reformats them into TinyGS-compatible JSON, and publishes to the TinyGS MQTT broker. Include RSSI/SNR estimation from SDR measurements, proper timestamp formatting, and station identification.
3. Implement an alternative ESP32 serial bridge approach: the SDR receiver sends decoded packet data over serial/USB to an ESP32 running stock TinyGS firmware, which handles MQTT authentication and submission.
4. Test end-to-end TinyGS integration with live satellite passes, verifying packet submission and visibility on tinygs.com dashboard.

**Phase 3: LoRaWAN Virtual Gateway Module (Weeks 8–9)**

1. Implement the Semtech UDP Packet Forwarder Protocol (v2) in Python. Build a virtual gateway (lorawan_gateway.py) that constructs PUSH_DATA UDP datagrams from gr-lora_sdr decoded output and sends them to a configurable LoRaWAN network server endpoint.
2. Integrate with ChirpStack Gateway Bridge, including proper gateway EUI registration, PUSH_DATA/PUSH_ACK handling, and PULL_DATA keepalive management. Provide Docker Compose configuration for a self-hosted ChirpStack stack.
3. Handle the raw LoRa vs. LoRaWAN framing distinction: detect packets with LoRaWAN MAC headers (DevAddr, FCnt, MIC) and forward them natively; for raw LoRa satellite packets, provide configurable wrapping or direct raw output.
4. Write test scripts that validate forwarding with both local LoRaWAN devices and simulated satellite packets.

**Phase 4: Documentation, Testing, and Polish (Weeks 10–12)**

1. Create comprehensive user documentation: installation guide (Ubuntu, Fedora, macOS via conda), quick-start tutorial, hardware setup guide with photos for RTL-SDR + LNA + antenna, and a satellite mission database with pre-configured parameters.
2. Write developer documentation: architecture overview, API reference for integration modules, flowgraph design guide, and contribution guidelines following LibreCube standards.
3. Build an automated test suite: unit tests for packet parsing/formatting, integration tests using recorded IQ files from real satellite passes, and regression tests for Doppler correction accuracy across SF7–SF12.
4. Create a command-line interface (CLI) tool (lora-groundstation) that wraps the full pipeline: satellite selection → pass prediction → automated reception → packet decoding → optional network forwarding. Supports headless/unattended operation.
5. Performance benchmarking: measure packet error rates across different SDR hardware, spreading factors, and Doppler conditions. Compare with TinyGS hardware receiver performance on the same satellite passes.
6. Final code review, merge request preparation, and blog post documenting the project.

### **Technical Details**

#### 1. Core Architecture

The system architecture follows a modular pipeline design with four distinct layers:

- **RF Front-End Layer**: SDR Source (gr-osmosdr or SoapySDR) → AGC → Band-pass Filter (Frequency Xlating FIR Filter centered on satellite downlink). Supports RTL-SDR, Airspy, PlutoSDR, USRP, HackRF, and LimeSDR via unified source block abstraction.
- **Doppler Correction Layer**: Accepts frequency correction input from three configurable sources: (a) pre-computed TLE-based Doppler file via File Source → Rotator block, (b) real-time GPredict tracking via gr-gpredict-doppler → Frequency Xlating FIR Filter, or (c) bypass mode for low-SF passes where Doppler is negligible.
- **Demodulation Layer**: gr-lora_sdr receiver chain: Frame Sync → Dechirp → FFT Demod → Gray Decode → Deinterleave → Hamming Decode → Dewhiten → CRC Check → Decoded Payload Output. Configurable SF, BW, CR, sync word, and implicit/explicit header.
- **Integration Layer**: Decoded packets are published via ZMQ PUB socket. Three optional subscriber modules: (a) tinygs_bridge.py for TinyGS MQTT submission, (b) lorawan_gateway.py for Semtech UDP forwarding to ChirpStack/TTN, (c) file_logger.py for local storage in SatNOGS-compatible format. Modules are independently configurable and can run simultaneously.

#### 2. Key Implementation Decisions

- **ZMQ for Inter-Process Communication**: Decoded packets are published on a ZMQ PUB socket (tcp://localhost:5555) as JSON messages containing: raw_payload (hex), rssi, snr, frequency, sf, bw, cr, crc_ok, timestamp_utc, satellite_name. This allows any number of subscriber modules to process packets independently.
- **Satellite Configuration Database**: A YAML-based satellite database (satellites.yml) maps satellite NORAD IDs to LoRa parameters (frequency, SF, BW, CR, sync word, implicit header mode). Pre-populated for 40+ active TinyGS-tracked satellites.
- **GNU Radio 3.10 Target**: The project targets GNU Radio 3.10.x (current stable: 3.10.12.0) for maximum compatibility. A compatibility assessment for GNU Radio 4.0 will be documented but porting to GR4 is out of scope.
- **Python + C++ Hybrid**: Core signal processing (gr-lora_sdr blocks) remains in C++ for performance. Integration modules, CLI tool, and Doppler computation are in Python for rapid development and user accessibility.

#### 3. Doppler Correction Implementation

**Coarse Correction (TLE-based)**: Before each satellite pass, the system computes the Doppler frequency shift curve using the SGP4 orbital propagator (python-sgp4) with current TLE data fetched from CelesTrak or Space-Track. The curve is sampled at 1 Hz resolution and written to a frequency correction file. During reception, a File Source block reads this file and drives a Rotator block for phase-continuous frequency shifting. This removes the bulk Doppler (typically 95%+ of the total shift) before the signal reaches the LoRa demodulator.

**Fine Correction (Preamble-based)**: Residual frequency offset after coarse correction (typically <1 kHz) is handled by gr-lora_sdr's built-in synchronization block, which uses the preamble chirps for CFO estimation. The fractional FFT bin offset of the preamble peak directly provides the residual carrier frequency error, which is then compensated in the dechirping stage.

#### 4. TinyGS MQTT Bridge: Protocol Details

The TinyGS MQTT bridge module reverse-engineers the station-to-server communication protocol from the open-source firmware. Key elements include: MQTT topic structure (tinygs/[username]/[station_id]/cmnd/frame/0 for packet submission), JSON payload fields (data, RSSI, SNR, frequency, sf, bw, cr, crc, timestamp, satellite NORAD ID), authentication via Telegram bot credentials, station heartbeat messages for keep-alive, and satellite pass scheduling via autotune subscription commands.

#### 5. LoRaWAN Virtual Gateway: Protocol Details

The virtual gateway implements the Semtech UDP Packet Forwarder Protocol v2 on port 1700. Each received packet generates a PUSH_DATA message (protocol version 0x02, random token, gateway EUI, JSON rxpk payload). The module maintains a PULL_DATA keepalive every 10 seconds. The gateway EUI is derived from the host machine's MAC address by default but is configurable. CRC status mapping: gr-lora_sdr CRC pass → stat=1, fail → stat=-1, no CRC → stat=0.

### **Timeline**

#### **Community Bonding Period (May 8 – June 1):**

- Introduce myself and this project on the LibreCube Forum, LibreCube IRC/Matrix channel, and the GNU Radio mailing list.
- Establish regular communication schedule with mentor Shayan (weekly video calls + async updates via Matrix/email).
- Set up complete development environment: Ubuntu 24.04, GNU Radio 3.10.12, gr-lora_sdr (conda install), RTL-SDR V3 + SPF5189Z LNA + QFH antenna at home station.
- Study gr-lora_sdr source code in depth (C++ blocks: frame_sync_impl, fft_demod_impl, deinterleaver_impl, hamming_dec_impl, header_decoder_impl).
- Set up TinyGS test station (ESP32 + SX1276) alongside SDR receiver for parallel comparison testing during satellite passes.
- Create project tracking board (GitLab Issues or Taiga) with weekly milestones.

#### **Official Coding Period**

| Period | Weeks | Activities | Deliverable |
|---|---|---|---|
| Coding Phase 1 | Week 1 (Jun 2–8) | Build core SDR → LoRa receiver flowgraph. Test with local ESP32 LoRa beacon at 433 MHz. | Working flowgraph decoding local LoRa |
| | Week 2 (Jun 9–15) | Implement TLE-based Doppler correction module using python-sgp4. | Doppler correction module v1 |
| | Week 3 (Jun 16–22) | Integrate GPredict real-time Doppler tracking. Test with recorded IQ from known satellites. | GPredict integration working |
| | Week 4 (Jun 23–29) | Create satellite config database (YAML). Build parameterized GRC templates. First live satellite reception test. | Satellite config DB + live reception demo |
| Midterm Eval | Week 5 (Jun 30–Jul 6) | Midterm evaluation: Working satellite LoRa receiver, Doppler correction (TLE + GPredict), Parameterized flowgraph templates. | Midterm: Core receiver complete |
| Coding Phase 2 | Week 6 (Jul 7–13) | Reverse-engineer TinyGS MQTT protocol. Develop Python MQTT bridge module. | MQTT protocol documentation |
| | Week 7 (Jul 14–20) | Complete TinyGS bridge; test live. Implement ESP32 serial bridge fallback. | TinyGS integration working |
| | Week 8 (Jul 21–27) | Implement Semtech UDP Packet Forwarder. Integrate with ChirpStack Gateway Bridge. | LoRaWAN virtual gateway module |
| | Week 9 (Jul 28–Aug 3) | Test LoRaWAN forwarding end-to-end. Handle raw LoRa vs LoRaWAN framing. | LoRaWAN integration complete |
| Coding Phase 3 | Week 10 (Aug 4–10) | Build CLI tool (lora-groundstation). Automated test suite with IQ recordings. | CLI tool + test suite |
| | Week 11 (Aug 11–17) | Write comprehensive documentation. Performance benchmarking vs TinyGS HW. | Full documentation package |
| | Week 12 (Aug 18–25) | Final code review and cleanup. Blog post. Merge request preparation. Buffer time. | Final submission: All deliverables |

### **Deliverables**

1. **Core GNU Radio LoRa Satellite Receiver**: A complete, parameterized GNU Radio flowgraph that receives LoRa signals from LEO satellites using standard SDR hardware (RTL-SDR, Airspy, USRP, HackRF, PlutoSDR, LimeSDR) with integrated Doppler correction (TLE-based + GPredict real-time) and configurable LoRa parameters (SF5–SF12, BW 125/250/500, all CRs, implicit/explicit header, CRC).

2. **TinyGS Integration Module**: A Python-based MQTT bridge that forwards decoded LoRa packets to the TinyGS network, including protocol documentation, authentication handling, and an alternative ESP32 serial bridge for plug-and-play compatibility.

3. **LoRaWAN Virtual Gateway Module**: A Semtech UDP Packet Forwarder implementation enabling decoded packets to be forwarded to any LoRaWAN network server (ChirpStack, TTN), with Docker Compose deployment configuration.

4. **Command-Line Interface (CLI) Tool**: A user-friendly lora-groundstation CLI supporting satellite selection, automated pass scheduling, headless reception, and multi-output forwarding.

5. **Satellite Configuration Database**: A YAML-based database of 40+ active LoRa satellite missions with pre-configured reception parameters, regularly updatable from TLE sources.

6. **Comprehensive Documentation**: Installation guides for multiple platforms, quick-start tutorials, hardware setup guides with photos, developer API reference, architecture diagrams, and a project blog post.

7. **Automated Test Suite**: Unit tests, integration tests using recorded IQ files, Doppler correction validation tests, and performance benchmarks comparing SDR receiver against TinyGS hardware.

### **Personal Inspiration for the Project**

My journey into satellite ground station technology began with building my first TinyGS station, which made me the first LoRa satellite ground station operator in Telangana. The excitement of receiving a FossaSat telemetry packet for the first time, decoded on a $15 ESP32 board with a simple wire antenna, was transformative. But it also revealed a fundamental limitation: TinyGS is locked to hardware LoRa chips. When I wanted to experiment with different demodulation parameters, receive on non-standard frequencies, or decode packets that the SX1276 was marginally missing, I had no options.

This frustration deepened when I became a SatNOGS operator and discovered that the largest open-source ground station network deliberately excludes LoRa support. My work as the primary contributor to the VayuVani ground station project gave me extensive experience with GNU Radio flowgraph design, SDR hardware integration, and automated satellite reception pipelines, and I realized that the tools to bridge this gap already exist individually but have never been assembled into an integrated solution.

This project represents the convergence of my direct operational experience with TinyGS, SatNOGS, and VayuVani; my technical skills in SDR operations and GNU Radio development; and my deep motivation to democratize LoRa satellite reception beyond the constraints of fixed hardware. An SDR-based LoRa satellite ground station would give operators the flexibility that hardware receivers cannot provide: simultaneous multi-SF decoding, custom Doppler compensation algorithms, wideband recording and offline processing, frequency agility across bands, and research-grade signal analysis. I am committed to building this tool not just as a GSoC project, but as an enduring contribution to the open-source space community that I am proud to be part of.

### **References**

1. [gr-lora_sdr](https://github.com/tapparelj/gr-lora_sdr)
2. [gr-lorawan](https://github.com/tapparelj/gr-lorawan)
3. [TinyGS](https://tinygs.com/)
4. [LibreCube GSoC 2026 Projects](https://librecube.org/)
5. [gr-satellites (Daniel Estevez)](https://github.com/daniestevez/gr-satellites)
6. [gr-gpredict-doppler](https://github.com/ghostop14/gr-gpredict-doppler)
7. [ChirpStack LoRaWAN Server](https://chirpstack.io/)
8. [Semtech UDP Packet Forwarder Protocol](https://github.com/Lora-net/packet_forwarder)
9. Tapparel et al., "An Open-Source LoRa Physical Layer Prototype on GNU Radio," IEEE SPAWC 2020
10. Tapparel, "Design and Implementation of LoRa Physical Layer in GNU Radio," GRCon 2024
11. Farhat et al., "Doppler Estimation and Compensation Techniques in LoRa Direct-to-Satellite Communications," arXiv 2506.20858, June 2025
12. MDPI Electronics, "LoRa-to-LEO Satellite: A Review with Performance Analysis," Dec 2025
13. Robyns et al., "Multi-channel LoRa decoding," IoTBDS 2018
14. Conway, "Building a Full Transceiver Stack with GR-lora for Meshtastic Networks," GRCon 2024
15. [SatNOGS Wiki](https://wiki.satnogs.org/)
16. [GNU Radio 3.10 Documentation](https://wiki.gnuradio.org/)
17. [python-sgp4](https://github.com/brandon-rhodes/python-sgp4)
18. [Skyfield](https://rhodesmill.org/skyfield/)
