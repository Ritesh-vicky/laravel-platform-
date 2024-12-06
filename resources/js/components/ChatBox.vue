<template>
    <div class="chat-box">
        <div ref="messagesBox" class="chat-messages">
            <div 
                v-for="(message, index) in messages" 
                :key="index" 
                :class="['chat-message', { 'own-message': message.sender_id === sender.id }]"
            >
                {{ message.text }}
                <span class="timestamp">{{ formatTimestamp(message.created_at) }}</span>
            </div>
        </div>
        <div class="chat-input">
            <input 
                v-model="newMessage" 
                @keyup.enter="sendMessage" 
                placeholder="Type a message..." 
            />
            <button @click="sendMessage">Send</button>
        </div>
    </div>
</template>

<script>
import axios from "axios";
import { nextTick, onMounted, ref, watch } from "vue";

export default {
    props: {
        receiver: {
            type: Object,
            required: true,
        },
        sender: {
            type: Object,
            required: true,
        },
    },
    setup(props) {
        const messages = ref([]);
        const newMessage = ref("");
        const messagesBox = ref(null);

        const scrollToBottom = () => {
            if (messagesBox.value) {
                messagesBox.value.scrollTop = messagesBox.value.scrollHeight;
            }
        };

        watch(
            messages,
            () => {
                nextTick(() => {
                    scrollToBottom();
                });
            },
            { deep: true }
        );

        const sendMessage = () => {
            if (newMessage.value.trim() !== "") {
                axios
                    .post(`/messages/${props.receiver.id}`, {
                        message: newMessage.value,
                    })
                    .then((response) => {
                        messages.value.push(response.data);
                        newMessage.value = "";
                    })
                    .catch((error) => {
                        console.error("Error sending message:", error);
                    });
            }
        };

        const formatTimestamp = (timestamp) => {
            const date = new Date(timestamp);
            const hours = date.getHours().toString().padStart(2, "0");
            const minutes = date.getMinutes().toString().padStart(2, "0");
            return `${hours}:${minutes}`;
        };

        onMounted(() => {
            if (props.receiver?.id) {
                axios
                    .get(`/messages/${props.receiver.id}`)
                    .then((response) => {
                        messages.value = response.data;
                    })
                    .catch((error) => {
                        console.error("Error fetching messages:", error);
                    });

                if (typeof Echo !== "undefined") {
                    Echo.private("chat").listen("MessageSent", (response) => {
                        if (response.message) {
                            const messageExists = messages.value.some(
                                (msg) => msg.id === response.message.id
                            );
                            if (!messageExists) {
                                if (
                                    (response.message.sender_id === props.sender.id &&
                                        response.message.receiver_id === props.receiver.id) ||
                                    (response.message.sender_id === props.receiver.id &&
                                        response.message.receiver_id === props.sender.id)
                                ) {
                                    messages.value.push(response.message);
                                }
                            }
                        }
                    });
                } else {
                    console.warn("Echo is not defined. Real-time updates may not work.");
                }
            }
        });

        return {
            messages,
            newMessage,
            messagesBox,
            sendMessage,
            formatTimestamp,
        };
    },
};
</script>

<style scoped>
.chat-box {
    display: flex;
    flex-direction: column;
    border: 1px solid #ccc;
    padding: 10px;
    margin: 0 auto;
    height: 500px;
    max-width: 600px;
}

.chat-messages {
    flex: 1;
    overflow-y: auto;
    padding: 15px;
    display: flex;
    flex-direction: column;
}

.chat-input {
    display: flex;
    align-items: center;
    border-top: 1px solid #ccc;
    padding: 10px 0;
}

.chat-input input {
    flex: 1;
    padding: 10px;
    margin-right: 5px;
    border: 1px solid #ccc;
    border-radius: 5px;
}

.chat-input button {
    padding: 10px 15px;
    background-color: #007bff;
    color: white;
    border: none;
    border-radius: 5px;
    cursor: pointer;
}

.chat-input button:hover {
    background-color: #0056b3;
}

.chat-message {
    display: inline-block;
    padding: 10px;
    margin: 5px 0;
    border-radius: 15px;
    word-wrap: break-word;
}

.chat-message.own-message {
    align-self: flex-end;
    background-color: #d1e7ff;
    color: #0056b3;
    text-align: right;
}

.chat-message:not(.own-message) {
    align-self: flex-start;
    background-color: #e1e1e1;
    color: #333;
}

.timestamp {
    font-size: 0.8rem;
    color: #aaa;
    margin-left: 10px;
}

@media (max-width: 600px) {
    .chat-box {
        height: 80vh;
    }

    .chat-input input,
    .chat-input button {
        font-size: 14px;
    }
}
</style>
