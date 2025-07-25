<template>
  <div
    class="avatar"
    :style="{ width: avatarSize + 'px', height: avatarSize + 'px' }"
    @click="handleAvatarClick"
    @longpress="longpress"
    @touchend="touchend"
  >
    <!-- 使用遮罩层避免android长按头像会出现保存图片的弹窗 -->
    <div class="img-mask"></div>
    <img
      :lazy-load="true"
      class="avatar-img"
      v-if="avatarUrl"
      :src="avatarUrl"
      mode="aspectFill"
    />
    <div class="avatar-name-wrapper" :style="{ backgroundColor: color }">
      <div class="avatar-name-text" :style="{ fontSize: fontSize + 'px' }">
        {{ appellation }}
      </div>
    </div>
  </div>
</template>

<script lang="ts" setup>
import { getAvatarBackgroundColor } from "../utils";
import { autorun } from "mobx";
import { ref, computed, onUnmounted, getCurrentInstance } from "vue";
import type { V2NIMUser } from "nim-web-sdk-ng/dist/esm/nim/src/V2NIMUserService";
import { useRouter } from "vue-router";
import { neUiKitRouterPath } from "../utils/uikitRouter";

const props = withDefaults(
  defineProps<{
    account: string;
    teamId?: string;
    avatar?: string;
    size?: string;
    gotoUserCard?: boolean;
    fontSize?: string;
    isRedirect?: boolean;
  }>(),
  {
    teamId: "",
    avatar: "",
    size: "",
    gotoUserCard: false,
    fontSize: "",
    isRedirect: false,
  }
);

const $emit = defineEmits(["onLongpress"]);

const { proxy } = getCurrentInstance()!; // 获取组件实例
const router = useRouter();

const avatarSize = props.size || 42;
const user = ref<V2NIMUser>();
let isLongPress = false;
const appellation = computed(() => {
  return proxy?.$UIKitStore.uiStore
    .getAppellation({
      account: props.account,
      teamId: props.teamId,
      ignoreAlias: false,
      nickFromMsg: "",
    })
    ?.slice(0, 2);
});
const uninstallUserInfoWatch = autorun(async () => {
  proxy?.$UIKitStore?.userStore?.getUserActive(props.account).then((data) => {
    user.value = data;
  });
});

const avatarUrl = computed(() => {
  user.value = proxy?.$UIKitStore?.userStore?.users?.get(props.account);
  return props.avatar || user.value?.avatar;
});

const color = computed(() => {
  return getAvatarBackgroundColor(props.account);
});

const handleAvatarClick = () => {
  if (props.gotoUserCard && !isLongPress) {
    if (props.account === proxy?.$UIKitStore?.userStore?.myUserInfo.accountId) {
      router.push(neUiKitRouterPath.myDetail);
    } else {
      router.push({
        path: neUiKitRouterPath.friendCard,
        query: {
          accountId: props.account,
        },
      });
    }
  }
};

const longpress = () => {
  isLongPress = true;
  $emit("onLongpress");
};

const touchend = () => {
  setTimeout(() => {
    isLongPress = false;
  }, 200);
};

onUnmounted(() => {
  uninstallUserInfoWatch();
});
</script>

<style scoped>
.avatar {
  overflow: hidden;
  border-radius: 50%;
  flex-shrink: 0;
  position: relative;
}

.img-mask {
  position: absolute;
  z-index: 10;
  left: 0;
  right: 0;
  top: 0;
  bottom: 0;
  opacity: 0;
}

.avatar-img {
  width: 100%;
  height: 100%;
  border-radius: 50%;
}

.avatar-name-wrapper {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar-name-text {
  color: #fff;
  size: 14px;
}
</style>
