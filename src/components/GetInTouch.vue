<template>
  <div>
    <vue-frame
      text="Api Vue"
      url="https://vuejs.org/v2/api"
      frame="myframe"
      type="a"
    />
    <iframe
      id="myframe"
      width="800"
    />
    <h1>Chat room</h1>
  </div>
</template>

<script>
  import firebase from 'firebase/firestore'
  import VueFrame from 'vue-frame'
  export default {
    components: { VueFrame },
    data () {
      return {
        peerConnection: null,
        localStream: null,
        remoteStream: null,
        roomDialog: null,
        roomId: null,
        configuration: {
          iceServers: [
            {
              urls: [
                'stun:stun1.l.google.com:19302',
                'stun:stun2.l.google.com:19302',
              ],
            },
          ],
          iceCandidatePoolSize: 10,
        },
      }
    },
    created () {
      this.registerPeerConnectionListeners()
    },

    methods: {
      registerPeerConnectionListeners () {
        this.peerConnection.addEventListener('icegatheringstatechange', () => {
          console.log(
            `ICE gathering state changed: ${this.peerConnection.iceGatheringState}`)
        })

        this.peerConnection.addEventListener('connectionstatechange', () => {
          console.log(`Connection state change: ${this.peerConnection.connectionState}`)
        })

        this.peerConnection.addEventListener('signalingstatechange', () => {
          console.log(`Signaling state change: ${this.peerConnection.signalingState}`)
        })

        this.peerConnection.addEventListener('iceconnectionstatechange ', () => {
          console.log(
            `ICE connection state change: ${this.peerConnection.iceConnectionState}`)
        })
      },
      async createRoom () {
        document.querySelector('#createBtn').disabled = true
        document.querySelector('#joinBtn').disabled = true
        const db = firebase.firestore()
        const roomRef = await db.collection('rooms').doc()

        console.log('Create this.peerConnection with this.configuration: ', this.configuration)
        this.peerConnection = new RTCPeerConnection(this.configuration)

        this.localStream.getTracks().forEach(track => {
          this.peerConnection.addTrack(track, this.localStream)
        })

        // Code for collecting ICE candidates below
        const callerCandidatesCollection = roomRef.collection('callerCandidates')

        this.peerConnection.addEventListener('icecandidate', event => {
          if (!event.candidate) {
            console.log('Got final candidate!')
            return
          }
          console.log('Got candidate: ', event.candidate)
          callerCandidatesCollection.add(event.candidate.toJSON())
        })
        // Code for collecting ICE candidates above

        // Code for creating a room below
        const offer = await this.peerConnection.createOffer()
        await this.peerConnection.setLocalDescription(offer)
        console.log('Created offer:', offer)

        const roomWithOffer = {
          offer: {
            type: offer.type,
            sdp: offer.sdp,
          },
        }
        await roomRef.set(roomWithOffer)
        this.roomId = roomRef.id
        console.log(`New room created with SDP offer. Room ID: ${roomRef.id}`)
        document.querySelector(
          '#currentRoom').innerText = `Current room is ${roomRef.id} - You are the caller!`
        // Code for creating a room above

        this.peerConnection.addEventListener('track', event => {
          console.log('Got remote track:', event.streams[0])
          event.streams[0].getTracks().forEach(track => {
            console.log('Add a track to the this.remoteStream:', track)
            this.remoteStream.addTrack(track)
          })
        })

        // Listening for remote session description below
        roomRef.onSnapshot(async snapshot => {
          const data = snapshot.data()
          if (!this.peerConnection.currentRemoteDescription && data && data.answer) {
            console.log('Got remote description: ', data.answer)
            const rtcSessionDescription = new RTCSessionDescription(data.answer)
            await this.peerConnection.setRemoteDescription(rtcSessionDescription)
          }
        })
        // Listening for remote session description above

        // Listen for remote ICE candidates below
        roomRef.collection('calleeCandidates').onSnapshot(snapshot => {
          snapshot.docChanges().forEach(async change => {
            if (change.type === 'added') {
              const data = change.doc.data()
              console.log(`Got new remote ICE candidate: ${JSON.stringify(data)}`)
              await this.peerConnection.addIceCandidate(new RTCIceCandidate(data))
            }
          })
        })
        // Listen for remote ICE candidates above
      },
      joinRoom () {
        document.querySelector('#createBtn').disabled = true
        document.querySelector('#joinBtn').disabled = true

        document.querySelector('#confirmJoinBtn')
          .addEventListener('click', async () => {
            this.roomId = document.querySelector('#room-id').value
            console.log('Join room: ', this.roomId)
            document.querySelector(
              '#currentRoom').innerText = `Current room is ${this.roomId} - You are the callee!`
            await this.joinRoomById(this.roomId)
          }, { once: true })
        this.roomDialog.open()
      },

      async joinRoomById (roomId) {
        const db = firebase.firestore()
        const roomRef = db.collection('rooms').doc(`${this.roomId}`)
        const roomSnapshot = await roomRef.get()
        console.log('Got room:', roomSnapshot.exists)

        if (roomSnapshot.exists) {
          console.log('Create this.peerConnection with this.configuration: ', this.configuration)
          this.peerConnection = new RTCPeerConnection(this.configuration)
          this.registerPeerConnectionListeners()
          this.localStream.getTracks().forEach(track => {
            this.peerConnection.addTrack(track, this.localStream)
          })

          // Code for collecting ICE candidates below
          const calleeCandidatesCollection = roomRef.collection('calleeCandidates')
          this.peerConnection.addEventListener('icecandidate', event => {
            if (!event.candidate) {
              console.log('Got final candidate!')
              return
            }
            console.log('Got candidate: ', event.candidate)
            calleeCandidatesCollection.add(event.candidate.toJSON())
          })
          // Code for collecting ICE candidates above

          this.peerConnection.addEventListener('track', event => {
            console.log('Got remote track:', event.streams[0])
            event.streams[0].getTracks().forEach(track => {
              console.log('Add a track to the this.remoteStream:', track)
              this.remoteStream.addTrack(track)
            })
          })

          // Code for creating SDP answer below
          const offer = roomSnapshot.data().offer
          console.log('Got offer:', offer)
          await this.peerConnection.setRemoteDescription(new RTCSessionDescription(offer))
          const answer = await this.peerConnection.createAnswer()
          console.log('Created answer:', answer)
          await this.peerConnection.setLocalDescription(answer)

          const roomWithAnswer = {
            answer: {
              type: answer.type,
              sdp: answer.sdp,
            },
          }
          await roomRef.update(roomWithAnswer)
          // Code for creating SDP answer above

          // Listening for remote ICE candidates below
          roomRef.collection('callerCandidates').onSnapshot(snapshot => {
            snapshot.docChanges().forEach(async change => {
              if (change.type === 'added') {
                const data = change.doc.data()
                console.log(`Got new remote ICE candidate: ${JSON.stringify(data)}`)
                await this.peerConnection.addIceCandidate(new RTCIceCandidate(data))
              }
            })
          })
          // Listening for remote ICE candidates above
        }
      },

      async openUserMedia (e) {
        const stream = await navigator.mediaDevices.getUserMedia(
          { video: true, audio: true })
        document.querySelector('#localVideo').srcObject = stream
        this.localStream = stream
        this.remoteStream = new MediaStream()
        document.querySelector('#remoteVideo').srcObject = this.remoteStream

        console.log('Stream:', document.querySelector('#localVideo').srcObject)
        document.querySelector('#cameraBtn').disabled = true
        document.querySelector('#joinBtn').disabled = false
        document.querySelector('#createBtn').disabled = false
        document.querySelector('#hangupBtn').disabled = false
      },

      async hangUp (e) {
        const tracks = document.querySelector('#localVideo').srcObject.getTracks()
        tracks.forEach(track => {
          track.stop()
        })

        if (this.remoteStream) {
          this.remoteStream.getTracks().forEach(track => track.stop())
        }

        if (this.peerConnection) {
          this.peerConnection.close()
        }

        document.querySelector('#localVideo').srcObject = null
        document.querySelector('#remoteVideo').srcObject = null
        document.querySelector('#cameraBtn').disabled = false
        document.querySelector('#joinBtn').disabled = true
        document.querySelector('#createBtn').disabled = true
        document.querySelector('#hangupBtn').disabled = true
        document.querySelector('#currentRoom').innerText = ''

        // Delete room on hangup
        if (this.roomId) {
          const db = firebase.firestore()
          const roomRef = db.collection('rooms').doc(this.roomId)
          const calleeCandidates = await roomRef.collection('calleeCandidates').get()
          calleeCandidates.forEach(async candidate => {
            await candidate.ref.delete()
          })
          const callerCandidates = await roomRef.collection('callerCandidates').get()
          callerCandidates.forEach(async candidate => {
            await candidate.ref.delete()
          })
          await roomRef.delete()
        }

        document.location.reload(true)
      },

    },
  }
</script>

<style>

</style>
