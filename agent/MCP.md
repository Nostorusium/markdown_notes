# Introduction to MCP

MCP是一个托管协议，当用户提出问题，Agent会先从MCP服务器搜索可用的工具，并转换为一定格式与用户输入一同发送给LLM。

这样引导LLM根据需求返回信息，引导Agent调用本地的工作函数。

MCP的作用是将Agent与工具解耦，Agent和工具可以自由更换。