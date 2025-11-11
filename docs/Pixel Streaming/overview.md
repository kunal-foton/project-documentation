---
title: overview
sidebar_position: 0 
---

This is the core browser-side class for Pixel Streaming applications that stream real-time content from Unreal
Engine to web browsers via WebRTC. It acts as a central orchestrator managing connections, input, video quality,
and communication between the browser and UE application.

Key Components

1. Core Controllers (lines 66-68)

- WebRtcPlayerController: Manages WebRTC peer connections and streaming
- WebXRController: Handles VR/AR functionality
- DataChannelLatencyTestController: Tests network latency

2. Configuration System (lines 129-268)

The class sets up extensive configuration listeners that:
- Monitor quality parameters (MinQP, MaxQP, MinQuality, MaxQuality)
- Handle input settings (keyboard, mouse, touch, gamepad)
- Manage WebRTC settings (bitrate, FPS)
- Support both legacy QP (Quantization Parameter) and newer quality factor systems
- Convert between quality percentages (0-100) and QP values (0-51)

3. Connection Management (lines 297-337)

- connect(): Initiates signaling server connection
- reconnect(): Tears down and re-establishes connection
- disconnect(): Closes connections cleanly
- play(): Starts video stream playback
- Auto-connect support for seamless startup


4. Media Device Control (lines 347-435)

Manages browser-to-UE media streaming:
- Microphone: muteMicrophone(), unmuteMicrophone()
- Camera: muteCamera(), unmuteCamera()
- Both support forceEnable parameter to reconnect with new media tracks

5. Initial Settings Handling (lines 565-655)

When connecting to UE, receives and processes:
- Encoder settings (QP ranges, quality factors)
- WebRTC parameters (bitrate limits, FPS)
- Console command permissions
- URL parameter overrides (URL params take precedence over UE defaults)

6. Event System (lines 438-551)

Emits events for lifecycle stages:
- Connection states: webRtcAutoConnect, webRtcConnecting, webRtcConnected, webRtcFailed, webRtcDisconnected
- Stream events: streamLoading, videoInitialized
- Performance: statsReceived, latencyCalculated, videoEncoderAvgQP
- SDP negotiation: webRtcSdp, webRtcSdpOffer, webRtcSdpAnswer

7. Network Diagnostics (lines 673-698)

_setupWebRtcTCPRelayDetection: Detects if connection is using TCP relay (TURN) instead of direct connection, which
indicates suboptimal network conditions.

8. Communication with UE (lines 705-825)

Multiple methods to interact with the UE application:
- emitUIInteraction(): Send arbitrary data to UE
- emitCommand(): Send structured commands
- emitConsoleCommand(): Execute UE console commands (security-gated)
- sendTextboxEntry(): Update UE text widgets
- requestLatencyTest(), requestShowFps(), requestIframe()
- addResponseEventListener(): Handle UE → browser messages


9. Quality Control (lines 131-232)

Sophisticated quality management:
- Dynamic QP adjustment (affects video compression)
- Quality factor settings (newer, more intuitive 0-100 scale)
- Backward compatibility between systems via CompatQualityMin/Max
- Instant propagation of changes to UE via data channel

10. Message Registration System (lines 901-925)

registerMessageHandler(): Allows custom bidirectional message protocols between browser and UE streamer.

Architecture Insights

Design Pattern: This follows a facade pattern - it provides a simplified interface to complex subsystems (WebRTC,
signaling, input handling, XR).

Event-Driven: Heavy use of event emitters allows decoupled components and easy extension.

Security: Console command execution is gated by UE permission flag (allowConsoleCommands), preventing arbitrary
code execution.

Flexibility:
- URL parameters can override UE settings (line 576-633)
- Custom signaling URL builder support (line 878)
- Extensible message handlers (line 901-925)

Backwards Compatibility: Maintains both QP-based (0-51) and quality-factor (0-100) systems, converting between
them automatically.

Key Workflows

1. Connection: Constructor → configureSettings() → setWebRtcPlayerController() → checkForAutoConnect() → connect()
→ play()
2. Quality Adjustment: User changes setting → Config listener fires → Send command to UE → UE adjusts encoder →
Stats received → UI updated
3. Reconnection: Network issue → Disconnect event → reconnect() → Re-establish connection with same settings

