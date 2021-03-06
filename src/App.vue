<template>
  <a-scene :networked-scene="'serverURL: https://nxr.lifescope.io; app: lifescope-xr; room: ls-room; audio: true; adapter: easyrtc; connectOnLoad: true;'">

    <!-- Load assets -->
    <a-assets class="aframe-assets">
      <img id="sky" src="../static/gallery/nightsky.jpg">
      <img id="floor" src="../static/gallery/wood-panel.jpg">
      <img id="earth" src="../static/gallery/Albedo.jpg">
      <!-- video controls -->
      <img id="video-play" src="https://s3.amazonaws.com/lifescope-static/static/xr/gallery/video_play.png"
        crossorigin="anonymous">
      <img id="video-pause" src="https://s3.amazonaws.com/lifescope-static/static/xr/gallery/video_pause.png"
        crossorigin="anonymous">
      <!-- gltf -->
      <!-- logo -->
      <a-gltf-model id="logo" src="https://s3.amazonaws.com/lifescope-static/static/xr/logo/logo.gltf"
                    crossorigin="anonymous">
      </a-gltf-model>
    </a-assets>

    <!-- gallery -->
    <gallery :LSObjs='LSObjs' :rooms='rooms' :roomConfig='roomConfig'/>

    <!-- Sky id="Sky" -->
    <a-sky src="#sky" rotation="90 0 90">
    </a-sky>

  </a-scene>
</template>

<script>
import axios from 'axios';

import socketIO from 'socket.io-client';
import easyrtc from '../static/easyrtc/easyrtc.js';

import gallery from "./components/gallery.vue";

var CONFIG = {};
CONFIG.DEBUG = true;
import debugListeners from './dev/listeners.js';

if (CONFIG.DEBUG) {console.log("from App.vue <script>");}
export default {
    components: {
        gallery
    },
    data() {
      return {
        LSObjs: [],
        rooms: [],
        roomConfig: {},
        roomName: 'ls-room'
      }
    },

    beforeCreate () {
      if (CONFIG.DEBUG) {console.log("beforeCreate");};
    },

    created () {
      if (CONFIG.DEBUG) {console.log("created");};
    },

    beforeMount () {
      if (CONFIG.DEBUG) {console.log("beforeMount");};
    },

    mounted () {
      if (CONFIG.DEBUG) {console.log("App.vue mounted");};
      // set userHeight after a-scene is available
      document.body.addEventListener('renderstart', function (evt) {
        if (CONFIG.DEBUG) {console.log('renderstart');};
        AFRAME.scenes[0].renderer.vr.userHeight = 0;
      });

      //if (CONFIG.DEBUG) {debugListeners();}
      document.body.addEventListener('move', function (evt) {
        // console.log('move');
        // console.log(evt);
      });
      document.body.addEventListener('rotateY', function (evt) {
        // console.log('rotateY');
        // console.log(evt);
        // console.log(evt.detail.value);
        // var playerRig = document.getElementById("playerRig");
        // console.log(playerRig.getAttribute('rotation'));
      });
      document.body.addEventListener('rotateX', function (evt) {
        // console.log('rotateX');
        // console.log(evt);
        // console.log(evt.detail.value);
      });


      //
      // Add hand when user enters vr mode
      var self = this;
      document.body.addEventListener('enter-vr', function (evt) {
        if (CONFIG.DEBUG) {console.log('entered vr');};
        var rightHand = document.getElementById('rightHandController');
        //typeof array != "undefined" && array != null && array.length != null && array.length > 0
        if (rightHand == null) {
          if (CONFIG.DEBUG) {console.log('adding hand...');};
          self.createRightHandNetworked();
        }
      });


      document.body.addEventListener('connected', function (evt) {
        if (CONFIG.DEBUG) {console.log('connected event. clientId =', evt.detail.clientId);};
        if (CONFIG.DEBUG) {console.log('roomName: ' + self.roomName);};

      });

      // make eyes invisible to user when the avatar is created
      document.body.addEventListener('entityCreated', function (evt) {
        if (evt.detail.el.id === 'playerRig') {
          document.getElementsByClassName('player')[0].getElementsByClassName('face')[0].setAttribute('visible', 'false');
          document.getElementsByClassName('player')[0].getElementsByClassName('head')[0].setAttribute('visible', 'false');
        }
      });
      

      this.getRoomConfig().then((res) => {
          if (CONFIG.DEBUG) {console.log("getRoomConfig().then")};

          this.roomConfig = res.roomConfig;
          //this.roomName = res.roomConfig.ROOM_NAME;

          this.getObjs().then((res) => {
            this.LSObjs = res.LSObjs;
            this.rooms = res.rooms;

            this.createAvatarRigTemplate();
            this.addAvatarRigTemplate();
            this.createNetworkedPlayer();

            if (AFRAME.utils.device.isMobile()) {
              if (CONFIG.DEBUG) {console.log("isMobile");};
              var playerRig = document.getElementById('playerRig');
              playerRig.setAttribute("virtual-gamepad-controls", {});
              var camera = document.getElementById('player-camera');
              var sceneEl = document.getElementsByTagName('a-scene')[0];
              //this.eventHandlers.push(new TouchEventsHandler(this.cursor, this.cameraController, this.cursor.el));
              sceneEl.setAttribute("look-on-mobile", "camera", camera);
              sceneEl.setAttribute("look-on-mobile", "verticalLookSpeedRatio", 3);
            } else {
              if (CONFIG.DEBUG) {console.log("!isMobile");};
              var playerRig = document.getElementById('playerRig');
              playerRig.setAttribute("look-controls", "reverseMouseDrag:true");
            }
            }
          );
        }
      );
    },


    methods: {
      textString: function (value) {
            return 'width: 1.5; color: white; value: ' + value
      },
      logText: function (value) {
        var logtext = document.getElementById('log-text');
        logtext.setAttribute('text', this.textString(value));
      },
      testButtonAction: function () {
        if (CONFIG.DEBUG) {console.log("testButtonAction");};
      },
      
      getRoomConfig () {
        if (CONFIG.DEBUG) {console.log("getRoomConfig");};
        return axios.get("/roomconfig")
        .then((res) => {
          //console.log(res.data);
          return {roomConfig: res.data}
        })
      },

      getObjs () {
        if (CONFIG.DEBUG) {console.log("getObjs");};
        
        var x = '/' + this.roomConfig.BUCKET_PATH;

        if (!this.$route.query.room){
          this.$route.query.room = 'ls-room';
        }

        this.roomName = this.$route.query.room || 'ls-room';

        if (CONFIG.DEBUG) {console.log(x);};
        return axios.get(x)
        .then((res) => {
          var result = [];
          var rooms = Object.keys(res.data);
          var someData = res.data[this.roomName].forEach(element => {
            result.push(element);
          });
          return { LSObjs: result, rooms: rooms }
        })
      },

      createAvatarRigTemplate() {
        var frag = this.fragmentFromString(`
        <template id="avatar-rig-template" v-pre>
          <a-entity class="player">

            <a-entity class="avatar" networked-audio-source >
              <a-sphere class="head"
                color="#5985ff"
                scale="0.45 0.5 0.4"
              ></a-sphere>
              <a-entity class="face"
                position="0 0.05 0"
              >
                <a-sphere class="eye"
                  color="#efefef"
                  position="0.16 0.1 -0.35"
                  scale="0.12 0.12 0.12"
                >
                  <a-sphere class="pupil"
                    color="#000"
                    position="0 0 -1"
                    scale="0.2 0.2 0.2"
                  ></a-sphere>
                </a-sphere>
                <a-sphere class="eye"
                  color="#efefef"
                  position="-0.16 0.1 -0.35"
                  scale="0.12 0.12 0.12"
                >
                  <a-sphere class="pupil"
                    color="#000"
                    position="0 0 -1"
                    scale="0.2 0.2 0.2"
                  ></a-sphere>
                </a-sphere>
                <a-sphere class="eye dev"
                  color="#efefef"
                  position="0 0.1 0.35"
                  scale="0.12 0.12 0.12"
                >
                  <a-sphere class="pupil dev"
                    color="#000"
                    position="0 0 1"
                    scale="0.2 0.2 0.2"
                  ></a-sphere>
                </a-sphere>
              </a-entity>
            </a-entity>

          </a-entity>
        </template> 
        `);

        document.getElementsByClassName('aframe-assets')[0].appendChild(frag);

      },

      addAvatarRigTemplate() {
        if (CONFIG.DEBUG) {console.log("addAvatarRigTemplate");};
        NAF.schemas.add({
          template: '#avatar-rig-template',
          components: [
            {
              component: 'position'
            },
            {
              component: 'rotation'
            },
            {
              selector: '.avatar',
              component: 'rotation'
            }
          ]
       });
      },


          // <a-entity id="cursor"
          //       cursor="fuse: true; fuseTimeout: 500"
          //       position="0 0 -1"
          //       geometry="primitive: ring; radiusInner: 0.02; radiusOuter: 0.03"
          //       material="color: black; shader: flat">
          //     </a-entity>

      // avatar-rig="camera:#player-camera;"
      // pitch-yaw-rotator
      // look-controls="reverseMouseDrag:true"
      
      createNetworkedPlayer() {
        var frag = this.fragmentFromString(`
        <a-entity id="playerRig"
          position="0 1.6 -2"
          wasd-controls
          networked="template:#avatar-rig-template;attachTemplateToLocal:true;"
          
          character-controller="pivot: #player-camera"
          >
         
         <a-entity id="player-camera"
            class="camera"
            camera
            pitch-yaw-rotator
          >
          
          <a-gui-cursor id="cursor"
					  raycaster="interval: 1000; objects: gui-interactable, .clickable"
					  fuse="true" fuse-timeout="2000"
					  design="dot"
			      >
			    </a-gui-cursor>
          
          </a-entity>
          <a-entity id="rightHandRig"
            class="hand"
          ></a-entity>
        </a-entity>`);
        document.getElementsByTagName('a-scene')[0].appendChild(frag);
      },


      createRightHandNetworked() {
        var frag = this.fragmentFromString(`
        <a-entity id="rightHandController"
           teleport-controls="cameraRig: #playerRig; teleportOrigin: #player-camera; startEvents: teleportstart; endEvents: teleportend; collisionEntities:.boundry; landingNormal: 0 0 1;"
            daydream-controls="hand: right;"
            oculus-touch-controls="hand: right"
            vive-controls="hand: right"
            windows-motion-controls="hand: right"
            gearvr-controls="hand: right"
            oculus-go-controls="hand: right">
         </a-entity>`);
        document.getElementById('playerRig').appendChild(frag);
      },

      fragmentFromString(strHTML) {
            return document.createRange().createContextualFragment(strHTML);
      },

    }
  }
</script>

<style src="./app.css"></style>

