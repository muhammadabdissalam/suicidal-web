<template>
  <div class="relative">
    <p class="inter-14-400-alt md:inter-16-400-alt text-black">Layanan Darurat <span class="font-bold">24 jam</span></p>
    <h3 class="mt-4 merri-20-700 lg:merri-30-700 max-w-[40rem] text-black mb-4">
      Bicara dengan konselor kami jika merasa
      <span class="text-[#007474]"> kondisi mentalmu sedang tidak baik </span>
    </h3>
    <p v-for="item in helpPoint" :key="item" class="text-black inter-14-400 lg:inter-18-400">{{ item }}</p>
    <p class="text-black inter-16-500 md:inter-20-500 mt-8">
      Ceritakan keluh-kesahmu dan kita cari jalan keluarnya bersama
    </p>
    <ul class="flex flex-col list-disc mt-4 pl-4">
      <li v-for="item in actionItem" :key="item" class="inter-14-400 lg:inter-18-400 text-black">
        {{ item }}
      </li>
    </ul>
    <NuxtLink
      to="https://api.whatsapp.com/send/?phone=6281380073120&text=halo%20kak%2C%20saya%20ingin%20bercerita%20mengenai...&type=phone_number&app_absent=0"
      target="_blank"
      class="w-full max-w-[17.5rem] relative mt-4 block mx-auto md:mx-0"
    >
      <PrimaryCTA label="Chat via WhatsApp" :is-width-parent="true">
        <template #left-icon>
          <IcChatWa />
        </template>
      </PrimaryCTA>
    </NuxtLink>
  </div>
</template>

<script setup lang="ts">
import IcChatWa from '~/components/elements/icons/IcChatWa.vue'
import PrimaryCTA from '../common/PrimaryCTA.vue'

const helpPoint = [
  '✔️ Bercerita dengan konselor profesional dapat membuatmu merasa lebih baik',
  '✔️ Cari tahu langkah yang tepat untuk mendapatkan hidup yang lebih berkualitas',
]

const actionItem = [
  'Kamu tersambung dengan konselor dari RS Marzoeki Mahdi via website Whatsapp',
  'Semua keluh-kesahmu akan kami dengarkan dengan tulus',
  'Tempat bercerita yang aman dan nyaman',
  'Membantumu menemukan jalan keluar',
]
</script>
