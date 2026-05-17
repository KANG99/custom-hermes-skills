# custom-hermes-skills
该项目主要介绍Hermes实际使用过程中总结归纳出的能够稳定产出的自定义skills，以及使用Hermes的部分实践经验。

## skills
- web-docs-to_zh:将网页内容完整翻译成中文md文档，移除部分无效html标签，内容及主要格式保持不变。
- resume_creator:用来生成符合reactive-resume开源平台的json格式数据，最终将导出json格式的简历及PDF格式的简历

```diff
-ps:实际上resume-creator可以和opencli skill结合使用，查询用户想要找到的工作信息，然后再结合用户工作经历生成符合个性化的简历，这部分内容还在创建中…………
```

## usage
### web-docs-to_zh
hermes安装skill指令如下：
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
prompt
```
请将https://xxx.xxx/...翻译成中文md文档，并且保存在xxx
```
### resume-creator
实际上resume-reactive项目已经提供了[resume-builder](https://github.com/amruthpillai/reactive-resume/tree/main/skills/resume-builder)这个技能，但是存在以下缺陷：
- 该技能是通过询问的方式创建简历的，需要一步步按照该技能提示输入内容
- 最终生成json文件，还需要手动导入到reactive-resume服务端渲染生成简历
- 检验生成的json文件只提供了schema.json及部分示例，生成的json文件基本不可用
- 即使生成的结果会校验检查，但是无法检查出实际出错的字段
resume-creator完成了简历的一站式生成，确保正常简历输出：
- 直接提供完整的个人信息，不通过询问方式生成简历json数据
- 提供简历Ditgar、Chikorita、Azurill模版的json数据
- 通过sh脚本验证json的schema字段是否合规
- 通过访问reactive resume提供的API接口对生成的数据进行导入reactive resume平台验证
- 成功导入数据后，会将简历以PDF格式导出保存在桌面上
这个skill会用到比较多的终端指令，Hermes的安全性是由llm去审核的，比较严格，需要用户审批，所以运行这个skill最好在docker运行绕过

## ssh远程沙箱
如果需要验证skill及代码安全性，建议使用远程沙箱或者docker运行的方式，以ubuntu作为远程沙箱为例：

**ubuntu端**
```
#查看内网地址，找到类似http://192.168.xxx.xxx这串
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
