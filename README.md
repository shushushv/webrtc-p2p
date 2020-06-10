```
npm install                 // 安装依赖

npm run dev                 // 访问 localhost:8080
```

# 50行代码完成视频通话 (WebRTC + WebSocket)
## 前言
>*“它（WebRTC）允许网络应用或者站点，在不借助中间媒介的情况下，建立浏览器之间点对点（Peer-to-Peer）的连接，实现视频流和（或）音频流或者其他任意数据的传输”*。

这是 MDN 上对 WebRTC 的描述，初次接触时无法理解 WebRTC 为什么要和 WebSocket 搭配，明明说的很清楚 **不借助中间媒介** ，那 WebSocket 充当的是什么角色？整个 WebRTC 通话建立的流程又是怎样的？

## 开始

**先看一下最终效果**：

![](https://image-static.segmentfault.com/140/741/140741630-5db028fbdabca_articlex)

### 1.相关技术

本示例主要使用了 `WebRTC` 和 `WebSocket`：
- `WebRTC`（Web Real-Time Communication）即网页即时通信，是一个支持网页浏览器进行实时语音对话或视频对话的API。
- `WebSocket`是一种在单个TCP连接上进行全双工通信的协议。在 WebSocket 中，浏览器和服务器只需要完成一次握手，两者之间就直接可以创建持久性的连接，并进行双向数据传输。

### 2.通话建立流程
简单说一下流程，如浏览器 A 想和浏览器 B 进行音视频通话：

 1. A、B 都连接信令服务器（ws）；
 2. A 创建本地视频，并获取会话描述对象（`offer sdp`）信息；
 3. A 将 `offer sdp` 通过 ws 发送给 B；
 4. B 收到信令后，B 创建本地视频，并获取会话描述对象（`answer sdp`）信息；
 5. B 将 `answer sdp` 通过 ws 发送给 A；
 6. A 和 B 开始打洞，收集并通过 ws 交换 ice 信息；
 7. 完成打洞后，A 和 B 开始为安全的媒体通信协商秘钥；
 8. 至此， A 和 B 可以进行音视频通话。

引用网上的有关`WebRTC`建立的时序图，可能更加直观：

![](https://user-gold-cdn.xitu.io/2019/10/23/16df7589c2b09544?w=813&h=708&f=png&s=165081)

从上述流程，可以发现**通信双方在建立连接前需要交换信息**，这也就是开头提到的 `WebSocket` 充当的角色：信令服务器，用于转发信息。而 WebRTC **不借助中间媒介** 的意思是，在建立对等连接后，不需要借助第三方服务器中转，而是直接在两个实体（浏览器）间进行传输。
### 3.代码

#### 第一步
获取视频标签，连接信令服务器，创建 `RTCPeerConnection` 对象。其中 [RTCPeerConnection]('https://developer.mozilla.org/zh-CN/docs/Web/API/RTCPeerConnection') 的作用是在两个对等端之间建立连接，其构造函数支持传一个配置对象，包含ICE“打洞”（由于本示例在本机进行测试，故不需要）。

```
const localVideo = document.querySelector('#local-video');
const remoteVideo = document.querySelector('#remote-video');
const socket = new WebSocket('ws://localhost:8080');
const peer = new RTCPeerConnection();

socket.onmessage = () => { // todo }
peer.ontrack = () => { // todo }
peer.onicecandidate = () => { // todo }
```

#### 第二步
获取本地摄像头/麦克风（需要允许使用权限），拿到本地媒体流（[MediaStream](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaStream)）后，需要将其中所有媒体轨道（[MediaStreamTrack](https://developer.mozilla.org/zh-CN/docs/Web/API/MediaStreamTrack)）添加到轨道集，这些轨道将被发送到另一对等方。

```
navigator.mediaDevices.getUserMedia({ video: true, audio: true })
	.then(stream => {
		localVideo.srcObject = stream;
		stream.getTracks().forEach(track => {
			peer.addTrack(track, stream);
		});
	});
```

#### 第三步
创建发起方会话描述对象（[createOffer](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/createOffer)），设置本地SDP（[setLocalDescription](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/setLocalDescription)），并通过信令服务器发送到对等端，以启动与远程对等端的新WebRTC连接。

```
peer.createOffer().then(offer => {
	peer.setLocalDescription(offer);
	socket.send(JSON.stringify(offer));
});
```

*当调用 setLocalDescription 方法，PeerConnection 开始收集候选人（ice信息），并发送**offer_ice**到对等方。这边补充第一步中的`peer.onicecandidate`和`socket.onmessage`*

*对等方收到ice信息后，通过调用 [addIceCandidate](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/addIceCandidate) 将接收的候选者信息传递给浏览器的ICE代理。*

```
peer.onicecandidate = e => {
	if (e.candidate) {
		socket.send(JSON.stringify({
			type: 'offer_ice',
			iceCandidate: e.candidate
		}));
	} 
};

socket.onmessage = e => {
	const { type, sdp, iceCandidate } = JSON.parse(e.data);
	if (type === 'offer_ice') {
		peer.addIceCandidate(iceCandidate);
	}
}
```

#### 第四步
接收方收到了`offer`信令后，开始获取摄像头/麦克风，与发起方操作一致。同时将收到`offer SDP`指定为连接的远程对等方属性（[setRemoteDescription](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/setRemoteDescription)），并创建应答SDP（[createAnswer](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/createAnswer)），发送到对等端。这边补充第一步中的`socket.onmessage`。

```
socket.onmessage = e => {
	const { type, sdp, iceCandidate } = JSON.parse(e.data);
	if (type === 'offer') {
		navigator.mediaDevices.getUserMedia();		// 与发起方一致，省略
		const offerSdp = new RTCSessionDescription({ type, sdp });
		peer.setRemoteDescription(offerSdp).then(() => {
			peer.createAnswer(answer => {
				socket.send(JSON.stringify(answer));
				peer.setLocalDescription(answer)
			});
		});
	}
}
```
*注意：当 setLocalDescription 方法调用后，开始收集候选人信息，并发送 **answer_ice** 到对等方。与发送方同理，不赘述。*

#### 第五步
通过不断收集ICE信息（[onicecandidate](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/onicecandidate)），发起方和应答方最终将建立一条最优的连接方式，此时会触发 [ontrack](https://developer.mozilla.org/en-US/docs/Web/API/RTCPeerConnection/ontrack) 回调，即可获取到对等方的媒体流。

```
peer.ontrack = e => {
	if (e && e.streams) {
		remoteVideo.srcObject = e.streams[0];
	}
};
```
### 4.最后

整合发起方和应答方的代码，差不多50行，不算标题党！哈哈哈哈哈哈...

![](https://user-gold-cdn.xitu.io/2019/10/23/16df7e38e3edbe0c?w=190&h=190&f=png&s=24880)

完整示例相关代码已上传 [github.com/shushushv/webrtc-p2p](https://github.com/shushushv/webrtc-p2p) ，⭐🦆~
