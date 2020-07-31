---
title: Upload 组件
category:
  - 前端
  - antd
tags:
  - 前端
  - antd
  - validator
keywords: '组件库,antd'
abbrlink: 100830c9
date: 2020-05-16 06:00:00
updated: 2020-05-16 06:00:00
---

Upload 组件构成如下：

* [rc-upload](https://github.com/react-component/upload) 提供基本的文件上传功能。
* Upload 组件在 rc-upload 基础上创建外观，实现文件与表单交互，并集成 UploadList 文件列表。

### rc-upload

rc-upload 的实现逻辑基于：由隐藏的 input 元素获取上传文件，再通过 ajax 上传 FormData 数据。上传时包含常规的 ajax 功能点以及 input 表单控件的功能点。ajax 功能点为：设置 withCredentials 跨域、headers 请求头、method 请求方式、data 附带属性；处理如 onProgress 进度、onSuccess 成功、onError 错误事件，abort 中断操作等。input 表单控件的功能点为：accept 支持上传的文件类型；multiple 是否允许多文件上传。

除此而外，rc-upload 支持拖拽上传、文件夹上传。

拖拽上传时，rc-upload 会基于 props.accept 属性过滤文件类型，因为拖拽上传不会经由 input 控件进行交互。同样一个原因，上传文件后的返回值若想作为表单数据，input 既存储了文件内容，又不能全然涵盖拖拽上传的情景，所以仍需通过 beforeUpload、onSuccess、onError、onStart、onProgress 等事件将返回值透出外围。

文件夹上传借助了 [DataTransferItem.webkitGetAsEntry 方法](https://developer.mozilla.org/zh-CN/docs/Web/API/DataTransferItem/webkitGetAsEntry)，以此遍历文件并上传。

### Upload 组件

如上文所述，Upload 组件实现了 beforeUpload、onSuccess、onError、onStart、onProgress 等方法以与表单完成交互。每次负责与表单交互的 onChange 事件传递数据内容为 { file, fileList } 对象，file 当前正在上传的文件，fileList 处理中的文件列表。因此与 ant design 表单进行交互时，getValueFromEvent 方法用于获取 fileList 存入表单数据中；valuePropName = ‘fileList’ 用于将表单数据注入到 props.fileList 中。

* beforeUpload：校验或剔除上传文件（操控 info.fileList），阻止提交。
* onStart：更新上传文件状态为 uploading。
* onProgress：更新上传文件的上传进度。
* onSuccess：更新上传文件状态为 done。
* onError：更新上传文件状态为 error，即上传失败的文件仍会存于表单数据中，如果想剔除，需要通过 props.onChange 方法实现。

Upload 组件对于拖拽上传的处理，只是切换外观。

除了基本的上传功能外，Upload 组件内还集成了 UploadList 文件列表展示功能。UploadList 可用于预览、下载、删除文件；它借助 Animate 组件实现动效。

* 缩略图展示、文件预览：文件经 props.previewFile 处理后获得 file.thumbUrl，thumbUrl 以高于 url 的优先级作为图片缩略图、预览文件展示。props.onPreview 用于设置预览规则。默认的图片预览通过 canvas 接口将图片内容转成 url 实现，参见 [previewImage 函数](https://github.com/ant-design/ant-design/blob/4.2.2/components/upload/utils.tsx#L71)。
* 文件下载：如果提供了 props.onDownload，使用该方法下载文件；否则，使用 window.open 打开 file.url。
* 文件删除：移除 fileList 中被删除的文件，并使用 rc-upload 中的 abort 机制中断文件上传操作。