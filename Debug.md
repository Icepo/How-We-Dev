# 配置 FastAPI 应用在 Docker 中的调试环境 (使用 debugpy 和 VS Code)

本文档以调试一个在 Docker 中运行的 FastAPI 应用（如 `open-webui`）为例，介绍如何配置 `debugpy` 和 VS Code 进行远程调试。

## 一、 先决条件

1.  **安装 `debugpy` 包:** 确保你的 Python 环境（通常是在 `Dockerfile` 中）安装了 `debugpy` 包。
    *   可以在 `requirements.txt` 中添加 `debugpy`。
    *   或者在 `Dockerfile` 中使用 `pip install debugpy`。
    *   *提示: 可以利用 Docker 多阶段构建来优化，仅在最终镜像中包含必要的运行时依赖，避免包含调试工具，但这超出了基础调试配置的范围。*

## 二、 配置步骤

### 1. 修改应用启动方式

你需要修改容器启动应用的方式，从直接使用 `uvicorn` 启动改为通过 `debugpy` 包装 `uvicorn` 启动。

*   **原正常启动命令 (示例):**
    ```shell
    uvicorn open_webui.main:app --host "$HOST" --port "$PORT" --forwarded-allow-ips '*' --workers "${UVICORN_WORKERS:-1}"
    ```

*   **修改后的调试启动命令 (示例):**
    (通常在容器启动脚本 `start.sh` 或 Dockerfile 的 `CMD`/`ENTRYPOINT` 中修改)
    ```shell
    # 使用 exec 替换 shell 进程，python -m debugpy 启动调试服务器
    # --listen 0.0.0.0:15678: 在容器所有网络接口监听 15678 端口
    # --wait-for-client: 启动后暂停，等待调试器连接
    # -m uvicorn ...: debugpy 连接成功后要执行的命令
    # --workers 1: **强烈建议**调试时将 worker 数量设为 1，避免调试器连接错进程
    # --reload --reload-dir ...: 启用热重载 (可选，但开发时常用)
    exec python -m debugpy --listen 0.0.0.0:15678 --wait-for-client -m uvicorn open_webui.main:app --host "$HOST" --port "$PORT" --forwarded-allow-ips '*' --workers 1 --reload --reload-dir "/app/backend"
    ```
    *注意：*
    *   调试端口 `15678` 是自定义的（默认 `5678` 可能冲突），确保与后续配置一致。
    *   `--reload-dir` 后面的路径 `/app/backend` 需要是你容器内**实际存放**后端代码的路径。请根据你的 Dockerfile 进行确认。
    *   **`--workers 1` 对于可靠调试至关重要。**

### 2. 配置 VS Code 调试文件 (`launch.json`)

1.  在 VS Code 中，打开你的项目文件夹。
2.  切换到 "运行和调试" 视图 (侧边栏图标)。
3.  点击齿轮图标 ⚙️ 或 "创建 launch.json 文件" 链接。
4.  选择 "Python" 环境（如果提示）。
5.  VS Code 会在项目根目录下创建或打开 `.vscode/launch.json` 文件。
6.  添加或修改 `configurations` 数组，包含以下 "attach" 配置：

    ```json
    {
        "version": "0.2.0",
        "configurations": [
            {
                "name": "Python: Attach to Docker (debugpy)", // 自定义配置名称
                "type": "debugpy",                           // 调试器类型
                "request": "attach",                         // 请求类型：附加到已运行进程
                "connect": {
                    "host": "localhost", // 连接本地主机 (Docker端口映射后)
                    "port": 15678        // 必须与 debugpy --listen 指定的端口一致
                },
                "pathMappings": [        // **极其重要：本地与容器路径映射**
                    {
                        // "localRoot": "你本地机器上代码的路径",
                        // 例如: "${workspaceFolder}/backend"
                        // `${workspaceFolder}` 代表 VS Code 打开的项目根目录。
                        // 这个路径必须准确指向你本地存放后端代码的文件夹。
                        "localRoot": "${workspaceFolder}/backend", 

                        // "remoteRoot": "容器内部代码对应的绝对路径"
                        // 这个路径由你的 Dockerfile (`WORKDIR`, `COPY`) 决定。
                        // 必须和容器内 Python 实际运行的代码路径一致。
                        "remoteRoot": "/app/backend" 
                    }
                    // 如果项目结构复杂，可以添加更多映射规则
                ],
                "jinja": true,           // (可选) 启用 Jinja2 模板调试
                "justMyCode": true       // 默认: true (只调试你的代码，跳过库代码)
                                         // 设置为 false 可以进入第三方库或框架内部调试。
            }
            // 可能还有其他配置...
        ]
    }

    ```

### 3. 配置 Docker 端口

确保 Docker 配置暴露并映射了**应用端口**和**调试端口**。

*   **Dockerfile:** 使用 `EXPOSE` 指令声明端口（主要用于文档和自动化）。
    ```dockerfile
    EXPOSE 8080  # 应用端口 (示例)
    EXPOSE 15678 # 调试端口
    ```
*   **`docker-compose.yml` (推荐):** 使用 `ports` 映射宿主机端口到容器端口。
    ```yaml
    services:
      open-webui:
        # ... 其他配置 ...
        ports:
          - "[宿主机应用端口]:8080"  # 例如: "3000:8080" 或 "8080:8080"
          - "15678:15678"         # 映射调试端口
        # ... 其他配置 ...
    ```
*   **`docker run`:** 使用 `-p` 参数映射。
    ```bash
    docker run -p [宿主机应用端口]:8080 -p 15678:15678 ... your_image_name
    ```

### 4. 启动项目和调试

1.  **启动容器:** 使用 `docker-compose up --build` (或 `docker run`) 启动你的应用容器。由于启动命令包含 `--wait-for-client`，应用会在日志中打印 `debugpy` 相关信息后暂停，等待连接。
2.  **启动 VS Code 调试:**
    *   回到 VS Code 的 "运行和调试" 视图。
    *   从顶部的下拉菜单中选择你配置的名称 (例如 "Python: Attach to Docker (debugpy)")。
    *   点击绿色的播放按钮 ▶️ (或按 F5)。
3.  **连接成功:**
    *   VS Code 底部状态栏会显示调试连接状态。
    *   容器的日志会继续输出 Uvicorn 启动信息 (如 `Uvicorn running on ...`)。
    *   现在可以在代码中设置断点，并通过浏览器访问应用来触发调试了。

## 三、 常见问题排查

1.  **断点是空心圆圈/灰色，无法命中:**
    *   **原因:** 最常见的是 `launch.json` 中的 `pathMappings` 配置错误。
    *   **解决:** 仔细检查 `localRoot` (本地路径) 和 `remoteRoot` (容器内路径) 是否完全正确，注意大小写和绝对路径。可以通过 `docker exec -it <container_id> ls <path>` 确认容器内路径。

2.  **可以连接调试器，但请求时断点不命中:**
    *   **原因 1:** Uvicorn 运行了多个 worker 进程 (`--workers` 大于 1)。`debugpy` 只能附加到一个进程，请求可能被其他未附加调试器的进程处理了。
    *   **解决 1:** 确保调试启动命令中使用了 `--workers 1`。需要重新构建并启动容器。
    *   **原因 2:** 断点设置的位置不正确，或者浏览器的操作并未实际执行到包含断点的代码路径。
    *   **解决 2:** 确认断点在处理请求的路由函数内部，并使用浏览器开发者工具 (F12 -> Network) 确认请求的 URL 和方法是否正确。可以在断点前加 `print` 语句验证代码流。
    *   **原因 3:** `justMyCode` 设置为 `true`，而断点设置在了你认为是“自己代码”但被调试器判定为“库代码”（例如通过 `pip install .` 安装的本地项目）的地方。
    *   **解决 3:** 尝试将 `justMyCode` 设置为 `false`。

## 四、 其他 `launch.json` 配置项说明

*   **`"request": "launch"`**: 直接启动并调试脚本/模块（与 `attach` 相对）。
*   **`"stopOnEntry": true`**: 连接/启动成功后立即在第一行暂停。
*   **`"console"`**: (用于 `launch`) 指定程序输出位置 (`internalConsole`, `integratedTerminal`, `externalTerminal`)。
*   **`"env"` / `"envFile"`**: (用于 `launch`) 设置/加载环境变量。
*   **`"args"`**: (用于 `launch`) 传递命令行参数。
*   **`"subProcess": true`**: (实验性) 尝试调试子进程。
*   **`"logToFile": true`**: 将 `debugpy` 自身的日志写入文件，用于排查调试器问题。