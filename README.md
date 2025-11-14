# 2025IntelAIPC-Contest
2025IntelAIPC-Contest Log &amp; Summary

参考：
https://www.bilibili.com/video/BV1iKeJzSEMs Intel Graphics Software里设置 共享显存
https://www.bilibili.com/video/BV1PDAteWESE 使用Ollama Intel iGPU/dGPU优化版 github.com/intel/ipex-llm

拿到的AIPC硬件信息如下：
CPU：Ultra7 二代258V
iGPU: Arc-140V(16G)
内存：32GB-8533
<img width="1074" height="255" alt="image" src="https://github.com/user-attachments/assets/54b4336f-421b-43b1-ada6-625833ca92e9" />

Intel Graphics Software里设置共享显存 为24GB
<img width="1730" height="436" alt="image" src="https://github.com/user-attachments/assets/177d6f03-4d2a-4d13-b3ea-2ef3822610cb" />

去ollama.com 网站下载windows版的官方ollama版本

打开PowerShell （以管理员身份运行）
Ollama run qwen3:30b-a3b-thinking-2507-q4_K_M --verbose
测得生成速度为~16tokens/s
<img width="1145" height="755" alt="image" src="https://github.com/user-attachments/assets/e1ff4909-f33f-4858-a653-09dbcf766d7c" />

**去https://github.com/intel/ipex-llm/blob/main/docs/mddocs/Quickstart/ollama_portable_zip_quickstart.md 下载最新的Ollama Intel优化版 （ollama-ipex-llm-2.3.0b20250725-win.zip 对应OllamaV0.9.3，之前版本有可能不支持MoE）**
解压，然后start-ollama.bat中加入“ set OLLAMA_NUM_CTX=16384 ” 来扩大上下文到16K（>单词表9.5K+词汇记忆chunckTOP10*0.25K；缺省Ollama只给2K），最后运行start-ollama.bat，注意在运行前要关闭之前的Ollama官方版本
然后Ollama run qwen3:30b-a3b-thinking-2507-q4_K_M --verbose
测得生成速度为\~20tokens/s，的确Ollama Intel优化版比官方版快了~25%，此时VRAM消耗达到20.9GB
<img width="1141" height="760" alt="image" src="https://github.com/user-attachments/assets/5ccc9a7d-665d-4235-b3a5-bcd9f7e31962" />

打开另一个PowerShell （以管理员身份运行）
ollama pull qwen3-embedding:0.6b-q8_0 （Q8大小700MB，context达32K）
失败，发现官方ollama 运行embedding模型是成功的，ollama run qwen3-embedding:0.6b-q8_0 "hi"。说明可能目前Ollama Intel优化版支持embedding还不够，https://github.com/ollama/ollama/issues/12368 里说HF的模型是OK的
但实际用 ollama create qwen3-embedding-0.6b-q8_0 -f Modelfile 指向HF模型还是不行，所以放弃Ollama Intel优化版！！！

Wait...
后来发现改用 nomic-embed-text 模型就可以了！！！

用下面的命令来下载nomic-embed-text模型 https://ollama.com/library/nomic-embed-text （FP16也只有270MB，Context只有2K）
C:\Ollama4Intel> ./ollama pull nomic-embed-text  

用下面的命令来测试
curl -Uri "http://localhost:11434/api/embed" -Method POST -ContentType "application/json" -Body '{"model":"nomic-embed-text","input":["你好世界","嵌入模型测试"]}'

研究了Embedding 模型的榜单，发现一些对中英文友好且在Ollama直接能用的Embedding模型
https://github.com/embeddings-benchmark/mteb
https://huggingface.co/spaces/mteb/leaderboard

```bash
./ollama pull shaw/dmeta-embedding-zh                1K上下文，409MB，2025年出的，其small版283MB在MTEB中英文排名与Qwen3-0.6b-embedding相当，平均分为66
ollama pull EntropyYue/jina-embeddings-v2-base-zh    8K上下文，322MB，2024年出的，其v3多语言版在MTEB总榜上比Qwen3-0.6b-embedding平均分差6分为58分
C:\Ollama4Intel> ./ollama pull nomic-embed-text      2K上下文，270MB，2024年出的，中英文性能未知
```
根据https://ollama.com/dengcao/Qwen3-Reranker-0.6B 所描述的“截止2025年6月11日，ollama暂不支持重排模型。经测试Ragflow 0.19和Dify不支持ollama的重排模型，添加所有ollama的重排模型时均会提示错误。FastGPT虽然可以添加，但是重排无效。”所以暂时先不安装rerank模型了。

## 2. 安装Dify

### 2.1 安装WSL2的Ubuntu系统

打开Windows Powershell
wsl --install -d Ubuntu

wsl -d Ubuntu 打开并进入Ubuntu系统

### 2.2 安装Docker Compose，参考 https://docs.docker.com/desktop/setup/install/windows-install/#wsl-2-backend

### 2.3 通过WSL2 在Docker Compose中安装和运行 Dify ，参考https://legacy-docs.dify.ai/getting-started/install-self-hosted/docker-compose 

打开WSL终端
sudo su   进入root再运行上面URL中的命令！！！
按照https://legacy-docs.dify.ai/getting-started/install-self-hosted/docker-compose  做有可能再docker compose ps时发现db起不来，看log后发现是permission错误，咨询AI后使用下面命令搞定，只是后面都要进~/apps/dify/docker 启动docker compose up -d
```bash
# 在你的 WSL 发行版（Ubuntu 等）里：
mkdir -p ~/apps
mv /mnt/c/Users/devcloud/dify ~/apps/dify

cd ~/apps/dify/docker
# 清掉容器和卷（会删除数据库数据）
docker compose down -v   (一般情况下不要加-v，这样会删数据库，这次是迁移目录所以要加-v)
docker compose up -d

docker compose ps
```

重启电脑后要
打开 Taskbar上的Docker Desktop
打开 WSL的Ubuntu
sudo su -   （默认passwd= devcloud）
cd ~/apps/dify/docker
docker compose up -d

docker compose ps

http://localhost/install
临时的email用test1@test.com / test12345
登录后配置Dify使用模型时，发现在ollama run qwen3-30b时内存不够了，Task Manager中其实是够的，原因是WSL2虚拟机缺省占用了16G（通过free -h能看到），所以用以下配置再重启WSL2

调整 WSL 配额，在 Windows 编辑 C:\Users\devcloud\.wslconfig（没有就新建）：
```bash
[wsl2]
memory=3GB          # 按你机器内存设置，比如 Dify需要2C4G,但此场景为了省内存所以极限可以到3GB
swap=1GB            # 建议给足，避免峰值 OOM
localhostForwarding=true
```

**另外Dify集成LLM时，要使用WSL虚拟机的宿主机的名字host.docker.internal 而不是127.0.0.1或localhost**
http://host.docker.internal:11434
**注意配置Ollama模型时要把缺省类型从“Completion”改为“Chat”，还要设置context 长度（缺省只有4096）到16384**

# 重启机器后启动所有项目

```bash
打开Windows Powershell
cd C:\Ollama4Intel
./start-ollama.bat

打开 Taskbar上的Docker Desktop
打开Windows Powershell
打开 WSL的Ubuntu
sudo su -   （默认passwd= devcloud）
cd ~/apps/dify/docker
docker-compose up -d    #用docker-compose start 有时表面up了，但浏览器中出不来？？？
docker-compose ps
此时Docker和Dify应该都已经up了，如果没有up，则 docker-compose restart 或者 docker-compose up -d

http://localhost/apps    临时的email用test1@test.com / test12345
另外Dify集成LLM时，要使用WSL虚拟机的宿主机的名字host.docker.internal 而不是127.0.0.1或localhost
http://host.docker.internal:11434
qwen3:30b-a3b-thinking-2507-q4_K_M
qwen3:30b-a3b-instruct-2507-q4_K_M
EntropyYue/jina-embeddings-v2-base-zh
```

# 关机时运行的命令
```bash
docker-compose stop    关闭Dify，而不是docker-compose down，会把docker删除，里面的数据就没了！！！
关闭ollama窗口
```

## 实测效果
- **当提示词含有#工作流程 #输出格式 #输出示例 和 9.5K tokens的《高考词汇表》时；qwen3-30b-Q8（可用） >> qwen3-30b-AWQ4 > qwen3-30b-INT4**
  gpt5mini@gptgod                                                  输出格式和输出内容的RAG效果都很好？？？<BR>
  qwen3-30b-a3b-2507-Q8:ctx32k-mlock@Ollama                        单个词/词根效果好，追问输出格式对但输出内容RAG效果一般？？？，词转题也OK<BR>
  Qwen3-30B-A3B-Thinking-2507-AWQ-4bit@vLLM                        单个词效果好，词转题也OK；输出格式都对，但单个词根和追问输出内容不行；另外可能是vLLM的缘故输出think都显示出来的<BR>
  **qwen3-30b-a3b-thingking-INT4@Ollama Intel优化版+16K上下文      完全没做到指令跟随，说明INT4量化模型对长文本的指令跟随不行！<BR>**
  Qwen3-30BThink-2507-FP8@vLLM                    <BR>
- **当提示词只含有#工作流程 #输出格式 #输出示例，去掉9.5K tokens的《高考词汇表》时**
  **qwen3-30b-a3b-thingking-INT4@Ollama+16K上下文                  单个词效果好，词转题OK，追问时输出格式OK但RAG都不行；输出think都显示出来的<BR>，有时可能因VRAM短暂不够要过10分钟（ollama自动退出模型时间）才能继续问？？？**
  **qwen3-30b-a3b-instruct-INT4@Ollama+16K上下文                   单个词效果好，词转题OK，追问时输出格式OK但RAG都不行;输出无Think，比thinking模型干净实用速度快最多等不到1min模型加载时间**
                                                                  有意思的是词转题在“Instruction"里加个仔细就能做对题！（1、对于。。。或者题目的答案涉及到某一单词需补充学习时（先**仔细**分析题目再把答案单词做为”该单词“进入下面的工作流程）“）

  























