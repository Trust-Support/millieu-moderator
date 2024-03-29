<script>
  // # # # # # # # # # # # # #
  //
  //  TSoaP Moderator
  //
  // # # # # # # # # # # # # #

  // IMPORTS
  import { onMount } from "svelte"
  import * as Colyseus from "colyseus.js"
  import get from "lodash/get"
  import { loadData, client } from "./sanity.js"
  import { Router, Route, links } from "svelte-routing"
  import {
    Header,
    HeaderNav,
    HeaderNavItem,
    SkipToContent,
    Content,
  } from "carbon-components-svelte"
  import Chance from "chance"
  const chance = new Chance()

  // ROUTES
  import Dashboard from "./Dashboard.svelte"
  import Users from "./Users.svelte"
  import Textchat from "./Textchat.svelte"
  import Audiochat from "./Audiochat.svelte"
  import Livestreams from "./Livestreams.svelte"
  import Banned from "./Banned.svelte"

  let users = {}
  let blackList = []
  let chatMessages = []

  // *** STORES
  import { gameRoom } from "./stores.js"

  // *** GLOBALS
  import { QUERY } from "./global.js"

  // *** COLYSEUS
  const gameClient = new Colyseus.Client("wss://gameserver.tsoap.dev")

  // *** SANITY
  const areas = loadData(QUERY.AREAS).catch(err => {
    console.log(err)
  })
  const audioRoomNames = loadData(QUERY.AUDIOROOM_NAMES).catch(err => {
    console.log(err)
  })
  const userList = loadData(QUERY.USERS).catch(err => {
    console.log(err)
  })

  let currentStream = false

  let activeStreams = loadData(QUERY.ACTIVE_STREAMS)
    .catch(err => {
      console.log(err)
    })
    .then(activeStreams => {
      currentStream = activeStreams.mainStream
    })

  // __ Listen for changes to the active streams post
  client.listen(QUERY.ACTIVE_STREAMS).subscribe(update => {
    currentStream = false
    setTimeout(() => {
      activeStreams = loadData(QUERY.ACTIVE_STREAMS)
        .then(aS => {
          if (aS.mainStream) {
            currentStream = aS.mainStream
          } else {
            currentStream = false
          }
        })
        .catch(err => {
          console.log(err)
        })
    }, 1000)
  })

  // FUNCTIONS
  let submitChat = () => {}

  // CREATE PLAYER
  const createPlayer = (player, sessionId) => {
    let avatar = {}
    avatar.x = player.x
    avatar.y = player.y
    avatar.waypoints = []
    avatar.area = player.area
    avatar.tint = player.tint
    avatar.name = player.name
    avatar.uuid = player.uuid
    avatar.ip = player.ip
    avatar.connected = player.connected
    avatar.authenticated = player.authenticated
    avatar.npc = player.npc
    avatar.carrying = player.carrying
    avatar.id = sessionId
    return avatar
  }

  onMount(async () => {
    // => GAME ROOM
    gameRoom.set(
      await gameClient.joinOrCreate("game", {
        moderator: true,
        uuid: chance.guid(),
        name: "Moderator",
        avatar: 0,
        tint: "0x000000",
      })
    )

    // PLAYER: REMOVE
    $gameRoom.state.players.onRemove = function (player, sessionId) {
      // console.log("REMOVE")
      // console.dir(users[sessionId])
      delete users[sessionId]
      // FORCE RENDER
      users = users
    }

    // PLAYER: ADD
    $gameRoom.state.players.onAdd = function (player, sessionId) {
      users[sessionId] = createPlayer(player, sessionId)

      // PLAYER: STATE CHANGE
      player.onChange = changes => {
        users[sessionId].x = player.x
        users[sessionId].y = player.y
        users[sessionId].area = player.area
        users[sessionId].carrying = player.carrying
      }
    }

    // BLACKLIST: ADD
    $gameRoom.state.blacklist.onAdd = function (bannedIP, sessionId) {
      blackList = [...blackList, bannedIP]
    }

    // BLACKLIST: REMOVE
    $gameRoom.state.blacklist.onRemove = function (unBannedIP, sessionId) {
      blackList = blackList.filter(ip => ip.address !== unBannedIP.address)
    }

    // GAME ROOM: ERROR
    $gameRoom.onError((code, message) => {
      console.error("!!! COLYSEUS ERROR:")
      console.error(message)
    })

    // CHAT: ADD
    $gameRoom.state.messages.onAdd = message => {
      chatMessages = [message, ...chatMessages]
    }

    $gameRoom.onMessage("nukeMessage", msgIdToRemove => {
      // console.log("!!!! MESGS")
      // console.dir(msgIdToRemove)
      const itemIndex = chatMessages.findIndex(m => m.msgId === msgIdToRemove)
      // console.log(itemIndex)
      // console.dir(chatMessages[itemIndex])
      chatMessages.splice(itemIndex, 1)
      chatMessages = chatMessages
    })

  })
</script>

<style lang="scss" global>
  @import "./variables.scss";
  @import "@carbon/themes/scss/themes";
  @include carbon--theme($carbon--theme--g100);

  * {
    box-sizing: border-box;
  }
</style>

<div use:links>
  <Header company="HkW" platformName="The Shape of a Practice">
    <div slot="skip-to-content">
      <SkipToContent />
    </div>
    <HeaderNav>
      <HeaderNavItem href="/" text="Dashboard" />
      <HeaderNavItem href="/users" text="Users" />
      <HeaderNavItem href="/banned" text="Banned" />
      <HeaderNavItem href="/livestreams" text="Livestreams" />
      <HeaderNavItem href="/textchat" text="Textchat" />
      <!-- <HeaderNavItem href="/audiochat" text="Audiochat" /> -->
    </HeaderNav>
  </Header>

  <Content>
    <Router>
      <Route path="/">
        <Dashboard {users} {blackList} {chatMessages} {currentStream} />
      </Route>
      {#await areas then areas}
        <Route path="/users">
          <Users {users} {areas} />
        </Route>
      {/await}
      <Route path="/banned">
        <Banned {blackList} />
      </Route>
      <Route path="/textchat">
        {#await userList then userList}
          <Textchat {chatMessages} 
            users={userList} 
            on:message={event=> {
              console.dir(event.detail)
              $gameRoom.send("submitChatMessage", {
                msgId: chance.guid(),
                uuid: chance.guid(),
                directed: true,
                directedTo: get(event, 'detail.recipient', ''),
                name: 'Moderator',
                username: 'moderator',
                authenticated: false,
                room: 1,
                text: get(event, 'detail.message', ''),
              });
          }}/>
        {/await}
      </Route>
      <Route path="/audiochat">
        <Audiochat />
      </Route>
      <Route path="/livestreams">
        <Livestreams {currentStream} />
      </Route>
    </Router>
  </Content>
</div>
