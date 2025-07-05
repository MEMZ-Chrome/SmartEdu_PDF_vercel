<template>
  <div>
    <!-- 教材列表与选择复选框 -->
    <div v-if="materials.length > 0">
      <h2>请选择要下载的教材</h2>
      <div v-for="mat in materials" :key="mat.id">
        <!-- 复选框绑定到 selectedIds 数组 -->
        <input type="checkbox" :value="mat.id" v-model="selectedIds" />
        <span>{{ mat.name || ('教材 ID:' + mat.id) }}</span>
      </div>
    </div>

    <!-- 如果教材列表为空，显示手动输入提示 -->
    <div v-else>
      <h2>无法自动获取教材列表</h2>
      <p>请在 SmartEdu 平台复制教材链接并粘贴到下方， 每行一个链接：</p>
      <textarea v-model="manualInput" placeholder="https://basic.smartedu.cn/tchMaterial/detail?contentId=XXXXX"></textarea>
      <button @click="parseManual">解析列表</button>
    </div>

    <!-- 下载链接展示区域 -->
    <div v-if="selectedIds.length > 0">
      <h2>下载链接</h2>
      <div v-for="id in selectedIds" :key="id">
        <!-- 构造的下载链接 -->
        <a :href="buildLink(id)" target="_blank">{{ buildLink(id) }}</a>
        <button @click="copyLink(buildLink(id))">复制链接</button>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue';

// 响应式数据：教材列表和选中ID列表
const materials = ref([]);      // 存放 {id, name} 对象
const selectedIds = ref([]);    // 存放用户选中的教材 ID
const manualInput = ref('');    // 用户粘贴的教材链接列表文本

// 页面加载时尝试自动抓取教材列表
onMounted(async () => {
  try {
    // 使用 Fetch 获取 SmartEdu 教材列表页面
    const response = await fetch('https://basic.smartedu.cn/tchMaterial');
    if (response.ok) {
      const html = await response.text();
      // 解析 HTML
      const parser = new DOMParser();
      const doc = parser.parseFromString(html, 'text/html');
      // 查找所有包含 contentId 的链接
      const linkElements = Array.from(doc.querySelectorAll('a[href*="contentId="]'));
      // 去重用的 Set
      const idSet = new Set();
      linkElements.forEach(a => {
        try {
          const url = new URL(a.href);
          const id = url.searchParams.get('contentId');
          if (id && !idSet.has(id)) {
            idSet.add(id);
            // 链接文本作为教材名称
            const name = a.textContent.trim();
            materials.value.push({ id, name });
          }
        } catch {}
      });
    }
  } catch (error) {
    console.error('抓取教材列表失败（可能跨域）:', error);
    // 不做处理，保持 materials 为空，以便切换到手动模式
  }
});

// 解析手动粘贴的链接列表
function parseManual() {
  materials.value = [];
  const lines = manualInput.value.split(/\r?\n/).map(s => s.trim()).filter(s => s);
  lines.forEach(line => {
    try {
      const url = new URL(line);
      const id = url.searchParams.get('contentId');
      if (id) {
        materials.value.push({ id, name: '' });
      }
    } catch {}
  });
}

// 根据 contentId 构造下载链接字符串
function buildLink(id) {
  return `https://r1-ndr.ykt.cbern.com.cn/edu_product/esp/assets_document/${id}.pkg/pdf.pdf`;
}

// 复制链接到剪贴板并提示
async function copyLink(link) {
  try {
    await navigator.clipboard.writeText(link);
    alert('链接已复制到剪贴板');
  } catch {
    alert('复制失败，请手动复制');
  }
}
</script>

<style scoped>
textarea {
  width: 100%;
  height: 100px;
  margin-top: 8px;
}
button {
  margin-left: 8px;
}
h2 {
  margin-top: 16px;
}
</style>
