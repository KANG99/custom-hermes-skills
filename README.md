# custom-hermes-skills
Hermes实际使用过程中能够稳定产出的自定义skills

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

