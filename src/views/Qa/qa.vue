<template>
  <div class="qa-container" >
    <div class="qa-list" >
      <div v-for="question in questions" :key="question.id" class="qa-item">
        <div class="question-header">
          <h3>{{ question.content }}</h3>
          <div class="button-group">
            <a-button size="small" @click="expandQuestion(question)">
              {{ question.expanded ? '收起' : '展开' }}
            </a-button>
            <a-button type="primary" size="small" @click="showAnswerDialog(question)">
              回答
            </a-button>
            <a-button v-if="isAdmin" size="small" @click="toggleQuestionStatus(question)">
              {{ question.finished ? '标记为未解决' : '标记为已解决' }}
            </a-button>
          </div>
        </div>

        <div v-if="question.expanded" class="answers-container">
          <div v-if="question.answers.length === 0" class="no-answers">无回答</div>
          <div v-for="answer in question.answers" :key="answer.id" class="answer-card">
            <div class="answer-header">
              <span class="username">{{ answer.username }}</span>
              <span class="time">{{ formatDate(answer.createTime) }}</span>
            </div>
            <div class="answer-content">{{ answer.content }}</div>
            <div v-if="answer.username === currentUser" class="answer-status">
              <span :class="getStatusClass(answer.status)">
                {{ getStatusText(answer.status) }}
              </span>
            </div>
            <div v-if="isAdmin" class="admin-actions">
              <a-button size="small" @click="approveAnswer(answer)"> 通过 </a-button>
              <a-button size="small" @click="rejectAnswer(answer)"> 不通过 </a-button>
            </div>
          </div>
        </div>
      </div>
    </div>
    <div v-if="loading" class="loading">加载中...</div>
    <div v-if="!loading && noMoreQuestions" class="no-more">没有更多问题了</div>

    <a-modal
      v-model:visible="answerDialogVisible"
      title="回答问题"
      @ok="submitAnswer"
      @cancel="answerDialogVisible = false"
    >
      <div class="original-question">
        <h4>原问题：</h4>
        <p>{{ currentQuestion?.content }}</p>
      </div>
      <a-form>
        <a-form-item>
          <a-textarea v-model:value="newAnswer" :rows="4" placeholder="请输入您的回答..." />
        </a-form-item>
      </a-form>
    </a-modal>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted } from 'vue';
import { message } from 'ant-design-vue';
import type { Question } from './types';
import { useQaStore } from '@/store/qa';
import { storeToRefs } from 'pinia';
import moment from 'moment';
import { useUser } from '@/store/useUser';

const userStore = useUser();
const qaStore = useQaStore();
const { questions, loading, noMoreQuestions } = storeToRefs(qaStore);
const answerDialogVisible = ref(false);
const currentQuestion = ref<Question | null>(null);
const newAnswer = ref('');
const isAdmin = ref(true); // 假设从某处获取用户角色信息
const currentUser = '当前用户'; // 假设从某处获取当前用户名

const handleScroll = async (event: Event) => {
  const target = event.target as HTMLElement;
  if (target.scrollHeight - target.scrollTop <= target.clientHeight + 1) {
    await qaStore.loadMoreQuestions();
  }
};
onMounted(async () => {
  document.addEventListener('scroll', handleScroll);
  await userStore.updateUserInfo(); // 确保在获取问题之前更新用户信息
  isAdmin.value = userStore.userInfo.role === 'admin'; // 根据角色设置 isAdmin
  await qaStore.fetchQuestions();
});

const expandQuestion = async (question: Question) => {
  if (!question.expanded) {
    await qaStore.fetchAnswers(question.id);
  }
  question.expanded = !question.expanded;
};

const showAnswerDialog = (question: Question) => {
  currentQuestion.value = question;
  answerDialogVisible.value = true;
  newAnswer.value = '';
};

const submitAnswer = async () => {
  if (!currentQuestion.value || !newAnswer.value.trim()) {
    message.warning('请输入回答内容');
    return;
  }

  try {
    await qaStore.submitAnswer(currentQuestion.value.id, newAnswer.value);
    message.success('回答提交成功');
    answerDialogVisible.value = false;
    await expandQuestion(currentQuestion.value);
  } catch (error) {
    message.error('提交失败，请重试');
  }
};

const approveAnswer = async (answer) => {
  try {
    await qaStore.approveAnswer(answer);
    message.success('答案已通过审核');
  } catch (error) {
    message.error('审核失败，请重试');
  }
};

const rejectAnswer = async (answer) => {
  try {
    await qaStore.rejectAnswer(answer);
    message.success('答案已被拒绝');
  } catch (error) {
    message.error('审核失败，请重试');
  }
};

const formatDate = (date: string) => {
  return moment(date).format('YYYY-MM-DD HH:mm:ss');
};



const toggleQuestionStatus = async (question: Question) => {
  try {
    const newStatus = !question.finished;
    console.log(111);
    await qaStore.setQuestionfinished(question.id, newStatus);
    console.log(question.id);
    question.finished = newStatus;
  } catch (error) {
    message.error('更新问题状态失败，请重试');
  }
};

const getStatusClass = status => {
  switch (status) {
    case 'approved':
      return 'status-approved';
    case 'rejected':
      return 'status-rejected';
    default:
      return 'status-pending';
  }
};

const getStatusText = status => {
  switch (status) {
    case 'approved':
      return '✔ 审核通过';
    case 'rejected':
      return '✘ 审核不通过';
    default:
      return '⏳ 待审核';
  }
};
</script>

<style lang="scss" scoped>
.qa-container {
  padding: 20px;
  max-width: 1200px;
  margin: 0 auto;
  overflow-y:auto;
  height:100vh;
  // border: 1px solid #ddd; // 可选：添加边框以更明显地显示滚动区域
}

.qa-item {
  border: 1px solid #f0f0f0;
  border-radius: 4px;
  margin-bottom: 16px;
  padding: 16px;
  background: #fff;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1); // 添加阴影
}

.qa-list {
  display: flex;
  flex-direction: column;
  gap: 16px; // 增加间距
}

.question-header {
  display: flex;
  justify-content: space-between;
  align-items: center;

  h3 {
    margin: 0;
  }
}

.button-group {
  display: flex;
  gap: 8px;

  a-button {
    background-color: #8a66f4; // 修改按钮背景颜色
    color: #fff; // 修改按钮字体颜色
  }
}

.answers-container {
  margin-top: 16px;
}

.answer-card {
  background: #fafafa;
  border-radius: 4px;
  padding: 12px;
  margin-top: 8px;
}

.answer-header {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  color: rgba(0, 0, 0, 0.45);
  font-size: 14px;
}

.answer-content {
  line-height: 1.6;
}

.admin-actions {
  margin-top: 8px;
  display: flex;
  justify-content: flex-end;
}

.original-question {
  margin-bottom: 20px;
  padding: 12px;
  background: #fafafa;
  border-radius: 4px;

  h4 {
    margin: 0 0 8px;
  }

  p {
    margin: 0;
  }
}

.loading,
.no-more {
  text-align: center;
  margin-top: 20px;
  color: #888;
}

.no-answers {
  text-align: center;
  color: #888;
  margin-top: 10px;
}

.status-approved {
  color: green;
}
.status-rejected {
  color: red;
}
.status-pending {
  color: gray;
}
</style>
