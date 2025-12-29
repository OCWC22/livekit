# LiveKit Real-Time Audio/Video Infrastructure: End-to-End Explanation

## Executive Summary

**LiveKit** is an open-source WebRTC **Selective Forwarding Unit (SFU)** server written in Go that enables scalable, low-latency real-time communication. Unlike traditional video conferencing that mixes and re-encodes all streams on the server (MCU), LiveKit **forwards packets as-is**, achieving O(N) rather than O(N²) complexity.

This explanation uses **16 kHz PCM audio** as a concrete example to trace the journey from physical sound waves to business value.

---

# 1️⃣ Engineering Intern View: Physical Foundations

**Goal:** Build physical and systems intuition for live audio streaming.

## What Sound Actually Is

Sound is **pressure waves** traveling through air. When someone speaks:

```
┌─────────────────────────────────────────────────────────────────┐
│  Physical Journey of Sound                                      │
└─────────────────────────────────────────────────────────────────┘

  Vocal cords       Air pressure       Microphone        ADC
   vibrate          propagates         diaphragm       (quantize)
      │                 │                 │               │
      ▼                 ▼                 ▼               ▼
  ┌───────┐       ┌──────────┐       ┌─────────┐     ┌─────────┐
  │       │  ~~~  │ ~~~~~~~~ │  ~~~  │    ∿    │ ─→  │ 0x1234  │
  │  🗣️   │ ───→ │  ~~~~~~  │ ───→ │   ∿∿∿   │ ─→  │ 0x5678  │
  │       │  ~~~  │ ~~~~~~~~ │  ~~~  │  ∿∿∿∿∿  │ ─→  │ 0xFEDC  │
  └───────┘       └──────────┘       └─────────┘     └─────────┘
   100-8kHz         343 m/s           Analog           Digital
   frequency        speed           voltage          numbers
```

1. **Vocal cords vibrate** → air molecules compress and expand
2. **Pressure variations** propagate at ~343 m/s (speed of sound)
3. **Microphone diaphragm** moves with pressure changes
4. **Analog-to-Digital Converter (ADC)** converts voltage to numbers

## Why 16 kHz PCM?

Human speech typically ranges from ~100 Hz to ~8 kHz. The **Nyquist theorem** tells us we must sample at **at least 2× the maximum frequency** to capture it accurately:

```
┌─────────────────────────────────────────────────────────────────┐
│  16 kHz PCM Audio Math                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Sampling rate:    16,000 samples/second                        │
│  Bit depth:        16 bits (signed integer: -32,768 to +32,767) │
│                                                                 │
│  One second of audio:                                           │
│    16,000 samples × 2 bytes = 32,000 bytes = 256 kbps          │
│                                                                 │
│  After Opus compression:                                        │
│    ~24-32 kbps (8-10× smaller!)                                 │
│                                                                 │
│  20ms audio frame (typical):                                    │
│    320 samples × 2 bytes = 640 bytes uncompressed              │
│    → ~60-80 bytes compressed                                    │
└─────────────────────────────────────────────────────────────────┘
```

**Sample values represent pressure:**
- **Silence** ≈ 0 (fluctuating around zero)
- **Normal speech** ≈ ±4,000 to ±16,000
- **Loud speech** ≈ ±24,000
- **Clipping (bad!)** = ±32,767

## What Latency Means in Human Perception

Latency is the delay between when Alice speaks and when Bob hears it:

```
┌──────────────────┬──────────────────────────────────────────────┐
│     Latency      │              Human Experience                │
├──────────────────┼──────────────────────────────────────────────┤
│    < 100ms       │  Feels like in-person conversation           │
│   100-200ms      │  Noticeable but acceptable (phone quality)   │
│   200-500ms      │  Walkie-talkie feel, awkward pauses          │
│    > 500ms       │  Frustrating, people talk over each other    │
└──────────────────┴──────────────────────────────────────────────┘

Target end-to-end: < 150ms one-way

┌─────────┐   ~50ms    ┌─────────┐   ~50ms    ┌─────────┐
│  Alice  │ ────────→  │   SFU   │ ────────→  │   Bob   │
│speaking │  capture   │ forward │  network   │  hears  │
└─────────┘  +encode   └─────────┘  +decode   └─────────┘
                                   +playback
```

## Why Packets Get Lost

Network packets are like letters sent through mail—most arrive, but some don't:

```
Packets sent:     1  2  3  4  5  6  7  8  9  10
                  ↓  ↓  ✗  ↓  ↓  ↓  ✗  ✗  ↓  ↓
Packets received: 1  2     4  5  6        9  10
                  
                  Missing: 3, 7, 8 (dropped by congested routers)

┌──────────────────────────────────────────────────────────────────┐
│  Why packets get lost:                                           │
├──────────────────────────────────────────────────────────────────┤
│  • Router buffers overflow (congestion)                          │
│  • Different network paths (reordering)                          │
│  • Queuing delays (late arrival)                                 │
│  • Wireless interference                                         │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│  Solutions in real-time audio:                                   │
├──────────────────┬───────────────────────────────────────────────┤
│  NACK            │  "Please resend packets 3, 7, 8"              │
│  FEC             │  Forward error correction (redundant data)    │
│  PLC             │  Packet loss concealment (extrapolate audio)  │
│  Jitter Buffer   │  Hold packets briefly to reorder              │
└──────────────────┴───────────────────────────────────────────────┘
```

## Why "Real-Time" is Fundamentally Different from REST APIs

```
┌─────────────────────────────────────────────────────────────────┐
│  Traditional Streaming (YouTube, Netflix, Spotify)              │
├─────────────────────────────────────────────────────────────────┤
│  • Buffer 10-30 seconds before playback                         │
│  • Re-request lost chunks (TCP guarantees delivery)             │
│  • Latency doesn't matter (content already happened)            │
│  • Uses: TCP, HLS, DASH                                         │
│  • Optimize for: QUALITY                                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  Real-Time Communication (LiveKit, Zoom, Discord)               │
├─────────────────────────────────────────────────────────────────┤
│  • Play immediately as packets arrive                           │
│  • Can't wait for retransmission (conversation keeps moving)    │
│  • Latency is everything (determines conversation quality)      │
│  • Uses: UDP, WebRTC, RTP                                       │
│  • Optimize for: SPEED (accept some quality loss)               │
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
│  16-bit signed integers representing pressure at each sample point   │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  Opus Encoder: Compress 640 bytes → ~60-80 bytes (32 kbps target)    │
│  Uses psychoacoustic models to remove imperceptible information      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  RTP Packet: [Header (12 bytes) | Opus Payload (~70 bytes)]          │
│  Header contains:                                                    │
│    - Sequence Number: 12345 (increments each packet)                │
│    - Timestamp: 1234567800 (when audio was captured)                │
│    - SSRC: 0x12345678 (unique stream identifier)                    │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  UDP Packet: Sent to SFU server via Internet                        │
│  ~82 bytes total, one packet every 20ms = 50 packets/second         │
└──────────────────────────────────────────────────────────────────────┘
```

---

# 2️⃣ Engineer View: Code and Implementation

**Goal:** Understand how these concepts appear in actual LiveKit code.

## LiveKit's Core Components

Based on the actual codebase structure:

```
pkg/
├── service/              # HTTP/WebSocket endpoints
│   ├── rtcservice.go         # WebSocket signal endpoint
│   ├── roomservice.go        # Room management API
│   └── signal.go             # Signal handling
├── rtc/                  # WebRTC logic
│   ├── room.go               # Room state management
│   ├── participant.go        # Participant connections
│   ├── transport.go          # PeerConnection wrapper (PCTransport)
│   └── mediatrack.go         # Track management
├── sfu/                  # Media forwarding
│   ├── receiver.go           # WebRTCReceiver - receives from publishers
│   ├── downtrack.go          # DownTrack - sends to each subscriber
│   ├── forwarder.go          # Routes packets with translation
│   └── buffer/               # Jitter buffer and packet storage
└── routing/              # Multi-node coordination
    ├── redisrouter.go        # Redis-based routing for multi-node
    ├── localrouter.go        # Single-node routing
    └── interfaces.go         # Router/MessageSink/MessageSource interfaces
```

## The Dual Connection Model

LiveKit uses **two separate connections** for different purposes:

```
┌────────────────────────────────────────────────────────────────┐
│  SIGNAL PATH (WebSocket over TCP)                              │
├────────────────────────────────────────────────────────────────┤
│  Location: pkg/service/rtcservice.go                           │
│  Protocol: WebSocket (ws:// or wss://)                         │
│  Purpose: Control messages, setup, coordination                │
│                                                                │
│  Messages (from actual code):                                  │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ • SignalRequest_Offer:  SDP offer from client             │ │
│  │ • SignalRequest_Answer: SDP answer from client            │ │
│  │ • SignalRequest_Trickle: ICE candidate from client        │ │
│  │ • SignalRequest_Mute: Mute/unmute request                 │ │
│  │ • SignalRequest_Ping/Pong: Connection keepalive           │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  Properties: Reliable (TCP), low bandwidth, always needed      │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│  MEDIA PATH (WebRTC over UDP)                                  │
├────────────────────────────────────────────────────────────────┤
│  Location: pkg/rtc/transport.go (PCTransport)                  │
│  Protocol: RTP/RTCP over ICE/DTLS/SRTP                        │
│  Purpose: Audio/video data, low latency                        │
│                                                                │
│  Components (from actual code):                                │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │ • ICE: NAT traversal (STUN/TURN via Pion)                │ │
│  │ • DTLS: Key exchange for encryption                      │ │
│  │ • SRTP: Encrypted RTP packets                            │ │
│  │ • Data Channels: "_reliable" and "_lossy"                │ │
│  │ • RTCP: Feedback (NACK, PLI, receiver reports)           │ │
│  └───────────────────────────────────────────────────────────┘ │
│                                                                │
│  Properties: Low latency (UDP), high bandwidth, lossy OK       │
└────────────────────────────────────────────────────────────────┘
```

## RTCService: WebSocket Connection Handler

From `pkg/service/rtcservice.go`, the actual flow:

```go
// RTCService handles WebSocket connections for signaling
type RTCService struct {
    router        routing.MessageRouter    // Routes to room nodes
    roomAllocator RoomAllocator            // Allocates rooms to nodes
    upgrader      websocket.Upgrader       // HTTP → WebSocket upgrade
    connections   map[*websocket.Conn]struct{} // Active connections
}

// serve() - actual connection handling flow:
func (s *RTCService) serve(w http.ResponseWriter, r *http.Request, needsJoinRequest bool) {
    // 1. Validate JWT token and parse join request
    roomName, pi, code, err := s.validateInternal(...)
    
    // 2. Start participant signal connection (find or create room)
    cr, initialResponse, err := s.startConnection(ctx, roomName, pi, timeout)
    
    // 3. Upgrade HTTP to WebSocket
    conn, err := s.upgrader.Upgrade(w, r, nil)
    
    // 4. Send initial response (room info, other participants)
    sigConn.WriteResponse(initialResponse)
    
    // 5. Enter bidirectional message loop
    // - Goroutine: Read from ResponseSource → Write to WebSocket
    // - Main loop: Read from WebSocket → Write to RequestSink
    for {
        req, count, err := sigConn.ReadRequest()
        // Route based on message type: Offer, Answer, Trickle, Mute, etc.
        cr.RequestSink.WriteMessage(req)
    }
}
```

## PCTransport: WebRTC PeerConnection Wrapper

From `pkg/rtc/transport.go`, the actual structure:

```go
// PCTransport wraps a Pion WebRTC PeerConnection
type PCTransport struct {
    params       TransportParams
    pc           *webrtc.PeerConnection    // Pion's PeerConnection
    iceTransport *webrtc.ICETransport      // ICE layer
    me           *webrtc.MediaEngine       // Codec configuration
    
    // Data channels for reliable/lossy messaging
    reliableDC   *datachannel.DataChannelWriter  // "_reliable" 
    lossyDC      *datachannel.DataChannelWriter  // "_lossy"
    
    // Connection timing for quality metrics
    iceStartedAt   time.Time
    iceConnectedAt time.Time
    connectedAt    time.Time
    
    // Stream allocation for bandwidth management
    streamAllocator *streamallocator.StreamAllocator
    bwe             bwe.BWE      // Bandwidth estimator
    pacer           pacer.Pacer  // Controls send rate
}

// AddTrack adds a media track for sending to subscriber
func (t *PCTransport) AddTrack(
    trackLocal webrtc.TrackLocal,
    params types.AddTrackParams,
    enabledCodecs []*livekit.Codec,
    rtcpFeedbackConfig RTCPFeedbackConfig,
) (sender *webrtc.RTPSender, transceiver *webrtc.RTPTransceiver, err error) {
    // Creates RTPSender attached to PeerConnection
    sender, err = t.pc.AddTrack(trackLocal)
    // Configure codecs based on negotiation
    t.queueOrConfigureSender(transceiver, enabledCodecs, ...)
    return sender, transceiver, nil
}
```

## WebRTCReceiver: Receiving Published Media

From `pkg/sfu/receiver.go`:

```go
// WebRTCReceiver handles incoming RTP from a publisher
type WebRTCReceiver struct {
    *ReceiverBase
    
    receiver       *webrtc.RTPReceiver
    upTracks       [4]TrackRemote           // Up to 4 spatial layers (simulcast)
    connectionStats *connectionquality.ConnectionStats
    onRTCP         func([]rtcp.Packet)
}

// AddUpTrack adds a new spatial/temporal layer
func (w *WebRTCReceiver) AddUpTrack(track TrackRemote, buff *buffer.Buffer) error {
    layer := buffer.GetSpatialLayerForRid(w.Mime(), track.RID(), w.TrackInfo())
    
    w.upTracks[layer] = track
    w.ReceiverBase.AddBuffer(buff, layer)
    
    // Buffer handles packet processing and callbacks
    buff.OnRtcpFeedback(w.sendRTCP)
    w.ReceiverBase.StartBuffer(buff, layer)
    return nil
}
```

## DownTrack: Sending to Each Subscriber

From `pkg/sfu/downtrack.go`, the core forwarding logic:

```go
// DownTrack represents a single subscriber's view of a track
// Each subscriber gets their own DownTrack (independent quality, NACK handling)
type DownTrack struct {
    params          DownTrackParams
    id              livekit.TrackID
    ssrc            uint32           // Unique SSRC for this down track
    ssrcRTX         uint32           // RTX SSRC for retransmissions
    
    receiver        TrackReceiver    // The published track source
    forwarder       *Forwarder       // Handles layer selection, header munging
    pacer           pacer.Pacer      // Controls send rate
    
    rtpStats        *rtpstats.RTPStatsSender
    connectionStats *connectionquality.ConnectionStats
}

// WriteRTP is called for each packet from the source track
func (d *DownTrack) WriteRTP(extPkt *buffer.ExtPacket, layer int32) int32 {
    if !d.writable.Load() {
        return 0
    }

    // 1. Get translation params (layer matching, header munging)
    tp, err := d.forwarder.GetTranslationParams(extPkt, layer)
    if tp.shouldDrop {
        return 0  // Skip this packet (wrong layer, etc.)
    }

    // 2. Build translated RTP header
    hdr := &rtp.Header{
        Version:        extPkt.Packet.Version,
        Marker:         tp.marker,
        PayloadType:    d.getTranslatedPayloadType(extPkt.Packet.PayloadType),
        SequenceNumber: uint16(tp.rtp.extSequenceNumber),  // Remapped!
        Timestamp:      uint32(tp.rtp.extTimestamp),       // Preserved
        SSRC:           d.ssrc,  // This DownTrack's unique SSRC
    }

    // 3. Add extensions (dependency descriptor, playout delay, etc.)
    if d.dependencyDescriptorExtID != 0 && tp.ddBytes != nil {
        hdr.SetExtension(uint8(d.dependencyDescriptorExtID), tp.ddBytes)
    }

    // 4. Cache for potential retransmission
    d.sequencer.push(extPkt.Arrival, extPkt.ExtSequenceNumber, ...)

    // 5. Enqueue for paced sending
    pacerPacket := &pacer.Packet{
        Header:  hdr,
        Payload: payload,
        WriteStream: d.writeStream,
    }
    d.pacer.Enqueue(pacerPacket)

    return 1  // Successfully queued
}

// handleRTCP processes feedback from subscriber
func (d *DownTrack) handleRTCP(bytes []byte) {
    pkts, _ := rtcp.Unmarshal(bytes)
    for _, pkt := range pkts {
        switch p := pkt.(type) {
        case *rtcp.ReceiverReport:
            for _, r := range p.Reports {
                if r.SSRC == d.ssrc {
                    rtt, _ := d.rtpStats.UpdateFromReceiverReport(r)
                    // Use RTT to adjust NACK timing
                }
            }
            
        case *rtcp.TransportLayerNack:
            // Subscriber requesting packet retransmission
            go d.retransmitPackets(p.Nacks)
            
        case *rtcp.PictureLossIndication:
            // Subscriber needs keyframe
            d.Receiver().SendPLI(layer, false)
        }
    }
}
```

## Buffer: Jitter Buffer and Packet Storage

From `pkg/sfu/buffer/buffer.go`:

```go
// Buffer contains all packets and handles reordering
type Buffer struct {
    *BufferBase

    twcc      *twcc.Responder  // Transport-wide congestion control
    twccExtID uint8

    // Callbacks
    onRtcpFeedback func([]rtcp.Packet)
}

// Write adds an RTP Packet - ordering is NOT guaranteed
func (b *Buffer) Write(pkt []byte) (n int, err error) {
    var rtpPacket rtp.Packet
    rtpPacket.Unmarshal(pkt)

    now := mono.UnixNano()
    
    // Handle TWCC for congestion control
    if b.twcc != nil && b.twccExtID != 0 {
        if ext := rtpPacket.GetExtension(b.twccExtID); ext != nil {
            b.twcc.Push(rtpPacket.SSRC, binary.BigEndian.Uint16(ext), now, rtpPacket.Marker)
        }
    }

    // Process packet: detect gaps, reorder, notify downstream
    b.calc(pkt, &rtpPacket, now, false, false)
    return
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
   640 bytes → ~70 bytes compressed
         │
         ▼
3. Browser creates RTP packet
   Header (12B) + Payload (70B) = 82 bytes
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
                              8. WebRTCReceiver.AddUpTrack()
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
                                    │    - WriteRTP() called
                                    │    - Translate headers (new SSRC)
                                    │    - pacer.Enqueue()
                                    │
                                    ├──► DownTrack (Charlie)
                                    │    - Independent quality/NACK
                                    │
                                    └──► DownTrack (Dana)
                                         - Independent quality/NACK
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
                                                              70B → 640B PCM
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
| **Sequence numbers increment** | Packets detected as duplicates, dropped | Gap detection in Buffer |
| **Timestamps monotonic** | Jitter buffer underrun, audio gaps | Negative delta check |
| **Payload type matches** | Codec mismatch, silence or noise | SDP negotiation failure |
| **ICE candidates valid** | Connection never establishes | ICE failed state |
| **SSRC is unique** | Packets misrouted to wrong track | Collision detection |

---

# 3️⃣ CTO View: System Design and Architecture

**Goal:** Understand architectural decisions, tradeoffs, and scalability.

## Why SFU Architecture?

LiveKit uses a **Selective Forwarding Unit (SFU)** instead of an **MCU (Multipoint Control Unit)**:

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
│                │  re-encodes)│                                      │
│                └─────────────┘                                      │
│                                                                     │
│  Server CPU: VERY HIGH (decode all, mix, re-encode per viewer)      │
│  Latency: +100-300ms (encoding delay)                               │
│  Scalability: ~10-20 participants per server                        │
│  Flexibility: Low (everyone sees same composition)                  │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  SFU (Selective Forwarding Unit) - LiveKit's Approach               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Alice ───┐                         ┌─── Alice receives: Bob, Charlie│
│           │    ┌─────────────┐      │    (separate streams)         │
│  Bob   ───┼───►│  SFU Server │──────┼─── Bob receives: Alice, Charlie│
│           │    │  (forwards  │      │    (separate streams)         │
│  Charlie ─┘    │   only, NO  │      └─── Charlie receives: Alice, Bob│
│                │   decoding) │                                       │
│                └─────────────┘                                       │
│                                                                     │
│  Server CPU: LOW (just routing packets)                             │
│  Latency: MINIMAL (no transcoding)                                  │
│  Scalability: 100-500+ participants per server                      │
│  Flexibility: HIGH (each client controls their view)                │
└─────────────────────────────────────────────────────────────────────┘
```

**Why LiveKit chose SFU:**
- **Scalability**: CPU usage is O(N), not O(N²)
- **Latency**: No encoding delay on server
- **Flexibility**: Each subscriber can choose quality independently
- **Cost**: Much cheaper compute per participant

## Multi-Node Architecture

From `pkg/routing/interfaces.go` and `pkg/routing/redisrouter.go`:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Global Architecture                            │
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

**From actual code** (`pkg/routing/interfaces.go`):

```go
// Router allows multiple nodes to coordinate the participant session
type Router interface {
    MessageRouter
    
    RegisterNode() error
    UnregisterNode() error
    RemoveDeadNodes() error
    
    ListNodes() ([]*livekit.Node, error)
    
    GetNodeForRoom(ctx context.Context, roomName livekit.RoomName) (*livekit.Node, error)
    SetNodeForRoom(ctx context.Context, roomName livekit.RoomName, nodeId livekit.NodeID) error
    ClearRoomState(ctx context.Context, roomName livekit.RoomName) error
}

// CreateRouter factory - uses Redis if available, else local
func CreateRouter(rc redis.UniversalClient, ...) Router {
    lr := NewLocalRouter(node, signalClient, roomManagerClient, nodeStatsConfig)
    
    if rc != nil {
        return NewRedisRouter(lr, rc, kps)  // Multi-node with Redis
    }
    
    logger.Infow("using single-node routing")  // Single-node fallback
    return lr
}
```

## Scalability Analysis

**Single Room Scaling:**

```
┌─────────────────────────────────────────────────────────────────────┐
│  10 participants, all publishing video (1.5 Mbps each)              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Publisher bandwidth (each): 1.5 Mbps × 1 = 1.5 Mbps                │
│  Total inbound to server: 10 × 1.5 = 15 Mbps                        │
│                                                                     │
│  Each participant subscribes to 9 others                            │
│  Per subscriber outbound: 9 × 1.5 = 13.5 Mbps                       │
│  Total outbound from server: 10 × 13.5 = 135 Mbps                   │
│                                                                     │
│  Total server bandwidth: 15 + 135 = 150 Mbps                        │
│  Server CPU: ~10-20% (mostly packet forwarding)                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  100 participants, 10 publishing video                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Inbound: 10 × 1.5 = 15 Mbps                                        │
│  Outbound: 100 × 10 × 1.5 = 1,500 Mbps (1.5 Gbps)                   │
│                                                                     │
│  With adaptive quality (StreamAllocator):                           │
│  - Active speaker: 720p (1.5 Mbps)                                  │
│  - Others: 180p thumbnails (0.2 Mbps)                               │
│  Optimized outbound: 100 × (1.5 + 9×0.2) = 330 Mbps                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Failure Modes and Recovery

```
┌─────────────────────────────────────────────────────────────────────┐
│  Failure: Publisher loses network                                   │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: ICE state change (from actual code: iceDisconnectedTimeout │
│             = 10 seconds, iceFailedTimeout = 5 seconds)              │
│  Recovery:                                                          │
│    1. Subscribers get "track muted" event                           │
│    2. Publisher reconnects via WebSocket                            │
│    3. ICE restart or new connection                                 │
│    4. Resume publishing from same track                             │
│  User impact: Audio/video freezes briefly, then resumes             │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Failure: LiveKit node crashes                                      │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: Redis heartbeat expires (via KeepalivePubSub)           │
│  Recovery:                                                          │
│    1. Redis marks node as dead                                      │
│    2. Clients auto-reconnect (SDKs handle this)                    │
│    3. Room migrates to new node                                     │
│    4. State restored via SetPreviousSdp() migration support        │
│  User impact: ~2-5 second interruption                              │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Failure: Network congestion (10%+ packet loss)                     │
├─────────────────────────────────────────────────────────────────────┤
│  Detection: RTCP receiver reports show high loss (in DownTrack)     │
│  Recovery:                                                          │
│    1. Send NACKs for important missing packets                      │
│    2. Request keyframe if video corruption (PLI)                   │
│    3. StreamAllocator switches subscriber to lower quality layer   │
│    4. BWE notifies publisher to reduce encoding quality            │
│  User impact: Quality degrades gracefully instead of freezing       │
└─────────────────────────────────────────────────────────────────────┘
```

## Latency vs Reliability Tradeoffs

LiveKit makes these tradeoffs configurable via `pkg/sfu/buffer/` and BWE:

| Strategy | Latency Impact | Reliability | When Used |
|----------|---------------|-------------|-----------|
| **Immediate forward** | Lowest | Accepts loss | Default for audio |
| **Small jitter buffer** | Very low | Handles reordering | Good networks |
| **Large jitter buffer** | Medium | Handles loss+reorder | Poor networks |
| **NACK retransmission** | Variable (+RTT) | High | When RTT is low |
| **FEC** | +10% bandwidth | Medium | Moderate loss |
| **Opus PLC** | None | Conceals 1-2 packets | Always for audio |

## Why Go? Language Choice and Tradeoffs

LiveKit is written in **Go** instead of C++, Rust, or Java. This wasn't arbitrary—it's a deliberate architectural decision optimized for LiveKit's specific workload.

### The Key Insight: SFUs Are I/O-Bound, Not CPU-Bound

```
┌─────────────────────────────────────────────────────────────────┐
│  WHERE DOES TIME GO IN A MEDIA SERVER (SFU)?                    │
└─────────────────────────────────────────────────────────────────┘

  ┌────────────────────────────────────────────────────────────┐
  │                                                            │
  │   Network I/O          ████████████████████████  85%       │
  │   (waiting for packets)                                    │
  │                                                            │
  │   Packet routing       ████                      10%       │
  │   (forwarding logic)                                       │
  │                                                            │
  │   CPU processing       █                          5%       │
  │   (actual computation)                                     │
  │                                                            │
  └────────────────────────────────────────────────────────────┘

  LiveKit does NOT encode/decode media (that's the CPU-heavy part)
  It just ROUTES packets from publisher → subscribers
  This is I/O-bound, not CPU-bound!
```

**Critical distinction:**
- **MCU (Multipoint Control Unit)**: Decodes, mixes, re-encodes → CPU-intensive → C++/Rust beneficial
- **SFU (Selective Forwarding Unit)**: Forwards packets as-is → I/O-intensive → Go is optimal

### What LiveKit Actually Does with Audio

```
  Publisher                    LiveKit (SFU)                 Subscribers
     │                              │                              │
     │   Opus packet ──────────────►│                              │
     │   (already encoded by        │                              │
     │    browser/client)           │                              │
     │                              │──── Forward same bytes ─────►│
     │                              │──── Forward same bytes ─────►│
     │                              │──── Forward same bytes ─────►│
     │                              │                              │
     
  NO TRANSCODING! Just forwarding bytes.
  The 10% performance gain from C++/Rust doesn't matter here.
```

### Language Comparison for SFU Workload

```
┌─────────────────────────────────────────────────────────────────┐
│  LANGUAGE TRADEOFFS FOR MEDIA SERVERS (SFU)                     │
└─────────────────────────────────────────────────────────────────┘

                    │ C++      │ Rust     │ Go       │ Java
────────────────────┼──────────┼──────────┼──────────┼──────────
Raw Performance     │ ★★★★★   │ ★★★★★   │ ★★★★☆   │ ★★★☆☆
Memory Safety       │ ★★☆☆☆   │ ★★★★★   │ ★★★★☆   │ ★★★★☆
Development Speed   │ ★★☆☆☆   │ ★★★☆☆   │ ★★★★★   │ ★★★★☆
Concurrency Model   │ ★★★☆☆   │ ★★★★☆   │ ★★★★★   │ ★★★☆☆
WebRTC Libraries    │ ★★★★★   │ ★★★☆☆   │ ★★★★★   │ ★★☆☆☆
Deployment          │ ★★★☆☆   │ ★★★★★   │ ★★★★★   │ ★★☆☆☆
Hiring/Maintenance  │ ★★☆☆☆   │ ★★★☆☆   │ ★★★★★   │ ★★★★★
────────────────────┴──────────┴──────────┴──────────┴──────────
```

### Why NOT Each Alternative?

**C++ - Too Dangerous, Too Slow to Develop:**
```
┌─────────────────────────────────────────────────────────────────┐
│  C++ DOWNSIDES                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ Memory bugs (use-after-free, buffer overflows)              │
│    → Security vulnerabilities in network-facing code           │
│                                                                 │
│  ✗ Much slower development (3-5x longer to write)              │
│                                                                 │
│  ✗ Complex build systems (CMake, cross-platform hell)          │
│                                                                 │
│  When C++ IS used:                                              │
│  → Encoding/decoding (FFmpeg, libvpx, libopus)                 │
│  → Browser WebRTC (Chrome's native implementation)             │
│  → When you need every last CPU cycle                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Rust - Great Language, But Ecosystem Wasn't Ready:**
```
┌─────────────────────────────────────────────────────────────────┐
│  RUST DOWNSIDES (for this use case, in 2020-2021)              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ No mature WebRTC library at the time                        │
│    → webrtc-rs existed but wasn't production-ready             │
│    → Pion (Go) was battle-tested in production                 │
│                                                                 │
│  ✗ Steeper learning curve                                      │
│    → Harder to hire, slower onboarding                         │
│                                                                 │
│  ✗ Slower iteration speed                                      │
│    → Fighting the borrow checker during rapid development      │
│                                                                 │
│  Note: Today Rust would be more viable (ecosystem matured)     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Java - Wrong Tool for Real-Time Systems:**
```
┌─────────────────────────────────────────────────────────────────┐
│  JAVA DOWNSIDES                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ✗ JVM startup time and memory overhead                        │
│    → Go binary: ~20MB, starts in milliseconds                  │
│    → Java: 200MB+ heap, slower cold start                      │
│                                                                 │
│  ✗ Garbage collection pauses                                   │
│    → Can cause latency spikes in real-time systems             │
│    → Go's GC is designed for low-latency (<1ms pauses)         │
│                                                                 │
│  ✗ No good WebRTC libraries in Java ecosystem                  │
│                                                                 │
│  ✗ Verbose code, slower iteration                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Why Go is Optimal for LiveKit

**1. Goroutines = Perfect for Network I/O**

Each connection needs its own processing loop. Go makes this trivial:

```go
// Handle 10,000+ concurrent connections easily
for {
    conn, _ := listener.Accept()
    go handleConnection(conn)  // Spawn lightweight goroutine
}

// Each goroutine uses ~2KB stack (vs ~1MB for OS thread)
// Can run millions of goroutines on one server
```

**2. Pion - Battle-Tested WebRTC in Go**

```
┌─────────────────────────────────────────────────────────────────┐
│  PION WebRTC Library                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  • Pure Go implementation of WebRTC (no C dependencies)        │
│  • Used in production by Discord, Cloudflare, and others       │
│  • Active development, excellent community                     │
│  • LiveKit is built directly on top of Pion                    │
│                                                                 │
│  This was THE deciding factor - Pion existed and worked.       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**3. Simple Deployment**

```
┌─────────────────────────────────────────────────────────────────┐
│  DEPLOYMENT COMPARISON                                          │
└─────────────────────────────────────────────────────────────────┘

  Go:
  ┌──────────────────────────────────────────────────────────────┐
  │  $ go build -o livekit-server                                │
  │  $ ./livekit-server                                          │
  │                                                              │
  │  Single binary, no dependencies, runs anywhere               │
  │  Docker image: ~20MB                                         │
  └──────────────────────────────────────────────────────────────┘

  Java:
  ┌──────────────────────────────────────────────────────────────┐
  │  $ mvn package                                               │
  │  $ java -jar -Xmx2g -XX:+UseG1GC ... server.jar             │
  │                                                              │
  │  Needs JVM, tuning, more memory                              │
  │  Docker image: 200MB+                                        │
  └──────────────────────────────────────────────────────────────┘

  C++:
  ┌──────────────────────────────────────────────────────────────┐
  │  $ cmake .. && make                                          │
  │  $ ./server  (hope you have the right .so files...)         │
  │                                                              │
  │  Dependency hell, different builds per platform              │
  └──────────────────────────────────────────────────────────────┘
```

### Performance Reality Check for Audio Streaming

For 16 kHz PCM audio specifically, here's why Go's performance is more than sufficient:

```
┌─────────────────────────────────────────────────────────────────┐
│  16 kHz AUDIO PERFORMANCE REQUIREMENTS                          │
└─────────────────────────────────────────────────────────────────┘

  Per audio stream:
  • 50 packets/second (one every 20ms)
  • ~82 bytes per packet (RTP header + Opus payload)
  • ~4 kbps per stream
  
  For 1000 concurrent audio streams:
  • 50,000 packets/second
  • 4 Mbps total bandwidth
  
  Go can easily handle:
  • 1,000,000+ packets/second per core
  • Network card becomes bottleneck before CPU
  
  ┌────────────────────────────────────────────────────────────┐
  │  The "10% faster" from C++/Rust doesn't matter when:       │
  │  • You're I/O bound, not CPU bound                         │
  │  • Network latency (50-200ms) is 100x your processing time │
  │  • Development speed and safety matter more                │
  └────────────────────────────────────────────────────────────┘
```

### What IS Written in C/C++ (and Should Be)

The CPU-intensive codec work is still done in C/C++, but on the **client side**:

```
┌─────────────────────────────────────────────────────────────────┐
│  COMPONENTS WRITTEN IN C/C++                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  libopus     → Audio encoding/decoding (Opus codec)            │
│  libvpx      → Video encoding/decoding (VP8/VP9)               │
│  libaom      → Video encoding (AV1)                            │
│  OpenH264    → Video encoding (H.264)                          │
│  libsrtp     → Encryption (SRTP)                               │
│                                                                 │
│  These run in the BROWSER or MOBILE APP (client-side)          │
│  The SFU server doesn't encode/decode - it just forwards!      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Summary: Go Tradeoff Decision

| Factor | Winner | Why It Matters for LiveKit |
|--------|--------|---------------------------|
| **Raw speed** | C++/Rust | Only 10-20% faster, not needed for I/O-bound SFU |
| **Development speed** | **Go** | 3-5x faster to iterate, ship features |
| **WebRTC ecosystem** | **Go (Pion)** | Battle-tested, production-ready |
| **Concurrency** | **Go** | Goroutines perfectly match connection-per-client model |
| **Memory safety** | Go/Rust | Prevents security vulnerabilities |
| **Deployment** | **Go** | Single binary, Docker-friendly |
| **Hiring** | **Go** | Easier to find devs than Rust/C++ |

**Bottom line:** For an I/O-bound SFU handling audio/video forwarding, Go's tradeoffs are nearly optimal. The 10% performance gain from Rust/C++ isn't worth the 3-5x slower development and harder maintenance.

---

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
│  Use Cases Requiring Real-Time Infrastructure                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ✓ Video Conferencing (Zoom/Meet alternative)                      │
│    - 500+ participants with adaptive quality                        │
│    - Sub-200ms latency for natural conversation                     │
│                                                                     │
│  ✓ Telehealth                                                       │
│    - HIPAA compliance possible (E2E encryption)                     │
│    - Reliable enough for medical consultations                      │
│                                                                     │
│  ✓ Gaming Voice Chat                                                │
│    - Low latency critical for competitive play                      │
│    - Scales with player count                                       │
│                                                                     │
│  ✓ AI-Powered Conversations (LiveKit Agents)                       │
│    - AI participant in room (speech-to-text, LLM, TTS)             │
│    - Real-time voice AI without custom infrastructure               │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  What Simpler Stacks CANNOT Do                                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Pure WebSocket / HTTP Streaming:                                   │
│  ✗ Latency 2-10 seconds (not conversational)                        │
│  ✗ Server must decode/encode all media (expensive)                  │
│  ✗ No NAT traversal (many users can't connect)                      │
│                                                                     │
│  Peer-to-Peer WebRTC:                                               │
│  ✗ N×N connections don't scale past 4-5 people                      │
│  ✗ No server-side recording                                         │
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

With adaptive quality (StreamAllocator):
  Reduced to ~200 GB/hour = $10-24/hour
```

**Total Cost of Ownership:**

| Scale | LiveKit Cloud | Self-Hosted |
|-------|--------------|-------------|
| **< 100 concurrent** | ~$0.005/min/participant = $20-50/month | 1-2 servers = $50-200/month |
| **100-500 concurrent** | $200-1,000/month | 2-4 servers + Redis = $200-500/month |
| **1000+ concurrent** | $1,000+/month | Self-hosting economical, need DevOps |

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Latency spikes** | Medium | User frustration | Multi-region, TURN, adaptive quality |
| **Node failure** | Low | Brief interruption | Auto-reconnect, migration (SetPreviousSdp) |
| **DDoS attack** | Low | Service outage | CDN, rate limiting, cloud protection |
| **Vendor lock-in** | N/A (open source) | None | Self-hostable, standard protocols |
| **Scaling ceiling** | Low | Can't grow | Add nodes, LiveKit Cloud handles |
| **Security breach** | Low | Data exposure | E2E encryption, JWT auth, audits |

## Strategic Value: LiveKit's Defensible Moats

1. **Open Source + Managed Service**
   - No vendor lock-in concerns
   - Start self-hosted, scale to managed (or vice versa)

2. **Protocol Expertise**
   - WebRTC is notoriously complex (ICE, DTLS, SRTP, SDP)
   - Years of edge case handling in Pion integration
   - Hard for competitors to replicate quickly

3. **Platform Ecosystem**
   - SDKs for JS, Swift, Kotlin, Flutter, React Native, Unity
   - Each SDK maintained and tested
   - Network effects: more users → more bug reports → better quality

4. **AI Integration (Agents Framework)**
   - First-mover in AI + real-time
   - Voice AI in production today
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
│  ✓ Want control over infrastructure (self-hostable)                │
│  ✓ Building AI-powered voice/video features                        │
│                                                                     │
│  Consider alternatives when:                                        │
│  ✓ Only 1:1 calls → Simpler WebRTC (PeerJS, SimpleWebRTC)          │
│  ✓ Broadcast only → HLS/DASH streaming services                    │
│  ✓ Low volume, non-critical → Simpler SaaS (Daily, Whereby)        │
│  ✓ Already deep in AWS → Chime SDK might integrate better          │
└─────────────────────────────────────────────────────────────────────┘
```

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
└─────────────┴───────────────────────────────────────────────────────┘
```

---

# Summary: The Complete 16 kHz Audio Journey

```
┌─────────────────────────────────────────────────────────────────────┐
│                    End-to-End Journey                               │
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
│  PCM (640 bytes) → Opus Encoder (70 bytes) → RTP Header (12 bytes) │
│  WebSocket: JWT auth, SDP offer/answer, ICE candidates             │
│  WebRTC: ICE → DTLS → SRTP → RTP packets over UDP                  │
│                                                                     │
│  LiveKit: RTCService → Room → Participant → PCTransport            │
│           WebRTCReceiver → Buffer → DownTrack → Subscriber         │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
LAYER 3 - ARCHITECTURE (CTO View)
┌─────────────────────────────────────────────────────────────────────┐
│  SFU architecture: Forward, don't mix (O(N) not O(N²))             │
│  Why Go: I/O-bound workload, Pion WebRTC, goroutines, fast deploy  │
│  Multi-node: Redis coordination, room affinity, migration          │
│  Reliability: NACK, FEC, adaptive jitter buffer, PLC               │
│  Scalability: Add nodes, StreamAllocator, per-subscriber quality   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
LAYER 4 - BUSINESS (CEO View)
┌─────────────────────────────────────────────────────────────────────┐
│  Enables: Video conferencing, telehealth, gaming, AI voice         │
│  Economics: Bandwidth is main cost, self-host at scale             │
│  Competitive: Open source differentiator, AI integration           │
│  Risk: Mitigated by standards compliance, multi-region, E2E encrypt│
└─────────────────────────────────────────────────────────────────────┘

Total latency: < 150ms (sound waves to eardrums across the internet)
```

---

# Key Takeaways by Audience

| Audience | Key Understanding |
|----------|------------------|
| **Intern** | Sound → numbers → compression → packets → network → sound. Latency matters. |
| **Engineer** | RTP/RTCP protocols, dual WebSocket+WebRTC paths, NACK/FEC reliability, Go codebase with Pion |
| **CTO** | SFU scales, Go chosen because I/O-bound (not CPU-bound), Pion WebRTC mature, bandwidth dominates cost |
| **CEO** | Enables $B market (video calls), open source = no lock-in, AI integration is differentiator |

---

# Appendix: Key Code Locations

| Component | File | Purpose |
|-----------|------|---------|
| WebSocket signaling | `pkg/service/rtcservice.go` | Handles client connections, JWT auth |
| PeerConnection wrapper | `pkg/rtc/transport.go` | Manages WebRTC transport (PCTransport) |
| Receive from publisher | `pkg/sfu/receiver.go` | WebRTCReceiver for incoming RTP |
| Send to subscriber | `pkg/sfu/downtrack.go` | DownTrack per subscriber |
| Packet buffering | `pkg/sfu/buffer/buffer.go` | Jitter buffer, reordering |
| Multi-node routing | `pkg/routing/redisrouter.go` | Redis-based room coordination |
| Node selection | `pkg/routing/selector/` | CPU load, region-aware selection |
| Bandwidth estimation | `pkg/sfu/bwe/` | REMB, TWCC feedback processing |
| Stream allocation | `pkg/sfu/streamallocator/` | Adaptive quality per subscriber |
