<template lang="pug">
#app
  navbar
  vk-modal(:show.sync="modalShow")
    vk-modal-title
      | Error
    p
      | {{modalText}}
    p.uk-text-right
      vk-button.uk-margin-small-right(@click="modalShow = false")
        | Close
  .container
    .panel
      div(v-if="stage === 'prepare'")
        input.hidden(type="file", ref="textLoader", accept="text/plain", @change="readText")
        vk-button(@click="loadText")
          | Load Text File
        .uk-margin
          textarea.uk-textarea(rows="20", placeholder="Subtitle Preview", v-model="subtitleText")
        p.uk-margin
          | Lines of Subtitle: {{ subtitleText.split('\n').length }} line(s)
        vk-button.uk-margin(type="primary", @click="startEdit")
          | Start Editing
      div(v-if="stage === 'edit'")
        h4
          | React Time: {{ reactTime }}
        input.uk-range(type="range", min="0.0" max="1.0", step="0.01", v-model="reactTime")
        h2
          | Current Line
        h4.alt-text(v-if="currentLine === null")
          | [Empty]
        h4(v-else)
          | {{ subtitles[currentLine] }}
        h2
          | Coming Lines
        h4.alt-text(v-for="subtitle in nextLines")
          | {{ subtitle }}
        h4.alt-text(v-if="nextLines.length < 4")
          | [End of File]
        vk-button.uk-margin(type="primary", @click="startReview")
          | Start Reviewing
      div(v-if="stage === 'review'")
        textarea.uk-textarea(rows="20", placeholder="Subtitle Preview", v-model="subtitleReview")
        vk-button.uk-margin(type="primary", @click="saveFile")
          | Save File
        a.hidden(ref="download", href="")
    .panel
      input.hidden(
        type="file",
        ref="videoLoader",
        accept="audio/mp4, video/mp4",
        @change="readVideo")
      vk-button(@click="loadVideo", v-if="stage === 'prepare'")
        | Load Video
      video.video.uk-margin(
        ref="video",
        src="http://commondatastorage.googleapis.com/gtv-videos-bucket/sample/BigBuckBunny.mp4",
        controls)
      shortcut(v-show="stage === 'edit'")
</template>

<script>
import Navbar from './components/Navbar';
import Shortcut from './components/Shortcut';

export default {
  name: 'App',
  components: {
    Navbar,
    Shortcut,
  },
  data() {
    return {
      modalShow: false,
      modalText: '',
      stage: 'prepare',
      subtitleText: '',
      subtitles: [],
      subtitleStarts: [],
      subtitleEnds: [],
      currentLine: null,
      nextLine: 0,
      previousTiming: 0,
      reactTime: 0.3,
      subtitleReview: '',
    };
  },
  computed: {
    nextLines() {
      return this.subtitles.slice(this.nextLine, this.nextLine + 4);
    },
  },
  beforeDestroy() {
    // Makesure remove all event handlers
    window.removeEventListener('keypress', this.keyHandler);
  },
  methods: {
    loadText() {
      this.$refs.textLoader.click();
    },
    loadVideo() {
      this.$refs.videoLoader.click();
    },
    readText(evt) {
      const filename = evt.target.files[0];
      this.subtitleText = ''; // Empty the previous results

      const reader = new FileReader();
      reader.onload = () => {
        this.subtitleText = reader.result;
      };

      reader.readAsText(filename);
    },
    readVideo(evt) {
      const filename = evt.target.files[0];
      const url = URL.createObjectURL(filename);
      this.$refs.video.src = url;
      this.$refs.video.load();
    },
    startEdit() {
      if (this.subtitleText.length === 0) {
        this.modalShow = true;
        this.modalText = 'Please import the subtitle text first.';
      } else {
        this.subtitles = this.subtitleText.split('\n');
        this.subtitleStarts = new Array(this.subtitles.length).fill(null);
        this.subtitleEnds = new Array(this.subtitles.length).fill(null);
        this.stage = 'edit';
        window.addEventListener('keypress', this.keyHandler);
      }
    },
    startReview() {
      window.removeEventListener('keypress', this.keyHandler);
      this.stage = 'review';
      this.generateSubtitle();
    },
    keyHandler(e) {
      const pressed = String.fromCharCode(e.keyCode).toLowerCase();

      switch (pressed) {
        case 'k':
          // Switch to Next Line
          if (this.currentLine !== null) {
            this.subtitleEnds[this.currentLine] = this.$refs.video.currentTime - 0.01;
          }
          this.currentLine = this.nextLine;
          this.nextLine += 1;
          if (this.currentLine < this.subtitles.length) {
            this.subtitleStarts[this.currentLine] = this.$refs.video.currentTime;
          } else {
            this.currentLine = null;
          }
          break;
        case 'l':
          // Stop Current Line
          if (this.currentLine !== null) {
            this.subtitleEnds[this.currentLine] = this.$refs.video.currentTime - 0.01;
          }
          this.currentLine = null;
          break;
        case 'u':
          // Prev 3 Secs
          this.$refs.video.currentTime -= 3;
          break;
        case 'p':
          // Skip 3 Secs
          this.$refs.video.currentTime += 3;
          break;
        case 'i':
          if (this.nextLine > 0) {
            this.currentLine = this.nextLine - 2;
            this.nextLine = this.nextLine - 1;
          }

          if (this.currentLine === -1) {
            this.currentLine = null;
          }
          break;
        case 'o':
          if (this.nextLine < this.subtitles.length) {
            this.currentLine = this.nextLine;
            this.nextLine = this.nextLine + 1;
          }
          break;
        default:
          break;
      }
    },
    timeFormat(secs) {
      if (secs === null) {
        return '0:0:0,0';
      }
      const hour = Math.floor(secs / 60 / 60);
      const min = Math.floor((secs / 60) % 60);
      const sec = Math.floor(secs % 60);
      const mil = Math.floor((secs * 1000) % 1000);
      return `${hour}:${min}:${sec},${mil}`;
    },
    generateSubtitle() {
      this.subtitleReview = '';
      for (let i = 0; i < this.subtitles.length; i += 1) {
        this.subtitleReview += `${i + 1}\n`;
        this.subtitleReview += `${this.timeFormat(this.subtitleStarts[i])} --> ${this.timeFormat(this.subtitleEnds[i])}\n`;
        this.subtitleReview += `${this.subtitles[i]}\n\n`;
      }
    },
    saveFile() {
      const a = this.$refs.download;
      const file = new Blob([this.subtitleReview], { type: 'text/plain' });
      a.href = URL.createObjectURL(file);
      a.download = 'result.srt';
      a.click();
    },
  },
};
</script>

<style lang="stylus" scoped>
.container
  display flex
  margin 20px
.hidden
  display none
.panel
  flex 1
  margin 10px
.uk-textarea
  resize vertical
.video
  width 100%
.alt-text
  color #aaa
</style>