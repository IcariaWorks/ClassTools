<script setup lang="tsx">
import homeworks from '../stores/homeworks';
import { computed, watch } from 'vue';
import settings from '../stores/settings';
import Weather from '../components/Weather/index.vue';
import SloganDisplay from '../components/SloganDisplay.vue';
import DateAndCountDown from '../components/DateAndCountDown.vue';
import HomeworkDisplay from '../components/HomeworkDisplay.vue';
import ScheduleDisplay from '../components/ScheduleDisplay.vue';
import slogan from '../stores/slogan';
import { useNotification } from 'naive-ui';
import { h, Fragment } from 'vue';
import useSystemWallpaper from '../hooks/systemWallpaper';
import { formatDate, useNow } from '@vueuse/core';
import { isToday } from 'date-fns';
import newDay from '../utils/newDay';
import sendHomeworkWebhook from '../utils/sendHomeworkWebhook';
import getTime from '../utils/getTime';

const systemWallpaper = useSystemWallpaper();
const notification = useNotification();
if (window.ipcRenderer) {
  window.ipcRenderer.on('update:installing', (_, { remoteVersion, packageVersion }: { [k: string]: string }) =>
    notification.info({
      closable: false,
      title: 'ClassTools 正在更新…',
      content: () => (<>当前版本：<code>{packageVersion}</code>
        <br/>最新版本：<code>{remoteVersion}</code></>),
      duration: 30 * 1000,
    }));
  window.ipcRenderer.on('update:failed', (_, { reason }: { [k: string]: string }) =>
    notification.info({
      closable: false,
      title: 'ClassTools 无法更新',
      content: reason,
      duration: 30 * 1000,
    }));
  window.ipcRenderer.on('update:needRestart', () =>
    notification.info({
      closable: false,
      title: 'ClassTools 更新完成',
      content: '下次启动时将使用新版本',
      duration: 30 * 1000,
    }));
}

const now = useNow();
watch(() => now.value.getMinutes(), () => {
  if (!settings.value.autoNewDay) return;
  const isTimeReached = getTime(new Date()) >= getTime(settings.value.autoNewDayTime);
  if (isToday(homeworks.value.date) || !isTimeReached) return;
  console.log('开始新的一天');
  newDay();
}, { immediate: true });
watch(() => now.value.getMinutes(), async () => {
  // WebHook
  if (formatDate(now.value, 'HH:mm') !== settings.value.homeworkWebHookTime[now.value.getDay()]) return;
  console.log('webhook');
  await sendHomeworkWebhook(notification);
});

const weatherOnLeft = computed(() => !!slogan.value.content);
const backgroundStyle = computed(() => {
  let css: { [p: string]: string } = {};
  switch (settings.value.backgroundType) {
    case 'image':
      css.backgroundImage = `url('file://${settings.value.backgroundImage}')`;
      break;
    case 'color':
      css.backgroundColor = settings.value.backgroundColor;
      break;
    case 'advanced':
      css.background = settings.value.backgroundCss;
      break;
    case 'system':
      css.background = `url('file://${systemWallpaper.value}')`;
      break;
  }
  css.fontSize = settings.value.fontSize + 'vw';
  css.color = settings.value.fontColor;
  css.fontFamily = settings.value.font;
  css.textShadow = `0 0 3px ${settings.value.shadowColor}`;
  return css;
});
const scheduleStyle = computed(() => ({
  flexShrink: 0,
  fontSize: settings.value.scheduleSize + 'em',
}));
</script>

<template>
  <n-layout has-sider>
    <n-layout-content>
      <div class="wallpaperContainer" :style="backgroundStyle">
        <n-grid x-gap="20" cols="3">
          <!-- -webkit-right 可以把天气的 div 也居右 -->
          <n-gi style="text-align: -webkit-right">
            <DateAndCountDown/>
            <Weather v-if="weatherOnLeft"/>
          </n-gi>
          <n-gi style="text-align: -webkit-center">
            <SloganDisplay v-if="weatherOnLeft"/>
            <Weather v-else style="margin-top: 1em"/>
          </n-gi>
          <n-gi style="display: flex">
            <HomeworkDisplay :homeworks="homeworks.homeworks" style="width: 0; flex-grow: 1"/>
            <ScheduleDisplay :style="scheduleStyle"/>
          </n-gi>
        </n-grid>
      </div>
    </n-layout-content>
  </n-layout>
</template>

<style lang="scss">
.wallpaperContainer {
  min-height: 100vh;
  max-height: 100vh;
  overflow: hidden;
  background-size: cover;
  background-repeat: no-repeat;
  background-position: center;
}
</style>
