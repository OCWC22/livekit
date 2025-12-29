<chatName="LiveKit Real-Time Infrastructure Deep Dive"/>

# LiveKit Real-Time Audio/Video Infrastructure: A Comprehensive Explanation

## Executive Summary

LiveKit is an open-source WebRTC Selective Forwarding Unit (SFU) server written in Go that enables scalable, low-latency real-time communication. This document explains how LiveKit works from physical sound waves to business value, using **16 kHz PCM audio** as a concrete example throughout.

---

# 1️⃣ Engineering Intern View: Physical Foundations

**Goal:** Build physical and systems intuition for live audio streaming.

## What Sound Actually Is

Sound is **pressure waves** in air. When someone speaks into a microphone:

1. Vocal cords vibrate → air molecules compress and expand
2. Pressure variations propagate at ~343 m/s (speed of sound)
3. Microphone diaphragm moves with pressure changes
4. Movement converted to electrical signal (analog voltage)
5. Analog-to-Digital Converter (ADC) converts voltage to numbers (PCM)

```
┌─────────────┐     ┌──────────┐     ┌─────────┐     ┌─────────┐
│   Speaking  │ →→→ │  Micro-  │ →→→ │   ADC   │ →→→ │   PCM   │
│  (waves)    │     │  phone   │     │(quantize│     │ Samples │
└─────────────┘     └──────────┘     └─────────┘     └─────────┘
   343 m/s            analog           16 kHz        digital
   pressure          voltage          sampling      numbers
```

## Why 16 kHz PCM?

Human speech ranges from ~100 Hz to ~8 kHz. The **Nyquist theorem** tells us we must sample at least 2× the maximum frequency to capture it accurately:

- **16 kHz sampling** captures everything important for voice
- Each sample is a **16-bit signed integer** (-32,768 to +32,767)
- Silence ≈ 0, loud speech ≈ ±16,000, clipping = ±32,767

```
┌────────────────────────────────────────────────────────────────┐
│  1 second of 16 kHz mono audio:                               │
│                                                                │
│  16,000 samples × 2 bytes = 32,000 bytes                     │
│  = 256 kbps uncompressed                                      │
│  = ~24-32 kbps with Opus compression (8-10× smaller!)        │
└────────────────────────────────────────────────────────────────┘
```

## What Latency Means (And Why It Matters)

Latency is the time delay between when Alice speaks and when Bob hears it:

| Latency | Experience |
|---------|------------|
| **< 100ms** | Feels like in-person conversation |
| **100-200ms** | Noticeable but acceptable (phone call quality) |
| **200-500ms** | Walkie-talkie feel, awkward pauses |
| **> 500ms** | Frustrating, people talk over each other |

```
┌─────────┐    50ms    ┌─────────┐    50ms    ┌─────────┐
│  Alice  │ ────────→ │   SFU   │ ────────→ │   Bob   │
│speaking │  capture   │ forward │  network   │  hears  │
└─────────┘  +encode   └─────────┘  +decode   └─────────┘
                                   +playback
                        
Target: < 150ms one-way for natural conversation
Round-trip: Alice speaks → Bob hears → Bob replies → Alice hears ≈ 300ms
```

## Why Packets Get Lost

Network packets are like letters sent through mail:
- **Most arrive safely** (99%+ typically)
- **Some get lost** (dropped by congested routers)
- **Some arrive out of order** (different network routes)
- **Some arrive late** (queuing delays)

```
Packets sent:     1  2  3  4  5  6  7  8  9  10
                  ↓  ↓  ✗  ↓  ↓  ↓  ✗  ✗  ↓  ↓
Packets received: 1  2     4  5  6        9  10
                  Missing: 3, 7, 8

Solutions in real-time audio:
┌──────────────────┬─────────────────────────────────────┐
│ NACK             │ "Please resend packets 3, 7, 8"     │
│ FEC              │ Send redundant data with each packet│
│ PLC              │ Conceal gaps by extrapolating audio │
└──────────────────┴─────────────────────────────────────┘
```

## Real-Time vs Traditional Streaming

```
┌─────────────────────────────────────────────────────────────────┐
│  Traditional Streaming (YouTube, Netflix, Spotify)             │
├─────────────────────────────────────────────────────────────────┤
│  • Buffer 10-30 seconds before playback                        │
│  • Re-request lost chunks (TCP guarantees delivery)            │
│  • Latency doesn't matter (content already happened)           │
│  • Optimize for quality, not speed                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Real-Time Communication (LiveKit, Zoom, Discord)              │
├─────────────────────────────────────────────────────────────────┤
│  • Play immediately as packets arrive                           │
│  • Can't wait for retransmission (conversation keeps moving)   │
│  • Latency is everything (determines conversation quality)      │
│  • UDP preferred (don't block for lost packets)                │
│  • Accept some quality loss for speed                           │
└─────────────────────────────────────────────────────────────────┘
```

## The Audio Processing Pipeline (20ms Frame)

```
┌──────────────────────────────────────────────────────────────────────┐
│                        20ms of audio (320 samples)                   │
│                    Human speaks: "Hello" (first syllable)            │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  PCM Buffer: [0x0123, 0x0456, ..., 0xFEDC]  (640 bytes uncompressed) │
│  16-bit signed integers representing pressure at each sample         │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Opus Encoder: Compress 640 bytes → ~80 bytes (32 kbps target)      │
│  Uses psychoacoustic models to remove imperceptible information      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  RTP Packet: [Header (12 bytes) | Opus Payload (~80 bytes)]         │
│  Header contains:                                                    │
│    - Sequence Number: 12345 (increments each packet)                │
│    - Timestamp: 1234567800 (when audio was captured)                │
│    - SSRC: 0x12345678 (unique stream identifier)                    │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  UDP Packet: Sent to SFU server via Internet                        │
│  ~92 bytes total, one packet every 20ms = 50 packets/second         │
└──────────────────────────────────────────────────────────────────────┘
```

---

# 2️⃣ Engineer View: Code and Implementation

**Goal:** Understand how these concepts appear in actual LiveKit code.

## LiveKit's Core Components

Based on the codebase structure at `/Users/chen/Projects/livekit/pkg/`:

```
pkg/
├── service/          # HTTP/WebSocket endpoints
│   ├── rtcservice.go     # WebSocket signal endpoint
│   ├── roomservice.go    # Room management API
│   └── signal.go         # Signal handling
├── rtc/              # WebRTC logic
│   ├── room.go           # Room state management
│   ├── participant.go    # Participant connections
│   ├── transport.go      # PeerConnection wrapper
│   └── mediatrack.go     # Track management
├── sfu/              # Media forwarding
│   ├── receiver.go       # Receives from publishers
│   ├── downtrack.go      # Sends to subscribers
│   ├── forwarder.go      # Routes packets
│   └── buffer/           # Jitter buffer
└── routing/          # Multi-node coordination
    ├── redisrouter.go    # Redis-based routing
    └── localrouter.go    # Single-node routing
```

## Platform Audio Capture (Client Side)

Before audio reaches LiveKit, the client captures and encodes it:

**iOS / Swift (AVAudioEngine):**
```swift
// 16 kHz, mono, PCM format
let format = AVAudioFormat(
    commonFormat: .pcmFormatInt16,
    sampleRate: 16000.0,
    channels: 1
)

// Install a tap to receive audio buffers
sourceNode.installTap(
    onBus: 0,
    bufferSize: 320,  // 20ms at 16kHz = 320 samples
    format: format
) { [weak self] buffer, _ in
    // buffer.frameLength = 320 samples
    // buffer.audioBufferList → raw PCM bytes (640 bytes)
    // WebRTC SDK handles Opus encoding and RTP packaging
}
```

**React Native / Web (getUserMedia):**
```javascript
// WebRTC takes over audio capture
const stream = await navigator.mediaDevices.getUserMedia({
    audio: {
        sampleRate: 16000,
        channelCount: 1,
        echoCancellation: true,
        noiseSuppression: true,
    }
});

// LiveKit SDK handles the rest
const room = new Room();
await room.connect(wsUrl, token);
await room.localParticipant.publishTrack(stream.getAudioTracks()[0]);
```

## PCM Buffers in Memory

At 16 kHz mono, 16-bit samples:

```c
// 20ms frame = 320 samples = 640 bytes
int16_t pcm_buffer[320];  // Signed 16-bit samples

// Memory layout (little-endian):
// Sample 0: [0x34, 0x12] = 0x1234 = 4660 (positive pressure)
// Sample 1: [0x78, 0x56] = 0x5678 = 22136 (louder)
// Sample 2: [0xDC, 0xFE] = 0xFEDC = -292 (slightly negative)

// Values:
// Silence ≈ 0 (fluctuating around zero)
// Normal speech ≈ ±4000-16000
// Loud speech ≈ ±24000
// Clipping (bad!) = ±32767
```

## RTP Packet Structure

From `pkg/sfu/receiver.go` - LiveKit receives RTP packets from publishers:

```go
// WebRTCReceiver wraps Pion's RTP receiver
type WebRTCReceiver struct {
    receiver        *webrtc.RTPReceiver  // Pion WebRTC receiver
    upTracks        [4]TrackRemote       // Up to 4 spatial layers (simulcast)
    connectionStats *connectionquality.ConnectionStats
    buffers         [4]*buffer.Buffer    // Jitter buffers per layer
}

// RTP Header structure (12 bytes minimum):
// ┌─────────┬─────────┬─────────────────────┐
// │  V P X  │   PT    │   Sequence Number   │ (4 bytes)
// │ (2bits) │ (7bits) │     (16 bits)       │
// ├─────────┴─────────┴─────────────────────┤
// │           Timestamp (32 bits)           │ (4 bytes)
// ├─────────────────────────────────────────┤
// │              SSRC (32 bits)             │ (4 bytes)
// └─────────────────────────────────────────┘

// For 16 kHz Opus audio:
// - PayloadType: 111 (Opus)
// - SequenceNumber: 12345, 12346, 12347... (detects loss)
// - Timestamp: increments by 960 per 20ms packet (at 48kHz Opus clock)
// - SSRC: uniquely identifies this audio stream
```

## The Dual Connection Model

LiveKit uses two separate connections for different purposes:

```
┌────────────────────────────────────────────────────────────────┐
│  SIGNAL PATH (WebSocket over TCP)                             │
├────────────────────────────────────────────────────────────────┤
│  Location: pkg/service/rtcservice.go                          │
│  Protocol: WebSocket (ws:// or wss://)                        │
│  Purpose: Control messages, setup, coordination               │
│                                                                │
│  Messages:                                                     │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ • JoinRequest:  { room: "daily", token: "jwt..." }     │  │
│  │ • Offer/Answer: SDP with codec negotiation             │  │
│  │ • ICE Candidates: Network path options                 │  │
│  │ • TrackPublished: "Alice published audio track"        │  │
│  │ • Mute/Unmute: Control commands                        │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Properties: Reliable (TCP), low bandwidth, always needed     │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  MEDIA PATH (WebRTC over UDP)                                 │
├────────────────────────────────────────────────────────────────┤
│  Location: pkg/rtc/transport.go                               │
│  Protocol: RTP/RTCP over ICE/DTLS/SRTP                       │
│  Purpose: Audio/video data, low latency                       │
│                                                                │
│  Components:                                                   │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │ • ICE: NAT traversal (STUN/TURN)                       │  │
│  │ • DTLS: Key exchange for encryption                    │  │
│  │ • SRTP: Encrypted RTP packets                          │  │
│  │ • RTP: Actual audio/video data                         │  │
│  │ • RTCP: Feedback (NACK, PLI, receiver reports)         │  │
│  └─────────────────────────────────────────────────────────┘  │
│                                                                │
│  Properties: Low latency (UDP), high bandwidth, lossy OK      │
└────────────────────────────────────────────────────────────────┘
```

## RTCService: WebSocket Connection Handler

From `pkg/service/rtcservice.go`:

```go
// RTCService handles WebSocket connections for signaling
func (s *RTCService) serve(w http.ResponseWriter, r *http.Request) {
    // 1. Validate JWT token from query string
    roomName, participantInfo, code, err := s.validateInternal(...)
    if err != nil {
        handleError(w, code, err.Error())
        return
    }
    
    // 2. Start participant connection (find or create room)
    connectionResult, initialResponse, err := s.startConnection(
        ctx, roomName, participantInfo,
    )
    
    // 3. Upgrade HTTP to WebSocket
    conn, err := s.upgrader.Upgrade(w, r, nil)
    
    // 4. Send initial response (room info, other participants)
    sigConn.WriteResponse(initialResponse)
    
    // 5. Enter signal message loop
    for {
        // Read message from WebSocket
        req, _, err := sigConn.ReadRequest()
        if err != nil {
            break  // Connection closed
        }
        
        // Route message based on type
        switch m := req.Message.(type) {
        case *livekit.SignalRequest_Offer:
            // Client sent SDP offer, negotiate WebRTC
        case *livekit.SignalRequest_Answer:
            // Client responded to our offer
        case *livekit.SignalRequest_Trickle:
            // ICE candidate from client
        case *livekit.SignalRequest_Mute:
            // Mute/unmute request
        case *livekit.SignalRequest_TrackSetting:
            // Quality preference change
        }
        
        // Forward to room/participant for processing
        connectionResult.RequestSink.WriteMessage(req)
    }
}
```

## PCTransport: WebRTC PeerConnection Wrapper

From `pkg/rtc/transport.go`:

```go
// PCTransport wraps a Pion WebRTC PeerConnection
type PCTransport struct {
    pc              *webrtc.PeerConnection
    iceTransport    *webrtc.ICETransport
    me              *webrtc.MediaEngine
    
    // Data channels for reliable/lossy messaging
    reliableDC      *datachannel.DataChannelWriter  // "_reliable"
    lossyDC         *datachannel.DataChannelWriter  // "_lossy"
    
    // Connection state
    iceConnectedAt  time.Time
    connectedAt     time.Time
    
    // Migration support
    previousAnswer  *webrtc.SessionDescription
}

// AddTrack adds a media track for sending to subscriber
func (t *PCTransport) AddTrack(
    trackLocal webrtc.TrackLocal,
    params types.AddTrackParams,
) (sender *webrtc.RTPSender, transceiver *webrtc.RTPTransceiver, err error) {
    
    // Create RTPSender attached to PeerConnection
    transceiver, err = t.pc.AddTransceiverFromTrack(
        trackLocal,
        webrtc.RTPTransceiverInit{Direction: webrtc.RTPTransceiverDirectionSendonly},
    )
    sender = transceiver.Sender()
    
    // Configure codecs based on negotiation
    t.queueOrConfigureSender(transceiver, params.EnabledCodecs, ...)
    
    return sender, transceiver, nil
}

// OnRTCP handles RTCP feedback from remote peer
func (t *PCTransport) OnRTCP(cb func([]rtcp.Packet)) {
    t.pc.OnRTCP(func(rtcpPackets []rtcp.Packet) {
        cb(rtcpPackets)  // Forward to forwarder for processing
    })
}
```

## WebRTCReceiver: Receiving Published Media

From `pkg/sfu/receiver.go`:

```go
// WebRTCReceiver handles incoming RTP from a publisher
type WebRTCReceiver struct {
    receiver        *webrtc.RTPReceiver
    kind            webrtc.RTPCodecType  // Audio or Video
    
    // One buffer per spatial layer (for simulcast)
    upTracks        [DefaultMaxLayerSpatial + 1]TrackRemote
    buffers         [DefaultMaxLayerSpatial + 1]*buffer.Buffer
    
    // Statistics for quality monitoring
    connectionStats *connectionquality.ConnectionStats
    rtpStats        [DefaultMaxLayerSpatial + 1]*rtpstats.RTPStatsReceiver
    
    // Callbacks
    onPacket        func(pkt *buffer.ExtPacket)  // When packet ready
}

// ReadRTP continuously reads packets from the WebRTC receiver
func (w *WebRTCReceiver) ReadRTP(layer int32) {
    defer w.onClose()
    
    for {
        // Read RTP packet from Pion
        pkt, _, err := w.upTracks[layer].ReadRTP()
        if err != nil {
            return  // Track closed
        }
        
        // Push into jitter buffer for reordering
        w.buffers[layer].PushRTP(pkt)
    }
}

// Buffer callbacks when packet is ready for forwarding
func (w *WebRTCReceiver) onBufferPacket(layer int32, pkt *buffer.ExtPacket) {
    // Notify all DownTracks subscribed to this track
    w.onPacket(pkt)
}
```

## DownTrack: Sending to Each Subscriber

From `pkg/sfu/downtrack.go`:

```go
// DownTrack represents a single subscriber's view of a track
// Each subscriber gets their own DownTrack (independent quality, NACK handling)
type DownTrack struct {
    // Identity
    subscriberID    livekit.ParticipantID
    ssrc            uint32  // Unique SSRC for this down track
    
    // Source
    receiver        TrackReceiver       // The published track
    
    // Forwarding logic
    forwarder       *Forwarder          // Handles layer selection, munging
    
    // Pacing and sending
    rtpWriter       *webrtc.RTPSender   // Pion sender
    pacer           pacer.Pacer         // Controls send rate
    
    // Statistics
    rtpStats        *rtpstats.RTPStatsSender
}

// WriteRTP is called for each packet from the source track
func (d *DownTrack) WriteRTP(extPkt *buffer.ExtPacket, layer int32) int32 {
    // 1. Check if packet should be forwarded (layer matching, keyframe, etc.)
    tp, err := d.forwarder.GetTranslationParams(extPkt, layer)
    if tp.shouldDrop {
        return 0  // Skip this packet (wrong layer, etc.)
    }
    
    // 2. Create new RTP header (translate seq/timestamp/SSRC)
    hdr := rtp.Header{
        Version:        2,
        Marker:         tp.marker,
        PayloadType:    d.getTranslatedPayloadType(...),
        SequenceNumber: uint16(tp.rtp.extSequenceNumber),  // Remapped
        Timestamp:      uint32(tp.rtp.extTimestamp),       // Preserved
        SSRC:           d.ssrc,  // This DownTrack's unique SSRC
    }
    
    // 3. Enqueue packet for paced sending
    pacerPacket := &pacer.Packet{
        Header:  hdr,
        Payload: extPkt.Packet.Payload,
    }
    d.pacer.Enqueue(pacerPacket)
    
    return 1  // Successfully queued
}

// handleRTCP processes feedback from subscriber
func (d *DownTrack) handleRTCP(pkts []rtcp.Packet) {
    for _, pkt := range pkts {
        switch p := pkt.(type) {
        case *rtcp.ReceiverReport:
            // Update RTT and loss statistics
            for _, r := range p.Reports {
                if r.SSRC == d.ssrc {
                    rtt := d.rtpStats.UpdateFromReceiverReport(r)
                    // Use RTT to adjust NACK timing
                }
            }
            
        case *rtcp.TransportLayerNack:
            // Subscriber requesting packet retransmission
            go d.retransmitPackets(p.Nacks)
            
        case *rtcp.PictureLossIndication:
            // Subscriber needs keyframe (video)
            d.receiver.SendPLI(layer)
        }
    }
}
```

## Complete Packet Flow (16 kHz Audio Example)

```
Publisher's Browser                  LiveKit Server                    Subscriber's Browser
──────────────────                  ──────────────                    ────────────────────

1. Microphone captures 20ms audio
   320 samples @ 16kHz = 640 bytes PCM
         │
         ▼
2. Browser Opus encoder compresses
   640 bytes → ~80 bytes compressed
         │
         ▼
3. Browser creates RTP packet
   Header (12B) + Payload (80B) = 92 bytes
   SeqNum: 12345, TS: 1234567800
         │
         ▼
4. SRTP encrypts packet
         │
         ▼
5. UDP sends to LiveKit server
   ──────────────────────────────►
                                    │
                                    ▼
                              6. ICE layer receives
                                 (NAT traversal complete)
                                    │
                                    ▼
                              7. DTLS decrypts → SRTP
                                    │
                                    ▼
                              8. WebRTCReceiver.ReadRTP()
                                 Pushes to Buffer
                                    │
                                    ▼
                              9. Buffer reorders, handles gaps
                                 Jitter buffer absorbs timing variation
                                    │
                                    ▼
                              10. For each subscriber's DownTrack:
                                    │
                                    ├──► DownTrack (Bob)
                                    │    - Translate headers
                                    │    - New SSRC: 0xBOB00001
                                    │    - Pace outgoing
                                    │
                                    ├──► DownTrack (Charlie)
                                    │    - New SSRC: 0xCHARLIE1
                                    │
                                    └──► DownTrack (Dana)
                                         - New SSRC: 0xDANA0001
                                    │
                                    ▼
                              11. SRTP encrypts (per subscriber)
                                    │
                                    ▼
                              12. UDP sends to subscribers
                                 ──────────────────────────────►
                                                                │
                                                                ▼
                                                          13. ICE receives
                                                              │
                                                              ▼
                                                          14. DTLS/SRTP decrypts
                                                              │
                                                              ▼
                                                          15. Opus decodes
                                                              80B → 640B PCM
                                                              │
                                                              ▼
                                                          16. Audio plays through
                                                              speaker
                                                              
Total latency target: < 150ms end-to-end
```

## What Breaks If Assumptions Are Wrong

| Assumption | What Happens If Wrong | Detection Method |
|------------|----------------------|------------------|
| **Opus uses 48kHz clock** | Timestamp drift, audio glitches | Monitor timestamp deltas |
| **Sequence numbers increment** | Packets detected as duplicates, dropped | Gap detection |
| **Timestamps monotonic** | Jitter buffer underrun, audio gaps | Negative delta check |
| **Payload type matches** | Codec mismatch, silence or noise | SDP negotiation failure |
| **ICE candidates valid** | Connection never establishes | ICE failed state |
| **SSRC is unique** | Packets misrouted to wrong track | Collision detection |

```go
// From pkg/sfu/buffer/buffer.go - Detecting packet loss
func (b *Buffer) PushRTP(pkt *rtp.Packet) {
    // Check sequence number continuity
    expectedSeq := b.lastSeq + 1
    actualSeq := pkt.SequenceNumber
    
    if actualSeq != expectedSeq {
        // Gap detected! Either packet loss or reordering
        gap := actualSeq - expectedSeq
        if gap > 0 && gap < 100 {
            // Forward jump: packets were lost
            b.lostPackets += int(gap)
            // Send NACK for missing packets
            b.generateNACK(expectedSeq, actualSeq)
        } else if gap < 0 && gap > -100 {
            // Backward jump: late arrival (reordering)
            // Insert into correct position in buffer
        }
    }
}
```

---

# 3️⃣ CTO View: System Design and Architecture

**Goal:** Understand architectural decisions, tradeoffs, and scalability.

## Why SFU Architecture?

LiveKit uses a **Selective Forwarding Unit (SFU)** architecture instead of alternatives:

```
┌─────────────────────────────────────────────────────────────────────┐
│  MCU (Multipoint Control Unit) - Traditional Approach              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Alice ───┐                         ┌─── Alice sees: Bob+Charlie   │
│           │    ┌─────────────┐      │    (mixed by server)         │
│  Bob   ───┼───►│  MCU Server │──────┼─── Bob sees: Alice+Charlie   │
│           │    │  (decodes,  │      │    (mixed by server)         │
│  Charlie ─┘    │   mixes,    │      └─── Charlie sees: Alice+Bob   │
│                │  re-encodes)│           (mixed by server)         │
│                └─────────────┘                                      │
│                                                                     │
│  Server CPU: VERY HIGH (decode all, mix, re-encode per viewer)     │
│  Latency: +100-300ms (encoding delay)                              │
│  Scalability: ~10-20 participants per server                       │
│  Flexibility: Low (everyone sees same composition)                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  SFU (Selective Forwarding Unit) - LiveKit's Approach              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Alice ───┐                         ┌─── Alice receives: Bob, Charlie │
│           │    ┌─────────────┐      │    (separate streams)        │
│  Bob   ───┼───►│  SFU Server │──────┼─── Bob receives: Alice, Charlie │
│           │    │  (forwards  │      │    (separate streams)        │
│  Charlie ─┘    │   only, no  │      └─── Charlie receives: Alice, Bob │
│                │   decoding) │           (separate streams)        │
│                └─────────────┘                                      │
│                                                                     │
│  Server CPU: LOW (just routing packets)                             │
│  Latency: MINIMAL (no transcoding)                                  │
│  Scalability: 100-500+ participants per server                     │
│  Flexibility: HIGH (each client controls their view)               │
└─────────────────────────────────────────────────────────────────────┘
```

**Why LiveKit chose SFU:**
- **Scalability**: CPU usage is O(N), not O(N²)
- **Latency**: No encoding delay on server
- **Flexibility**: Each subscriber can choose quality
- **Cost**: Much cheaper compute per participant

## Multi-Node Architecture

From `pkg/routing/`:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Global Architecture                           │
└─────────────────────────────────────────────────────────────────────┘

              Client connects to nearest region
                            │
                            ▼
                  ┌─────────────────┐
                  │  Load Balancer  │
                  │  (DNS/Anycast)  │
                  └────────┬────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
   │  LiveKit    │  │  LiveKit    │  │  LiveKit    │
   │  Node       │  │  Node       │  │  Node       │
   │  (US-East)  │  │  (US-West)  │  │  (EU-West)  │
   │             │  │             │  │             │
   │ Room: daily │  │ Room: eng   │  │ Room: sales │
   │ Users: 10   │  │ Users: 25   │  │ Users: 15   │
   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
          │                │                │
          └────────────────┼────────────────┘
                           │
                  ┌────────▼────────┐
                  │     Redis       │
                  │  ┌───────────┐  │
                  │  │ Room Map  │  │
                  │  │daily → US-E│  │
                  │  │eng → US-W  │  │
                  │  │sales → EU-W│  │
                  │  └───────────┘  │
                  └─────────────────┘
```

**How room assignment works** (from `pkg/routing/redisrouter.go`):

```go
// RedisRouter coordinates multiple LiveKit nodes
type RedisRouter struct {
    rc        redis.UniversalClient
    currentNode string  // This node's ID
}

// StartParticipantSignal routes a joining participant
func (r *RedisRouter) StartParticipantSignal(
    ctx context.Context,
    roomName string,
    pi routing.ParticipantInit,
) (routing.StartParticipantSignalResults, error) {
    
    // 1. Check if room already exists somewhere
    existingNode, err := r.rc.HGet(ctx, "room:"+roomName, "node").Result()
    
    if err == redis.Nil {
        // Room doesn't exist - create on this node
        err = r.rc.HSet(ctx, "room:"+roomName, "node", r.currentNode).Err()
        return r.localRouter.StartParticipantSignal(ctx, roomName, pi)
    }
    
    if existingNode == r.currentNode {
        // Room is on this node - handle locally
        return r.localRouter.StartParticipantSignal(ctx, roomName, pi)
    }
    
    // Room is on another node - redirect client
    return routing.StartParticipantSignalResults{
        RedirectTo: existingNode,
    }, nil
}
```

## Scalability Analysis

**Single Room Scaling:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  10 participants, all publishing video (1.5 Mbps each)             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Publisher bandwidth (each): 1.5 Mbps × 1 = 1.5 Mbps               │
│  Total inbound to server: 10 × 1.5 = 15 Mbps                       │
│                                                                     │
│  Each participant subscribes to 9 others                            │
│  Per subscriber outbound: 9 × 1.5 = 13.5 Mbps                      │
│  Total outbound from server: 10 × 13.5 = 135 Mbps                  │
│                                                                     │
│  Total server bandwidth: 15 + 135 = 150 Mbps                        │
│  Server CPU: ~10-20% (mostly packet forwarding)                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  100 participants, 10 publishing video                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Inbound: 10 × 1.5 = 15 Mbps                                        │
│  Outbound: 100 subscribers × 10 streams × 1.5 = 1,500 Mbps (1.5 Gbps)│
│                                                                     │
│  With adaptive quality (subscribers get lower res):                 │
│  - Active speaker: 720p (1.5 Mbps)                                  │
│  - Others: 180p thumbnails (0.2 Mbps)                               │
│  Optimized outbound: 100 × (1.5 + 9×0.2) = 330 Mbps                │
└─────────────────────────────────────────────────────────────────────┘
```

**Horizontal Scaling:**

```go
// Node selection logic from pkg/routing/selector/
type NodeSelector interface {
    SelectNode(nodes []*livekit.Node) (*livekit.Node, error)
}

// CPULoadSelector picks node with lowest CPU
type CPULoadSelector struct{}

func (s *CPULoadSelector) SelectNode(nodes []*livekit.Node) (*livekit.Node, error) {
    var selected *livekit.Node
    lowestLoad := math.MaxFloat64
    
    for _, node := range nodes {
        load := node.Stats.CpuLoad
        if load < lowestLoad && load < 0.8 {  // Don't use nodes above 80%
            lowestLoad = load
            selected = node
        }
    }
    
    if selected == nil {
        return nil, selector.ErrNoAvailableNodes
    }
    return selected, nil
}

// RegionAwareSelector prefers geographically close nodes
type RegionAwareSelector struct {
    CurrentRegion string
}

func (s *RegionAwareSelector) SelectNode(nodes []*livekit.Node) (*livekit.Node, error) {
    // First: try nodes in same region
    // Second: try nodes in nearby regions
    // Third: any available node
    // ...
}
```

## Failure Modes and Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│  Failure: Publisher loses network                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: RTCP timeout (no packets for ~5 seconds)               │
│  Recovery:                                                          │
│    1. Subscribers get "track muted" event                          │
│    2. Publisher reconnects via WebSocket                            │
│    3. ICE restart or new connection                                 │
│    4. Resume publishing from same track                             │
│  User impact: Audio/video freezes briefly, then resumes             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Failure: LiveKit node crashes                                     │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: Redis heartbeat expires                                 │
│  Recovery:                                                          │
│    1. Redis marks node as dead                                     │
│    2. Room migrates to new node                                    │
│    3. Clients reconnect (auto-reconnect in SDKs)                   │
│    4. State restored from migration cache                          │
│  User impact: ~2-5 second interruption                             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Failure: Network congestion (10%+ packet loss)                    │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: RTCP receiver reports show high loss                   │
│  Recovery:                                                          │
│    1. Send NACKs for important missing packets                      │
│    2. Request keyframe if video corruption                         │
│    3. Switch subscriber to lower quality layer                      │
│    4. Notify publisher to reduce encoding quality (dynacast)        │
│  User impact: Quality degrades gracefully instead of freezing       │
└─────────────────────────────────────────────────────────────────────┘
```

**Migration Support** (from `pkg/rtc/transport.go`):

```go
// SetPreviousSdp enables seamless migration between nodes
func (t *PCTransport) SetPreviousSdp(
    localDescription, remoteDescription *webrtc.SessionDescription,
) {
    // Restore previous session state
    // - Track subscriptions
    // - Transceiver MIDs (media identifiers)
    // - SSRC mappings
    
    if t.pc.RemoteDescription() == nil && t.previousAnswer == nil {
        t.previousAnswer = remoteDescription
        
        // Parse previous SDP to restore track mappings
        senders, err := t.initPCWithPreviousAnswer(*remoteDescription)
        
        // Client resumes with same SSRCs - seamless transition
        for mid, sender := range senders {
            t.midToSender[mid] = sender
        }
    }
}
```

## Latency vs Reliability Tradeoffs

LiveKit makes these tradeoffs configurable:

| Strategy | Latency Impact | Reliability | When Used |
|----------|---------------|-------------|-----------|
| **Immediate forward** | Lowest | Accepts loss | Default for audio |
| **Small jitter buffer (20-50ms)** | Very low | Handles reordering | Good networks |
| **Large jitter buffer (100-200ms)** | Medium | Handles loss+reorder | Poor networks |
| **NACK retransmission** | Variable (+RTT) | High | When RTT is low |
| **FEC (Forward Error Correction)** | +10% bandwidth | Medium | Moderate loss |
| **Opus PLC** | None | Conceals 1-2 packets | Always for audio |

**Adaptive Strategy** (from `pkg/sfu/buffer/`):

```go
// Buffer dynamically adjusts based on network conditions
type Buffer struct {
    minLatency    time.Duration  // Floor
    maxLatency    time.Duration  // Ceiling
    currentLatency time.Duration
    
    lossRate      float64  // Recent packet loss percentage
    jitter        time.Duration  // Variance in arrival time
}

func (b *Buffer) adaptLatency() {
    if b.lossRate > 0.05 || b.jitter > 30*time.Millisecond {
        // Network issues: increase buffer
        b.currentLatency = min(b.currentLatency * 1.5, b.maxLatency)
    } else if b.lossRate < 0.01 && b.jitter < 10*time.Millisecond {
        // Good network: reduce buffer for lower latency
        b.currentLatency = max(b.currentLatency * 0.9, b.minLatency)
    }
}
```

## Bandwidth Estimation and Allocation

From `pkg/sfu/streamallocator/`:

```go
// StreamAllocator manages bandwidth across all subscriptions
type StreamAllocator struct {
    tracks          []*Track
    availableBW     int64  // Estimated available bandwidth
    allocatedBW     int64  // Currently allocated
}

// AllocateOptimal distributes bandwidth fairly
func (sa *StreamAllocator) AllocateOptimal() {
    // 1. Get bandwidth estimate from BWE (Bandwidth Estimator)
    available := sa.bwe.GetEstimate()
    
    // 2. Calculate each track's ideal needs
    type trackNeed struct {
        track    *Track
        current  int64  // Current allocation
        optimal  int64  // Ideal allocation
        minimum  int64  // Bare minimum (audio priority)
        priority int    // Active speaker = high priority
    }
    
    needs := make([]trackNeed, len(sa.tracks))
    for i, t := range sa.tracks {
        needs[i] = trackNeed{
            track:    t,
            current:  t.CurrentBitrate(),
            optimal:  t.OptimalBitrate(),  // Highest quality
            minimum:  t.MinimumBitrate(),  // Lowest acceptable
            priority: t.Priority(),
        }
    }
    
    // 3. Allocate: priorities first, then fair share
    // Audio always gets minimum (small, critical)
    // Active speaker video gets priority
    // Others share remaining proportionally
    
    // 4. Apply allocations
    for _, need := range needs {
        allocation := calculateAllocation(need, available)
        need.track.SetTargetLayer(allocation)
    }
}
```

## Architectural Decisions: Hard to Change Later

| Decision | Why It's Hard to Change | LiveKit's Choice |
|----------|------------------------|------------------|
| **SFU vs MCU** | Entire media path architecture | SFU (scalability) |
| **WebSocket signaling** | All client SDKs depend on it | Protocol buffers over WS |
| **Room model** | Application logic built on it | Room → Participants → Tracks |
| **Per-subscriber DownTrack** | Core to scalability | Yes (independent quality) |
| **WebRTC (not custom)** | Protocol compliance | Pion WebRTC (Go) |
| **Redis for coordination** | Multi-node discovery | Redis (with fallback to local) |

---

# 4️⃣ CEO View: Business and Strategic Value

**Goal:** Understand product, cost, scalability, and risk implications.

## What LiveKit Enables That Simpler Stacks Cannot

```
┌─────────────────────────────────────────────────────────────────────┐
│  Use Cases Requiring Real-Time Infrastructure                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Video Conferencing (Zoom/Meet alternative)                       │
│    - 500+ participants                                              │
│    - Sub-200ms latency                                              │
│    - Screen sharing, multiple presenters                            │
│                                                                     │
│  ✓ Live Classrooms / Webinars                                       │
│    - Teacher broadcasts, students can speak                         │
│    - Breakout rooms                                                 │
│    - Recording for later                                            │
│                                                                     │
│  ✓ Telehealth                                                       │
│    - HIPAA compliance possible (E2E encryption)                     │
│    - Reliable enough for medical consultations                      │
│                                                                     │
│  ✓ Gaming Voice Chat                                                │
│    - Low latency critical for gameplay                              │
│    - Scales with player count                                       │
│                                                                     │
│  ✓ AI-Powered Conversations                                         │
│    - LiveKit Agents framework                                       │
│    - AI participant in room (speech-to-text, LLM, text-to-speech)  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  What Simpler Stacks CANNOT Do                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pure WebSocket / HTTP Streaming:                                  │
│  ✗ Latency 2-10 seconds (not conversational)                       │
│  ✗ Server must decode/encode all media (expensive)                 │
│  ✗ Doesn't scale past ~50 participants                             │
│  ✗ No NAT traversal (many users can't connect)                     │
│                                                                     │
│  Peer-to-Peer WebRTC:                                              │
│  ✗ N×N connections don't scale past 4-5 people                      │
│  ✗ No server-side recording                                        │
│  ✗ Can't broadcast to many viewers                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Cost and Economics

**Resource Usage Per 100 Active Participants:**

| Resource | Typical Usage | Cost Driver |
|----------|--------------|-------------|
| **CPU** | 4-8 cores | Packet forwarding, RTCP |
| **Memory** | 2-4 GB | Buffers, connection state |
| **Bandwidth** | 500 Mbps - 2 Gbps | Main cost (egress) |
| **Redis** | Minimal | Coordination only |

**Bandwidth Dominates Cost:**

```
Example: 100 participants, 10 publishing 720p video

Cloud bandwidth cost: ~$0.05-0.12/GB (AWS/GCP/Azure)

Outbound bandwidth:
  100 subscribers × 10 streams × 1.5 Mbps × 1 hour
  = 100 × 10 × 1.5 × 3600 / 8 / 1000 GB
  = 675 GB/hour
  = $33-81/hour in bandwidth alone

With optimization (adaptive quality):
  Reduced to ~200 GB/hour = $10-24/hour
```

**Total Cost of Ownership:**

| Scale | LiveKit Cloud | Self-Hosted |
|-------|--------------|-------------|
| **< 100 concurrent** | ~$0.005/min/participant = $20-50/month | 1-2 servers = $50-200/month |
| **100-500 concurrent** | $200-1,000/month | 2-4 servers + Redis = $200-500/month |
| **1000+ concurrent** | $1,000+/month | Self-hosting economical, need DevOps |

**Self-hosting becomes cheaper at scale**, but requires:
- DevOps expertise
- Monitoring setup
- On-call rotation
- Multi-region complexity

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Latency spikes** | Medium | User frustration | Multi-region, TURN, adaptive quality |
| **Node failure** | Low | Brief interruption | Auto-reconnect, migration |
| **DDoS attack** | Low | Service outage | CDN, rate limiting, cloud protection |
| **Vendor lock-in** | N/A (open source) | None | Self-hostable, standard protocols |
| **Scaling ceiling** | Low | Can't grow | Add nodes, LiveKit Cloud handles |
| **Security breach** | Low | Data exposure | E2E encryption, JWT auth, audits |

## Competitive Landscape

```
┌─────────────────────────────────────────────────────────────────────┐
│  Competitor Analysis                                                │
├─────────────┬───────────────────────────────────────────────────────┤
│  Provider   │  Positioning                                          │
├─────────────┼───────────────────────────────────────────────────────┤
│  LiveKit    │  Open source + optional managed service               │
│             │  Best for: Teams wanting control + flexibility        │
│             │  Differentiator: AI Agents, open source               │
├─────────────┼───────────────────────────────────────────────────────┤
│  Daily.co   │  SaaS only, simple integration                       │
│             │  Best for: Quick implementation, no DevOps            │
│             │  Limitation: Closed source, less customizable         │
├─────────────┼───────────────────────────────────────────────────────┤
│  Agora      │  Enterprise, global network                           │
│             │  Best for: Large scale, China access                  │
│             │  Limitation: Expensive, proprietary                   │
├─────────────┼───────────────────────────────────────────────────────┤
│  Twilio     │  Broad platform, video is one product                │
│             │  Best for: Already using Twilio                       │
│             │  Limitation: Video not primary focus                  │
├─────────────┼───────────────────────────────────────────────────────┤
│  Jitsi      │  Open source, simpler                                 │
│             │  Best for: Basic use cases, budget                    │
│             │  Limitation: Less scalable, fewer features            │
├─────────────┼───────────────────────────────────────────────────────┤
│  Amazon Chime│  AWS native                                          │
│  SDK        │  Best for: All-in AWS shops                           │
│             │  Limitation: AWS lock-in                              │
└─────────────┴───────────────────────────────────────────────────────┘
```

## Strategic Value

**LiveKit's Defensible Moats:**

1. **Open Source + Managed Service**
   - No vendor lock-in concerns
   - Start self-hosted, scale to managed (or vice versa)
   - Community contributions improve the product

2. **Protocol Expertise**
   - WebRTC is notoriously complex
   - Years of edge case handling
   - Hard for competitors to replicate quickly

3. **Platform Ecosystem**
   - SDKs for JS, Swift, Kotlin, Flutter, React Native, Unity
   - Each SDK is maintained and tested
   - Network effects: more users → more bug reports → better quality

4. **AI Integration (Agents)**
   - First-mover in AI + real-time
   - Agent framework for voice AI
   - Integration with speech-to-text, LLMs, text-to-speech

5. **Enterprise Features**
   - E2E encryption
   - SSO integration
   - Compliance (SOC2, HIPAA possible)

## When to Use LiveKit vs Alternatives

```
┌─────────────────────────────────────────────────────────────────────┐
│  Decision Matrix                                                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Use LiveKit when:                                                  │
│  ✓ Product needs real-time interaction (< 500ms latency)           │
│  ✓ More than 5 participants per room                                │
│  ✓ Need screen sharing, recording, or live streaming               │
│  ✓ Want control over infrastructure                                 │
│  ✓ Building AI-powered voice/video features                         │
│                                                                     │
│  Consider alternatives when:                                        │
│  ✓ Only 1:1 calls → Simpler WebRTC (PeerJS, SimpleWebRTC)          │
│  ✓ Broadcast only → HLS/DASH streaming services                    │
│  ✓ Low volume, non-critical → Simpler SaaS (Daily, Whereby)        │
│  ✓ Already deep in AWS → Chime SDK might integrate better          │
└─────────────────────────────────────────────────────────────────────┘
```

## Future-Proofing

1. **WebRTC is a W3C Standard**
   - Supported by all major browsers
   - Active development and improvement
   - Not going away

2. **Emerging Standards**
   - **WHIP/WHEP**: Standardized WebRTC ingestion/playback
   - LiveKit can adopt these for broader interoperability

3. **Codec Evolution**
   - **AV1**: Better compression, royalty-free
   - **VP9 SVC**: Scalable video coding
   - LiveKit architecture supports new codecs

4. **Edge Computing**
   - Deploy closer to users for even lower latency
   - LiveKit's multi-node architecture supports edge

5. **AI Integration**
   - Voice AI, real-time translation
   - LiveKit Agents framework is production-ready
   - First-mover advantage in AI + real-time

---

# Summary: The Complete 16 kHz Audio Journey

```
┌─────────────────────────────────────────────────────────────────────┐
│                    End-to-End Journey                              │
└─────────────────────────────────────────────────────────────────────┘

LAYER 1 - PHYSICS (Intern View)
┌─────────────────────────────────────────────────────────────────────┐
│  Alice speaks → Sound waves (343 m/s) → Microphone → ADC           │
│  16,000 samples/sec × 16 bits = 256 kbps uncompressed              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
LAYER 2 - CODE (Engineer View)
┌─────────────────────────────────────────────────────────────────────┐
│  PCM (640 bytes) → Opus Encoder (80 bytes) → RTP Header (12 bytes) │
│  WebSocket: JWT auth, SDP offer/answer, ICE candidates             │
│  WebRTC: ICE → DTLS → SRTP → RTP packets over UDP                  │
│                                                                     │
│  LiveKit: RTCService → Room → Participant → PCTransport            │
│           WebRTCReceiver → Buffer → DownTrack → Subscriber          │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
LAYER 3 - ARCHITECTURE (CTO View)
┌─────────────────────────────────────────────────────────────────────┐
│  SFU architecture: Forward, don't mix (O(N) not O(N²))             │
│  Multi-node: Redis coordination, room affinity, migration          │
│  Reliability: NACK, FEC, adaptive jitter buffer, PLC               │
│  Scalability: Add nodes, load balance, per-subscriber quality      │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
LAYER 4 - BUSINESS (CEO View)
┌─────────────────────────────────────────────────────────────────────┐
│  Enables: Video conferencing, telehealth, gaming, AI voice         │
│  Economics: Bandwidth is main cost, self-host at scale              │
│  Competitive: Open source differentiator, AI integration            │
│  Risk: Mitigated by standards compliance, multi-region, E2E encrypt│
└─────────────────────────────────────────────────────────────────────┘

Total latency: < 150ms (sound waves to eardrums across the internet)
```

---

# Key Takeaways by Audience

| Audience | Key Understanding |
|----------|------------------|
| **Intern** | Sound → numbers → compression → packets → network → sound. Latency matters. |
| **Engineer** | RTP/RTCP protocols, dual WebSocket+WebRTC paths, NACK/FEC reliability, Go codebase structure |
| **CTO** | SFU scales, bandwidth dominates cost, Redis coordinates nodes, migration handles failures |
| **CEO** | Enables $B market (video calls), open source = no lock-in, AI integration is differentiator |