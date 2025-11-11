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

去https://github.com/intel/ipex-llm/blob/main/docs/mddocs/Quickstart/ollama_portable_zip_quickstart.md 下载最新的Ollama Intel优化版
解压后运行start-ollama.bat，注意在运行前要关闭之前的Ollama官方版本
然后Ollama run qwen3:30b-a3b-thinking-2507-q4_K_M --verbose
测得生成速度为\~20tokens/s，的确Ollama Intel优化版比官方版快了~25%
<img width="1141" height="760" alt="image" src="https://github.com/user-attachments/assets/5ccc9a7d-665d-4235-b3a5-bcd9f7e31962" />







