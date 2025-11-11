---
title: Improvements
sidebar_position: 1
---

## 1. Performance Optimizations

### A. Cache Transceiver References (lines 380-386, 429-435)

**Issue:** Every time you mute/unmute mic or camera, it loops through ALL transceivers.

```typescript
// Current - inefficient
private setMicrophoneMuted(mute: boolean): void {
    for (const transceiver of this._webRtcController?.peerConnectionController?.peerConnection?.getTransceivers() ?? []) {
        if (RTCUtils.canTransceiverSendAudio(transceiver)) {
            transceiver.sender.track.enabled = !mute;
        }
    }
}
```

**Improvement:** Cache audio/video transceivers on connection:

```typescript
private _audioTransceiver?: RTCRtpTransceiver;
private _videoTransceiver?: RTCRtpTransceiver;

// Cache on connection
private cacheMediaTransceivers(): void {
    const transceivers = this._webRtcController?.peerConnectionController?.peerConnection?.getTransceivers() ?? [];
    this._audioTransceiver = transceivers.find(t => RTCUtils.canTransceiverSendAudio(t));
    this._videoTransceiver = transceivers.find(t => RTCUtils.canTransceiverSendVideo(t));
}

private setMicrophoneMuted(mute: boolean): void {
    if (this._audioTransceiver?.sender.track) {
        this._audioTransceiver.sender.track.enabled = !mute;
    }
}
```

### B. Event Listener Cleanup Issue (lines 106-112)

**Issue:** _setupWebRtcTCPRelayDetection adds itself as listener every time webRTC connects, potential memory leak on reconnections.

**Fix:** Remove listener properly or use once pattern:

```typescript
this._eventEmitter.addEventListener('webRtcConnected', (_: WebRtcConnectedEvent) => {
    // Use { once: true } if EventEmitter supports it, or ensure proper cleanup
    this._eventEmitter.addEventListener('statsReceived', this._setupWebRtcTCPRelayDetection, { once: false });
}, { once: true });
```


## 2. Error Handling & Robustness

### A. Add Validation and Better Error Messages

```typescript
public connect() {
    if (!this._webRtcController) {
        Logger.Error('Cannot connect: WebRTC controller not initialized');
        return;
    }

    if (this._webRtcController.isConnected()) {
        Logger.Warning('Already connected. Use reconnect() to re-establish connection.');
        return;
    }

    this._eventEmitter.dispatchEvent(new StreamPreConnectEvent());
    this._webRtcController.connectToSignallingServer();
}
```

### B. Return Error Details Instead of Just Boolean

```typescript
// Current
public emitUIInteraction(descriptor: object | string): boolean {
    if (!this._webRtcController.videoPlayer.isVideoReady()) {
        return false; // Why did it fail?
    }
    this._webRtcController.emitUIInteraction(descriptor);
    return true;
}

// Better
public emitUIInteraction(descriptor: object | string): { success: boolean; error?: string } {
    if (!this._webRtcController.videoPlayer.isVideoReady()) {
        return { success: false, error: 'Video not ready' };
    }
    this._webRtcController.emitUIInteraction(descriptor);
    return { success: true };
}
```

## 3. Memory Management

### A. Add Disposal/Cleanup Method

```typescript
public dispose(): void {
    // Clean up event listeners
    this._eventEmitter.removeEventListener('webRtcConnected', this._onWebRtcConnectedHandler);
    this._eventEmitter.removeEventListener('statsReceived', this._setupWebRtcTCPRelayDetection);

    // Dispose controllers
    this._webRtcController?.dispose();
    this._webXrController?.dispose();
    this._dataChannelLatencyTestController?.dispose();

    // Clear cached references
    this._audioTransceiver = undefined;
    this._videoTransceiver = undefined;
    this._videoElementParent = undefined;
}
```

## 4. Architecture Improvements

### A. Use Dependency Injection

  Dependency Injection is a design pattern where instead of a class creating its own dependencies (the objects it needs), those dependencies are "injected" (passed in) from the outside.

```typescript
// Current - tight coupling
constructor(config: Config, overrides?: PixelStreamingOverrides) {
    this._webRtcController = new WebRtcPlayerController(this.config, this);
    this._webXrController = new WebXRController(this._webRtcController);
}

// Better - injectable for testing
constructor(
    config: Config,
    overrides?: PixelStreamingOverrides,
    dependencies?: {
        webRtcController?: WebRtcPlayerController;
        webXrController?: WebXRController;
    }
) {
    this._webRtcController = dependencies?.webRtcController ??
        new WebRtcPlayerController(this.config, this);
    this._webXrController = dependencies?.webXrController ??
        new WebXRController(this._webRtcController);
}
```

