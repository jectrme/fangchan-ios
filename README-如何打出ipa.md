# 房产文案 · iOS 打包工程（免费出 ipa）

这个仓库不含后端、不含登录，就是一个把已经做好的本地版网页装进 iOS 原生壳的工程。
它的唯一作用：让 GitHub 的免费 macOS 机器帮你编译出一个 **未签名 .ipa**，
你再用自己的 Apple ID（Sideloadly / 爱思助手 / AltStore 之类工具）签名装到手机上。

- 应用名：房产文案
- Bundle ID：`com.fangchan.wenan`
- 最低系统：iOS 14
- 页面资源：全部在 `www/` 里，安装后完全离线（只有调用 AI 时需要联网）

---

## 第一步：注册 GitHub 并新建仓库

1. 打开 https://github.com 注册（邮箱 + 密码即可，免费）。
2. 右上角 `+` → **New repository**。
3. Repository name 随便填，比如 `fangchan-ios`。
4. 可见性选 **Public**（公开仓库的 macOS 编译额度不扣分钟数；Private 也能用，但会消耗免费额度）。
5. 点 **Create repository**。

## 第二步：把本文件夹的内容传上去

最简单的方式（网页上传）：

1. 解压我给你的 zip，得到 `ios-ipa-repo` 文件夹（里面是 `www/`、`res/`、`capacitor.config.json`、`package.json`、`.github/`）。
2. 在新建的仓库页面点 **uploading an existing file**。
3. 把 `ios-ipa-repo` 里的**全部文件和文件夹**拖进上传区（包括 `.github` 这个文件夹，它是隐藏的，一定要带上）。
4. 点 **Commit changes**。

> 如果网页上传漏掉了 `.github` 文件夹（这会导致没有编译按钮），改用下面任一方式：
> - 在仓库页面点 **Add file → Create new file**，文件名填 `.github/workflows/build-ios.yml`，把本文件末尾附的 YAML 全文粘进去，提交；
> - 或者装 **GitHub Desktop**，把文件夹拖进去点 Publish。

## 第三步：点一下编译

1. 仓库顶部点 **Actions** 标签。
2. 左侧会看到 **Build iOS IPA (unsigned)**，点开。
3. 右边点 **Run workflow** → 绿色按钮 **Run workflow**。
4. 等 10～20 分钟（第一次要下载 Pods，会慢一些）。
5. 全绿之后，点进这次运行，页面底部 **Artifacts** 里下载 `fangchan-wenan-unsigned-ipa.zip`。
6. 解压得到 `fangchan-wenan-unsigned.ipa`——这就是你要的那个文件。

## 第四步：用 Apple ID 签名装到 iPhone

Windows 电脑上（Mac 也行）：

1. 先装好苹果官方驱动：Microsoft Store 里的 **Apple Devices**（或旧版 iTunes），并让电脑认出手机、在手机上点“信任”。
2. 打开签名工具（Sideloadly 最省事；爱思助手的「IPA 签名」也行）。
3. 把 `fangchan-wenan-unsigned.ipa` 拖进去，Apple ID 填你的邮箱，设备选你的 iPhone。
4. 点开始，中途可能要你输入 Apple ID 密码 / 双重验证码，按提示走。
5. 装完第一次打开如果提示「不受信任的开发者」，去 **设置 → 通用 → VPN与设备管理** 里信任你的 Apple ID 证书。
6. iOS 16 以上若提示需要开启开发者模式：**设置 → 隐私与安全性 → 开发者模式** 打开并重启。

## 必须知道的三个限制（免费 Apple ID 的规则，不是我这边加的）

1. **7 天失效**：到期 App 打不开，重新签一次即可（数据一般保留，因为是同一个 Bundle ID 覆盖安装）。
2. **一台设备最多 3 个自签 App**，多出来的要删掉旧的。
3. **不能上架 App Store，也不能发给别人装**（别人的设备没有你的证书）。想发给别人，必须买开发者账号（$99/年）或用 TestFlight。

## 编译失败怎么办

Actions 里那一步变红，点进去看日志，把**第一处红色报错**发我，我改工程再给你。常见几种：

- `No signing certificate`：说明签名没关干净——本工程已强制不签名，若仍出现请把日志给我；
- `pod install` 超时：重跑一次即可（网络抖动）；
- `Bundle ID 冲突`：只是签名阶段的问题，签名时把 Bundle ID 改成 `com.你的名字.wenan` 即可。

---

## 附：workflow 文件全文（备用，手动创建时粘贴）

文件路径：`.github/workflows/build-ios.yml`

```yaml
name: Build iOS IPA (unsigned)

on:
  workflow_dispatch:
  push:
    branches: [ main ]

permissions:
  contents: read

jobs:
  build:
    runs-on: macos-latest
    timeout-minutes: 60
    steps:
      - name: 拉取代码
        uses: actions/checkout@v4

      - name: 安装 Node 20
        uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: 安装依赖
        run: npm install --no-audit --no-fund

      - name: 检查 CocoaPods
        run: pod --version || (sudo gem install cocoapods -v 1.16.2 --no-document)

      - name: 生成 iOS 工程
        run: npx cap add ios

      - name: 同步网页资源
        run: npx cap sync ios

      - name: 安装 Pods
        run: cd ios/App && pod install

      - name: 设置中文应用名
        run: |
          P="ios/App/App/Info.plist"
          /usr/libexec/PlistBuddy -c "Set :CFBundleDisplayName 房产文案" "$P" 2>/dev/null || /usr/libexec/PlistBuddy -c "Add :CFBundleDisplayName string 房产文案" "$P"

      - name: 替换应用图标
        run: cp res/icon-1024.png "ios/App/App/Assets.xcassets/AppIcon.appiconset/AppIcon-512@2x.png"

      - name: 编译（不签名）
        run: |
          cd ios/App
          xcodebuild \
            -workspace App.xcworkspace \
            -scheme App \
            -configuration Release \
            -sdk iphoneos \
            -destination 'generic/platform=iOS' \
            -derivedDataPath ./build \
            CODE_SIGNING_ALLOWED=NO \
            CODE_SIGNING_REQUIRED=NO \
            CODE_SIGN_IDENTITY="" \
            build

      - name: 打包 IPA
        run: |
          APP=$(find ios/App/build/Build/Products/Release-iphoneos -maxdepth 1 -name "*.app" | head -n 1)
          echo "打包的 App：$APP"
          mkdir -p out/Payload
          cp -R "$APP" out/Payload/
          cd out
          zip -qry ../fangchan-wenan-unsigned.ipa Payload
          ls -lh ../fangchan-wenan-unsigned.ipa

      - name: 上传产物
        uses: actions/upload-artifact@v4
        with:
          name: fangchan-wenan-unsigned-ipa
          path: fangchan-wenan-unsigned.ipa
          retention-days: 30
```

---

## 工程里有什么

```
www/                   已构建好的网页（六个页面 + 图标 + 字体）
res/icon-1024.png      iOS 桌面图标
capacitor.config.json  App 配置（bundle id / 名字 / 资源目录）
package.json           只装了 @capacitor/core、@capacitor/ios、@capacitor/cli
.github/workflows/     云端编译脚本
```

想改 App 名字或 Bundle ID，改 `capacitor.config.json`：
`"appId"` 是 Bundle ID（签名时也可覆盖），`"appName"` 保持英文，中文名由 workflow 里的 PlistBuddy 那一步写入。
