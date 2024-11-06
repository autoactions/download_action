# 远程下载上传助手

## 项目简介

这是一个基于 GitHub Actions 的远程下载上传工具，可以自动下载指定链接的文件并上传至云存储。该项目使用 Cloudflare Workers 作为API入口，通过 GitHub Repository Dispatch 触发工作流程。

## 主要功能

- 支持远程URL文件下载
- 自动上传至云存储（支持多种云存储服务）
- 自动按日期分类存储
- 支持断点续传
- 多线程下载加速
- 自动重试机制

## 技术架构

- 前端API: Cloudflare Workers
- 工作流引擎: GitHub Actions
- 下载工具: aria2
- 云存储工具: rclone

## 部署步骤

### 1. 配置 GitHub 仓库

1. Fork 本仓库
2. 配置以下 GitHub Secrets:
   - `GITHUB_TOKEN`: GitHub Personal Access Token
   - `RCLONE_CONFIG`: rclone 配置文件内容
   - `REMOTE_NAME`: 远程存储名称
   - `UPLOAD_PATH`: 上传路径

### 2. 部署 Cloudflare Worker

1. 在 Cloudflare Workers 中创建新的 Worker
2. 将 `_worker.js` 的内容部署到 Worker
3. 配置以下环境变量：
   - `GITHUB_TOKEN`
   - `GITHUB_OWNER`
   - `GITHUB_REPO`

## 使用方法

### API 调用

```http
GET https://your-worker.workers.dev/[encoded-download-url]
```

### 响应格式

成功响应：
```json
{
    "success": true,
    "message": "工作流已触发",
    "url": "下载链接"
}
```

错误响应：
```json
{
    "success": false,
    "message": "错误信息",
    "error": "详细错误描述"
}
```

## 核心代码说明

1. Worker 入口处理（参考 `_worker.js`）：

```1:82:_worker.js
export default {
    async fetch(request, env) {
        const url = new URL(request.url);
        const downloadUrl = decodeURIComponent(url.pathname.substring(1));

        if (!downloadUrl) {
            return new Response(JSON.stringify({ message: '缺少下载链接' }), {
                status: 400,
                headers: { 
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                }
            });
        }

        try {
            const githubToken = env.GITHUB_TOKEN;
            const owner = env.GITHUB_OWNER;
            const repo = env.GITHUB_REPO;

            if (!githubToken || !owner || !repo) {
                throw new Error('缺少必要的环境变量配置');
            }

            // 使用验证过的 repository dispatch API 调用方式
            const response = await fetch(
                `https://api.github.com/repos/${owner}/${repo}/dispatches`,
                {
                    method: 'POST',
                    headers: {
                        'Authorization': `token ${githubToken}`,
                        'User-Agent': 'Mozilla/5.0 (compatible; DownloadBot/1.0)',
                        'Content-Type': 'application/json',
                    },
                    body: JSON.stringify({
                        event_type: 'download_file',
                        client_payload: {
                            download_url: downloadUrl,
                            timestamp: new Date().toISOString()
                        }
                    })
                }
            );
            // 记录响应信息用于调试
            console.log('GitHub API Response:', {
                status: response.status,
                statusText: response.statusText,
                headers: Object.fromEntries(response.headers.entries())
            });

            if (response.ok || response.status === 204) {
                return new Response(JSON.stringify({ 
                    success: true,
                    message: '工作流已触发',
                    url: downloadUrl
                }), {
                    headers: { 
                        'Content-Type': 'application/json',
                        'Access-Control-Allow-Origin': '*'
                    }
                });
            } else {
                throw new Error(`无法触发 GitHub Action: ${response.status} ${response.statusText}`);
            }

        } catch (error) {
            console.error('Error:', error);
            return new Response(JSON.stringify({ 
                success: false,
                message: '服务器错误',
                error: error.message
            }), {
                status: 500,
                headers: { 
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                }
            });
        }
    }
}
```


2. GitHub Actions 工作流配置（参考 `.github/workflows/download-upload.yml`）：

```1:119:.github/workflows/download-upload.yml
name: Download and Upload to Cloud Storage

on:
  workflow_dispatch:
    inputs:
      download_url:
        description: '下载链接'
        required: true
  repository_dispatch:
    types: [download_file]

jobs:
  transfer:
    runs-on: ubuntu-latest
    steps:
      - name: 获取下载链接
        run: |
          if [ "${{ github.event_name }}" = "repository_dispatch" ]; then
            echo "DOWNLOAD_URL=${{ github.event.client_payload.download_url }}" >> $GITHUB_ENV
          else
            echo "DOWNLOAD_URL=${{ github.event.inputs.download_url }}" >> $GITHUB_ENV
          fi
          
      - name: 安装 aria2 和 rclone
        run: |
          sudo apt-get update
          sudo apt-get install -y aria2
          curl https://rclone.org/install.sh | sudo bash
          
      - name: 配置 rclone
        run: |
          mkdir -p ~/.config/rclone
          # 确保配置文件格式正确
          echo '${{ secrets.RCLONE_CONFIG }}' > ~/.config/rclone/rclone.conf
          
          # 验证配置文件
          echo "验证 rclone 配置..."
          rclone config show
          
          # 列出可用的远程存储
          echo "可用的远程存储:"
          rclone listremotes
          
      - name: 下载文件
        run: |
          mkdir -p downloads
          cd downloads
          
          # 从URL中提取文件名
          FILENAME=$(basename "$DOWNLOAD_URL" | sed 's/\?.*//')
          echo "下载文件名: $FILENAME"
          
          # 使用 --allow-overwrite 参数避免重名文件报错
          # 使用 --max-tries 参数设置重试次数
          aria2c --allow-overwrite=true --max-tries=5 \
                --max-connection-per-server=16 \
                --split=16 --min-split-size=1M \
                --connect-timeout=10 --timeout=600 \
                --auto-file-renaming=false \
                "$DOWNLOAD_URL"
          
          # 检查下载是否成功
          if [ $? -ne 0 ]; then
            echo "下载失败！"
            exit 1
          fi
          
      - name: 上传到网盘
        run: |
          echo "检查下载目录内容..."
          ls -la downloads
          
          # 检查文件是否存在
          if [ -z "$(ls -A downloads)" ]; then
            echo "错误：下载目录为空！"
            exit 1
          fi
          
          # 使用实际的远程存储名称，确保路径格式正确
          REMOTE="${{ secrets.REMOTE_NAME }}"
          UPLOAD_PATH="${{ secrets.UPLOAD_PATH }}"
          
          # 去除路径首尾的空格
          UPLOAD_PATH=$(echo "$UPLOAD_PATH" | sed 's/^[[:space:]]*//;s/[[:space:]]*$//')
          
          # 确保路径以 / 开头
          [[ "$UPLOAD_PATH" != /* ]] && UPLOAD_PATH="/$UPLOAD_PATH"
          
          # 获取当前日期作为子目录
          DATE_DIR=$(date +"%Y-%m-%d")
          UPLOAD_PATH="${UPLOAD_PATH}/${DATE_DIR}"
          
          echo "准备上传到: $REMOTE:$UPLOAD_PATH"
          
          # 验证远程存储是否可访问
          echo "验证远程存储..."
          rclone lsd "$REMOTE:" || exit 1
          
          echo "开始上传..."
          # 添加更多的上传参数来提高可靠性
          rclone copy --progress \
                     --transfers 4 \
                     --checkers 8 \
                     --tpslimit 10 \
                     --retries 3 \
                     --low-level-retries 10 \
                     --stats 1s \
                     downloads/ "$REMOTE:$UPLOAD_PATH"
          
          # 检查上传是否成功
          if [ $? -eq 0 ]; then
            echo "上传完成！"
            # 列出上传后的文件
            echo "已上传的文件:"
            rclone ls "$REMOTE:$UPLOAD_PATH"
          else
            echo "上传失败！"
            exit 1
          fi
```


## 性能优化

- 使用 aria2 多线程下载
- rclone 多线程上传
- 自动重试机制
- 断点续传支持

## 注意事项

1. 确保 GitHub Token 具有足够的权限
2. rclone 配置文件格式需要正确
3. 建议设置合适的超时时间
4. 注意云存储的容量和带宽限制

## 错误处理

系统实现了完整的错误处理机制：
- 下载失败自动重试
- 上传失败重试
- API 调用错误处理
- 详细的日志记录

## 贡献指南

欢迎提交 Pull Request 或提出 Issue。在提交代码前，请确保：
1. 代码符合项目规范
2. 添加必要的测试
3. 更新相关文档

## 许可证

MIT License