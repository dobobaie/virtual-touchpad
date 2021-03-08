<template>
  <div id="board">
    <div class="frequencies logo"></div>
    <div
      v-for="(track, index) in Object.values(tracks).reverse()"
      :key="'frequencies' + index"
      class="frequencies"
      :class="{
        [track.type]: true,
        [track.currentColor.name]: true,
      }"
      @click="pressPad(track.note)"
    >
      <span class="track-note">{{ track.note }}</span>
      <div
        v-for="(frequency, index2) in track.frequencies"
        :key="'frequency' + index + '_' + index2"
        :style="{ height: ((frequency * 100) / 256) + '%' }"
        class="p"
      ></div>
      <span class="track-border"></span>
    </div>
  </div>
</template>

<script>
import WebMidi from 'webmidi';
import MediaStreamRecorder from 'msr';

console.log('REQUIRE', MediaStreamRecorder);

// Notes
// const SIZE_SPACE = 1;
// const NUMBER_TOUCH_PAD = 8;
const MAX_NUMBER_PAD = 98;
const MIN_NUBMBER_PAD = 11;
// const TIME_INTERVAL_NUMBER = 500;
const NOTE_SYSTEM = 240;
const NOTE_CONTROL = 176;
const NOTE_ON = 0x90;
const NOTE_OFF = 0x80;

const notes = {
  NONE: 'none',
  SYSTEM: 'system',
  CONTROL: 'control',
  CONTROL_2: 'control_2',
  PAD: 'pad',
};

const colors = {
  none: {
    name: 'none',
    number: 0,
  },
  white: {
    name: 'white',
    number: 3,
  },
  grey: {
    name: 'grey',
    number: 3,
  },
  pink: { 
    name: 'pink',
    number: 4,
  },
  red: { 
    name: 'red',
    number: 5,
  },
  beige: { 
    name: 'beige',
    number: 8,
  },
  orange: { 
    name: 'orange',
    number: 9,
  },
  beige_bis: { 
    name: 'beige_bis',
    number: 11,
  },
  green_light: { 
    name: 'green_light',
    number: 13,
  },
  turquoise_light: { 
    name: 'turquoise_light',
    number: 16,
  },
  green_medium: { 
    name: 'green_medium',
    number: 18,
  },
  turquoise_medium: { 
    name: 'turquoise_medium',
    number: 20,
  },
  green: { 
    name: 'green',
    number: 22,
  },
  turquoise: { 
    name: 'turquoise',
    number: 24,
  },
  green_dark: { 
    name: 'green_dark',
    number: 26,
  },
  turquoise_dark: { 
    name: 'turquoise_dark',
    number: 28,
  },
  blue_light: { 
    name: 'blue_light',
    number: 30,
  },
  blue: { 
    name: 'blue',
    number: 32,
  },
  blue_dark: { 
    name: 'blue_dark',
    number: 34,
  },
  purple: { 
    name: 'purple',
    number: 49,
  },
  purple_dark: { 
    name: 'purple_dark',
    number: 58,
  },
};

export default {
  data: () => ({
    notes,

    devices: {
      input: null,
      output: null,
    },

    controlFocused: null,
    noteFocused: null,
    tracks: {},

    recordingModeInterval: null,
    setLoopModeEnabled: false,
    recordTrackModeEnabled: false,
    recordAudioModeEnabled: false,

    mediaRecorder: null,
    audioChunks: [],
  }),
  methods: {
    pressPad(noteNumber) {
      this.tracks[noteNumber].press();
    },
    initPad() {
      // Create and switch to session
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 13, 16, 1, 247]);
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 13, 0, 0, 247]);
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 14, 16, 1, 247]);
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 14, 0, 0, 247]);
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 12, 16, 1, 247]);
      this.devices.output.send(NOTE_SYSTEM, [0, 32, 41, 2, 12, 0, 0, 247]);

      // Initialization pad
      this.initPads();
      this.initControl();
      this.resetPad();
      this.initBoard();
      this.initPadEvent();
      this.initControlEvent();
    },
    getTouchColor(pos) {
      switch (this.tracks[pos].type) {
        case notes.PAD:
          if (!this.recordAudioModeEnabled && this.noteFocused === pos) {
            return colors.green;
          }
          if (this.setLoopModeEnabled) {
            return (this.tracks[pos].loopEnabled && colors.green) || colors.orange;
          }
          if (this.recordTrackModeEnabled) {
            return (this.tracks[pos].recordingEnabled && colors.red) || colors.pink;
          }
          return (this.tracks[pos].audio && colors.blue) || colors.white;
        case notes.CONTROL:
          if (this.recordingModeInterval && pos === 91) {
            return (this.tracks[pos].currentColor.name === colors.red.name && colors.pink) || colors.red;
          }
          if (this.tracks[pos].isEnabled) {
            return colors.green;
          }
          return colors.pink;
        case notes.CONTROL_2:
          if (this.tracks[pos].isEnabled) {
            return colors.green;
          }
          return colors.orange;
        default:
          return colors.grey;
      }
    },
    setColor(entry, pos, color) {
      const _color = typeof color === 'function' ? color(pos) : color;
      this.tracks[pos].currentColor = _color;
      this.devices.output.send(entry, [pos, _color?.number]);
    },
    isControl(notePad) {
      return notePad % 10 === 0 ||
        (notePad <= 98 && notePad >= 91) ||
        notePad % 10 === 9;
    },
    initPads() {
      for (let i = MIN_NUBMBER_PAD; i <= MAX_NUMBER_PAD; i++) {
        if (this.isControl(i)) continue;
        this.tracks[i] = {
          type: notes.PAD,
          note: i,
          track: null,
          audio: null,
          audioChunks: [],
          frequencies: [],
          loopEnabled: false,
          currentColor: colors.none,
          turnOff: () => this.setColor(NOTE_OFF, i, colors.none),
          setColor: (color) => this.setColor(NOTE_ON, i, color),
          press: () => this.execPadEvent({ noteNumber: i }),
        };
      }
    },
    initControl() {
      for (let i = MIN_NUBMBER_PAD; i <= MAX_NUMBER_PAD; i++) {
        if (!this.isControl(i)) continue;
        this.tracks[i] = {
          type: (i <= 98 && i >= 95 && notes.SYSTEM) ||
            (i <= 94 && i >= 91 && notes.CONTROL) ||
            (i % 10 === 9 && notes.CONTROL_2) ||
            (i % 10 === 0 && notes.NONE),
          note: i,
          isEnabled: false,
          currentColor: colors.none,
          turnOff: () => this.setColor(NOTE_CONTROL, i, colors.none),
          setColor: (color) => this.setColor(NOTE_CONTROL, i, color),
          press: () => this.execControlEvent({ noteNumber: i }),
        };
      }
    },
    resetPad() {
      for (let i = MIN_NUBMBER_PAD; i <= MAX_NUMBER_PAD; i++) {
        this.tracks[i].turnOff();
      }
    },
    initBoard() {
      this.fillBoardColor(notes.PAD);
      this.fillBoardColor(notes.SYSTEM);
      this.fillBoardColor(notes.CONTROL);
      this.fillBoardColor(notes.CONTROL_2);
    },
    fillBoardColor(type, color) {
      for (let i = MIN_NUBMBER_PAD; i <= MAX_NUMBER_PAD; i++) {
        if (this.tracks[i].type !== type) continue;
        this.tracks[i].setColor(color || this.getTouchColor(i));
      }
    },
    initPadEvent() {
      this.devices.input.addListener('noteon', 'all', (e) => {
        console.log('Pad event', e);
        this.execPadEvent({
          noteNumber: e.note.number
        });
      });
    },
    execPadEvent({ noteNumber }) {
      // ---
      if (this.recordAudioModeEnabled && this.noteFocused !== noteNumber) {
        this.stopAudioRecording(this.noteFocused);
        this.tracks[this.noteFocused].recordingEnabled = false;
        this.tracks[this.noteFocused].setColor(this.getTouchColor(noteNumber));
      }

      // ---
      this.noteFocused = this.noteFocused !== noteNumber && noteNumber;

      // ---
      this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));
      
      // ---
      if (this.recordTrackModeEnabled) {
        this.noteFocused
          ? this.startAudioRecording(noteNumber)
          : this.stopAudioRecording(noteNumber);
        this.tracks[noteNumber].recordingEnabled = !!this.noteFocused;
      }
      
      // ---
      if (this.setLoopModeEnabled) {
        this.tracks[noteNumber].loopEnabled = !this.tracks[noteNumber].loopEnabled;
      }
      
      // ---
      if (!this.recordTrackModeEnabled && !this.setLoopModeEnabled) {
        this.playAudioTrack(noteNumber);
      }

      // ---
      setTimeout(() => {
        this.noteFocused = this.recordTrackModeEnabled && this.noteFocused;
        this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));
      }, 100);
    },
    initControlEvent() {
      this.devices.input.addListener('controlchange', 'all', (e) => {
        console.log('Control event', e);
        if (e.data[2] === 0) return;
        this.execControlEvent({
          noteNumber: e.controller.number
        });
      });
    },
    execControlEvent({ noteNumber }) {
      // ---
      if (this.recordAudioModeEnabled && this.noteFocused) {
        const note = this.noteFocused;
        this.stopAudioRecording(note);
        this.noteFocused = null;
        this.tracks[note].recordingEnabled = false;
        this.tracks[note].setColor(this.getTouchColor(note));
      }

      // ---
      if (this.controlFocused && this.controlFocused !== noteNumber) {
        this.tracks[this.controlFocused].isEnabled = false;
        this.tracks[this.controlFocused].setColor(this.getTouchColor(noteNumber));
      }      

      // ---
      this.tracks[noteNumber].isEnabled = !this.tracks[noteNumber].isEnabled;
      this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));

      // ---
      this.controlFocused = noteNumber === this.controlFocused ? null : noteNumber;

      // ---
      if (noteNumber === 91 && this.recordingModeInterval === null) {
        this.recordingModeInterval = setInterval(() =>
          this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber))
        , 500);
        this.recordTrackModeEnabled = false;
        this.setLoopModeEnabled = false;
        this.fillBoardColor(notes.PAD);
      } else if (
        this.recordingModeInterval &&
        this.tracks[noteNumber].type === notes.CONTROL
      ) {
        if (noteNumber === 91) {
          // save
        }
        clearInterval(this.recordingModeInterval);
        this.recordingModeInterval = null;
        this.tracks[91].setColor(this.getTouchColor(noteNumber))
      }

      // ---
      if (noteNumber === 92) {
        this.setLoopModeEnabled = false;
        this.recordTrackModeEnabled = !this.recordTrackModeEnabled;
        this.fillBoardColor(notes.PAD);
      }
      
      // ---
      if (noteNumber === 93) {
        this.recordTrackModeEnabled = false;
        this.setLoopModeEnabled = !this.setLoopModeEnabled;
        this.fillBoardColor(notes.PAD);
      }
    },

    startFrequenciesRecording(noteNumber) {
      const audioCtx = new AudioContext();
      const analyser = audioCtx.createAnalyser();
      const source = audioCtx.createMediaElementSource(this.tracks[noteNumber].audio);

      source.connect(analyser);
      analyser.connect(audioCtx.destination);
      analyser.fftSize = 32;
      
      
      if (this.tracks[noteNumber].frequencyInterval) {
        clearInterval(this.tracks[noteNumber].frequencyInterval);
      }
      this.tracks[noteNumber].frequencies = new Uint8Array(analyser.frequencyBinCount);
      this.tracks[noteNumber].frequencyInterval = setInterval(() => {
        analyser.getByteFrequencyData(this.tracks[noteNumber].frequencies);
        this.$forceUpdate();
      }, 50);
    },
    stopFrequenciesRecording(noteNumber) {
      this.tracks[noteNumber].frequencies.fill(0);
      this.$forceUpdate();
      clearInterval(this.tracks[noteNumber].frequencyInterval);
    },

    
    playAudioTrack(noteNumber) {
      let audio = this.tracks[noteNumber].audio;
      const track = this.tracks[noteNumber].track;
      if (track) {
        if (audio) {
          audio.pause();
        }
        this.tracks[noteNumber].audio = new Audio(track);
        audio = this.tracks[noteNumber].audio;
        audio.currentTime = 0.16;
        audio.play();
        this.startFrequenciesRecording(noteNumber);
        this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));
        audio.addEventListener('ended', () => {
          if (!this.tracks[noteNumber].loopEnabled) {
            this.tracks[noteNumber].audio = null;
            this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));
            this.stopFrequenciesRecording(noteNumber);
            return ;
          }
          audio.currentTime = 0.20;
          audio.play();
          this.tracks[noteNumber].setColor(this.getTouchColor(noteNumber));
        }, false);
        this.tracks[noteNumber].audio = audio;
      }
    },
    startAudioRecording(noteNumber) {
      console.log('Start recording', noteNumber);

      this.recordAudioModeEnabled = true;
      navigator.mediaDevices.getUserMedia({ audio: true })
        .then(stream => {
          this.mediaRecorder = new MediaRecorder(stream);
          this.mediaRecorder.start();
          this.mediaRecorder.addEventListener('dataavailable', event =>
            this.tracks[noteNumber].audioChunks.push(event.data)
          );
        })
    },
    stopAudioRecording(noteNumber) {
      console.log('Stop recording', noteNumber);

      this.mediaRecorder.addEventListener('stop', () => {
        const audioBlob = new Blob(this.tracks[noteNumber].audioChunks);
        const audioUrl = URL.createObjectURL(audioBlob);
        this.tracks[noteNumber].track = audioUrl;
        this.tracks[noteNumber].audioChunks = [];
        console.log(audioUrl);
      })
      this.recordAudioModeEnabled = false;
      this.mediaRecorder.stop();
    }
  },
  mounted() {
    const self = this;
    WebMidi.enable(function (err) {
      console.log('Error', err);
      WebMidi.addListener('connected', function(e) {
        console.log('Device connected', e);
        if (e.port.type === 'input' && e.port.name === 'LPMiniMK3 MIDI') {
          self.devices.input = e.port;
          self.devices.output && self.initPad();
        }
        if (e.port.type === 'output' && e.port.name === 'LPMiniMK3 MIDI') {
          self.devices.output = e.port;
          self.devices.input && self.initPad();
        }
      });
      WebMidi.addListener('disconnected', function(e) {
        console.log('Device disconnected', e);
        if (e.port.type === 'input' && e.port.name === 'LPMiniMK3 MIDI') {
          self.devices.input = null;
        }
        if (e.port.type === 'output' && e.port.name === 'LPMiniMK3 MIDI') {
          self.devices.output = null;
        }
      });
    }, true);
  }
}
</script>

<style scoped>
#board, .frequencies {
  display: flex;
  flex-direction: row;
  flex-wrap: wrap;
}

#board {
  flex-direction: row-reverse;
  max-width: 800px;
  border: 1px solid #dadada;
  margin: 0 auto;
  background: #f6f6f6;
  justify-content: center;
}

.frequencies {
  cursor: pointer;
  position: relative;
  margin: 3px;
  background-color: #dddddd;
  width: calc(100% / 10.3);
  height: 75px;
  flex-wrap: initial;
}

.frequencies.logo {
  background-color: #ffffff;
}

.frequencies.none {
  display: none;
}

.frequencies.red {
  background-color: #f45b5b;
}

.frequencies.pink {
  background-color: #ffc0cb;
}

.frequencies.grey {
  background-color: #7b7b7b;
}

.frequencies.orange {
  background-color: #ffb000;
}

.frequencies.green {
  background-color: #3fff00;
}

.frequencies .track-note {
  position: absolute;
  top: -2px;
}

.frequencies.pad .track-border {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 10px;
  background-color: #00dcff;
}

.frequencies .p {
  overflow: hidden;
  background-color: #00dcff;
  min-height: 10px;
  width: inherit;
  margin-top: auto;
}
</style>
