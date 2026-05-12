# custom-hermes-skills
该项目主要介绍Hermes实际使用过程中总结归纳出的能够稳定产出的自定义skills，以及使用Hermes的部分实践经验。

## skills
- web-docs-to_zh:将网页内容完整翻译成中文md文档，移除部分html标签，格式基本不变。

## usage
以web-docs-to_zh为例，hermes安装skill指令如下：
```bash
hermes skills install KANG99/custom-hermes-skills/skills/web-docs-to-zh
```
理论上使用tab将仓库名称加入Hermes，可以直接指定skill名称进行安装仓库内的其他的skill，但是对自定义skill实际上会出现无法query的问题。
```bash
#添加GitHub仓库
hermes skills tap add KANG99/custom-hermes-skills
#安装仓库内的skill
hermes skills install web-docs-to-zh
```
直接报错无法查找
```
Resolving 'web-docs-to-zh'...
Error: No skill named 'web-docs-to-zh' found in any source.
```
如果需要将skills上传到自定义仓库，指令模版如下（实际上就是利用Hermes提交PR）
```
hermes skills publish your-skill-name --to github --repo owner/repo
```

## ssh远程沙箱
如果需要验证skill及代码安全性，建议使用远程沙箱或者docker运行的方式，以ubuntu作为远程沙箱为例：

**ubuntu端**
```
#查看内网地址，找到类似1[92.168.x.xxx](http://92.168.1.xxx/)这串
ip a
#在 Ubuntu 上新建低权限用户 hermes(自定义)
sudo adduser hermes
#添加安装软件、操作基础命令等基础管理员权限
sudo usermod -aG sudo hermes
#切换到 hermes 用户
sudo su - hermes
#建立 SSH 密钥目录
mkdir -p ~/.ssh
chmod 700 ~/.ssh
```
**本地电脑端**
```
#生成专属 SSH 密钥,全程直接回车，不要设密码
ssh-keygen -t ed25519 -f ~/.ssh/hermes_agent_key
#把密钥传到Ubuntu干活机,将ip替换成ip a获取的地址。提示输密码，输入Ubuntu hermes用户设的密码
ssh-copy-id -i ~/.ssh/hermes_agent_key hermes@192.168.x.xxx
#测试免密链接，本地端会进入远程段
ssh hermes@192.168.x.xxx -i ~/.ssh/hermes_agent_key
#终端退出远程终端
exit
```
**修改Hermes config.yaml文件及.env文件**
- `config.yaml文件`
```
#如果要换回本地模式backend设置为local
terminal:
  backend: ssh
```
- `.env文件`
```
TERMINAL_SSH_HOST=192.168.x.xxx
TERMINAL_SSH_USER=hermes
TERMINAL_SSH_KEY=~/.ssh/hermes_agent_key
```
