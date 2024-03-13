# WebRTC-RtpTransport Explainer

## Problem and Motivation

Today realtime communications and streaming technologies are converging within
ultra low latency use cases such as augmented reality/virtual reality,
game streaming or live streaming with fanout. These use cases may involve
peer-to-peer interactions traversing Network Address Translators (NATs)
so that they cannot be addressed solely with client/server APIs. 
For example, streaming a game from a console to a mobile device
or a live streaming application supporting peer-to-peer fanout to
improve scalability.

For these applications, today's WebRTC APIs may not be sufficient, due to: 

-- Lack of support for custom metadata. In AR/VR applications, the metadata
can be large enough to require custom packetization and rate control. 

-- Lack of codec support. For music, the AAC codec is popular,
but it is not supported in WebRTC implementations. However,
[AAC](https://www.w3.org/TR/webcodecs-aac-codec-registration/) is supported in WebCodecs.

-- Lack of custom rate control. While WebRTC's built-in rate control is
general purpose, it does not allow for rapid response to changes in bandwidth,
as is possible with [per-frame QP rate control in WebCodecs](https://docs.google.com/presentation/d/1FpCAlxvRuC0e52JrthMkx-ILklB5eHszbk8D3FIuSZ0/edit#slide=id.g2452ff65d17_0_1).

-- Inability to support custom RTCP messages. WebRTC implementations today do
not support feedback messages such as [LRR](https://datatracker.ietf.org/doc/html/draft-ietf-avtext-lrr/), [RPSI](https://datatracker.ietf.org/doc/html/rfc4585#page-39) or [SLI](https://datatracker.ietf.org/doc/html/rfc4585#page-37), or extended statistics as provided by RTCP-XR. 
 
Native applications can use raw UDP sockets, but those are not available on the
web because they lack encryption, congestion control, and a mechanism for
consent to send (to prevent DDoS attacks).

To enable new use cases, we think it would be useful to provide an API to
send and receive RTP and RTCP packets. 

## Goals

The WebRTC-RtpTransport API enables web applications to support: 

- Custom payloads (ML-based audio codecs)
- Custom packetization
- Custom FEC 
- Custom RTX
- Custom Jitter Buffer 
- Custom bandwidth estimate
- Custom rate control (with built-in bandwidth estimate)
- Custom bitrate allocation
- Custom metadata (header extensions)
- Custom RTCP messages
- Custom RTCP message timing
- RTP forwarding

## Non-goals

This is not [UDP Socket API](https://www.w3.org/TR/raw-sockets/).  We must have
encrypted and congestion-controlled communication.

## Key use-cases

WebRTC-RtpTransport can be used to implement the following WebRTC Extended Use Cases: 

- [Section 2.3](https://www.w3.org/TR/webrtc-nv-use-cases/#videoconferencing*): Video Conferencing with a Central Server
- [Section 3.2.1](https://www.w3.org/TR/webrtc-nv-use-cases/#game-streaming): Game streaming
- [Section 3.2.2](https://www.w3.org/TR/webrtc-nv-use-cases/#auction): Low latency Broadcast with Fanout
- [Section 3.5](https://www.w3.org/TR/webrtc-nv-use-cases/#vr*): Virtual Reality Gaming

WebRTC-RtpTransport enables these use cases by enabling applications to:

- Encode with a custom (WASM) codec, packetize and send
- Obtain frames from Encoded Transform API, packetize and send
- Obtain frames from Encoded Transform API, apply custom FEC, and send
- Observe incoming NACKs and resend with custom RTX behavior
- Observe incoming packets and customize when NACKs are sent
- Receive packets using a custom jitter buffer implementation
- Use WebCodecs for encode or decode, implement packetization/depacketization and a custom jitter buffer
- Receive packets, depacketize and inject into Encoded Transform (requires a constructor for EncodedAudioFrame/EncodedVideoFrame)
- Observe incoming feedback and/or estimations from built-in congestion control and implement custom rate control (as long as the sending rate is lower than the bandwidth estimate provided by built-in congestion control)
- Obtain frames from Encoded Transform API, packetize, attach custom metadata, and send
- Obtain a bandwidth estimate from RtpTransport, do bitrate allocation, and set bitrates of RtpSenders
- Forward RTP/RTCP packets from one PeerConnection to another, with full control over the entire packet (modulo SRTP/CC exceptions)

## API Outline 

### RtpPacket, RtcpPacket, RtxPacket, RtpHeaderExtension

```javascript
interface RtpPacket {
  constructor(required RtpPacketInit);
  readonly attribute bool marker;
  readonly attribute octet payloadType;
  readonly attribute unsigned short sequenceNumber;
  readonly attribute unsigned long timestamp;
  readonly attribute unsigned long ssrc;
  readonly attribute sequence<unsigned long> csrcs;
  readonly attribute sequence<RtpHeaderExtension> headerExtensions;
  readonly attribute ArrayBuffer payload;

  // Duplicate with header extensions, but conveniently parsed
  readonly attribute DOMString? mid;
  readonly attribute DOMString? rid;
  readonly attribute octet? audioLevel;  
  readonly attribute octet? videoRotation;
  readonly attribute unsigned long long? remoteSendTimestamp;

  // Extra information that may be useful to know
  readonly attribute DOMHighResTimeStamp receivedTime;
  readonly attribute unsigned long sequenceNumberRolloverCount;
}

dictionary RtpPacketInit {
  bool marker = false;
  reuired octet payloadType;
  unsigned short sequenceNumber;  // optional for sendRtp
  required unsigned long timestamp;
  sequence<unsigned long> csrcs = [];
  // Cannot be MID, RID, or congestion control sequence number
  sequence<RtpHeaderExtensionInit> headerExtensions = [];
  required ArrayBuffer payload;

  // Convenience for adding to headerExtensions
  octet audioLevel;    // optional
  octet videoRotation;  // optional
}

interface RtcpPacket {
  constructor(required RtcpPacketInit);
  readonly attribute octet type;
  readonly attribute octet subType;
  readonly attribute ArrayBuffer value;  
}

dictionary RtcpPacketInit {
  required octet type;  // TODO: Should we force the type APP?
  required octet subType;  // AKA FMT
  required ArrayBuffer value;  
}

interface RtxPacket {
  constructor(required RtxPacketInit, RtpPacket);
  readonly attribute octet payloadType;
  readonly attribute unsigned short sequenceNumber;
  readonly attribute unsigned long ssrc;
  readonly attribute RtpPacket originalRtp;  
}

dictionary RtxPacketInit {
  required octet payloadType;
  required unsigned short sequenceNumber;
  required unsigned long ssrc;
}

interface RtpHeaderExtension {
  constructor(required RtpHeaderExtensionInit);
  readonly attribute DOMString uri;
  readonly attribute ArrayBuffer value;
}

dictionary RtpHeaderExtensionInit {
  required DOMString uri;
  required ArrayBuffer value;
}
```
### RTCPeerConnection, RTCRtpSender, RTCRtpReceiver Extensions

```javascript
partial interface PeerConnection {
  // There may be an RtpTransport with no RtpSenders and no RtpReceivers
  readonly attribute sequence<RtpTransport> getRtpTransports();
}
partial interface RtpSender {
  // shared between RtpSenders in the same BUNDLE group
  readonly attribute RtpTransport? rtpTransport;
  sequence<RtpSendStream> replaceSendStreams();
}
partial interface RtpReceiver {
  // shared between RtpSenders in the same BUNDLE group
  readonly attribute RtpTransport? rtpTransport;
  sequence<RtpReceiveStream> replaceReceiveStreams();
}

interface RtpTransport {
  // For custom RTCP
  Promise<RtpSendStream> createRtpSendStream(RtpSendStreamInit);
  Promise<RtpReceiveStream> createRtpReceiveStream(RtpReceiveStreamInit);
}

[Exposed=(Window,Worker), Transferable]
interface RtpSendStream {
  readonly attribute DOMString mid?;  // Shared among RtpSendStreams
  readonly attribute DOMString rid?;  // Unique (scoped to MID)
  readonly attribute unsigned long ssrc;
  readonly attribute unsigned long rtxSsrc;

  // Goes to the network
  Promise<RtpSent> sendRtp(RtpPacketInit packet, optional RtpSendOptions options);
  void sendBye();
  
  // Comes from the network
  attribute EventHandler onreceivepli;  // Cancellable
  attribute EventHandler onreceivefir;  // Cancellable
  attribute EventHandler onreceivenack; // sequence<unsigned short>
  attribute EventHandler onreceivebye;

  // not needed if we go with frame-level APIs instead
  attribute EventHandler onrtppacketized;
  sequence<RtpPacket> readPacketizedRtp(maxNumberOfPackets);

  // If browser-owned, goes to the browser, 
  void receivePli();
  void receiveFir();
  void receiveNack(sequence<unsigned short>);
  void receiveBye();

  // If browser-owned, amount the browser expects to use for this RID
  readonly attribute unsigned long allocatedBandwidth;

  // If app-owned, de-allocates the MID, RID, and SSRC
  void close();
}

[Exposed=(Window,Worker), Transferable]
interface RtpReceiveStream {
  readonly attribute DOMString mid?;
  readonly attribute DOMString rid?;
  readonly attribute sequence<unsigned long> ssrcs;
  readonly attribute sequence<unsigned long> rtxSsrcs;

  attribute EventHandler onrtpreceived;
  sequence<RtpPacket> readReceivedRtp(maxNumberOfPackets);

  // Goes to the network
  void sendPli();
  void sendFir();
  void sendNack(sequence<unsigned short>);

  // If browser-owned Comes from the browser;  Cancellable
  attribute EventHandler onsendpli; // cancellable
  attribute EventHandler onsendfir; // cancellable
  attribute EventHandler onsendnack; // sequence<unsigned short> cancellable

  // not needed if we go with frame-level APIs instead
  void depacketizeRtp(RtpPacketInit packet);

  // If app-owned, de-allocates the MID, RID, and SSRC
  void close();
}

```

## Proposed solutions

## Example: Send with custom packetization

```javascript
const pc = new RTCPeerConnection({encodedInsertableStreams: true});
const rtpTransport = pc.createRtpTransport();
pc.getSenders().forEach((sender) => {
  pc.createEncodedStreams().readable.
      pipeThrough(createPacketizingTransformer()).pipeTo(rtpTransport.writable);
});

function createPacketizingTransformer() {
  return new TransformStream({
    async transform(encodedFrame, controller) {
      let rtpPackets = myPacketizer.packetize(frame);
      rtpPackets.forEach(controller.enqueue);
    }
  });
}
```

## Example: Receive with custom packetization

```javascript
const pc = new RTCPeerConnection({encodedInsertableStreams: true});
const rtpTransport = pc.createRtpTransport();
receiver.ontrack = event => {
  const esWriter = event.receiver.createEncodedStreams().writable.getWriter();
  rtpTransport.onrtppacket = (rtpPacket) => {
    let {vBuffer, esWriter} = receivers[getUniqueStreamIdentifier(rtpPacket)];
    vBuffer.insertPacket(rtpPacket);
    // Requires a constructor for EncodedVideoFrame/EncodedAudioFrame
    while (vBuffer.nextFrameReady()) esWriter.write(vBuffer.getFrame());
  }
}
```

## Example: Receive with custom jitter buffer and built-in depacketization

```javascript
const receiver = new RTCPeerConnection({encodedInsertableStreams: true});
receiver.ontrack = e => {
  if (e.track.kind == "video") {
    const es = event.receiver.createEncodedStreams({jitterBuffer: false});
    receiveVideo(es.readable.getReader(), es.writable.getWriter());
  }
  else {
    const es = event.receiver.createEncodedStreams();
    receiveAudio(es.readable.getReader(), es.writable.getWriter());
  }
}
function receiveVideo(reader, writer) {
  while (true) {
    const {value: frame, done} = await reader.read();
    if (done) return;
    vBuffer.insertFrame(frame);
    while (vBuffer.nextFrameReady()) writer.write(vBuffer.getFrame());
  }
}
```

## Example: Custom bitrate allocation

```javascript
const pc = new RTCPeerConnection();
const rtpTransport = pc.createRtpTransport();
rtpTransport.ontargetsendratechanged = () => {
  const rtpSender = pc.getTransceivers()[0];
  const parameters = rtpSender.getParameters();
  parameters.encodings[0].maxBitrate = rtpTransport.targetSendRate;
  rtpSender.setParameters(parameters);
};
```

## Alternative designs considered