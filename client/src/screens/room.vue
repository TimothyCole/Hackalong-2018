<template>
    <div class="container_wide room">


    <router-link to="/"><div class="logo small">ChangeMyMind<span>.io</span></div></router-link>

    <div class="biginput dropshadow">
        <input v-model="theTopic" type="text" disabled />
        <div class="vote">
            <div :class="{'notselected': myVote === true }" @click="vote(false)" class="button dropshadow skew red"><span class="noselect"> <div class="fa fa-angle-down"/> </span></div>
            <div :class="{'notselected': myVote === false }" @click="vote(true)" class="button dropshadow skew blue "><span class="noselect"> <div class="fa fa-angle-up"/> </span></div>
        <!-- add notselected class when user has made selection -->
        </div>
    </div>

    <div class="bar">
        <div v-bind:style="{width: percentFor + '%'}" class="dropshadow blue"><div>{{percentFor}}%</div></div>
        <div v-if="!hideRightBar" v-bind:style="{width: 100 - percentFor + '%'}" class="dropshadow red"><div>{{100 - percentFor}}%</div></div>
    </div>

    <div ref="messageContainer" class="message_container dropshadow">
        <div class="messageInner">
            <div class="message_body  alert_message">
                <div class="timestamp">{{niceTime(new Date())}}</div><div style="margin-left: 24px" class=" message"> Welcome to the chat!</div>
            </div>
            <div v-for="(i, index) in messages" :key="index">
                <div v-if="i.type === 'MEMBER_JOIN'" class="message_body member_join alert_message">
                    <div class="timestamp">{{niceTime(i.timestamp)}}</div>
                    <div class="username">{{i.username}}</div>
                    <div class="message">has joined the debate</div>
                </div>
                 <div v-else class="message_body">
                    <div class="timestamp">{{niceTime(i.timestamp)}}</div>

                    <div :class="{'redText': i.member.vote === -1, 'blueText': i.member.vote === 1 }" class="username">{{i.member.username}}</div>

                    <div class="message">{{i.message}}</div>
                </div>
            </div>
            <div class="bottom_padding"></div>
        </div>
        <div class="connected_users">
            <div v-for="(i, index) in members" :key="index"> 
                <div :class="{'red': i.vote === -1, 'grey': i.vote === 0}" class="button dropshadow skew blue"><span class="noselect"> {{i.username}} </span></div>
            </div>
        </div>
    </div>
    <div v-if="!dummy" class="dropshadow biginput messageBox">
      <input v-shortkey="['enter']" @shortkey="sendMessage()" v-model="message" type="text" placeholder="Be rational... or not."/>
    <div :class="{'red': myVote === false }" @click="sendMessage()" class="button send_button dropshadow skew blue"><span class="noselect"> SEND </span></div>

    </div>
    

    </div>
</template>

<script>
import moment from 'moment'
import momentDurationFormatSetup from 'moment-duration-format'

export default {
    mounted() {



        const vm = this

        this.$root.$on('response', (response) => {

            // console.log('Type: ', response.type)
        
          if (response.type === 'JOIN_CHANNEL') {
                console.log('setting the topic to: ' +  response.data.topic)
              vm.theTopic = response.data.topic
              console.log('it was set: ' +  vm.theTopic)
              
            // this.setTheTopic(response.data.topic) 
            // console.log('THIS IS THE FUCKING TOPIC', this.theTopic)
          }
          if (response.type === 'NEW_MESSAGE') {
            response.data.type = "NEW_MESSAGE"
             this.messages.push(response.data)
             this.scrollOnNewMessageButOnlyIfUserDidNotScrollUpMoreThan120()
          }
          if (response.type === 'MEMBER_JOIN') {
            response.data.type = "MEMBER_JOIN" // used to render a style in the DOM
            this.messages.push(response.data)
            this.members.push(response.data)
          }
          // WHEN A MEMBER LEAVES
          if (response.type === 'MEMBER_LEAVE') {
            response.data.type = "MEMBER_JOIN" // used to render a style in the DOM
            this.messages.push(response.data)
            this.members = this.members.filter(member => member.username !== response.data.username)
          }
          if (response.type === 'VOTE') {
            this.members = this.members.map(member => {
                if (member.username === response.data.member.username) member.vote = response.data.member.vote
                return member
            })
          }
        })
        this.$root.$on('roomData', (data) => {
            this.theTopic = data.topic
             this.members = data.members
        })



          if (window.dummy) {
                this.dummy = true
                this.members = window.dummy.members
                this.messages = window.dummy.history
                this.theTopic = window.dummy.topic
            }  



    },

    data() {
        return {
            socket: null,
            theTopic: '',
            message: '',
            messages: [],
            myVote: null,
            percentFor: 41,
            members: [],
            hideRightBar: true,
            dummy: false
        }
    },
    methods: {
        setTheTopic(topic) {
            this.theTopic = topic
            console.log('we set the topic!')
        },
        sendMessage() {
            if (this.message.length < 1) return false
          window.socket.send(JSON.stringify({
            type: "SEND_MESSAGE",
            data: {
                message: this.message
            }
          }))
          this.message = ''
        },
        vote(boool) {
          window.socket.send(JSON.stringify({
            type: "VOTE",
            data: {
                vote: boool
            }
          }))
          this.myVote = boool
        },
        removeDuplicates(array) {
            console.log('JEFF:',  array)
            return array.filter(
                (elem, index, self) =>
                    self.findIndex(item => {
                        return item.message === elem.message
                    }) === index
            )
        },
        niceTime(time) {
			moment.updateLocale('en', {
				relativeTime: {
					future: '1s',
					past: '%s',
					s: '1s',
					m: '1m',
					mm: '%dm',
					h: '1hr',
					hh: '%dhr',
					d: '1d',
					dd: '%dd',
					M: '1 month',
					MM: '%d months',
					y: '1 year',
					yy: '%d years'
				}
			})
			return moment(time).fromNow()
        },
       scrollOnNewMessageButOnlyIfUserDidNotScrollUpMoreThan120() {
        const node = this.$refs.messageContainer
        const scrollHeight = node.scrollHeight
        const clientHeight = node.clientHeight
        const scrollTop = node.scrollTop
            if (clientHeight + scrollTop + 120 >= scrollHeight) {
                node.scrollTop = scrollHeight
            }
        }

    },
    computed: {
    },
    watch: {
        members() {
            let forVote = 0
            this.members.forEach(member => {
                if (member.vote === 1) ++forVote
            })
            this.percentFor = forVote / this.members.length * 100

            if (this.percentFor === 100) this.hideRightBar = true
            else this.hideRightBar = false

            console.log(this.percentFor)
        }
    }
}
</script>



<style lang="scss">
.container_wide {
    margin: 0 40px ;
        display: flex;
    flex-direction: column;
        height: calc(100vh - 10px);
}
.room .biginput, .room .biginput input {
    background-color: #f5f5f5;
}
.room {
    .logo {
        margin: 18px 0px;
    }
}
.vote {
    position: absolute;
    right: 40px;
    top: 5px;
    .button {
        margin-left: 20px;
    }
}
.message_container {
    margin-top: 25px;
    background: #f9f9f9;
    padding: 20px 0;
    width: 100%;
    display: flex;
    height: calc(100vh - 390px);
        overflow-y: scroll;
    overflow-x: hidden;
    * {
        box-sizing: border-box;
    }
    .messageInner {
        width: calc(100% - 200px);
    }
    .connected_users {
    position: fixed;
    right: 50px;
        display: flex;
    flex-direction: column;
    align-items: flex-end; 
    * {
            font-size: 19px;
    padding: 3px 20px;
    box-shadow: none !important;
    margin: 0px;
    }
    }
}

.message_body {
    width: 100%;
    display: flex;
    padding: 3px;
    margin-left: 30px;
        text-align: left;
    line-height: 28px;
    padding-right: 60px;
    .username {
     font-size: 1.3rem;
    font-weight: bold;
    margin: -1px 20px 0px 20px;
    }
    .message {

    }
}
.alert_message {
    font-weight: bold;
    margin: 10px 30px;
    color: #6e6e6e;
    opacity: 0.8;
    .message {
        font-size: 1.3rem;
    }
}
.bottom_padding {
height: 100px;
}
.messageBox {
    margin: auto;
    width: 85%;
    margin-top: -40px;
    background: #f1f1f1 !important;
    padding: 0;
    input {
        background: #f1f1f1 !important
    }
}

.send_button {
    position: absolute;
    top: -16px;
    right: 15px;
}
.bar {
    width: 100%;
    z-index: 999;
    margin-top: 20px;
    *:hover {

    }
    * {
        height: 37px;
        float: left;
        display: flex;
        div {
            line-height: 2.3rem;
            color: #fff;
            margin: auto;
            font-weight: bold;
        }
    }
}
</style>
