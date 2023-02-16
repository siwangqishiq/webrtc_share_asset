## WebRtc分享

#### 音视频通话需要解决的问题 
    音视频简化版通话流程 
        场景 : 客户端A 向 客户端B 发起音视频通话
        通话流程 : 
                        F1: 请求通话
            A ------------------[Server]-------------------> B

                        F2: 接收邀请
            A <-----------------[Server]-------------------- B

                        F3: A B 端各自开启本地音视频硬件
            A............................................... B

                        F4: A B 端获取对端网络地址 
                            协商此次通讯采用何种编解码方案   
            A <------------------[Server]------------------> B

                        F5: A B 端开始采集原始音视频数据
            A .............................................. B

                        F6: A B 讲各自采集的数据编码
                            经直连网络发送给对端
                        原始数据 -> [编码器] ->编码后数据
            A [编解码器]<==========================>[编解码器] B

                        F7: A B端接收到对端数据后
                            经过解码 还原出音视频数据
                            View系统渲染
                            Voice系统播放
                        解码后的数据 -> [解码器] ->原始数据
            A [编解码器]<==========================>[编解码器] B

                        F8: A B通话结束 各自关闭设备
            A <------------------[Server]-------------------> B


    网络
        两客户端如何建立通话
        数据如何传输 

    多媒体
        音频视频数据采集
        音视频数据发送端编码
        音视频数据接收端解码

    
#### WebRtc介绍
    一套客户端API 用于辅助客户端完成音视频通话的开发
    
    解决上述通话流程中的两个问题
        1.媒体协商
            客户端通过SDP协议 互相告知对方支持的编解码格式等信息

        2.两客户端网络直连
            A -> B Socket网络直连 NAT穿透

主要API
```js 
    //音视频设备的管理
    MediaDevices.getUserMedia() 

    //WebRtc会话的建立与控制
    RTCPeerConnection()
```

#### WebRtc通话流程
    准备
        服务端:
            信令服务器 建立会话 交换数据
            ICE服务器 NAT穿透  
        客户端:
            集成各自平台对应的WebRTC SDK
            与信令服务器维持一条长连接 WebSocket / IM SDK均可 用于发送及接收信令服务的数据

    
##### A向B发起音视频通话

总体流程图

            [信令服务]                  [ICE服务]    
                |                           |
    A...........|...........................|............B      //初始状态 AB 都连接上信令服务
                |                           |
    A---------->|                           |                   //A端创建Offer 
                |--------------------------------------->B      //B端接收Offer  
                |                           |
    A           |<---------------------------------------B      //B端创建Answer 
    A<----------|                           |            B      //A端接收Answer
                |                           |
    A-------------------------------------->|<-----------B      //A B端通过ice服务 确定自己所处网络环境
                |                           |
    A<--------------------------------------|----------->B      // A B端获得iceCandidate
                |                           |
    A---------->|<---------------------------------------B      // A B端向信令服务器发送自身的iceCandidate
    A<----------|--------------------------------------->B      // A B端收到信令服务推送的对端的iceCandidate
    A<==================================================>B      //媒体传输通道建立 开始音视频通话
                |                           |
    A---------->|                                        B      //A结束通话 关闭媒体流 关闭PeerConnection 发送关闭信令
    A           |--------------------------------------->B      //B接到关闭信令 关闭媒体流 关闭PeerConnection
    A....................................................B      //通话结束



F1: 获取本地音视频流 

                [信令服务]          [ICE服务] 

    A ----------------------------------------------------------  B
```js
var constraints = {
            'video': true,
            'audio': true
        };

navigator.mediaDevices.getUserMedia(constraints)
    .then(stream => {
        console.log("getUserMedia onsuccess!");
        localVideo.srcObject = stream;
    })
    .catch(error => {
        console.log(error);
    });
```

F2: 创建RTCPeerConnection 

                [信令服务]          [ICE服务] 
        
    A ----------------------------------------------------------  B
```js
const iceConfiguration = {
        'iceServers': [{
            urls: 'turn:101.34.23.152:3478',
        }],
    }

peerConnection = new RTCPeerConnection(iceConfiguration);
stream.getTracks().forEach(track => {
    peerConnection.addTrack(track, stream);
});
```

F3: A产生Offer 

                        [信令服务]           [ICE服务] 
                    .       |
                .           |
            .               |       
        .                   | 
    A --------------------->|-----------------------------------  B
```js
peerConnection.createOffer().then(offer => {
    console.log("peer connection createOffer success");
    peerConnection.setLocalDescription(offer);
}).catch(error => {
    console.log("peer connection createOffer error");
    console.log(error);
});
```

F4: 向B端发送A产生的Offer 

                        [信令服务]           [ICE服务] 
                           |
                           |
                           |       
                           |        A产生的offer发送给B
    A ---------------------|------------------------------------>  B
```js
//F4 发送Offer
var msg = buildSignalData('offer', offer);
socket.send(msg);
```

F5: B端打开媒体流 

                        [信令服务]           [ICE服务] 
                           |
                           |
                           |    F5: B端打开媒体流    
                           |  
    A ---------------------|-------------------------------------  B
```js
function handleOfferSignal(jsonData) {
    console.log("handle offer : ");
    console.log(jsonData.data);

    var constraints = {
        'video': true,
        'audio': true
    };

    navigator.mediaDevices.getUserMedia(constraints)
        .then(stream => {
            localVideo.srcObject = stream;
            //add local track
            stream.getTracks().forEach(track => {
                peerConnection.addTrack(track, stream);
            });
        })
        .catch(error => {
            console.log(error);
        });
}
```

F6: B端创建RTCPeerConnection  

                        [信令服务]           [ICE服务] 
                           |
                           |
                           |    B端收到Offer创建RTCPeerConnection   
                           |  
    A ---------------------|-------------------------------------  B 

```js
const iceConfiguration = {
        'iceServers': [{
            urls: 'turn:101.34.23.152:3478',
            username: 'panyi',
            credential: '123456'
        }],
    }
peerConnection = new RTCPeerConnection(iceConfiguration);
//add local track
stream.getTracks().forEach(track => {
    peerConnection.addTrack(track, stream);
});

```

F7: B端通过RTCPeerConnection 创建Answer 设置setRemoteDescript 经由信令服务发送给A端

                        [信令服务]           [ICE服务] 
                           |
                           |    B端通过RTCPeerConnection 创建Answer 
                           |        经由 信令服务发送给A端
                           |  
    A ---------------------|<------------------------------------  B 

```js
peerConnection.setRemoteDescription(new RTCSessionDescription(offer));
peerConnection.createAnswer().then(answer => {
        peerConnection.setLocalDescription(answer);
        //发送answer
        var msg = buildSignalData('answer', answer);
        socket.send(msg);
    });
})
.catch(error => {
    console.log(error);
});
```

F7: 将由B创建的Answer 通过信令服务推送给A

                        [信令服务]           [ICE服务] 
                            |
                            |
                            |      
                            |  
    A <---------------------|-------------------------------------  B

F8: A获取到B端的Answer 调用RTCPeerConnection setRemoteDescript() 

                        [信令服务]           [ICE服务]
                            |
                            |
                            |
    A <---------------------|-------------------------------------  B
```js
function handleAnswerSignal(jsonData) {
    console.log("hanlde answer:");
    peerConnection.setRemoteDescription(new RTCSessionDescription(jsonData.data))
        .then((data) => {
            console.log("set remote description success");
        }).catch((err) => {
            console.log("set remote description error");
        });
}
```

自此A B两端的RTCPeerConnection 都完成了 setLocalDescription / setRemoteDescription 

A端
- [x] RTCPeerConnection
- [x] setLocalDescription
- [x] setRemoteDescription

B端
- [x] RTCPeerConnection
- [x] setLocalDescription
- [x] setRemoteDescription

WebRTC在上述步骤完成后会完成两端的媒体协商 商议出一个可在两端通用的媒体格式及编解码器

    
F9: 通过ICE服务 获取各端网络相关信息

                        [信令服务]           [ICE服务]
                                                |
                                                |
                                                |
    A ----------------------------------------->|<----------------  B
                                                |
                                                |
                                                |
                                                |
    A <-----------------------------------------|----------------> B

```js
var iceCandidates = [];
peerConnection.addEventListener('icecandidate', event => {
    console.log('icecandidate:');
    iceCandidates.push(event.candidate);
});
```

F10: 客户端将收集的iceCandidate数据发送至信令服务器

                 [信令服务]                  [ICE服务]
                     |                           
                     |                          
                     |                           
    A -------------->|<------------------------------------------- B


```js
iceCandidates.forEach(candidate => {
    var msg = buildSignalData("icecandidate", candidate);
    socket.send(msg);
});
```

F11: A 与 B 各自设置对端的IceCandidate

                [信令服务]                  [ICE服务]
                    |                           
                    |     
                    |                                                                   
    A <-------------|-------------------------------------------> B


```js
function handleIceCandidataSignal(jsonData) {
    console.log("handle IceCandidataSignal :");
    console.log(jsonData);
    if (peerConnection != null) {
        peerConnection.addIceCandidate(jsonData.data)
            .then(result => {
                console.log("addIceCandidate success");
            })
            .catch(error => {
                console.log("addIceCandidate error!");
            });
    }
}
```

完成ICE的交换后 WebRTC就可以根据这些信息完成NAT穿透建立端到端的直接连接

F12: 端到端连接建立完成 将媒体流加入PeerConnection 开始实际音视频通话

                [信令服务]                  [ICE服务]
                                               
                        RTP / RTCP开始音视频媒体流传输 
                            Media Session                                                       
    A <========================================================> B

```js
peerConnection.addEventListener('track', (event) => {
    console.log('peerConnection track');
    var remoteStream = event.streams[0];
    remoteVideoDiv.srcObject = remoteStream;
});
```

F13: B想结束通话 向信令服务发送close信令 并自行关闭RTCPeerConnection.close() 释放音视频流

                [信令服务]                  [ICE服务]
                    |     
                    |                                                                   
    A ------------->|                                             B
                    |
    A <========================================================== B
                    |
    A               | ------------------------------------------> B
                    |
    A =========================================================== B
                    |
    A ........................................................... B

```js
function closeRtc(sendSingal) {
    if (peerConnection != null) {
        peerConnection.close();
        peerConnection.onicecandidate = null;
        peerConnection.ontrack = null;
        peerConnection = null;
    }

    console.log("close camera");
    localVideoDiv.srcObject = null;
    remoteVideoDiv.srcObject = null;

    if (sendSingal) {
        var msg = buildSignalData('close', {});
        socket.send(msg);
    }
}

function handleCloseSignal(data) {
    closeRtc(false);
}

```


#### 不同平台的实现
    web 游览器自带

    Android java
    
    flutter dart

#### WebRTC 与 Sip协议
    两者是互补关系 
    WebRTC建立通信时并未定义信令数据的格式 
    WebRTC在信令传输时采用Sip协议
    利用Sip协议包含的 Register Invite Bye 完成WebRTC会话的创建 SDP协商 ICE的交换


    