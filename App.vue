<template>
  <header>
    <div>
      <h1>Mediasoup WebRTC Client</h1>
      <video ref='remoteVideo' autoplay></video>
      <video ref='localVideo' autoplay muted></video>
    </div>
  </header>
</template>
<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';
import ProToo from 'protoo-client'; // 使用 ES6 导入
import * as mediasoupClient from 'mediasoup-client';

let localVideo = ref(null);
let remoteVideo = ref(null);
let device;
let producers = []; // 存储生产者
let _protoo;
let _sendTransport;
let _recvTransport;

let mediaRecorder;
let recordedChunks = [];
let recordingInterval;

const serverUrl = 'wss://192.168.0.104:4443/?roomId=lsk01&peerId=123456&consumerReplicas=0'; // Mediasoup server URL

onMounted(async () => {
  try {
    remoteVideo.value.srcObject = new MediaStream()
    const protooTransport = new ProToo.WebSocketTransport(serverUrl);
    _protoo = new ProToo.Peer(protooTransport);
    _protoo.on('open', async () => {
      device = new mediasoupClient.Device();
      const routerRtpCapabilities = await _protoo.request('getRouterRtpCapabilities');
      await device.load({ routerRtpCapabilities });
      const stream = await navigator.mediaDevices.getUserMedia({ audio: true, video: true });
      localVideo.value.srcObject = stream;
      {
        const transportInfo = await _protoo.request(
            'createWebRtcTransport', 
            {
                forceTcp: false,
                producing: true,
                consuming: false,
                sctpCapabilities: device.sctpCapabilities,
            }
        );
        // console.log('transportInfo', transportInfo);
        const { id, iceParameters, iceCandidates, dtlsParameters, sctpParameters} = transportInfo;
        _sendTransport = device.createSendTransport({
          id,
          iceParameters,
          iceCandidates,
          dtlsParameters: { ...dtlsParameters, role: 'auto' },
          sctpParameters,
          iceServers: [],
          proprietaryConstraints: {},
          additionalSettings: {
            encodedInsertableStreams: false,
          },
        });
        // 设置连接发送传输的事件监听器
        _sendTransport.on('connect', async ({ dtlsParameters }, callback) => {
          try {
            await _protoo.request('connectWebRtcTransport', {
              transportId: _sendTransport.id,
              dtlsParameters,
            });
            callback();
          } catch (error) {
            console.error('Error connecting transport:', error);
            callback(error); 
          }
        });

      }

      {
          const recvTransportInfo = await _protoo.request(
            'createWebRtcTransport', 
            {
              forceTcp: false,
              producing: false, // 不生产，只消费
              consuming: true,   // 消费媒体流
              sctpCapabilities: device.sctpCapabilities,
            }
          );
          const {id, iceParameters, iceCandidates, dtlsParameters, sctpParameters} = recvTransportInfo;
          console.log("sctpParameters", sctpParameters)
          _recvTransport = device.createRecvTransport({
            id,
            iceParameters,
            iceCandidates,
            dtlsParameters:{ ...dtlsParameters, role: 'auto' },
            sctpParameters,
            iceServers: [],
            additionalSettings: {
              encodedInsertableStreams: false,
            }
          })
          _recvTransport.on('connect', async ({ iceParameters, dtlsParameters }, callback, errback) => {
              try {
                  await _protoo.request('connectWebRtcTransport', {
                      transportId: _recvTransport.id,
                      iceParameters,
                      dtlsParameters,
                  });
                  callback();
              } catch (error) {
                  console.error('Error connecting transport:', error);
                  errback(error);
              }
          });



      }
       _sendTransport.on('produce', async (parameters, callback, errback) => {
         try {
           const { id } = await _protoo.request(
             'produce',
             {
               transportId: _sendTransport.id,
               ...parameters,
             }
           );
           callback({ id });
         } catch (error) {
           console.error('Error producing:', error);
           errback(error); 
         }
       });
       const { peers } = await _protoo.request('join', {
         displayName: 'haha',
         device: {flag:'chrome', name:'Chrome', version:"131.0.0.0"},
         rtpCapabilities: device.rtpCapabilities,
         sctpCapabilities: undefined,
       });
       const audioTrack = stream.getAudioTracks()[0];
       const audioProducer = await _sendTransport.produce({ 
         track: audioTrack,
         codecOptions : {
           opusStereo : true,
           opusDtx    : true,
           opusFec    : true,
           opusNack   : true
         }
       });
       producers.push(audioProducer); 
       const videoTrack = stream.getVideoTracks()[0];
       const videoProducer = await _sendTransport.produce({ track: videoTrack });
       producers.push(videoProducer); 
    });



    _protoo.on('request', async (request, accept, reject) =>
    {
      switch (request.method){
        case 'newConsumer':
        {
          try{
            const { peerId, producerId, id, kind, rtpParameters, type, appData, producerPaused} = request.data;
            console.log("id, producerId, kind, rtpParameters",id, producerId, kind, rtpParameters)
            const consumer = await _recvTransport.consume(
							{
								id,
								producerId,
								kind,
								rtpParameters,
							}
            );
            remoteVideo.value.srcObject.addTrack(consumer.track);
            accept();
          }catch (error) {
            console.log('newConsumer', error);
          }
          break;
        }
      }
    }
  )
  } catch (error) {
    console.error('Error during setup:', error);
  }
});


onBeforeUnmount(() => {
  if (_protoo) {
    _protoo.close(); 
  }

  producers.forEach(producer => producer.close()); 
});



</script>

<style scoped>
video {
    width: 100%;
    height: auto; /* 或者设置为固定高度 */
    border: 1px solid black; /* 添加边框以便于调试 */
    display: block; /* 确保元素为块级元素 */
}
</style>
