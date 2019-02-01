<template>
  <div :style="{background: backgroundColor}">
    <beautiful-chat
      :participants="participants"
      :titleImageUrl="titleImageUrl"
      :onMessageWasSent="onMessageWasSent"
      :messageList="messageList"
      :newMessagesCount="newMessagesCount"
      :isOpen="isChatOpen"
      :close="closeChat"
      :open="login"
      :showEmoji="true"
      :showTypingIndicator="showTypingIndicator"
      :colors="colors"
      :alwaysScrollToBottom="alwaysScrollToBottom"
      :messageStyling="messageStyling" />
  </div>
</template>

<script>


import io from 'socket.io-client'

var socket = io('http://chat-chat-q7ig6gdd00ou.3zvkjbft7q.us-east-1.elasticbeanstalk.com');
import VueFocus from 'vue-focus'
import dateFormat from 'dateformat'

import messageHistory from './messageHistory'
import chatParticipants from './chatProfiles'
import Header from './Header.vue'
import Footer from './Footer.vue'
import TestArea from './TestArea.vue'
import availableColors from './colors'

export default {
  name: 'app',
  components: {
    Header, Footer, TestArea
  },
  data() {
    return {
      participants: chatParticipants,
      titleImageUrl: 'https://a.slack-edge.com/66f9/img/avatars-teams/ava_0001-34.png',
      messageList: messageHistory,
      newMessagesCount: 0,
      isChatOpen: false,
      showTypingIndicator: '',
      colors: null,
      availableColors,
      chosenColor: null,
      alwaysScrollToBottom: false,
      messageStyling: true,
      id: null,
        message: '',
        messages: [],
        unread: false,
        chatActive: false,
        loggedIn: false,
        details: { name: "" }
    }
  },
  created() {
    this.setColor('blue')
  },
  mixins: [VueFocus.mixin],
  methods: {
    sendMessage (text) {
      if (text.length > 0) {
        debugger;

            socket.emit('send', text);
            
        this.newMessagesCount = this.isChatOpen ? this.newMessagesCount : this.newMessagesCount + 1
        this.onMessageWasSent({ author: 'me', type: 'text', data: { text } })
      }
    },
    handleTyping (text) {
      this.showTypingIndicator = text.length > 0 ? this.participants[this.participants.length - 1].id : '';
    },
    onMessageWasSent (message) {
      
      socket.emit('send', message.data.text);
      this.messageList = [ ...this.messageList, message ]
    },
    openChat () {
      this.isChatOpen = true
      this.newMessagesCount = 0
    },
    closeChat () {
      debugger;
      socket.emit('logout', true);
      this.messageList.length=0;
      this.messageList=messageHistory;
      this.isChatOpen = false
    },
    setColor (color) {
      this.colors = this.availableColors[color]
      this.chosenColor = color
    },
    showStylingInfo() {
      this.$modal.show('dialog', {
        title: 'Info',
        text: 'You can use *word* to <strong>boldify</strong>, /word/ to <em>emphasize</em>, _word_ to <u>underline</u>, `code` to <code>write = code;</code>, ~this~ to <del>delete</del> and ^sup^ or ¡sub¡ to write <sup>sup</sup> and <sub>sub</sub>'
      })
    },
    messageStylingToggled(e) {
      this.messageStyling = e.target.checked
    },
    showChat: function() {
            this.chatActive = true;

            let style = widget.frame.style;
            style.width = '300px';
            if (this.loggedIn) {
                style.height = '500px';
            } else {
                style.height = (104 + (87 * Object.keys(this.participants).length)) + 'px';
            }
        },
        hideChat: function(logout) {
            this.chatActive = false;
            if (logout) {
                if (logout >= 2) {
                    socket.emit('logout', logout === 3);
                }

                this.loggedIn = false;
                this.message = '';
                this.messages = [];
            }

            let style = widget.frame.style;
            style.width = '500px';
            style.height = '700px';
        },
        login: function() {
          console.log('Logged In')
            
            if (!this.participants[0].name) return;

            socket.emit('login', Object.assign({}, this.participants[0], { id: this.id }));
            this.loggedIn = true;
            this.isChatOpen=true
            //this.showChat();
        },
        send: function() {
            if (!this.message) return;

            socket.emit('send', this.message);
            this.message = '';
        },
        formatDate: function(date) {
            return dateFormat(+date, "h:MM:ss tt");
        }
    },
    mounted: function() {
      
        socket.on('connect', function() {
            if (!this.id) {
                this.id = localStorage.getItem('widget-id');
                if (!this.id) {
                    this.id = (+new Date()).toString(36) + socket.id;
                    localStorage.setItem('widget-id', this.id);
                }
            } else {
                // this.hideChat(1);
            }

            socket.emit('hello', {
                key: 'testkey',
                id: this.id,
                url: window.top.location.href
            });
        }.bind(this));

        socket.on('online', function(details) {
          debugger;
            if (this.loggedIn) return;

            this.participants[0].name = details.name;
            this.login();
        }.bind(this));

        socket.on('offline', function() {
            if (!this.loggedIn) return;

            // this.hideChat(2);
        }.bind(this));

        socket.on('message', function(message) {
          // let participant={};
          // let userIsParticipant=false;
          // this.participants.forEach(function(item){
          //   if(message.user.name==item.name){
          //     userIsParticipant=true
          //   }
          // })
          // if(!userIsParticipant){
          //   participant.name=message.user.name
          //   this.participants.push(participant)
          // }
          let msg=message.message
            // if (!this.loggedIn) return;
            this.$nextTick().then(()=>{
              this.messageList = [ ...this.messageList, { author: message.user.name, type: 'text', data: { text:msg } } ]
            })
            
            // this.onMessageWasSent({ author: message.user, type: 'text', data: { text:msg } })
            this.messages.push(message);
            this.unread = true;
        }.bind(this));

        socket.on('room', function(room) {
            this.messages = room.messages;
            this.unread = true;
        }.bind(this));

        // let style = widget.frame.style;
        // style.right = '3px';
        // style.bottom = '3px';
        // style.position = 'fixed';
        // style.borderStyle = 'solid';
        // style.borderColor = '#f0f0f0';
        //this.hideChat(0);

        socket.on('active', function(active) {
            // style.display = active ? 'block' : 'none';
        });
 
  },
  computed: {
    linkColor() {
      return this.chosenColor === 'dark' ? this.colors.sentMessage.text : this.colors.launcher.bg
    },
    backgroundColor() {
      return this.chosenColor === 'dark' ? this.colors.messageList.bg : '#fff'
    }
  },
    updated: function () {
        this.$nextTick(function() {
            if (!this.unread) return;

            let messagesBody = this.$el.querySelector('.messages-body');
            if (messagesBody) messagesBody.scrollTop = messagesBody.scrollHeight;
            this.unread = false;
        });
  },
  computed: {
    linkColor() {
      return this.chosenColor === 'dark' ? this.colors.sentMessage.text : this.colors.launcher.bg
    },
    backgroundColor() {
      return this.chosenColor === 'dark' ? this.colors.messageList.bg : '#fff'
    }
  }
}
</script>

<style>
body {
  padding: 0px;
  margin: 0px;
}

* {
  font-family: Avenir Next, Helvetica Neue, Helvetica,sans-serif;
}

.demo-description {
  max-width: 500px;
}

.demo-description img {
  max-width: 500px;
}

.demo-test-area {
  width: 300px;
  box-sizing: border-box;
}

.demo-test-area--text {
  box-sizing: border-box;
  width: 100%;
  margin: 0px;
  padding: 0px;
  resize: none;
  font-family: Avenir Next, Helvetica Neue, Helvetica,sans-serif;
  background: #fafbfc;
  color: #8da2b5;
  border: 1px solid #dde5ed;
  font-size: 16px;
  padding: 16px 15px 14px;
  margin: 0;
  border-radius: 6px;
  outline: none;
  height: 150px;
  margin-bottom: 10px;
}

.demo-monster-img {
  width: 400px;
  display: block;
  margin: 60px auto;
}

.text-center {
  text-align: center;
}

.colors a {
  color: #fff;
  text-decoration: none;
  padding: 4px 10px;
  border-radius: 10px;
}

.toggle a {
  text-decoration: none;
}

.messageStyling {
  font-size: small;
}

</style>