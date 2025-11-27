# 2025IntelAIPC-Contest Log &amp; Summary

## 2025阿里端侧AI创新挑战赛参赛文章见： https://modelscope.cn/learn/2788 和 https://modelscope.cn/competition/145/talkArea 讨论区

## 0. 拿到的AIPC硬件信息如下：
CPU：Ultra7 二代258V
iGPU: Arc-140V(16G)
内存：32GB-8533
<img width="1074" height="255" alt="image" src="https://github.com/user-attachments/assets/54b4336f-421b-43b1-ada6-625833ca92e9" />

Intel Graphics Software里设置共享显存 为24GB；参考https://www.bilibili.com/video/BV1iKeJzSEMs Intel Graphics Software里设置共享显存
<img width="1730" height="436" alt="image" src="https://github.com/user-attachments/assets/177d6f03-4d2a-4d13-b3ea-2ef3822610cb" />

## 1. 安装Ollama Intel优化版，参考 https://www.bilibili.com/video/BV1PDAteWESE 使用Ollama Intel iGPU/dGPU优化版 github.com/intel/ipex-llm
### 1.1 去ollama.com 官网下载和安装windows版
打开PowerShell （以管理员身份运行）
Ollama run qwen3:30b-a3b-thinking-2507-q4_K_M --verbose   <BR>
**测得生成速度为~16tokens/s  <BR>**
<img width="1145" height="755" alt="image" src="https://github.com/user-attachments/assets/e1ff4909-f33f-4858-a653-09dbcf766d7c" />

### 1.2 下载和安装最新的Ollama Intel优化版
去https://github.com/intel/ipex-llm/blob/main/docs/mddocs/Quickstart/ollama_portable_zip_quickstart.md 下载最新的Ollama Intel优化版 （ollama-ipex-llm-2.3.0b20250725-win.zip 对应OllamaV0.9.3，之前版本有可能不支持MoE）；<BR>
解压，然后start-ollama.bat中加入“ set OLLAMA_NUM_CTX=16384 ” 来扩大上下文到16K（>单词表9.5K+词汇记忆chunckTOP10*0.25K；缺省Ollama只给2K），最后运行start-ollama.bat，注意在运行前要关闭之前的Ollama官方版本<BR>

### 1.3 下载和运行LLM、Embedding和Rerank（最后由于Ollama目前还不支持Rerank放弃）模型
然后Ollama run qwen3:30b-a3b-thinking-2507-q4_K_M --verbose <BR>
**测得生成速度为\~20tokens/s，的确Ollama Intel优化版比官方版快了~25%，此时VRAM消耗达到20.9GB<BR>**
<img width="1141" height="760" alt="image" src="https://github.com/user-attachments/assets/5ccc9a7d-665d-4235-b3a5-bcd9f7e31962" />

打开另一个PowerShell （以管理员身份运行）<BR>
ollama pull qwen3-embedding:0.6b-q8_0 （Q8大小700MB，context达32K） <BR>
**失败，发现官方ollama 运行embedding模型是成功的，ollama run qwen3-embedding:0.6b-q8_0 "hi"。说明可能目前Ollama Intel优化版支持embedding还不够**，https://github.com/ollama/ollama/issues/12368 里说HF的模型是OK的  <BR>
但实际用 ollama create qwen3-embedding-0.6b-q8_0 -f Modelfile 指向HF模型还是不行，所以放弃Ollama Intel优化版！！！ <BR>

Wait... 后来发现改用 nomic-embed-text 模型就可以了！！！ <BR>

用下面的命令来下载nomic-embed-text模型 https://ollama.com/library/nomic-embed-text （FP16也只有270MB，Context只有2K） <BR>
C:\Ollama4Intel> ./ollama pull nomic-embed-text   <BR>

用下面的命令来测试<BR>
curl -Uri "http://localhost:11434/api/embed" -Method POST -ContentType "application/json" -Body '{"model":"nomic-embed-text","input":["你好世界","嵌入模型测试"]}'  <BR>

**研究了Embedding 模型的榜单，发现一些对中英文友好且在Ollama直接能用的Embedding模型 <BR>
https://github.com/embeddings-benchmark/mteb
https://huggingface.co/spaces/mteb/leaderboard**

```bash
./ollama pull shaw/dmeta-embedding-zh                1K上下文，409MB，2025年出的，其small版283MB在MTEB中英文排名与Qwen3-0.6b-embedding相当，平均分为66
ollama pull EntropyYue/jina-embeddings-v2-base-zh    8K上下文，322MB，2024年出的，其v3多语言版在MTEB总榜上比Qwen3-0.6b-embedding平均分差6分为58分
C:\Ollama4Intel> ./ollama pull nomic-embed-text      2K上下文，270MB，2024年出的，中英文性能未知
```
**根据https://ollama.com/dengcao/Qwen3-Reranker-0.6B 所描述的“截止2025年6月11日，ollama暂不支持重排模型。经测试Ragflow 0.19和Dify不支持ollama的重排模型，添加所有ollama的重排模型时均会提示错误。FastGPT虽然可以添加，但是重排无效。”所以暂时先不安装rerank模型了。**

## 2. 安装Dify

### 2.1 安装WSL2的Ubuntu系统

打开Windows Powershell，运行 wsl --install -d Ubuntu <BR>
wsl -d Ubuntu 打开并进入Ubuntu系统 <BR>

### 2.2 安装Docker Compose，参考 https://docs.docker.com/desktop/setup/install/windows-install/#wsl-2-backend

### 2.3 通过WSL2 在Docker Compose中安装和运行 Dify ，参考https://legacy-docs.dify.ai/getting-started/install-self-hosted/docker-compose 

打开WSL终端 <BR>
sudo su   进入root再运行上面URL中的命令！！！ <BR>
按照https://legacy-docs.dify.ai/getting-started/install-self-hosted/docker-compose  做有可能再docker compose ps时发现db起不来，看log后发现是permission错误，咨询AI后使用下面命令搞定，只是后面都要进~/apps/dify/docker 启动docker compose up -d <BR>
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

**重启电脑后要**<BR>
打开 Taskbar上的Docker Desktop<BR>
打开 WSL的Ubuntu<BR>
sudo su -   （默认passwd= devcloud）<BR>
cd ~/apps/dify/docker<BR>
docker compose up -d<BR>

docker compose ps<BR>

http://localhost/install  <BR>
临时的email用test1@test.com / test12345  <BR>
登录后配置Dify使用模型时，发现在ollama run qwen3-30b时内存不够了，Task Manager中其实是够的，原因是WSL2虚拟机缺省占用了16G（通过free -h能看到），所以用以下配置再重启WSL2  <BR>

调整 WSL 配额，在 Windows 编辑 C:\Users\devcloud\.wslconfig（没有就新建）：<BR>
```bash
[wsl2]
memory=3GB          # 按你机器内存设置，比如 Dify需要2C4G,但此场景为了省内存所以极限可以到3GB
swap=1GB            # 建议给足，避免峰值 OOM
localhostForwarding=true
```

**另外Dify集成LLM时，要使用WSL虚拟机的宿主机的名字host.docker.internal 而不是127.0.0.1或localhost** <BR>
http://host.docker.internal:11434    <BR>
**注意配置Ollama模型时要把缺省类型从“Completion”改为“Chat”，还要设置context 长度（缺省只有4096）到16384** <BR>

## 3. 重启机器后启动所有项目

```bash
打开Windows Powershell
cd C:\Ollama4Intel
./start-ollama.bat

打开 Taskbar上的Docker Desktop
打开Windows Powershell
打开 WSL的Ubuntu
sudo su -   （默认passwd= devcloud）
cd ~/apps/dify/docker
docker-compose up  -d # 🟥重启Ubuntu后有些服务自动重启了，但Difyweb服务还没有起，所以要用这个命令彻底启动Dify；有时会碰到Dify无原来的配置数据了，这时要么等足够时间要么用docker-compose restart重启有时就好了！！！
docker-compose ps
此时Docker和Dify应该都已经up了，如果没有up（有时表面up了，但浏览器中出不来？？？），则 docker-compose restart 或者 docker-compose up -d 

http://localhost/apps    临时的email用test1@test.com / test12345
另外Dify集成LLM时，要使用WSL虚拟机的宿主机的名字host.docker.internal 而不是127.0.0.1或localhost
http://host.docker.internal:11434
qwen3:30b-a3b-instruct-2507-q4_K_M
shaw/dmeta-embedding-zh

EntropyYue/jina-embeddings-v2-base-zh
qwen3:30b-a3b-thinking-2507-q4_K_M
```

## 4. 关机时运行的命令
```bash
cd ~/apps/dify/docker
docker-compose stop    关闭Dify，而不是docker-compose down，会把docker删除，里面的数据就没了？？？
关闭ollama窗口
```

## 5. 实测效果 和 🔴结论
- **当提示词含有#工作流程 #输出格式 #输出示例 和 9.5K tokens的《高考词汇表》时；qwen3-30b-Q8（可用） >> qwen3-30b-AWQ4 > qwen3-30b-INT4；EntropyYue/jina-embeddings-v2-base-zh（embed效果不如qwen3-0.6b）**
  gpt5mini@gptgod                                                  输出格式和输出内容的RAG效果都很好？？？<BR>
  qwen3-30b-a3b-2507-Q8:ctx32k-mlock@Ollama                        单个词/词根效果好，追问输出格式对但输出内容RAG效果一般？？？，词转题也OK<BR>
  Qwen3-30B-A3B-Thinking-2507-AWQ-4bit@vLLM                        单个词效果好，词转题也OK；输出格式都对，但单个词根和追问输出内容不行；另外可能是vLLM的缘故输出think都显示出来的<BR>
  **qwen3-30b-a3b-thingking-INT4@Ollama Intel优化版+16K上下文      完全没做到指令跟随，说明INT4量化模型对长文本的指令跟随不行！<BR>**
  Qwen3-30BThink-2507-FP8@vLLM                    <BR>
- **当提示词只含有#工作流程 #输出格式 #输出示例，去掉9.5K tokens的《高考词汇表》时；EntropyYue/jina-embeddings-v2-base-zh（embed效果不如qwen3-0.6b）**
  **qwen3-30b-a3b-thingking-INT4@Ollama+16K上下文                  单个词、词转题效果好，追问时输出格式OK但RAG都不行；输出think都显示出来的<BR>，有时可能因VRAM短暂不够要过10分钟（ollama自动退出模型时间）才能继续问？？？**<BR>
  **qwen3-30b-a3b-instruct-INT4@Ollama+16K上下文                   单个词、词转题效果好，追问时输出格式OK但RAG都不行;输出无Think，比thinking模型干净实用速度快最多等不到1min模型加载时间**<BR>
                                                                  有意思的是词转题在“Instruction"里加个仔细就能做对题！（1、对于。。。或者题目的答案涉及到某一单词需补充学习时（先**仔细**分析题目再把答案单词做为”该单词“进入下面的工作流程）“）<BR>
- 🔴**当提示词只含有#工作流程 #输出格式 #输出示例，去掉9.5K tokens的《高考词汇表》时；shaw/dmeta-embedding-zh（embed效果接近qwen3-0.6b，果然后面的RAG召回比jina要好！！！）**<BR>
 🔴 **qwen3-30b-a3b-instruct-INT4@Ollama+16K上下文         单个词、词转题效果好，追问（cough拟声词/cess-为词根词/escape同根词）时输出格式和RAG召回都OK了！目前应用层面上无rerank模型时的最好效果，速度也不错**<BR>
- ”cess-为词根的单词有哪些？“这样问就能在TOP10中召回；”cess为词根的单词有哪些？”这样问就不能在TOP10内召回；所以对于智商勉强够的小模型，问题的文本差异对回答质量还有有一定关系的      <BR>                                                        
Tips：<BR>
- 因为偶尔会出现VRAM不够的情况，所以把ollama的模型上下文进一步减小到12000（减少300-400MB显存应该正好够了，因为以前只有当连续发问embed和chat模型一起被调用时才偶尔报VRAM不够）
- Dify 最好结果的Backup配置如下 <BR>
**Dify 知识库配置：《___高考词汇-记忆技巧AI校对版V3-Pub按类别-词根-词缀-合成词分割》.txt；分隔符“=====”、每块1Ktokens/500t重叠、用shaw/dmeta-embedding-zh嵌入、混合检索（权重关键词1.0语义0.0）TOP10** <BR>
**chatbot召回设置成混合检索：权重关键词1.0语义0.0（ 实践下来效果最好，根据https://www.bilibili.com/video/BV1AUCFYpEis 的理解关键词不是简单的CTRL+F，而是基于TF-IDF(词频逆文档频率）算法，即根据每个词在文本里的重要性来为文本块和Query计算高维向量然后再计算余弦相似度看是匹配程度，实践下来是最适合我们场景的配置）TOP10** <BR>
**Dify chatbot的提示词如下：** <BR>
```bash
# 本应用名称：高考单词王

# 本应用功能：
通过检索《Context》来学习和复习高考英语单词。不回答任何和英语学习无关的问题

#工作流程
1、对于给定的单词、词组或者题目的答案涉及到某一单词需补充学习时（先仔细分析题目再把答案单词做为”该单词“进入下面的工作流程）
1.1、从《Context》中查其记忆技巧包括合成词、构词分析、词源分析，如果查到该单词就使用粗体输出并输出关于这个单词的信息，有该单词的构词分析和词源分析信息就输出。
1.2、最后为这个单词的每一个词性和主要意思编写一句带有该单词的高考难度的简单例句并翻译，并且例句中这个单词用粗体显示

# 输出格式始终用MD格式，除必要外其他用于都始终用中文
# 输出示例1：(给定一个单词或词组）
**necessary：['nesa,seri] adj.必要的；必需的**
**【构词分析】**
necessary=ne（不）+cess（走开）+ary（形容词后缀）  →  不能离开的 →    必需的，必要的**
**【主要用法的例句】**
Hard work is **necessary** if you want to pass the exam. 如果你想通过考试，努力是**必要的**。
# 输出示例2：(给定一道答案为单个单词的题目）
**【原题】The fact that I didn't have enough experience was really a big ________ .(advantage)**
【解析】括号中给出的是 advantage（优势），但根据句意，“我没有足够经验”显然是一个不利条件，所以这里不能填“advantage”（优势），而应该填它的反义词。advantage（名词，优势）的反义词是 disadvantage（名词，劣势）,前缀 “dis-” 表示否定或相反含义，常用于构成反义词。原题的中文翻译是“我没有足够经验这一事实确实是一个很大的劣势。”
**【答案】disadvantage** [disəd'veantud3]n.缺点，劣势
**【构词分析】**
disadvantage=dis（反义）+ advantage（ 优点，优势）→ 缺点，劣势
**【主要用法的例句】**
One **disadvantage** of living in the city is the high cost of housing. 住在城市的一个**缺点**是房价高昂。
His lack of experience was a serious **disadvantage** in the job interview.  他缺乏经验在面试中是个严重**劣势**。
```                                                
**Dify chatbot的开场白如下：** <BR>
```bash
欢迎使用《高考单词王》通过词根词缀和词源故事以及精彩例句学习高考单词！请输入你的单词，例如escape
注：
1、本应用关于高考单词有关信息均来自网络和AI生成，仅供学习和参考。
2、本应用支持多轮追问，例如可以追问“和cough一样的拟声词高考里还有哪些？”、”和necessary有相同词缀的高考词汇还有哪些？“
3、本应用甚至还可以帮助你分析”词性转换“真题

escape
帮我分析下面这道词性转换真题：It's our family (_____) to exchange gifts on New Year's Eve. (traditional)
cess-为词根的单词有哪些？
和escape同词根的高考词汇还有哪些？
和cough一样的拟声词高考里还有哪些？
```
- **🔴目前Ollama占用~21GB显存 + Dify/Docker@WSL虚拟机上的3-4GB内存 + Windows开销3-4G已经达到28-29GB，占总内存的87%以上了且偶有OOM内存不够发生；可用说已经精打细算已是本次系统硬件的极限!**
  Ollama显存需求 = qwen3-30b-a3b-instruct-2507-Q4模型19GB + 12000上下文的KV Cache~1.2GB + Embedding模型0.4GB + Overhead开销0.XGB

## DEMO

  ![cough追问录屏 00_00_00-00_00_30~1](https://github.com/user-attachments/assets/c4469278-6807-48f7-bbda-700b75c7aba2)
  ![escape动画GIF](https://github.com/user-attachments/assets/f49d5e6c-b69d-4381-8e5a-46a65f58d3f3)
  ![tradition词转题动画GIF](https://github.com/user-attachments/assets/2002d36e-403a-4d0e-b0d1-02587b5df2aa)
  <img width="1870" height="769" alt="高考单词王后台截图" src="https://github.com/user-attachments/assets/7b58a65e-ea97-4f92-9951-ca46861af61a" />

## 6. 本地服务免费转公网（内网穿透）服务https://sunfast-malcolm-extemporarily.ngrok-free.dev/

🔴试了cpolar、Cloudflare Tunnel，最后还是ngrok搞定的！！！<BR>

```bash
winget install -e --id Ngrok.Ngrok

然后在 ngrok 注册账号（139邮箱不行,本次用的是outlook邮箱），在控制台(https://dashboard.ngrok.com/authtokens)Add并拿到 Authtoken，然后执行：
ngrok config add-authtoken <你的_authtoken>

最后
ngrok http 80
ngrok tcp 3389
```
<img width="1890" height="470" alt="image" src="https://github.com/user-attachments/assets/d4f02da2-8b63-4030-a704-6c8aae695a2a" />

第一次访问https://sunfast-malcolm-extemporarily.ngrok-free.dev/ 会得到下面这个页面<BR>
<img width="1133" height="656" alt="image" src="https://github.com/user-attachments/assets/0ef8cd72-32dc-470c-9243-8bd26b333c23" />


- Cloudflare Tunnel：临时隧道最简单，**下面一句话就能搞定，无需账号且只要这个进程在临时域名就有效，适合HTTP/HTTPS场景，但在本场景下云环境封了outgoing到7844端口的访问**，所以不行！
  cloudflared tunnel --protocol http2 --url http://localhost:80
- https://www.cpolar.com/download：国内厂商，也比较简单，注册账号后登录http://localhost:9200(win下）/cpolar authtoken xxxxxxx然后cpolar http <本地Dify Web端口=80>即可（Linux下）,**免费支持HTTP/HTTPS/TCP（含RDP等，同时支持4个隧道，1Mbps,_但有效期只有24小时_）场景，但不支持UDP场景，按理和ngrok一样只需要环境不封outgoing到443/80端口访问即可，但本场景下就是没搞定，莫非本次是使用US的云环境而cpolar更适合国内环境（139邮箱注册）？？？**
- **ngrok.com ：国外厂商，也比较简单且更强大，注册账号后，免费支持HTTP/HTTPS场景且永久XX.grok-free.dev域名（同时1个隧道1GB流量），_TCP隧道场景（需绑定信用卡）_，但不支持UDP场景，且只需要环境不封outgoing到443/80端口访问即可，所以通用性很强！！！**

- 自建型：FRP、Inlets（需要你有一台云服务器当“出口”）
FRP 支持 TCP/UDP/HTTP/HTTPS，能把 3389 这种 RDP 端口映射到你云主机的公网口，适合长期自控；但需要自己部署 frps（服务端）+ frpc（客户端）。​
Inlets 也是自建隧道方案（更 Cloud-Native），自己掌控数据与出口，同样需要有一台带公网 IP 的“出口”实例。​
DOCS.INLETS.DEV







