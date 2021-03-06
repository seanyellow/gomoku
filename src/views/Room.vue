<template lang="pug">
main
  Logo(:name='roomName' @clickRoomName='clickChangeRoomName()')
  .container(v-show='!isLoading')
    .row(v-show='page === `game`')
      #game-area.col-12.col-md.order-3.order-lg-2
        #board-area(:class='{ hidden: isChatShow }')
          Board(ref='board' :game='game' :chess='myChess' :fingerprint='fingerprint' @clickBlock='clickBlock' @sendGame='sendGame' @setRecord='setRecord')
          Result(v-if='$refs.board' :board='$refs.board')
        Status(v-if='$refs.board' :board='$refs.board')
      .col-12.col-md-12.col-lg-4.order-1.order-lg-3
        #chat-toggle-btn(:class='{ new: $refs.chat.notificationCount > 0 }' v-if='$refs.chat')
          a(href='#' @click='toggleChatShow()' :class='{ active: isChatShow }')
            fa.icon(icon='comment-alt')
            span(v-if='!isChatShow || 1')
        #player-list
          transition-group(name='player-item' tag='ul')
            li(v-for='(p, i, index) in players' :title='`加入遊戲時間: ${convertDate(p.date)}`' :key='index' :class='getPlayerItemClass(p.chess)') #[fa(icon='user')]  {{ p.info.name }}
        Chat(:roomId='roomId' :fingerprint='fingerprint' ref='chat' :class='{ hidden: !isChatShow }' :isShow='isChatShow')
        ShareLink(:class='{ hidden: !isChatShow }')
    .row.justify-content-center.align-items-center.mt-3(v-if='page === `name`')
      .col-12.col-md-6.col-lg-5.col-xl-4(v-if='fingerprint')
        Nickname(v-model.trim='newRoomName' @confirm='confirmRoomName' @cancel='page = `game`' :isRoomName='true')

  transition(name='full-loader')
    #full-loader(v-if='isLoading')
      Logo(:isForFullLoader='true').my-0
      .loader
        //- span {{ status[(status.length - 1)] }}
        .progress: .bar(:style='{ width: `${100 * (status.length / 8)}%` }')
</template>

<script>
import Logo from '@/components/Logo.vue'
import Chat from '@/components/Chat.vue'
import Nickname from '@/components/Nickname.vue'
import Board from '@/components/Board.vue'
import ShareLink from '@/components/ShareLink.vue'
import Status from '@/components/Status.vue'
import Result from '@/components/Result.vue'

import { common } from '@/mixins/common'
import fingerprint from '@/assets/js/fingerprint'
import db from '@/assets/js/db'
import room from '@/assets/js/room'
import firebase from 'firebase/app'
import moment from 'moment'
import randomstring from 'randomstring'
import _ from 'lodash'

export default {
  name: 'Room',
  data () {
    return {
      linstenList: [`info`, `players`, `game`],
      info: null,
      players: null,
      game: null,
      roomId: null,
      session: null,
      page: `game`,
      newRoomName: ``,
      isLoggedIn: false,
      isChatShow: false,
    }
  },
  created () {
    this.roomId = this.$route.params.id   // 根據網址參數設定 roomId
    this.session = randomstring.generate({ length: 6 })
  },
  mounted () {
    this.init()
  },
  methods: {
    // 初始化
    async init () {
      this.isLoading = true   // 設定正在載入中

      await this.setProfile()       // 取得玩家本人資料
      await this.checkRoom()        // 檢查房間是否存
      await this.checkRepeatLogin() // 檢查是否重複登入
      await this.checkPlayerCount()
      const chess = await this.chooseChess()
      await this.joinRoom(chess)         // 加入房間

      // 監聽房間內 linstenList 列表中所有子節點
      this.linstenList.forEach(child => {
        if (child === `game`)
          db.onRoom(this.roomId, child, data => {
            this[child] = data
            window.navigator.vibrate(100)
          })
        else
          db.onRoom(this.roomId, child, data => this[child] = data)
      })

      // 當 `本機` 離線時呼叫 disconnect()
      db.connectedRef.on('value', (snap) => {
        if (snap.val() !== true) this.disconnect()
      })

      // 當 `遠端` 偵測本機離線時
      // 1. 移除 players 中的玩家本人
      await db.roomRef.child(`${this.roomId}/players/${this.fingerprint}`).onDisconnect().remove()
      // 2. 聊天室插入離開訊息
      await db.roomRef.child(`${this.roomId}/chat/${this.session}`).onDisconnect().set({
        date: firebase.database.ServerValue.TIMESTAMP,
        content: `${this.profile.name} has accidentally left.`
      })

      this.$nextTick(() => {
        this.sendSystemInfo(`${this.profile.name} has joined.`)
      })

      this.isLoading = false
    },
    clickBlock (nextGame) {
      this.sendGame(nextGame)
    },
    async sendGame (game) {
      db.sendGame(this.roomId, game)
    },
    async setRecord () {
      const winner = this.fingerprint
      const loser = Object.keys(this.players).find((id) => id !== winner) || null
      console.log(winner, loser)
      db.setPlayerRecord(winner, loser)
    },
    async disconnect () {
      this.goToIndex()
    },
    async setProfile () {
      this.status.push(`取得裝置指紋`)
      this.fingerprint = await fingerprint.get()
      this.status.push(`取得玩家資訊`)
      this.profile = await db.getPlayer(this.fingerprint)
      if (!this.profile) {
        this.$router.push({ name: 'Register', query: { roomId: this.roomId } })
        throw `The player hasn't registered.`
      }
    },
    async checkRepeatLogin () {
      this.status.push(`檢查登入狀態`)
      if (await db.getIsLoggedIn(this.roomId, this.fingerprint)) {
        this.disconnect()
        throw `Repeated login!`
      } else this.isLoggedIn = true
    },
    async checkRoom () {
      this.status.push(`檢查房間ID`)
      if (!(await db.getIsRoomExist(this.roomId))) {
        this.disconnect()
        throw `The room doesn't exist.`
      }
    },
    async checkPlayerCount () {
      this.status.push(`檢查參加人數`)
      if (await db.getRoomPlayerCount(this.roomId) >= 2) {
        this.disconnect()
        throw `The room is full!`
      }
    },
    async joinRoom (chess) {
      this.status.push(`正在加入`)
      await db.joinRoom(this.roomId, this.fingerprint, chess)
    },
    async chooseChess () {
      this.status.push(`選擇棋色中`)
      const players = await db.getRoomPlayer(this.roomId)
      if (players === null) return 1
      else {
        for (let key of Object.keys(players)) {
          return (players[key].chess === 1) ? 2 : 1
        }
      }
    },
    async leaveRoom () {
      if(this.isLoggedIn) this.sendSystemInfo(`${this.profile.name} has left.`)
      this.linstenList.forEach(child => db.offRoom(this.roomId, child))
      if(this.isLoggedIn) await db.leaveRoom(this.roomId, this.fingerprint)
      db.roomRef.child(`${this.roomId}/players/${this.fingerprint}`).onDisconnect().cancel()  // 解除綁定
      db.roomRef.child(`${this.roomId}/chat/${this.session}`).onDisconnect().cancel()
    },
    goToIndex () {
      this.$router.push({ name: 'Index' })
    },
    sendSystemInfo (message) {
      this.$refs.chat.sendChat(this.roomId, message, null)
    },
    clickChangeRoomName () {
      this.newRoomName = this.roomName
      this.page = `name`
    },
    async confirmRoomName () {
      await this.changeRoomName(this.newRoomName)
      this.page = `game`
    },
    async changeRoomName (name = ``) {
      if (name.trim() === ``) throw `Empty room name!`
      const info = { name }
      await db.setRoomInfo(this.roomId, info)
      await this.$refs.chat.sendChat(this.roomId, `Room name has been changed to '${name}'`, null)
    },
    // 取得玩家列表項目的 class
    getPlayerItemClass (chess) {
      return {
        black: chess === 1,
        white: chess === 2,
        me: chess === this.myChess,
        turn: this.boardRef ? (this.boardRef.whosTurn === chess) : false,
      }
    },
    toggleChatShow () {
      this.isChatShow = !this.isChatShow
      if (this.isChatShow) this.$nextTick(() => this.$refs.chat.focus())
    }
  },
  computed: {
    roomName () {
      return (this.info) ? this.info.name : null
    },
    // 取得我的棋子種類
    myChess () {
      try {
        return (this.players && this.fingerprint) ? this.players[this.fingerprint].chess : null
      } catch (e) {
        return null
      }
    },
    boardRef () {
      return this.$refs.board ? this.$refs.board : null
    }
  },
  watch: {
    'game.board': {
      handler () {},
      deep: true
    }
  },
  beforeDestroy () {
    this.leaveRoom()
  },
  components: {
    Logo,
    Chat,
    Nickname,
    Board,
    ShareLink,
    Status,
    Result,
  },
  mixins: [common]
}
</script>

<style scoped lang="sass">
@import "../assets/css/room"

</style>
