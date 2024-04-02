<template>
  <div class="w-full max-w-[17.5rem] relative">
    <PrimaryCTA label="Call via healing119.id" :is-width-parent="true" @on-click="handleCallNumber"> </PrimaryCTA>
    <audio autoplay id="remote-stream-audio"></audio>
  </div>
</template>

<script setup lang="ts">
import { webrtcPhone } from '~/utils/webrtc-phone'
import Janus from '~/utils/janus'
import adapter from 'webrtc-adapter'
import PrimaryCTA from '../common/PrimaryCTA.vue'

const started = ref(false)
const registered = ref(false)
const sipcall = ref<any>()
const janus = ref<any>()
const counterpartNum = ref<string>()
const incoming = ref<any>()
const currentJsep = ref<any>()
const supportedDevices = ref<any>()

const ringing = new Audio('sounds/ringing.mp3')
const calling = new Audio('sounds/calling.mp3')

const login = () => {
  if (sipcall) {
    const register = {
      username: 'sip:admin@healing119.id',
      display_name: 'Web Agent',
      authuser: 'Web Agent',
      secret: '2001',
      proxy: 'sip:https://call.healing119.id/ippbx:50361',
      request: 'register',
      type: 'guest',
    }
    sipcall.value.send({
      message: register,
    })
  }
}
const handleRemote = (jsep) => {
  sipcall.value.handleRemoteJsep({
    jsep,
    error: function () {
      const body = {
        request: 'hangup',
      }
      sipcall.value.send({
        message: body,
      })
      sipcall.value.hangup()
    },
  })
}
const getSupportedDevices = function (origCallback) {
  if (navigator.mediaDevices && navigator.mediaDevices.enumerateDevices) {
    // Firefox 38+ seems having support of enumerateDevicesx
    navigator.enumerateDevices = function (callback) {
      navigator.mediaDevices.enumerateDevices().then(callback)
    }
  }

  let MediaDevices = []
  const isHTTPs = location.protocol === 'https:' // eslint-disable-line

  let canEnumerate = false

  if (typeof MediaStreamTrack !== 'undefined' && 'getSources' in MediaStreamTrack) {
    canEnumerate = true
    // eslint-disable-next-line
  } else if (navigator.mediaDevices && !!navigator.mediaDevices.enumerateDevices) {
    canEnumerate = true
  }

  let hasMicrophone = false
  let hasSpeakers = false //eslint-disable-line
  let hasWebcam = false

  let isMicrophoneAlreadyCaptured = false
  let isWebcamAlreadyCaptured = false

  function checkDeviceSupport(callback) {
    if (!canEnumerate) {
      return
    }

    if (!navigator.enumerateDevices && window.MediaStreamTrack && window.MediaStreamTrack.getSources) {
      navigator.enumerateDevices = window.MediaStreamTrack.getSources.bind(window.MediaStreamTrack)
    }

    if (!navigator.enumerateDevices && navigator.enumerateDevices) {
      navigator.enumerateDevices = navigator.enumerateDevices.bind(navigator)
    }

    if (!navigator.enumerateDevices) {
      if (callback) {
        callback()
      }
      return
    }

    MediaDevices = []
    navigator.enumerateDevices((devices) => {
      devices.forEach((_device) => {
        const device = {}
        // eslint-disable-next-line
        for (const d in _device) {
          device[d] = _device[d]
        }

        if (device.kind === 'audio') {
          device.kind = 'audioinput'
        }

        if (device.kind === 'video') {
          device.kind = 'videoinput'
        }

        let skip
        MediaDevices.forEach((d) => {
          if (d.id === device.id && d.kind === device.kind) {
            skip = true
          }
        })

        if (skip) {
          return
        }

        if (!device.deviceId) {
          device.deviceId = device.id
        }

        if (!device.id) {
          device.id = device.deviceId
        }

        if (!device.label) {
          device.label = 'Please invoke getUserMedia once.'
          if (!isHTTPs) {
            device.label = `HTTPs is required to get label of this ${device.kind} device.`
          }
        } else {
          if (device.kind === 'videoinput' && !isWebcamAlreadyCaptured) {
            isWebcamAlreadyCaptured = true
          }

          if (device.kind === 'audioinput' && !isMicrophoneAlreadyCaptured) {
            isMicrophoneAlreadyCaptured = true
          }
        }

        if (device.kind === 'audioinput') {
          hasMicrophone = true
        }

        if (device.kind === 'audiooutput') {
          hasSpeakers = true
        }

        if (device.kind === 'videoinput') {
          hasWebcam = true
        }
        // there is no 'videoouput' in the spec.
        MediaDevices.push(device)
      })

      if (callback) {
        callback()
      }
    })
  }

  // check for microphone/camera support!
  checkDeviceSupport(() => {
    supportedDevices.value = {
      audio: hasMicrophone,
      audioCap: isMicrophoneAlreadyCaptured,
      video: adapter.browserDetails.browser === 'chrome' ? hasWebcam : false,
      videoCap: adapter.browserDetails.browser === 'chrome' ? isWebcamAlreadyCaptured : false,
    }
    console.log(supportedDevices.value)
    origCallback()
  })
}
onMounted(() => {
  // initAndLogin, call audio, hangup, answer, logout
  // webrtcPhone().initAndLogin({
  //   server: 'janus.conf.meetecho.com',
  //   name: 'Web Agent',
  //   exten: 'exted',
  //   password: '2001',
  //   audioId: 'remote-stream-audio',
  //   localVideoId: 'local-stream-video',
  //   remoteVideoId: 'remote-stream-video',
  // })
  const server = 'janus.conf.meetecho.com'
  const janusUrl = `https://${server}/janus`
  const name = 'Web Agent'
  const password = '2001'
  const audioId = 'remote-stream-audio'
  const localVideoId = 'local-stream-video'
  const remoteVideoId = 'remote-stream-video'
  const localStream = document.getElementById(`#${localVideoId}`)
  const remoteStreamAudio = document.getElementById(`#${audioId}`)
  const remoteStreamVideo = document.getElementById(`#${remoteVideoId}`)
  if (sipcall.value) {
    login()
    return
  }
  Janus.init({
    debug: 'all',
    callback: function () {
      if (started.value) {
        return
      }
      started.value = true
      if (!Janus.isWebrtcSupported()) {
        console.error('No WebRTC support... ')
        return
      }
      // Create session
      janus.value = new Janus({
        server: janusUrl,
        success: function () {
          janus.value.attach({
            plugin: 'janus.plugin.sip',
            success: function (pluginHandle) {
              sipcall.value = pluginHandle as unknown
              console.log(`Plugin attached! (${sipcall.value.getPlugin()}, id=${sipcall.value.getId()})`)
              getSupportedDevices(() => {
                login()
              })
            },
            error: function (error) {
              console.error('  -- Error attaching plugin...', error)
            },
            onmessage: function (msg, jsep) {
              console.debug(' ::: Got a message :::')
              console.debug(JSON.stringify(msg))
              // Any error?
              const { error } = msg
              if (error !== null && error !== undefined) {
                if (!registered) {
                  console.log('User is not registered')
                } else {
                  // Reset status
                  sipcall.value.hangup()
                }
                return
              }
              const { result } = msg
              if (result !== null && result !== undefined && result.event !== undefined && result.event !== null) {
                // get event
                const { event } = result

                // switch event
                switch (event) {
                  case 'registration_failed':
                    console.error(`Registration failed: ${result.code} ${result.reason}`)
                    break

                  case 'registered':
                    console.log(`Successfully registered as ${result.username}!`)
                    if (!registered.value) {
                      registered.value = true
                      // $(document).trigger('registered')
                    }
                    break

                  case 'unregistered':
                    console.log(`Successfully unregistered as ${result.username}!`)
                    if (registered.value) {
                      registered.value = false
                      // $(document).trigger('unregistered')
                    }
                    break

                  case 'calling':
                    console.log('Waiting for the peer to answer...')
                    //   $(document).trigger('calling')
                    break

                  case 'incomingcall':
                    counterpartNum.value = msg.result.username.split('@')[0].split(':')[1] // eslint-disable-line
                    incoming.value = true
                    ringing.play()
                    console.log(`Incoming call from ${result.username}!`)
                    currentJsep.value = jsep
                    //   $(document).trigger('incomingcall', counterpartNum)
                    break

                  case 'progress':
                    console.log(`There's early media from ${result.username}, wairing for the call!`)
                    if (jsep !== null && jsep !== undefined) {
                      handleRemote(jsep)
                    }
                    break

                  case 'accepted':
                    calling.pause()
                    ringing.pause()
                    console.log(`${result.username} accepted the call!`)
                    if (jsep !== null && jsep !== undefined) {
                      handleRemote(jsep)
                    }
                    //   $(document).trigger('callaccepted')
                    break

                  case 'hangup':
                    incoming.value = null
                    calling.pause()
                    ringing.pause()
                    console.log(`Call hung up (${result.code} ${result.reason})!`)
                    sipcall.value.hangup()
                    //   $(document).trigger('hangup')
                    break

                  default:
                    break
                }
              }
            },
            onlocalstream: function (stream) {
              console.debug(' ::: Got a local stream :::')
              console.debug(JSON.stringify(stream))

              Janus.attachMediaStream(localStream, stream)

              /* IS VIDEO ENABLED ? */
              stream.getVideoTracks()
              /* */
            },
            onremotestream: function (stream) {
              console.debug(' ::: Got a remote stream :::')
              console.debug(JSON.stringify(stream))

              // retrieve stream track
              const audioTracks = stream.getAudioTracks()
              const videoTracks = stream.getVideoTracks()

              Janus.attachMediaStream(remoteStreamAudio, new MediaStream(audioTracks))
              Janus.attachMediaStream(remoteStreamVideo, new MediaStream(videoTracks))
            },
            oncleanup: function () {
              console.log(' ::: Got a cleanup notification :::')
            },
          })
        },
        error: function (error) {
          started.value = false
          registered.value = false
          console.error(error)
          console.error(`Janus error: ${error}`)
          // reject()
        },
        destroyed: function () {
          started.value = false
          registered.value = false
          // reject()
        },
      })
    },
  })
})
const call = (to, video) => {
  calling.play()
  const sipUri = `sip:admin@healing119.id`
  getSupportedDevices(() => {
    console.log('This is a SIP call')
    sipcall.value.createOffer({
      media: {
        audioSend: true,
        audioRecv: true,
        videoSend: video,
        videoRecv: video,
      },
      success: function (jsep) {
        console.debug('Got SDP!')
        console.debug(jsep)
        const body = {
          request: 'call',
          uri: sipUri,
        }
        sipcall.value.send({
          message: body,
          jsep,
        })
        counterpartNum.value = to
      },
      error: function (error) {
        console.error('WebRTC error...', error)
      },
    })
  })
}

const handleCallNumber = () => {
  console.log('testing call')
  call('085603638794', false)
}
</script>
