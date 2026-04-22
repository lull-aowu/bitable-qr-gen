## 多维表格二维码生成器

二维码生成工具，与飞书多维表格深度集成，允许用户创建各种模板的自定义二维码并直接导出到多维表格。

## 功能特点

- 多种不同风格的二维码模板
- 可自定义二维码内容和外观
- 实时预览和热更新
- 一键导出到多维表格


## 环境要求

- Node.js 14.0.0 或更高版本
- npm 6.0.0 或更高版本
- 具有多维表格访问权限的飞书账号

## 安装步骤

```bash
# 克隆仓库
git clone https://github.com/yourusername/bitable-qr-gen.git
cd bitable-qr-gen

# 安装依赖
npm install
```

## 开发指南

```bash
# 启动开发服务器，支持热重载
npm run dev

# 打开浏览器访问 http://localhost:5173
```

## 生产构建

```bash
# 构建生产版本应用
npm run build

# 预览生产构建
npm run preview --port 4173
```

## 配置说明

您可以通过编辑 `src/data/templates.json` 文件修改二维码模板。要添加新的语言，请在 `src/locales/` 目录中创建新的翻译文件。

## 发布到飞书应用市场

1. 运行 `npm run build` 生成生产构建
2. 提交包含 `dist` 目录的整个项目
3. 填写[提交表单](https://feishu.feishu.cn/share/base/form/shrcnGFgOOsFGew3SDZHPhzkM0e)

## 相关文档

- [飞书多维表格扩展开发指南](https://feishu.feishu.cn/docx/U3wodO5eqome3uxFAC3cl0qanIe)
- [Vue 3 文档](https://vuejs.org/guide/introduction.html)
- [Vite 文档](https://vitejs.dev/guide/)
- [Element Plus 文档](https://element-plus.org/zh-CN/)
