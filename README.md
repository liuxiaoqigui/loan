# 房贷计算器 - Vue 版本

一个功能完整的房贷计算器，支持等额本息、等额本金、提前还款、利率变更等功能。

## 功能特性

- ✅ 等额本息 / 等额本金两种还款方式
- ✅ 支持提前还款（减少月供 / 减少期数）
- ✅ 支持利率变更
- ✅ 美观的折线图展示还款趋势
- ✅ 配置导入/导出（JSON格式）
- ✅ 自动计算节省的利息
- ✅ 响应式设计，支持移动端

## 开发

### 安装依赖

```bash
cd loan-vue
npm install
```

### 启动开发服务器

```bash
npm run dev
```

访问 http://localhost:3000

## 构建生产版本

```bash
npm run build
```

构建完成后，`dist` 目录包含所有静态文件，可以直接部署到任何静态服务器。

## 部署到服务器

### 方式一：直接部署 dist 目录

将 `dist` 目录中的所有文件上传到服务器的 Web 根目录（如 `/var/www/html/`）。

### 方式二：使用 Nginx

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    root /path/to/dist;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # 安全头配置
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';";
}
```

### 方式三：使用 GitHub Pages

```bash
# 初始化 git 仓库
git init
git add .
git commit -m "Initial commit"

# 推送到 GitHub
git branch -M main
git remote add origin https://github.com/你的用户名/loan-calculator.git
git push -u origin main
```

然后在 GitHub 仓库设置中启用 GitHub Pages。

### 方式四：使用 Netlify / Vercel

将项目推送到 GitHub 后，可以直接在 Netlify 或 Vercel 上导入项目，会自动构建和部署。

## 安全建议

由于这是纯前端项目，安全风险很低：

1. ✅ 没有后端，没有数据库，没有 API
2. ✅ 没有用户输入执行（XSS 风险低）
3. ✅ 没有第三方依赖（除了 Chart.js）

额外防护措施：

```nginx
# 启用 HTTPS
listen 443 ssl;
ssl_certificate /path/to/cert.pem;
ssl_certificate_key /path/to/key.pem;

# 防止点击劫持
add_header X-Frame-Options "SAMEORIGIN";

# 防止 MIME 类型嗅探
add_header X-Content-Type-Options "nosniff";

# XSS 防护
add_header X-XSS-Protection "1; mode=block";
```

## 项目结构

```
loan-vue/
├── dist/                 # 构建输出目录
├── src/
│   ├── components/
│   │   └── LoanCalculator.vue   # 主组件
│   ├── App.vue                    # 根组件
│   ├── main.js                    # 入口文件
│   └── style.css                  # 样式文件
├── index.html             # HTML 模板
├── package.json           # 项目配置
└── vite.config.js        # Vite 配置
```

## 技术栈

- Vue 3 (Composition API)
- Vite
- Chart.js

## License

MIT
