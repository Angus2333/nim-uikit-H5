<template>
  <div class="valid-list-container">
    <NavBar :title="t('validMsgText')" />
    <div class="valid-list-content">
      <Empty v-if="validMsg.length === 0" :text="t('validEmptyText')" />
      <template v-else>
        <div class="valid-item" v-for="msg in validMsg" :key="msg.timestamp">
          <!-- 好友申请 已同意 -->
          <template
            v-if="
              msg.status ===
              V2NIMConst.V2NIMFriendAddApplicationStatus
                .V2NIM_FRIEND_ADD_APPLICATION_STATUS_AGREED
            "
          >
            <div class="valid-item-left">
              <Avatar :account="msg.applicantAccountId" />
              <div class="valid-name-container">
                <div class="valid-name">
                  <Appellation :account="msg.applicantAccountId" />
                </div>
                <div class="valid-action">{{ t("applyFriendText") }}</div>
              </div>
            </div>
            <div class="valid-state">
              <Icon type="icon-yidu" />
              <span class="valid-state-text">{{ t("acceptResultText") }}</span>
            </div>
          </template>
          <!--好友申请 已拒绝 -->
          <template
            v-else-if="
              msg.status ===
              V2NIMConst.V2NIMFriendAddApplicationStatus
                .V2NIM_FRIEND_ADD_APPLICATION_STATUS_REJECTED
            "
          >
            <template v-if="isMeApplicant(msg)">
              <div class="valid-item-left">
                <Avatar :account="msg.recipientAccountId" />
                <div class="valid-name-container">
                  <div class="valid-name">
                    <Appellation :account="msg.recipientAccountId" />
                  </div>
                  <div class="valid-action">{{ t("beRejectResultText") }}</div>
                </div>
              </div>
            </template>
            <template v-else>
              <div class="valid-item-left">
                <Avatar :account="msg.applicantAccountId" />
                <div class="valid-name-container">
                  <div class="valid-name">
                    <Appellation :account="msg.applicantAccountId" />
                  </div>
                  <div class="valid-action">{{ t("applyFriendText") }}</div>
                </div>
              </div>
              <div class="valid-state">
                <Icon type="icon-shandiao" />
                <span class="valid-state-text">{{
                  t("rejectResultText")
                }}</span>
              </div>
            </template>
          </template>
          <!-- 好友申请 未处理 -->
          <template
            v-else-if="
              msg.status ===
              V2NIMConst.V2NIMFriendAddApplicationStatus
                .V2NIM_FRIEND_ADD_APPLICATION_STATUS_INIT
            "
          >
            <template v-if="!isMeApplicant(msg)">
              <div class="valid-item-left">
                <Avatar :account="msg.applicantAccountId" />
                <div class="valid-name-container">
                  <div class="valid-name">
                    <Appellation :account="msg.applicantAccountId" />
                  </div>
                  <div class="valid-action">{{ t("applyFriendText") }}</div>
                </div>
              </div>
              <div class="valid-buttons">
                <div
                  class="valid-button button-reject"
                  @click="handleRejectApplyFriendClick(msg)"
                  :loading="applyFriendLoading"
                >
                  {{ t("rejectText") }}
                </div>
                <div
                  class="valid-button button-accept"
                  @click="handleAcceptApplyFriendClick(msg)"
                  :loading="applyFriendLoading"
                >
                  {{ t("acceptText") }}
                </div>
              </div>
            </template>
          </template>
        </div>
      </template>
    </div>
  </div>
</template>

<script lang="ts" setup>
/** 验证消息页面 */
import { autorun } from "mobx";
import { onUnmounted, getCurrentInstance, ref } from "vue";
import Empty from "../../CommonComponents/Empty.vue";
import Avatar from "../../CommonComponents/Avatar.vue";
import NavBar from "../../CommonComponents/NavBar.vue";
import Icon from "../../CommonComponents/Icon.vue";
import { t } from "../../utils/i18n";
import type { V2NIMFriendAddApplicationForUI } from "@xkit-yx/im-store-v2/dist/types/types";
import { V2NIMConst } from "nim-web-sdk-ng/dist/esm/nim";
import Appellation from "../../CommonComponents/Appellation.vue";
import type { V2NIMMessage } from "nim-web-sdk-ng/dist/esm/nim/src/V2NIMMessageService";
import RootStore from "@xkit-yx/im-store-v2";
import { toast } from "../../utils/toast";

const validMsg = ref<V2NIMFriendAddApplicationForUI[]>([]);
const applyFriendLoading = ref(false);

const { proxy } = getCurrentInstance()!;

const store = proxy?.$UIKitStore as RootStore;
const nim = proxy?.$NIM;
/** 是否是我发起的申请 */
const isMeApplicant = (data: V2NIMFriendAddApplicationForUI) => {
  return data.applicantAccountId === store.userStore.myUserInfo.accountId;
};

/** 拒绝好友申请 */
const handleRejectApplyFriendClick = async (
  msg: V2NIMFriendAddApplicationForUI
) => {
  applyFriendLoading.value = true;
  try {
    await store.friendStore.rejectAddApplicationActive(msg);
    toast.info(t("rejectedText"));
  } catch (error) {
    toast.info(t("rejectFailedText"));
  } finally {
    applyFriendLoading.value = false;
  }
};

/** 接受好友申请 */
const handleAcceptApplyFriendClick = async (
  msg: V2NIMFriendAddApplicationForUI
) => {
  applyFriendLoading.value = true;
  try {
    try {
      await store.friendStore.acceptAddApplicationActive(msg);
      toast.info(t("acceptedText"));
    } catch (error) {
      toast.info(t("acceptFailedText"));
    }

    const textMsg = nim.V2NIMMessageCreator.createTextMessage(
      t("passFriendAskText")
    ) as unknown as V2NIMMessage;

    await store.msgStore.sendMessageActive({
      msg: textMsg,
      conversationId: nim.V2NIMConversationIdUtil.p2pConversationId(
        msg.operatorAccountId
      ),
    });
  } catch (error) {
    console.log("error", error);
  } finally {
    applyFriendLoading.value = false;
  }
};

/** 监听验证消息 */
const validMsgWatch = autorun(() => {
  validMsg.value = store?.sysMsgStore.friendApplyMsgs;
  store?.sysMsgStore.friendApplyMsgs?.map((item) => {
    store?.userStore.getUserActive(item.applicantAccountId);
  });
});

onUnmounted(() => {
  validMsgWatch();
});
</script>

<style scoped>
.valid-list-container {
  height: 100vh;
  box-sizing: border-box;
  background-color: #fff;
  overflow-y: auto;
}

.valid-list-content {
  height: calc(100% - 60px - var(--status-bar-height));
  width: 100%;
  overflow-y: auto;
  overflow-x: hidden;
}

.valid-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  height: 60px;
  padding: 0 20px;
}

.valid-name-container {
  display: flex;
  flex-direction: column;
  margin-left: 10px;
  flex: 1;
  padding-right: 20px;
}

.valid-name {
  font-size: 16px;
  color: #000;
  max-width: 35vw;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.valid-action {
  color: #888;
  font-size: 14px;
  max-width: 40vw;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.valid-item-left {
  display: flex;
  align-items: center;
}

.valid-buttons {
  display: flex;
  align-items: center;
}

.valid-button {
  margin: 0;
  width: 60px;
  height: 32px;
  line-height: 32px;
  font-size: 14px;
  text-align: center;
  border-radius: 3px;
  background-color: #fff;
}

.button-reject {
  color: #000;
  border: 1px solid #d9d9d9;
  margin-right: 10px;
}

.button-accept {
  color: #337eef;
  border: 1px solid #337eef;
}

.valid-state {
  display: flex;
  align-items: center;
}

.valid-state-text {
  margin-left: 10px;
  color: #000;
}
</style>
