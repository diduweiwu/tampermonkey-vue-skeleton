<template>
  <div class="config-area">
    <n-space vertical>
      <n-popover placement="left">
        <template #trigger>
          <n-button round :type="runningReadConfig.isAutoReading?'primary':'default'" @click="switchAutoRead">阅读
          </n-button>
        </template>
        <span>
          点击切换阅读状态~
        </span>
      </n-popover>
      <n-popover placement="left" trigger="click">
        <template #trigger>
          <n-button round>配置</n-button>
        </template>
        <config-panel/>
      </n-popover>
      <n-popover placement="left">
        <template #trigger>
          <n-button round @click="clearReadConfig">清除</n-button>
        </template>
        <span>点击恢复所有本地设置成默认</span>
      </n-popover>
      <n-button round @click="switchPage">翻页</n-button>

    </n-space>
  </div>
</template>

<script>
import {onBeforeUnmount, onMounted, reactive} from "vue";
import ConfigPanel from "@/component/panel/ConfigPanel.vue";
import {eventKey, offEvent, onEvent} from "@/helper/eventHelper";
import {getItem, removeItem, setItem} from "@/helper/storageHelper";
import {defaultReadConfig, storageKey} from "@/config/autoReadConfig";
import {composeAutoReadWorker, startAutoReadTask, stopAutoReadTask} from "@/task/autoReadTask";
import {simulateArrowRight} from "@/helper/keyboardHelper";

export default {
  name: "app",
  components: {ConfigPanel},
  methods: {},
  setup() {
    const worker = composeAutoReadWorker();

    const runningReadConfig = reactive({})

    const reloadRunningReadConfig = () => {
      const storageReadConfig = getItem(storageKey.READ_CONFIG, defaultReadConfig)
      Object.assign(runningReadConfig, storageReadConfig)
    }

    const startAutoRead = () => {
      startAutoReadTask(worker, runningReadConfig)
      runningReadConfig.isAutoReading = true
      setItem(storageKey.READ_CONFIG, runningReadConfig)
    };

    const stopAutoRead = () => {
      runningReadConfig.isAutoReading = false
      setItem(storageKey.READ_CONFIG, runningReadConfig)
      stopAutoReadTask(worker);
    }

    reloadRunningReadConfig()
    setTimeout(() => {
      // 加载完成后，如果记录的状态是正在阅读中，自动开启阅读任务
      if (runningReadConfig.isAutoReading) {
        startAutoRead();
      }
    }, 2000)

    // 切换阅读任务开启/关闭
    const switchAutoRead = () => {
      if (runningReadConfig.isAutoReading) {
        stopAutoRead();
        return
      }
      if (!runningReadConfig.isAutoReading) {
        startAutoRead();
      }
    }

    const reloadAutoRead = () => {
      reloadRunningReadConfig()
      if (runningReadConfig.isAutoReading) {
        stopAutoRead();
        startAutoRead()
      }
    }

    const clearReadConfig = () => {
      removeItem(storageKey.READ_CONFIG)
      window.location.reload()
    }

    const readConfigChangedCallback = () => {
      reloadAutoRead()
    }

    onMounted(() => {
      // 添加事件监听
      onEvent(eventKey.READ_CONFIG_UPDATE_EVENT, readConfigChangedCallback)
    })

    onBeforeUnmount(() => {
      // 页面卸载前销毁 Worker
      if (worker) {
        worker.terminate()
      }
      // 取消事件监听
      offEvent(eventKey.READ_CONFIG_UPDATE_EVENT, readConfigChangedCallback)
    })

    const switchPage = () => simulateArrowRight()
    return {
      switchAutoRead,
      reloadAutoRead,
      runningReadConfig,
      clearReadConfig,
      switchPage,
    }
  },
};
</script>
<style scoped>
.config-area {
  flex-wrap: wrap;
  position: fixed;
  top: calc(100vh / 2);
  right: 10px;
  z-index: 1024;
}
</style>

