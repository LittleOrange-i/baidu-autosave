# 百度网盘自动转存工具 - 前端

基于 Vue 3 + TypeScript + Element Plus 的现代化前端重构版本。

## 🚀 快速开始

### 环境要求

- Node.js >= 16.0.0
- npm >= 8.0.0 (或 yarn >= 1.22.0)

### 安装依赖

```bash
cd frontend
npm install
```

### 开发模式

**⚠️ 重要：需要按顺序启动后端和前端**

#### 1. 启动后端 (必须先启动)
```bash
# 在项目根目录下
python web_app.py
```
后端将运行在 `http://localhost:5656`

#### 2. 启动前端
```bash
# 确保在 frontend 目录下
npm run dev

# 或使用快捷脚本
# Windows
start.bat
# Linux/Mac
./start.sh
```

启动后访问：http://localhost:3001  
API 请求会自动代理到后端的 5656 端口

### 生产构建

```bash
# 仅构建到 dist 目录
npm run build

# 构建并自动复制到后端使用的 static 目录（在项目根目录也可执行）
npm run build:static
# 或：npm --prefix frontend run build:static
```

`build:static` 会先清空后端根目录的 `static/`，再复制最新的 `dist/` 内容，避免残留旧哈希文件。

## 📁 项目结构

```
frontend/
├── src/
│   ├── components/          # 组件系统
│   │   ├── layout/         # 布局组件
│   │   │   ├── AppLayout.vue      # 主布局
│   │   │   ├── AppHeader.vue      # 头部导航
│   │   │   ├── AppSidebar.vue     # 侧边栏
│   │   │   └── AppBottomNav.vue   # 底部导航
│   │   └── business/       # 业务组件
│   │       └── AddTaskDialog.vue  # 任务对话框
│   ├── views/              # 页面组件
│   │   ├── login/          # 登录页面
│   │   ├── dashboard/      # 仪表盘
│   │   ├── tasks/          # 任务管理
│   │   ├── users/          # 用户管理
│   │   └── settings/       # 系统设置
│   ├── stores/             # 状态管理 (Pinia)
│   │   ├── auth.ts         # 认证状态
│   │   ├── tasks.ts        # 任务管理
│   │   ├── users.ts        # 用户管理
│   │   └── config.ts       # 配置管理
│   ├── services/           # API服务层
│   │   ├── http.ts         # HTTP客户端
│   │   ├── api.ts          # API接口
│   │   └── polling.ts      # 轮询服务
│   ├── composables/        # 组合式函数
│   │   ├── usePolling.ts   # 轮询管理
│   │   ├── useTasks.ts     # 任务操作
│   │   └── useVersionCheck.ts # 版本检查
│   ├── utils/              # 工具函数
│   ├── types/              # TypeScript类型定义
│   ├── router/             # 路由配置
│   ├── config/             # 配置文件
│   ├── App.vue             # 根组件
│   └── main.ts             # 应用入口
├── public/                 # 静态资源
│   └── favicon/           # 应用图标
├── package.json            # 依赖配置
├── vite.config.ts          # Vite配置
├── tsconfig.json          # TypeScript配置
├── start.sh               # Linux/Mac启动脚本
└── start.bat              # Windows启动脚本
```

## 🛠️ 技术栈

- **框架**: Vue 3 (Composition API)
- **语言**: TypeScript (严格模式)
- **构建工具**: Vite
- **UI组件库**: Element Plus
- **状态管理**: Pinia
- **路由**: Vue Router 4
- **工具库**: @vueuse/core

## 📋 主要功能

### 已实现功能

- ✅ 用户认证和登录
- ✅ 完整的布局系统（头部、侧边栏、底部导航）
- ✅ 响应式设计（桌面端/移动端自适应）
- ✅ 任务管理（添加、编辑、删除、执行）
- ✅ 批量操作（批量执行、批量删除）
- ✅ 用户管理（添加、切换、配额查看）
- ✅ 系统设置（通知、定时、分享配置）
- ✅ 实时状态更新（轮询机制）
- ✅ 版本检查和更新提醒
- ✅ 状态持久化（侧边栏折叠等）
- ✅ 无障碍设计支持

### API兼容性

- ✅ 完全兼容现有后端API
- ✅ 无需修改后端代码
- ✅ 保持数据格式一致
- ✅ 支持所有现有功能

## ⚙️ 配置说明

### 版本管理

版本信息在 `src/config/version.ts` 中管理：

```typescript
export const VERSION_CONFIG = {
  APP_VERSION: 'v1.1.3',
  BUILD_TIME: '2024-09-15T20:00:00Z',
  RELEASE_NOTES: '前端重构版本 - Vue 3 + TypeScript'
} as const
```

### API代理配置

开发环境API代理配置在 `vite.config.ts` 中：

```typescript
server: {
  port: 3000,
  proxy: {
    '/api': 'http://localhost:5656',
    '/login': 'http://localhost:5656'
  }
}
```

### 环境变量

可以在项目根目录创建以下环境变量文件：

- `.env.development` - 开发环境
- `.env.production` - 生产环境

## 🔧 开发指南

### 代码规范

- 使用 TypeScript 严格模式
- 遵循 Vue 3 Composition API 最佳实践
- 使用 ESLint + Prettier 进行代码格式化
- 组件名使用 PascalCase
- 文件名使用 kebab-case

### 状态管理

使用 Pinia 进行状态管理，按功能模块分离：

```typescript
// stores/tasks.ts
export const useTaskStore = defineStore('tasks', () => {
  const tasks = ref<Task[]>([])
  // ...
})
```

### API调用

统一使用服务层进行API调用：

```typescript
// services/api.ts
export class ApiService {
  async getTasks(): Promise<ApiResponse<{ tasks: Task[] }>> {
    return this.http.get('/api/tasks')
  }
}
```

### 组合式函数

将业务逻辑封装为可复用的组合式函数：

```typescript
// composables/useTasks.ts
export function useTasks() {
  const taskStore = useTaskStore()
  // 业务逻辑
  return { /* 导出的状态和方法 */ }
}
```

## 📱 响应式设计

项目采用响应式设计，支持多种设备：

- **移动端**: < 768px
- **平板端**: 768px - 1024px  
- **桌面端**: > 1024px

使用 CSS 媒体查询和 @vueuse/core 的 breakpoints 功能。

## 🚦 部署说明

### 开发环境部署

1. 启动后端服务（端口5656）
2. 启动前端开发服务器：`npm run dev`
3. 访问 http://localhost:3000

### 生产环境部署

1. 构建前端资源：`npm run build`
2. 构建文件将输出到 `dist` 目录
3. 配置后端服务器指向新的静态文件目录
4. 重启后端服务

## 🔍 调试指南

### Chrome DevTools

- Vue DevTools：查看组件状态和 Pinia store
- Network 面板：检查API请求
- Console 面板：查看日志和错误

### 常见问题

1. **API请求失败**：检查后端服务是否启动
2. **轮询不工作**：检查网络连接和API响应
3. **页面空白**：查看浏览器控制台错误信息

## 📖 更多文档

- [Vue 3 官方文档](https://vuejs.org/)
- [TypeScript 官方文档](https://www.typescriptlang.org/)
- [Element Plus 官方文档](https://element-plus.org/)
- [Pinia 官方文档](https://pinia.vuejs.org/)

## 🤝 贡献指南

1. Fork 项目
2. 创建功能分支：`git checkout -b feature/new-feature`
3. 提交更改：`git commit -m 'Add some feature'`
4. 推送分支：`git push origin feature/new-feature`
5. 提交 Pull Request

## 📄 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](../LICENSE) 文件了解详情。
