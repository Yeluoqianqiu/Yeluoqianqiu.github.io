NaotuWeb支持丰富的导入(txt、km、json、md)导出格式（如md、png、txt、svg、json和km），但不能对打开的文件进行保存，只能下载为新文件。

DesktopNaotu跨平台，可以新建文件、打开已有文件并保存修改，但只能导出为km。

一般使用DesktopNaotu，需要转为md时使用NaotuWeb。



DesktopNaotu style修改：

```
npm i -g asar
//解压asar
asar extract ./resource/app.asar  ./app
//修改 ./app/dist/style/vendor.css
#previewer-content{...} 修改max-width:600px max-height:400px font-size:13px
#previewer-content blockquote{...} 修改两处color:gray; padding:0 0 0 5px; font-size:12px; font-style:normal; margin:0 0 0 1em;
#previewer-content p{...} 修改 margin:1em 0 0 0;
#previewer-content ol{...} 修改padding:0 0 0 2em; 添加margin:0;
//打包asar
asar pack ./app app.asar
```

