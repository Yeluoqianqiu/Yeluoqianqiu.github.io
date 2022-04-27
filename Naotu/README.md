NaotuWeb支持丰富的导入(txt、km、json、md)导出格式（如md、png、txt、svg、json和km），但不能对打开的文件进行保存，只能下载为新文件。

可以使用vscode扩展eighthundreds的mindmap进行日常使用，支持保存、另存为km，因为是vscode扩展，支持跨平台。

修改样式：打开~/.vscode/extensions/eighthundreds.vscode-mindmap..../kityminder/webui/dist，修改两个css文件中的内容：

```
#previewer-content{...} 修改max-width:600px max-height:400px font-size:13px
#previewer-content blockquote{...} 修改两处color:gray; padding:0 0 0 5px; font-size:12px; font-style:normal; margin:0 0 0 1em;
#previewer-content p{...} 修改 margin:1em 0 0 0;
#previewer-content ol{...} 修改padding:0 0 0 2em; 添加margin:0;
```

