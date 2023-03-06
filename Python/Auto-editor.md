这个软件是用 Python 写的，开源的。试用了一下挺好的，它会自动剪掉没有声音和没有画面动态的地方，并且输出的视频体积小，节约二次剪辑导入导出时间。
## 安装
```bash
pip install auto-editor
```
## 使用
```bash
~/.local/bin/auto-editor video_file_name    //导出为普通视屏

~/.local/bin/auto-editor video_file_name --export_to_premiere    //导出给 Premiere 剪辑软件使用
```
