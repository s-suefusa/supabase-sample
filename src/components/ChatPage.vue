<template>
  <v-container class="fill-height flex-column">
    <v-col class="flex-grow-1" style="overflow-y: auto;">
      <v-container>
        <v-card v-for="item in messages" :key="item.id" class="mb-1">
          <v-card-text>{{item.name}} : {{item.message}}</v-card-text>
        </v-card>
      </v-container>
    </v-col>
    <v-col class="flex-grow-0">
      <v-form @submit.prevent="onSend" v-model="valid">
        <v-row>
          <v-col style="max-width: 25%;">
            <v-text-field placeholder="名前" density="compact" v-model="name" :rules="[required]"></v-text-field>
          </v-col>
          <v-col class="flex-grow-1">
            <v-text-field placeholder="メッセージ" density="compact" v-model="message" :rules="[required]"></v-text-field>
          </v-col>
          <v-col class="flex-grow-0">
            <v-btn type="submit" :disabled="!valid">送信</v-btn>
          </v-col>
          <v-col class="flex-grow-0">
            <v-btn @click="onClear" :disabled="messages.length === 0">クリア</v-btn>
          </v-col>
        </v-row>
      </v-form>
    </v-col>
  </v-container>
</template>

<script setup lang="ts">
import {createClient} from "@supabase/supabase-js";
import {onBeforeMount, ref} from "vue";

const supabase = createClient('https://bocalxayiikdawzxbhul.supabase.co', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6ImJvY2FseGF5aWlrZGF3enhiaHVsIiwicm9sZSI6ImFub24iLCJpYXQiOjE3MTcwNTM2NDIsImV4cCI6MjAzMjYyOTY0Mn0.VL4f8lMl_0o6aIq6Xnuu9gEt_TTzqG3a6N05Ek_7WIA')
const messages = ref([])
const name = ref('')
const message = ref('')
const valid = ref(false)

const required = (value) => !!value || '必ず入力してください'

onBeforeMount(async () => {
  // 最新のレコードを全件取得
  const {data, error} = await supabase
    .from('message')
    .select()
    .order('created_at', {ascending: false})
  if (error) {
    console.error(error)
    return
  }
  messages.value.push(...data)

  // リアルタイム通知をサブスクライブする
  supabase
    .channel('messages')
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'message' },
      (payload) => {
        if (payload.eventType === 'INSERT') {
          // INSERTレコードを追加する
          messages.value.unshift(payload.new)
        } else if (payload.eventType === 'UPDATE') {
          // UPDATEレコードを更新する
          const item = messages.value.find((item) => item.id === payload.new.id)
          if (item) {
            Object.assign(item, payload.new)
          }
        } else if (payload.eventType === 'DELETE') {
          // DELETEレコードを取り除く
          messages.value = messages.value.filter((item) => item.id !== payload.old.id)
        }
      })
    .subscribe()
})

const onSend = async () => {
  // レコード追加
  const {error} = await supabase
    .from('message')
    .insert({
      name: name.value,
      message: message.value,
    })
  if (error) {
    console.error(error)
    return
  }
  message.value = ''
}

const onClear = async () => {
  // レコード全件削除
  const {error} = await supabase
    .from('message')
    .delete()
    .gte('id', 0)   // 仕様上条件が必要
  if (error) {
    console.error(error)
    return
  }
}
</script>
