<template>
  <h1>CHAT</h1>

  <p>Current user is
    <base-badge :title="currentUser"></base-badge>
  </p>

  <div v-if="!isAuth">
    <base-card>
      <div>
        <label for="user">username</label>
        <input id="user" type="text" v-model="user">
        <base-button @click="handleLogin">login</base-button>
      </div>
    </base-card>

  </div>
  <div v-else-if="isAuth && canChat">
    <base-card>
      <div>
        <input id="message" type="text" v-model="message">
        <base-button type="button" @click="sendMsg">Send</base-button>
      </div>
    </base-card>


  </div>
<div class="area">
  <base-card v-if=" isAuth">

    <div v-if="messages.length > 0 ">
      <ul :key="msg.message" v-for="msg in messages">
        <li>from: [{{ msg.messageFrom }}] | {{ msg.message }}</li>
      </ul>
    </div>
    <div v-else-if="messages.length === 0 ">
      <p>No messages yet =(</p>
    </div>


  </base-card>

  <base-card>
    <div>
      <p>Users list</p>
      <ul :key="user" v-for="user in userList">
        <li @click="toggleConnection(user)">{{ user }}</li>
      </ul>
    </div>
  </base-card>
</div>


</template>

<script>
import BaseCard from "@/components/ui/BaseCard";
import BaseButton from "@/components/ui/BaseButton";
import BaseBadge from "@/components/ui/BaseBadge";

export default {
  name: 'HelloWorld',
  components: {BaseBadge, BaseButton, BaseCard},
  props: {
    msg: String
  },
  data() {
    return {
      user: null,
      webSocket: null,
      currentUser: '',
      localConnection: null,
      dataChannel: null,
      messages: [],
      connectedTo: '',
      connecting: false,
      userList: [],
      canChat: false,
      isAuth: false,
      message: ''
    }
  },
  created() {
    this.webSocket = new WebSocket("ws://localhost:9999/channels/TEST_1");
    this.webSocket.onopen = () => {
      console.log("clientWebSocket.onopen");
    }

    this.webSocket.onmessage = (message) => {
      console.log("new msg from socket: " + message.data)
      const data = JSON.parse(message.data);
      this.operateSocketMessage(data)
    }

    this.webSocket.onclose = () => {
      console.log("clientWebSocket.onclose");
      this.webSocket.close();
    }

  },
  methods: {

    updateUsersList(data) {
      console.log("updateUsersList: " + data.user)
      this.userList.push(data.user)
    },

    onLogin(data) {
      console.log("onLogin")
      if (data.user !== this.currentUser) {
        return
      }
      this.isAuth = true

      let configuration = {
        iceServers: [{url: "stun:stun.1.google.com:19302"}]
      }
      let localConnection = new RTCPeerConnection(configuration);

      console.log("try configureLocalConn")
      localConnection.onicecandidate = ({candidate}) => {

        if (candidate && !!this.connectedTo) {
          console.log("try to send candidate : " + candidate)
          console.log("to " + this.connectedTo)
          this.send({
            connectedTo: this.connectedTo,
            messageFrom: this.currentUser,
            type: "candidate",
            messageBody: candidate
          });
        }
      };

      let channel = localConnection.createDataChannel('chat', {negotiated: true, id: 0});
      channel.onopen = () => {
        console.log("Data channel is open and ready to be used.");
      };
      channel.onmessage = (data) => this.handleDataChannelMessageReceived(data);

      // localConnection.ondatachannel = (event) => {
      //   console.log("Data channel is created!");
      //   let receiveChannel = event.channel;
      //   receiveChannel.onopen = () => {
      //     console.log("Data channel is open and ready to be used.");
      //   };
      //   receiveChannel.onmessage = this.handleDataChannelMessageReceived;
      //   this.receiveChannel = receiveChannel
      // };

      this.localConnection = localConnection

      // let dataChannel = localConnection.createDataChannel("messenger");
      // dataChannel.onerror = error => {
      //   console.log(error)
      // };
      //
      // dataChannel.onmessage = this.handleDataChannelMessageReceived;
      this.dataChannel = channel

      this.send(
          {
            type: "updateUsers",
            user: this.user
          }
      )

    },
    handleLogin() {
      console.log("handleLogin")
      if (this.user === null) {
        console.log("user is null")
        return
      }
      this.currentUser = this.user
      this.send(
          {
            type: "login",
            user: this.user
          }
      )
    },

    send(data) {
      this.webSocket.send(JSON.stringify(data));
    },
    operateSocketMessage(data) {
      switch (data.type) {
          // case "connect":
          //   setSocketOpen(true);
          //   break;
        case "login":
          this.onLogin(data);
          break;
        case "updateUsers":
          this.updateUsersList(data);
          break;
          // case "removeUser":
          //   removeUser(data);
          //   break;
        case "offer":
          this.onOffer(data);
          break;
        case "answer":
          this.onAnswer(data);
          break;
        case "candidate":
          this.onCandidate(data);
          break;
        default:
          break;
      }
    },
    async onOffer(data) {
      let offer = data.messageBody
      let messageTo = data.messageTo
      let messageFrom = data.messageFrom

      if (this.currentUser !== messageTo) {
        return
      }

      console.log("on offer from: " + messageFrom)

      this.connectedTo = messageFrom;

      await this.localConnection.setRemoteDescription(new RTCSessionDescription(offer))
      let answer = await this.localConnection.createAnswer()
      await this.localConnection.setLocalDescription(answer)
      console.log("try to send answer to " + messageFrom)
      await this.send(
          {
            type: "answer",
            messageBody: this.localConnection.localDescription,
            messageTo: messageFrom,
            messageFrom: this.currentUser
          }
      )

    },

    async onAnswer(data) {

      let messageTo = data.messageTo
      let messageFrom = data.messageFrom
      let answer = data.messageBody;

      if (this.currentUser !== messageTo) {
        return
      }
      console.log("onAnswer from: " + messageFrom)
      this.connectedTo = messageFrom
      await this.localConnection.setRemoteDescription(new RTCSessionDescription(answer));
    },

    async onCandidate(data) {
      let connectedTo = data.connectedTo;
      let messageFrom = data.messageFrom

      if (connectedTo !== this.currentUser) {
        return
      }
      console.log("onCandidate from: " + messageFrom)

      await this.localConnection.addIceCandidate(new RTCIceCandidate(data.messageBody));

      console.log("Now you can chat with " + this.connectedTo)
      this.canChat = true
    },

    async handleConnection(name) {
      // var dataChannelOptions = {
      //   reliable: true
      // };

      console.log("handle connection")

      let offer = await this.localConnection.createOffer()
      await this.localConnection.setLocalDescription(offer)
      console.log("sending offer to " + name)
      await this.send({
        type: "offer",
        messageBody: this.localConnection.localDescription,
        messageTo: name,
        messageFrom: this.currentUser
      })
    },

    toggleConnection(userName) {
      console.log("try to connect to " + userName)
      if (this.currentUser === userName) {
        this.connecting = true;
        this.connectedTo = ""
        // connectedRef.current = "";
        this.connecting = false;
      } else {
        this.connecting = true;
        this.connectedTo = userName
        // connectedRef.current = "";
        this.handleConnection(userName);
        this.connecting = false;
      }
    },
    handleDataChannelMessageReceived(event) {
      console.log("new event from channel: " + event)

      console.log(" data from channel: " + event.data)

      let data = JSON.parse(event.data)
      this.messages.push(data)
    },
    sendMsg() {
      if (this.message) {

        let msg = {message: this.message, messageTo: this.connectedTo, messageFrom: this.currentUser};
        console.log("msg to send: " + msg)
        this.dataChannel.send(JSON.stringify(msg))
        this.messages.push(msg)
        this.message = ''
      }

    }
  }
}
</script>


<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}

ul {
  list-style-type: none;
  padding: 0;

}

li {
  display: inline-block;
  /* margin: 0 10px; */
  cursor: pointer;
  border-radius: 10px;
  background: darkseagreen;
  width: 100%;
  text-align: center;
}

li:hover {
  background-color: #42b983;
  transform: scale(1.4);
}

a {
  color: #42b983;
}

label {
  font-weight: bold;
  margin-bottom: 0.5rem;
  display: block;
}

input,
textarea {
  display: block;
  width: 100%;
  font: inherit;
  border: 1px solid #ccc;
  padding: 0.15rem;
  margin-top: 1rem;
  margin-bottom: 1rem;
}

.area {
  display: flex;
  align-content: space-between;

}
</style>
