## 0724 webrtc

### 1. 기본 socket.io, mediastream 사용
node.js 시그널링서버

내 화면만 보임, ssl 인증문제..
```html
<!-- call.html -->
<!DOCTYPE html>
<html>
<head>
  <title>WebRTC 화상 회의</title>
</head>
<body>
  <h1>WebRTC 화상 회의</h1>
  <button onclick="startConference()">회의 시작</button>
  <div id="localVideo"></div>
  <div id="remoteVideos"></div>

  <script>
    let localStream;
    const remoteStreams = {};

    // WebSocket 연결 설정
    const socket = io();

    // Offer 생성 및 전송
    async function createOfferAndSend(userId, pc) {
      try {
        const offer = await pc.createOffer();
        await pc.setLocalDescription(offer);

        // Offer를 시그널링 서버로 전송
        socket.emit('sendOffer', userId, offer);
      } catch (error) {
        console.error('Error creating and sending offer:', error);
      }
    }

    // Answer 수신 및 SDP 설정
    socket.on('receiveAnswer', async (senderId, answer) => {
      console.log(`Received answer from ${senderId}: ${answer}`);

      // 해당 사용자(userId)의 피어 연결 찾기
      const peer = peers.find(peer => peer.userId === senderId);
      if (peer) {
        // Answer를 해당 피어 연결의 원격 설명으로 설정
        await peer.pc.setRemoteDescription(answer);
      }
    });

    async function startConference() {
      try {
        // 미디어 스트림 가져오기
        localStream = await navigator.mediaDevices.getUserMedia({ video: true, audio: true });
        document.getElementById('localVideo').srcObject = localStream;

        // RTC Configuration 설정
        const configuration = { iceServers: [{ urls: 'stun:stun.stunprotocol.org' }] };

        // 피어 연결 설정
        const peers = [];

        function createPeerConnection(userId) {
          const pc = new RTCPeerConnection(configuration);

          // 로컬 미디어 스트림 추가
          localStream.getTracks().forEach(track => {
            pc.addTrack(track, localStream);
          });

          // 원격 스트림 수신 이벤트 핸들러
          pc.ontrack = event => {
            if (!remoteStreams[userId]) {
              remoteStreams[userId] = new MediaStream();
              document.getElementById('remoteVideos').appendChild(createVideoElement(userId));
            }
            remoteStreams[userId].addTrack(event.track);
            document.getElementById(`video_${userId}`).srcObject = remoteStreams[userId];
          };

          // ICE Candidate 수집 이벤트 핸들러
          pc.onicecandidate = event => {
            if (event.candidate) {
              // ICE candidate를 다른 사용자들에게 전송 (시그널링 서버 사용)
              socket.emit('sendIceCandidate', userId, event.candidate);
            }
          };

          return pc;
        }

        // 각 사용자들과 피어 연결 생성
        // users 배열에 4명의 사용자 정보가 있다고 가정
        const users = ['user1', 'user2', 'user3', 'user4'];

        users.forEach(userId => {
          const pc = createPeerConnection(userId);
          peers.push({ userId, pc });
        });

        // Offer 생성 및 전송
        peers.forEach(({ userId, pc }) => {
          createOfferAndSend(userId, pc);
        });

        // ICE Candidate 수신 및 추가
        socket.on('receiveIceCandidate', async (senderId, iceCandidate) => {
          console.log(`Received ICE candidate from ${senderId}: ${iceCandidate}`);

          // 해당 사용자(userId)의 피어 연결 찾기
          const peer = peers.find(peer => peer.userId === senderId);
          if (peer) {
            // 수신된 ICE candidate 추가
            try {
              await peer.pc.addIceCandidate(iceCandidate);
            } catch (error) {
              console.error('Error adding ICE candidate:', error);
            }
          }
        });

      } catch (error) {
        console.error('Error accessing media devices:', error);
      }
    }

    // 새로운 video element 생성
    function createVideoElement(userId) {
      const videoElement = document.createElement('video');
      videoElement.id = `video_${userId}`;
      videoElement.autoplay = true;
      videoElement.playsinline = true;
      return videoElement;
    }
  </script>

  <!-- Socket.io 클라이언트 라이브러리 -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/socket.io/4.1.2/socket.io.js"></script>
</body>
</html>
```

```javascript
// Node.js 서버를 위한 시그널링 서버 코드 (server.js)

const express = require('express');
const app = express();
const http = require('http').createServer(app);
const io = require('socket.io')(http);

const PORT = 3000;

// 웹 서버 설정
app.use(express.static(__dirname + '/public'));

// 웹 소켓 연결
io.on('connection', (socket) => {
  console.log('A user connected.');

  // 새로운 사용자가 연결되었음을 다른 사용자들에게 알립니다.
  socket.broadcast.emit('newUserJoined');

  // 연결이 끊어졌을 때의 처리를 정의할 수도 있습니다.
  socket.on('disconnect', () => {
    console.log('A user disconnected.');
  });
});

// 서버 시작
http.listen(PORT, () => {
  console.log(`Server is running on http://localhost:${PORT}`);
});se
```

### 2. openvidu 사용

화상회의 관련 라이브러리

docker 환경에서 시그널링 서버 구현 - port 4443

npm openvidu-library-react 3000 클라이언트

로컬환경에서 구현 확인

배포필요

<img width="800" alt="스크린샷 2023-07-25 오전 1 05 38" src="https://github.com/DongIkkk/TIL/assets/110454344/cbc0a09e-b3f8-4031-aa8e-2d0d169fca55">

1. 튜토리얼 파일 클론
```
git clone https://github.com/OpenVidu/openvidu-tutorials.git -b v2.21.0
```

2. 폴더 이동 후 실행
```
cd openvidu-tutorials/openvidu-library-react
npm install
npm start
```

3. 도커환경 시그널링서버
```
docker run -p 4443:4443 --rm -e OPENVIDU_SECRET=MY_SECRET openvidu/openvidu-server-kms:2.21.0
```

- node 버전확인, 도커환경 구축

## 0725 WebRTC 배포과정

1. **EC2 서버 생성**: AWS 콘솔 또는 CLI를 사용하여 EC2 인스턴스를 생성합니다. 인스턴스를 만들 때 SSH 키를 생성하고 EC2 보안 그룹에서 필요한 포트(예: 22번 포트는 SSH 접근, 80번 포트는 HTTP 접근 등)를 열어줍니다.
    1. **HTTP (80번 포트) 및 HTTPS (443번 포트)**: Nginx와 OpenVidu를 설치한 경우, 웹 서비스를 위해 HTTP와 HTTPS 포트가 필요합니다. HTTP 포트는 80번, HTTPS 포트는 443번을 사용하는 것이 일반적입니다.
    2. **OpenVidu 사용에 필요한 포트들**: OpenVidu는 WebRTC 기반의 화상회의를 지원하는 데 사용되며, 다양한 미디어와 시그널링 서버 간의 통신을 위해 여러 포트들이 필요할 수 있습니다. OpenVidu의 문서를 참조하여 필요한 포트들을 확인하세요.
2. **Docker 설치**: EC2 인스턴스에 Docker를 설치합니다. Docker는 애플리케이션을 컨테이너로 패키징하고 실행하는 데 도움이 됩니다.
3. **Nginx 설치**: Nginx는 웹 서버로 사용되며, HTTPS 통신을 위해 SSL 인증서를 적용할 수 있습니다. Nginx를 설치하고 설정합니다.
    1. 주요 설정 파일 - /etc/nginx 아래 nginx.conf / sites-available/default
4. **OpenVidu 설치**: OpenVidu는 WebRTC를 사용하여 실시간 화상회의를 구현하는 데 도움이 됩니다. OpenVidu를 Docker 컨테이너로 실행합니다.
5. **SSL 인증서 발급**: HTTPS 통신을 위해 SSL 인증서가 필요합니다. Let's Encrypt와 certbot을 사용하여 무료 SSL 인증서를 발급받을 수 있습니다.
6. **Nginx에서 SSL 적용**: 발급받은 SSL 인증서를 Nginx에 적용하여 HTTPS로 보안된 통신을 구현합니다.
7. **도메인 연결**: 만약 도메인을 보유하고 있다면, 해당 도메인을 EC2 인스턴스와 연결하여 도메인으로 서비스에 접근할 수 있도록 설정합니다.

실패 -> openvidu 접속불가 -> kms는 로컬용

openvidu on premise 참고 재배포
![openvidu](https://github.com/DongIkkk/TIL/assets/110454344/d751df6f-3c50-46fa-abec-44b7452b0ebb)