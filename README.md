# WeChat Channels Downloader

A lightweight, easy-to-use downloader for macOS and Windows.

## Usage

Download a [release build](https://github.com/ltaoo/wx_channels_download/releases) and **run it as an administrator**. On the first launch, it installs the certificate automatically and then starts the service.

When the terminal displays “代理服务启动成功” (“Proxy service started successfully”), the application is ready.

![Application running](./docs/assets/app_screenshot1.png)

> Certificate installation is skipped if the certificate is already installed.

Open the WeChat desktop client and select the video you want to download. A download button appears in the action bar below the video, as shown:

![Video download button](./docs/assets/screenshot1.png)

If it does not appear, use the floating button at the side or bottom of the page, which provides the same functionality.

| Home recommendations | Video details page |
| --- | --- |
| ![Home recommendations](docs/assets/fixed_btn1.jpg) | ![Video details page](docs/assets/fixed_btn2.jpg) |


Wait for playback to start, pause the video, and click the download button. When the download completes, the downloaded file appears above. The end of the filename indicates the video quality.

![Video downloaded successfully](./docs/assets/screenshot2.png)

By default, the download button downloads the quality currently playing in WeChat Channels, usually the smallest file. Use the dropdown menu to download another quality.


## Development

Start a terminal as an administrator, then run `go run main.go`.

## Packaging

See the `build/build.sh` script.

## Acknowledgments

The frontend decryption implementation is based on
<br>
https://github.com/kanadeblisst00/WechatVideoSniffer2.0
<br>

The backend decryption code comes from
<br>
https://github.com/Hanson/WechatSphDecrypt


## ⚠️ Disclaimer

```text
This is an open-source project.
It is intended solely for technical discussion, learning, and research.
Comply with applicable laws and regulations, and do not use it for illegal purposes.
You are solely responsible for any consequences of misuse.
By downloading and using this project, you acknowledge and agree to these terms.
```
