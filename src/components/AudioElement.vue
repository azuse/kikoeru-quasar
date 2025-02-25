<template>
  <vue-plyr
    ref="plyr"
    :emit="['loadedmetadata', 'canplay', 'timeupdate', 'ended', 'seeked', 'playing', 'waiting', 'pause']"
    @loadedmetadata="resumePlayHistory()"
    @canplay="onCanplay()"
    @timeupdate="onTimeupdate()"
    @ended="onEnded()"
    @seeked="onSeeked()"
    @playing="onPlaying()"
    @waiting="onWaiting()"
    @pause="onPause()"
  >
    <audio crossorigin="anonymous" >
      <source v-if="source" :src="source" />
    </audio>
  </vue-plyr>
</template>

<script>
import Lyric from 'lrc-file-parser'
import { mapState, mapGetters, mapMutations } from 'vuex'
import NotifyMixin from '../mixins/Notification.js'

export default {
  name: 'AudioElement',

  mixins: [NotifyMixin],

  data() {
    return {
      lrcObj: null,
      lrcAvailable: false,
    }
  },

  computed: {
    player () {
      return this.$refs.plyr.player
    },

    source () {
      // 从 LocalStorage 中读取 token
      const token = this.$q.localStorage.getItem('jwt-token') || ''
      // New API
      if (this.currentPlayingFile.mediaStreamUrl) {
        return `${this.currentPlayingFile.mediaStreamUrl}?token=${token}`
      } else if (this.currentPlayingFile.hash) {
        // Fallback to be compatible with old backend
        return `/api/media/stream/${this.currentPlayingFile.hash}?token=${token}`
      } else {
        return ""
      }
    },

    ...mapState('AudioPlayer', [
      'playing',
      'queue',
      'queueIndex',
      'playMode',
      'muted',
      'volume',
      'sleepTime',
      'sleepMode',
      'rewindSeekTime',
      'forwardSeekTime',
      'rewindSeekMode',
      'forwardSeekMode'
    ]),

    ...mapGetters('AudioPlayer', [
      'currentPlayingFile'
    ])
  },

  watch: {
    playing (flag) {
      if (this.player.duration) {
        // 缓冲至可播放状态
        flag ? this.player.play() : this.player.pause()
      }
      // this.playLrc(flag);
    },

    // watch source -> media.load() -> canPlay -> player.play()
    source (url) {
      if (url) {
        // 加载新音频/视频文件
        this.player.media.load();
        this.loadLrcFile();
      }
    },

    muted (flag) {
      // 切换静音状态
      this.player.muted = flag
    },

    volume (val) {
      // 屏蔽非法数值
      if (val < 0 || val > 1) {
        return
      }

      // 调节音量
      this.player.volume = val
    },
    rewindSeekMode(rewind) {
      if (rewind) {
        this.player.rewind(this.rewindSeekTime);
        this.SET_REWIND_SEEK_MODE(false);
      }
    },
    forwardSeekMode(forward) {
      if (forward) {
        this.player.forward(this.forwardSeekTime);
        this.SET_FORWARD_SEEK_MODE(false);
      }
    }
  },

  methods: {
    /**
     * 当 外部暂停（线控暂停、软件切换）、用户控制暂停、seek 时会触发本事件
     */
    onPause() {
      // console.log('onPause')
      this.playLrc(false)
      this.PAUSE()

      // save play history on pause
      this.lastHistoryUpdateTime = this.player.currentTime
      this.updatePlayHistory(this.player.currentTime, this.player.duration)
    },
    /**
     * 当播放器真正开始播放时会触发本事件
     */
    onPlaying() {
      // console.log('playing')
      this.playLrc(true)
      this.PLAY()
    },
    /**
     * 当播放器缓冲区空，被迫暂停加载时会触发本事件
     */
    onWaiting() {
      // console.log('waiting')
      this.playLrc(false)
      this.PLAY()
    },
    ...mapMutations('AudioPlayer', [
      'SET_DURATION',
      'SET_CURRENT_TIME',
      'PAUSE',
      'PLAY',
      'SET_TRACK',
      'NEXT_TRACK',
      'SET_CURRENT_LYRIC',
      'SET_VOLUME',
      'CLEAR_SLEEP_MODE',
      'SET_REWIND_SEEK_MODE',
      'SET_FORWARD_SEEK_MODE'
    ]),

    onCanplay () {
      // 缓冲至可播放状态时触发 (只有缓冲至可播放状态, 才能获取媒体文件的播放时长)
      this.SET_DURATION(this.player.duration)

      // 播放
      if (this.playing && this.player.currentTime !== this.player.duration) {
        this.player.play()
      }
    },

    onTimeupdate () {
      // 当目前的播放位置已更改时触发
      this.SET_CURRENT_TIME(this.player.currentTime)
      if (this.sleepMode && this.sleepTime) {
        const currentTime = new Date()
        const currentHourStr = currentTime.getHours().toString().padStart(2, '0')
        const currentMinuteStr = currentTime.getMinutes().toString().padStart(2, '0')
        const sleepHourStr = this.sleepTime.match(/\d+/g)[0]
        const sleepMinuteStr = this.sleepTime.match(/\d+/g)[1]
        if (currentHourStr === sleepHourStr && currentMinuteStr === sleepMinuteStr) {
          this.PAUSE()
          this.CLEAR_SLEEP_MODE()
          // Persist sleep mode settings
          this.$q.sessionStorage.set('sleepTime', null)
          this.$q.sessionStorage.set('sleepMode', false)
        }
      }

      // 更新历史记录
      const UPDATE_INTERVAL = 1
      if (this.lastHistoryUpdateTime == null) {
        this.lastHistoryUpdateTime = this.player.currentTime
        this.updatePlayHistory(this.player.currentTime, this.player.duration)
      } else if (this.player.currentTime > this.lastHistoryUpdateTime + UPDATE_INTERVAL) {
        this.lastHistoryUpdateTime = this.player.currentTime
        this.updatePlayHistory(this.player.currentTime, this.player.duration)
      }

    },

    onEnded () {
      console.log("onEnded")
      this.dontResume = true
      // 当前文件播放结束时触发
      switch (this.playMode.name) {
        case "all repeat":
          // 循环播放
          if (this.queueIndex === this.queue.length - 1) {
            this.SET_TRACK(0)
          } else {
            this.NEXT_TRACK()
          }
          break
        case "repeat once":
          // 单曲循环
          this.player.currentTime = 0
          this.player.play()
          this.PLAY()
          break
        case "shuffle": {
          // 随机播放
          const index = Math.floor(Math.random()*this.queue.length)
          this.SET_TRACK(index)
          if (index === this.queueIndex) {
            this.player.currentTime = 0
          }
          break
        }
        default:
          // 顺序播放
          if (this.queueIndex === this.queue.length - 1) {
            this.PAUSE()
          } else {
            this.NEXT_TRACK()
          }
      }
    },

    onSeeked() {
      // if (this.lrcAvailable) {
      //   this.lrcObj.play(this.player.currentTime * 1000);
      //   if (!this.playing) {
      //     this.lrcObj.pause();
      //   }
      // }
    },


    playLrc (playStatus) {
      if (this.lrcAvailable) {
        if (playStatus) {
          this.lrcObj.play(this.player.currentTime * 1000);
        } else {
          this.lrcObj.pause();
        }
      }
    },

    initLrcObj () {
        this.lrcObj = new Lyric({
          onPlay: (line, text) => {
            this.SET_CURRENT_LYRIC(text);
          },
        })
    },

    loadLrcFile () {
      const token = this.$q.localStorage.getItem('jwt-token') || '';
      const fileHash = this.queue[this.queueIndex].hash;
      const url = `/api/media/check-lrc/${fileHash}?token=${token}`;

      this.$axios.get(url)
        .then((response) => {
          if (response.data.result) {
            // 有lrc歌词文件
            this.lrcAvailable = true;
            console.log('读入歌词');
            const lrcUrl = `/api/media/stream/${response.data.hash}?token=${token}`;
            this.$axios.get(lrcUrl)
              .then(response => {
                console.log('歌词读入成功');
                this.lrcObj.setLyric(response.data);
                this.lrcObj.play(this.player.currentTime * 1000);
              });
          } else {
            // 无歌词文件
            this.lrcAvailable = false;
            this.lrcObj.setLyric('');
            this.SET_CURRENT_LYRIC('');
          }
        })
        .catch((error) => {
          if (error.response) {
            // 请求已发出，但服务器响应的状态码不在 2xx 范围内
            if (error.response.status !== 401) {
              this.showErrNotif(error.response.data.error || `${error.response.status} ${error.response.statusText}`);
            }
          } else {
            this.showErrNotif(error.message || error);
          }
        })
    },

    updatePlayHistory(currentTime = 0, totalTime = 0) {
      const fileName = this.currentPlayingFile.title
      const workID = this.currentPlayingFile.hash.split('/')[0]
      const fileIndex = this.currentPlayingFile.hash.split('/')[1]

      let payload = {
        "work_id": parseInt(workID),
        "file_index": parseInt(fileIndex),
        "file_name": fileName,
        "play_time": currentTime,
        "total_time": totalTime
      }

      this.$axios.put('/api/history', payload)

      console.log(fileName, fileIndex)
    },

    resumePlayHistory() {
      if (this.dontResume) {
        this.dontResume = false
        return
      }

      const workID = this.currentPlayingFile.hash.split('/')[0]
      const fileIndex = this.currentPlayingFile.hash.split('/')[1]

      if (workID == null || fileIndex == null) return

      this.$axios.get('/api/history/getByWorkIdIndex', {
        params: {
          work_id: workID,
          file_index: fileIndex
        }
      })
      .then((response) => {
        console.log(response)
        if (response.data.length < 1) {
          return
        }

        let continueTime = response.data[0].play_time
        let totalTime = response.data[0].total_time

        if (continueTime == 0) return
        if (totalTime <= continueTime) return

        this.player.currentTime = parseInt(continueTime);

        console.log('set current time to', continueTime)
        console.log('current time is ', this.player.currentTime)
        console.log(this.player)

        function formatSeconds (seconds) {
          let h = Math.floor(seconds / 3600) < 10
            ? '0' + Math.floor(seconds / 3600)
            : Math.floor(seconds / 3600)

          let m = Math.floor((seconds / 60 % 60)) < 10
            ? '0' + Math.floor((seconds / 60 % 60))
            : Math.floor((seconds / 60 % 60))

          let s = Math.floor((seconds % 60)) < 10
            ? '0' + Math.floor((seconds % 60))
            : Math.floor((seconds % 60))

          return h === "00"
            ? m + ":" + s
            : h + ":" + m + ":" + s
        }

        this.showSuccNotif(`继续从${formatSeconds(continueTime)}开始播放`)
      })
      .catch((err) => {
        console.log(err)
      })

    }

  },

  mounted () {
    // 初始化音量
    this.SET_VOLUME(this.player.volume);
    this.initLrcObj();
    if (this.source) {
      this.loadLrcFile();
    }
    console.log(this.player)
    this.player.vhs.options_.externHls.MAX_GOAL_BUFFER_LENGTH = 600;
  }
}
</script>
