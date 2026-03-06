# MindIE-SD Dockerfile 构建自动化 Workflow 生成 Skill

## Skill 概述

本 Skill 用于自动生成 MindIE-SD 项目的 GitHub Actions workflow，使用 Dockerfile 构建镜像并实现自动化构建 wheel 包的完整流水线。支持多种 Dockerfile 版本的并行构建和对比分析。

## 功能特性

- 自动生成 GitHub Actions workflow 文件
- 使用 Dockerfile 构建自定义镜像（不直接拉取预构建镜像）
- 在 workflow 执行时自动下载 Dockerfile，不存储到项目中
- 支持多种 Dockerfile 版本并行构建（Optimized 和 Original）
- 构建结果对比分析，自动生成对比报告
- Docker 环境配置和依赖检查
- 构建时长统计（镜像构建、wheel构建、制品归档）
- 时长统计展示在 GitHub Actions build summary
- 制品归档到 GitHub 默认制品仓
- Workflow 语法检查，确保无低级错误
- 自动提交代码并推送到远端仓库
- 完整的错误处理和故障排查

## Workflow 类型

### 1. 对比构建 Workflow (build-mindie-sd-comparison.yml)
- 并行执行 Optimized 和 Original 两种 Dockerfile 构建
- 自动生成对比分析报告
- 包含构建时间、镜像大小、wheel 包大小等对比
- 提供优化建议

### 2. 单独构建 Workflow
- `build-mindie-sd-optimized.yml`: 仅使用 Optimized Dockerfile 构建
- `build-mindie-sd-original.yml`: 仅使用 Original Dockerfile 构建
- `build-mindie-sd-dockerfile.yml`: 通用 Dockerfile 构建模板

## 环境要求

### 必需环境
- Git 工具
- GitHub 仓库访问权限
- Docker（本地测试用）

### Dockerfile 来源
```
https://github.com/DSLCI/test2/blob/docker-build/Dockerfile.py311.x86._2.6.0.optimized
```

## 配置参数

### Dockerfile 配置
```yaml
DOCKERFILE_URL: https://raw.githubusercontent.com/DSLCI/test2/docker-build/Dockerfile.py311.x86._2.6.0.optimized
IMAGE_NAME: mindie-sd-build
IMAGE_TAG: latest
```

### 代码仓库
```yaml
REPO_URL: https://gitcode.com/Ascend/MindIE-SD.git
```

### 环境变量
```yaml
SET_ENV_PATH: /usr/local/Ascend/ascend-toolkit/set_env.sh
TORCH_DEVICE_BACKEND_AUTOLOAD: 0
USER_ABI_VERSION: 1
```

## Workflow 结构

### 触发条件
```yaml
on:
  push:
    branches: [ main, master ]
    paths:
      - '.github/workflows/build-mindie-sd-dockerfile.yml'
      - 'build/**'
      - 'src/**'
      - 'setup.py'
      - 'pyproject.toml'
  pull_request:
    branches: [ main, master ]
  workflow_dispatch:
```

### 构建步骤

1. **Checkout code** - 检出代码
2. **Set up Docker Buildx** - 设置 Docker Buildx
3. **Check Docker environment dependencies** - 检查 Docker 环境
4. **Download Dockerfile** - 从远程仓库下载 Dockerfile
5. **Build Docker image from Dockerfile** - 使用 Dockerfile 构建镜像
6. **Build MindIE-SD wheel package** - 构建 wheel 包
7. **Upload wheel artifacts** - 上传构建产物到 GitHub 默认制品仓
8. **Calculate and display build statistics** - 显示构建统计到 build summary

## 核心实现

### 下载 Dockerfile
```bash
curl -o Dockerfile ${DOCKERFILE_URL}
```

### 构建镜像
```bash
docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -f Dockerfile .
```

### Docker 构建命令
```bash
docker run --rm \
  -v ${{ github.workspace }}:/workspace \
  ${IMAGE_NAME}:${IMAGE_TAG} \
  bash -c "cd /workspace && git clone https://gitcode.com/Ascend/MindIE-SD.git mindie-sd && cd mindie-sd && source /usr/local/Ascend/ascend-toolkit/set_env.sh && export TORCH_DEVICE_BACKEND_AUTOLOAD=0 && export USER_ABI_VERSION=1 && python setup.py bdist_wheel && ls -lh dist/"
```

## 常见问题与解决方案

### 1. Dockerfile 下载失败

**错误信息：**
```
curl: (22) The requested URL returned error: 404
```

**解决方案：**
确保 Dockerfile URL 正确且可访问：
```bash
# 验证 URL 是否可访问
curl -I ${DOCKERFILE_URL}
```

### 2. 镜像构建超时

**错误信息：**
```
Error: The operation was canceled.
```

**解决方案：**
- 增加构建超时时间
- 使用 GitHub Actions 缓存加速构建
- 优化 Dockerfile 减少层数

### 3. op_gen 模块找不到错误

**错误信息：**
```
ModuleNotFoundError: No module named 'op_gen'
```

**解决方案：**
使用 `set_env.sh` 脚本加载 Ascend 工具链环境：
```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
```

### 4. 文件权限错误

**错误信息：**
```
Error: You are not the owner of path /workspace/mindie-sd/build/ir_demo.json
```

**解决方案：**
在 Docker 容器内下载代码，避免宿主机和容器之间的文件权限冲突：
```bash
docker run --rm \
  -v ${{ github.workspace }}:/workspace \
  ${IMAGE_NAME}:${IMAGE_TAG} \
  bash -c "cd /workspace && git clone https://gitcode.com/Ascend/MindIE-SD.git mindie-sd && ..."
```

### 5. Workflow 语法检查

**要求：**
生成的 workflow 需要进行基本的语法检查，确保没有低级错误。

**实现方法：**
在提交前使用 YAML 语法检查工具：
```bash
# 安装 yamllint
pip install yamllint

# 检查 workflow 文件
yamllint .github/workflows/build-mindie-sd-dockerfile.yml
```

**验证要点：**
- YAML 缩进正确
- 变量引用格式正确
- 引号匹配
- 列表和字典格式正确
- GitHub Actions 语法符合规范

### 6. 制品归档到 GitHub 默认制品仓

**要求：**
构建产物需要归档到 GitHub 的默认制品仓。

**实现方法：**
使用 `actions/upload-artifact@v4`：
```yaml
- name: Upload wheel artifacts
  uses: actions/upload-artifact@v4
  with:
    name: mindie-sd-wheel
    path: mindie-sd/dist/*.whl
    retention-days: 30
```

**说明：**
- GitHub 默认制品仓存储限制：10GB
- 制品保留时间：30天（可配置）
- 制品可以在 GitHub Actions 界面直接下载
- 不需要额外的存储配置

### 7. 构建时长统计展示在 Build Summary

**要求：**
构建时长统计需要展示在 GitHub Actions 的 build summary 中。

**实现方法：**
使用 `$GITHUB_STEP_SUMMARY` 环境变量：
```yaml
- name: Calculate and display build statistics
  run: |
    BUILD_IMAGE_TIME=$(( ${{ steps.build_image.outputs.END_TIME }} - ${{ steps.build_image.outputs.START_TIME }} ))
    BUILD_WHEEL_TIME=$(( ${{ steps.build.outputs.END_TIME }} - ${{ steps.build.outputs.START_TIME }} ))
    UPLOAD_TIME=$(( ${{ steps.upload_end.outputs.END_TIME }} - ${{ steps.upload_start.outputs.START_TIME }} ))

    echo "## Build Time Statistics" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "| Phase | Duration (seconds) | Duration (minutes) |" >> $GITHUB_STEP_SUMMARY
    echo "|-------|-------------------|-------------------|" >> $GITHUB_STEP_SUMMARY
    echo "| Image Build (from Dockerfile) | $BUILD_IMAGE_TIME | $(awk "BEGIN {printf \"%.2f\", $BUILD_IMAGE_TIME/60}") |" >> $GITHUB_STEP_SUMMARY
    echo "| Build Wheel (includes code download) | $BUILD_WHEEL_TIME | $(awk "BEGIN {printf \"%.2f\", $BUILD_WHEEL_TIME/60}") |" >> $GITHUB_STEP_SUMMARY
    echo "| Artifact Upload | $UPLOAD_TIME | $(awk "BEGIN {printf \"%.2f\", $UPLOAD_TIME/60}") |" >> $GITHUB_STEP_SUMMARY

    TOTAL_TIME=$(( BUILD_IMAGE_TIME + BUILD_WHEEL_TIME + UPLOAD_TIME ))
    echo "| **Total** | **$TOTAL_TIME** | **$(awk "BEGIN {printf \"%.2f\", $TOTAL_TIME/60}")** |" >> $GITHUB_STEP_SUMMARY
```

**显示效果：**
- 统计表格会显示在 workflow 运行页面的 Summary 部分
- 包含各阶段的秒数和分钟数
- 支持表格格式，易于阅读
- 自动计算总时长

## 自动化脚本

### 使用方法
```bash
# 创建目录
mkdir -p /path/to/project/.github/workflows

# 生成 workflow 文件
# （参考上面的核心实现部分）

# 初始化 Git 仓库
cd /path/to/project
git init
git add .
git commit -m "Add MindIE-SD Dockerfile build workflow"

# 推送到远端
git remote add origin https://github.com/username/repo.git
git push -u origin main
```

## 文件结构

```
project/
├── .github/
│   └── workflows/
│       └── build-mindie-sd-dockerfile.yml
├── skills/
│   └── MindIE-SD-Dockerfile-Build/
│       └── SKILL.MD
└── README.md
```

## 构建统计

Workflow 会自动统计并显示各阶段的构建时长：

- **Image Build (from Dockerfile)** - 镜像构建时长
- **Build Wheel (includes code download)** - wheel构建时长（包含代码下载）
- **Artifact Upload** - 制品归档时长
- **Total** - 总时长

## 最佳实践

1. **在容器内操作**：所有文件操作都在 Docker 容器内进行，避免权限问题
2. **动态下载 Dockerfile**：在 workflow 执行时下载 Dockerfile，不存储到项目中
3. **环境变量配置**：使用 `set_env.sh` 脚本而不是手动设置环境变量
4. **语法检查**：提交前使用 yamllint 检查 workflow 语法
5. **制品归档**：使用 GitHub 默认制品仓，配置合理的保留时间
6. **时长统计**：使用 $GITHUB_STEP_SUMMARY 在 build summary 中显示统计信息
7. **错误处理**：每个步骤都有清晰的错误信息和解决方案
8. **文档维护**：及时更新 README 和故障排查指南

## 扩展功能

### 添加额外的构建步骤
```yaml
- name: Custom build step
  run: |
    echo "Custom build operations"
```

### 支持多个分支
```yaml
on:
  push:
    branches: [ main, master, develop ]
```

### 添加通知功能
```yaml
- name: Send notification
  if: always()
  uses: actions/github-script@v6
  with:
    script: |
      github.rest.issues.createComment({
        issue_number: context.issue.number,
        owner: context.repo.owner,
        repo: context.repo.repo,
        body: 'Build completed!'
      })
```

## 维护和更新

### 更新 Dockerfile URL
修改 `DOCKERFILE_URL` 变量并提交更改

### 更新环境变量
修改构建命令中的 `export` 语句

### 验证 Workflow 语法
使用 yamllint 检查语法：
```bash
yamllint .github/workflows/build-mindie-sd-dockerfile.yml
```

### 测试 Workflow
1. 推送代码到测试分支
2. 在 GitHub Actions 界面查看运行状态
3. 检查 build summary 中的时长统计
4. 验证制品是否正确上传到制品仓

## 参考资源

- [MindIE-SD 官方仓库](https://gitcode.com/Ascend/MindIE-SD)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [Docker 文档](https://docs.docker.com/)
- [华为云 Ascend 文档](https://www.huawei.com/en/product/ascend)

## 许可证

请参考 MindIE-SD 项目的许可证。
