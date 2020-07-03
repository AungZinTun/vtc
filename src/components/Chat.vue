<template>
  <v-container>
    <v-container
      fluid
    >
      <div v-show="connected">
        <span
          ref="remoteVideoBox"
          class="video-box remote-video-box"
        />
        <span
          ref="localVideoBox"
          class="video-box local-video-box"
        />
      </div>
    </v-container>
  </v-container>
</template>

<script>
  import config from '@/config.js'
  export default {
    data () {
      return {
        socketurl: '',
        turnServerUsername: '',
        turnServerPassword: '',
        connection: null,
        connected: false,
        roomId: 'nap-tgi',
        counter: 0,
      }
    },

    computed: {
    },

    mounted () {
      console.log('mounted')
      this.init()
      this.openOrJoinRoom()
    },

    methods: {
      getQueryParam (name) {
        const paramsString = window.location.href.split('?')
        const searchParams = new URLSearchParams(paramsString[1])
        return searchParams.has(name) ? searchParams.get(name) : false
      },

      reloadWindow () {
        const url = window.location
        window.location.href = `${url.origin}${url.pathname}?roomid=${this.roomId}&start=0`
      },
      leave () {
        console.log('onleave')
        this.roomId = null
        this.connected = false
      },

      init () {
        const self = this
        console.log('refs', this.$refs)
        // console.log('getQueryParam('roomid'):', this.roomId)

        const supports = navigator.mediaDevices.getSupportedConstraints()
        var constraints = {}
        if (supports.width && supports.height) {
          constraints = {
            width: config.width || 640,
            height: config.height || 480,
          }
        }
        console.log('constraints', constraints)

        // eslint-disable-next-line no-undef
        const connection = (this.connection = new RTCMultiConnection())

        // connection.applyConstraints({
        //   video: constraints,
        // })

        this.socketURL = this.getQueryParam('socketurl')
        connection.socketURL = this.socketURL || config.socketURL
        // connection.socketURL = 'https://rtcmulticonnection.herokuapp.com:443/'
        // connection.socketURL = 'https://rulokoba.me:9001/'
        console.log('connection.socketURL', connection.socketURL)
        connection.session = {
          audio: false,
          video: true,
        }

        // https://www.rtcmulticonnection.org/docs/iceServers/
        connection.iceServers = []
        const stunServers = config.stunServers || []
        for (const server of stunServers) {
          console.log('stunServer', server)
          connection.iceServers.push({
            urls: server,
          })
        }

        this.turnServerUsername = this.getQueryParam('turnuser')
        this.turnServerPassword = this.getQueryParam('turnpass')
        console.log('turnServerConfig', {
          urls: config.turnServerURL,
          username: this.turnServerUsername || config.turnServerUsername,
          credential: this.turnServerPassword || config.turnServerPassword,
        })
        connection.iceServers.push({
          urls: config.turnServerURL,
          username: this.turnServerUsername || config.turnServerUsername,
          credential: this.turnServerPassword || config.turnServerPassword,
        })

        connection.onstream = function (event) {
          console.log('onstream')
          self.counter++
          const video = event.mediaElement

          if (event.type === 'local') {
            console.log('Video: local')
            video.className += ' local'
            self.$refs.localVideoBox.appendChild(video)
          }

          if (event.type === 'remote') {
            console.log('Video: remote')
            video.className += ' remote'
            self.$refs.remoteVideoBox.appendChild(video)
          }
        // self.$refs['videoBox'].appendChild(video)
        }

        connection.onleave = function (event) {
          console.log('onleave')
          if (self.counter > 0) {
            self.counter--
          }
        }

        this.roomId = this.roomId || connection.token()
      },

      openOrJoinRoom () {
        this.connected = true
        this.connection.openOrJoin(this.roomId)
      },
    },
  }
</script>

<style>
[v-cloak] > * {
  display: none;
}
[v-cloak]::before {
  content: ". . .";
}

.video-box {
  display: flex;
}
.video-box video {
  width: 100%;
}
.video-box video.local {
  background-color: rgba(0, 0, 0, 0.1);
}
.video-box video.remote {
  background-color: rgba(0, 0, 0, 0.1);
}
@media (min-width: 768px) {
  .video-box {
    flex-direction: row;
  }
  .video-box video {
    width: 25%;
  }
}
</style>
