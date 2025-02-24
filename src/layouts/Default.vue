<template>
    <div class="layout">
        <transition name="slide-down">
            <div id="top" class="bg-frost" v-if="visible.top">
                <div class="x-top x-left x-right">
                    <div class="flex valign" v-if="visible.top">
                        <button class="pd-x-xs" @click="clearConversation()" :disabled="!conversation.length">
                            <g-image style="display: block; width: auto; height: 28px;" src="~/assets/images/trash.svg" alt="Clear conversation" />
                        </button>

                        <div class="grow text-center">
                            <g-image src="~/assets/images/profile.jpg" alt="Vincent Lejtzén" style="display: block; margin: 0 auto; width: auto; height: 38px; border-radius: 1000px;" />
                        </div>

                        <button class="pd-x-xs" @click="showInfo()">
                            <g-image style="display: block; width: auto; height: 28px;" src="~/assets/images/info.svg" alt="Information" />
                        </button>
                    </div>
                    <div class="text-xs text-center">{{ bot.name }}</div>
                </div>
            </div>
        </transition>

        <transition name="fade">
            <div id="view" v-if="visible.view" class="x-all":class="{ fullscreen: visible.wallpaper }">
                <div>
                    <transition-group name="list" tag="ul" class="messages pd-x-xs">
                        <li v-for="(message, index) in conversation" :key="message.id" :class="[
                            conversation[index -1] && conversation[index -1].sender == message.sender ? 'mg-t-min' : 'mg-t-xs'
                        ]">
                            <div class="br-radius-md" :class="[
                                message.type,
                                message.sender == 'bot' ? 'received bg-grey' : 'sent bg-blue text-light-primary'
                            ]">
                                <div v-if="message.type == 'attachment'">
                                    <div class="replay">
                                        <button class="text-blue" @click="replay(message.body)">
                                            <g-image src="~/assets/images/refresh.svg" alt="Replay" /> Replay
                                        </button>
                                    </div>
                                    <g-image class="br-radius-md" v-freeze="message.body" :src="message.body" />
                                </div>
                                <span v-else>
                                    {{ message.body }}
                                </span>
                            </div>
                        </li>
                    </transition-group>
                </div>

                <transition name="fade">
                    <div class="mg-t-xs pd-x-xs" v-if="bot.writing">
                        <div class="br-radius-md message received bg-grey">
                            <span class="writing"></span>
                            <span class="writing" style="animation-delay: .25s;"></span>
                            <span class="writing" style="animation-delay: .5s;"></span>
                        </div>
                    </div>
                </transition>
            </div>
        </transition>

        <transition name="slide-up">
            <div id="bottom" class="flex valign bg-frost" v-if="visible.bottom">
                <div class="x-right x-bottom x-left">
                    <form class="form" v-on:submit.prevent="handleSubmit">
                        <textarea class="textarea br-radius-md" name="message" rows="1" placeholder="Whats up?" v-model="message" v-on:keyup.enter.prevent.stop="handleSubmit" ref="textarea"></textarea>
                        <button class="button" type="submit" :disabled="!valid">
                            <g-image style="display: block; width: auto; height: 28px;" src="~/assets/images/angle-up.svg" alt="Send" />
                        </button>
                    </form>
                </div>
            </div>
        </transition>

        <transition name="fade">
            <div id="wallpaper" v-if="visible.wallpaper" :style="{ 'background-image': 'url(' + wallpaper + ')' }"></div>
        </transition>
    </div>
</template>

<script>
    import axios from 'axios'

    export default {
        data: function () {
            return {
                bot: {
                    name: 'Vincent Lejtzén',
                    writing: false,
                },
                user: {
                    name: ''
                },
                message: '',
                conversation: [],
                wallpaper: '',
                visible: {
                    top: false,
                    view: false,
                    bottom: false,
                    wallpaper: false
                }
            }
        },

        computed: {
            valid: function () {
                return this.message.trim().length > 0
            }
        },

        beforeMount: function () {
            this.retrieveConversation()
            this.retrieveUser()
        },

        mounted: function () {
            var self = this

            // TODO: Animera dessa när bilder är preloadade och konversation är hämtad från localStorage
            // med Promise istället för timeOut.
            setTimeout(function () {
                self.visible.top = true
                self.visible.view = true
                self.visible.bottom = true
            }, 1000)

            if (!this.conversation.length) {
                setTimeout(function () {
                    self.bot.writing = true
                }, 2000)

                setTimeout(function () {
                    self.bot.writing = false
                    self.sendMessage(self.createMessage('bot', 'message', 'Hi there! Ask me anything and I\'ll give you a yes or no. Lets go 👊'))
                }, 3500)
            }
        },

        watch: {
            message: function () {
                this.message = this.message.replace(/(\r\n|\n|\r)/gm, '')
            },
        },

        methods: {
            preload: function (attachment) {
                return new Promise (function (resolve, reject) {
                    var image = new Image()

                    image.onload = resolve
                    image.onerror = resolve

                    image.src = attachment
                })
            },

            handleSubmit: function () {
                if (!this.valid) {
                    return
                }

                this.sendMessage(this.createMessage('user', 'message', this.message))
                this.message = ''

                this.sendResponse()
            },

            createMessage: function (sender, type, message) {
                return {
                    id: Math.random().toString(36).substring(2) + Date.now().toString(36),
                    sender: sender,
                    type: type,
                    time: new Date(),
                    body: message
                }
            },

            sendMessage: function (message) {
                this.conversation.push(message)
                localStorage.setItem('conversation', JSON.stringify(this.conversation))
                this.scrollView()
            },

            transformResponse: function (answer) {
                var answers = {
                    yes : [
                        'Yes, that\'s right.',
                        'Yes ...',
                        'That\'s a yes',
                        'Yep!',
                        'Always!',
                        'You know that. Of course.',
                    ],
                    no : [
                        'Simply ... no',
                        'Never',
                        'I don\'t think so',
                        'I\'d say no actually.',
                        'No ...',
                        'No, but don\'t tell anyone 🤫',
                    ],
                    maybe : [
                        'Maybe 😉',
                    ]
                }

                return answers[answer][Math.floor( Math.random() * answers[answer].length )]
            },

            sendResponse: function () {
                var self = this

                setTimeout(function() {
                    self.bot.writing = true
                }, 500)

                this.$refs.textarea.blur()

                setTimeout(function() {
                    axios.get('https://yesno.wtf/api').then(function (response) {
                        self.bot.writing = false

                        self.preload(response.data.image).then(function () {
                            // Visa wallpaper
                            self.wallpaper = response.data.image
                            self.visible.top = false
                            self.visible.bottom = false
                            self.visible.wallpaper = true

                            // Skicka attachment
                            setTimeout(function () {
                                self.sendMessage(self.createMessage('bot', 'attachment', response.data.image))
                            }, 2000)

                            // Skicka message
                            setTimeout(function () {
                                self.sendMessage(self.createMessage('bot', 'message', self.transformResponse(response.data.answer)))
                            }, 3000)

                            // Lägg tillbaka allt
                            setTimeout(function () {
                                self.visible.top = true
                                self.visible.bottom = true
                                self.visible.wallpaper = false
                            }, 5000)
                        })
                    }).catch(function (error) {
                        console.log(error)
                        self.bot.writing = false
                        self.sendMessage(self.createMessage('bot', 'message', 'Sorry, I can\'t answer any questions now. The yesno.wtf API seems to be down :('))
                    })
                }, 1500)
            },

            replay: function (attachment) {
                var self = this

                this.preload(attachment).then(function () {
                    // Visa wallpaper
                    self.wallpaper = attachment
                    self.visible.top = false
                    self.visible.bottom = false
                    self.visible.wallpaper = true

                    // Lägg tillbaka allt
                    setTimeout(function () {
                        self.visible.top = true
                        self.visible.bottom = true
                        self.visible.wallpaper = false
                    }, 5000)
                })
            },

            scrollView: function () {
                window.scrollTo(0, document.getElementById('view').offsetHeight)
            },

            retrieveConversation: function () {
                var conversation = JSON.parse(localStorage.getItem('conversation'))

                if (conversation) {
                    this.conversation = conversation
                }
            },

            clearConversation: function () {
                if (this.conversation.length) {
                    var confirm = window.confirm('Are you sure you want to delete the conversation?')

                    if (confirm == true) {
                        this.conversation = []
                        localStorage.removeItem('conversation')
                    }
                }
            },

            retrieveUser: function () {
                var username = JSON.parse(localStorage.getItem('username'))

                if (username) {
                    this.user.name = username
                }
            },

            showInfo: function () {
                this.sendMessage(this.createMessage('bot', 'message', 'That button there. It should show some info about this project but i haven\'t made it that far yet. Come back soon.'))
            }
        }
    }
</script>
