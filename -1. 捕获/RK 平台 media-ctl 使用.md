
**media-ctl** 工具的操作是通过 /dev/medio0 等 media 设备，它管理的是 Media 拓扑结构中各个节点的 format、大小、链接。

**v4l2-ctl** 工具则是针对 /dev/video0，/dev/video1 等 video 设备，它在 video 设备上进行 set_fmt、reqbuf、qbuf、dqbuf、stream_on、stream_off 等一系列操作。

以下是几种使用方法：

1. 打印拓扑结构：
```bash
media-ctl -p -d /dev/media0
```
注：isp2 的设备节点较多，可能存在 media0/media1/media2 节点 ，需要逐个枚举查看设备信息。

2. 链接
```bash
media-ctl -l '"rkisp-isp-subdev":2->"rkisp-bridge-ispp":0[0]'
media-ctl -l '"rkisp-isp-subdev":2->"rkisp_mainpath":0[1]'
```

注： 把 ispp 的通路断开，链接到 main_path，从main_path 抓取 raw 图，media-ctl 没加-d指定设备，默认是 /dev/media0 设备，需要确认 rkisp-isp-subdev 挂在在哪个设备节点上，一般是 /dev/media1。

3. 修改 fmt/size
```bash
media-ctl -d /dev/media0 \  
	--set-v4l2 '"ov5695 7-0036":0[fmt:SBGGR10_1X10/640x480]'
```
注：需要确认 camera 设备节点（ov5695 7-0036）挂载在哪个media设备。

4. 设置 fmt 并抓帧
```bash
v4l2-ctl -d /dev/video0 \  
	--set-fmt-video=width=720,height=480,pixelformat=NV12 \  
	--stream-mmap=3 \  
	--stream-skip=3 \  
	--stream-to=/tmp/cif.out \  
	--stream-count=1 \  
	--stream-poll
```

5. 设置曝光、gain 等 control
```bash
v4l2-ctl -d /dev/video3 --set-ctrl 'exposure=1216,analogue_gain=10'
```
注： isp 驱动会调用 camera 子设备的 control 命令，所以指定设备为 video3（main_path or self_path）可以设置到曝光，vicap 不会去调用 camera 子设备的 control 命令，对采集节点直接设置 control 命令会失败。正确做法是找到 camera 设备节点是 /dev/v4l-subdevX，对终端节点直接配置。

