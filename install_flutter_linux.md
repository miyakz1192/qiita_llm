# flutterをLinux(ubuntsu)にインストールする
## 環境

  Ubuntu 22.04 LTS(Desktop)

## インストール手順

miyakz@flutter:~$ sudo apt update 
miyakz@flutter:~$ sudo snap install flutter --classic
flutter 0+git.2149a81 from Flutter Team✓ installed
miyakz@flutter:~$ 

ここでflutter doctorを実行するとissueが出る。これを放置はできない。

ctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.16.9, on Ubuntu 22.04 LTS 6.5.0-15-generic,
    locale ja_JP.UTF-8)
[✗] Android toolchain - develop for Android devices
    ✗ Unable to locate Android SDK.
      Install Android Studio from:
      https://developer.android.com/studio/index.html
      On first launch it will assist you in installing the Android SDK
      components.
      (or visit https://flutter.dev/docs/get-started/install/linux#android-setup
      for detailed instructions).
      If the Android SDK has been installed to a custom location, please use
      `flutter config --android-sdk` to update to that location.

[✗] Chrome - develop for the web (Cannot find Chrome executable at
    google-chrome)
    ! Cannot find Chrome. Try setting CHROME_EXECUTABLE to a Chrome executable.
[✓] Linux toolchain - develop for Linux desktop
[!] Android Studio (not installed)
[✓] Connected device (1 available)
[✓] Network resources

! Doctor found issues in 3 categories.
miyakz@flutter:~$ 

まず、Android Studioが必要とのことで、これをインストールする。

wget https://redirector.gvt1.com/edgedl/android/studio/ide-zips/2023.1.1.28/android-studio-2023.1.1.28-linux.tar.gz
tar xvfz android-studio-2023.1.1.28-linux.tar.gz 
cd android-studio/
./studio.sh 

studio.shはAndroid Studioを起動するものだが、初回起動時はインストール自体が走る。次回、studio.shを起動するとAndroid Studioが普通に起動してくる。

また、Android cmd line toolsの導入が必要


Android Studio > Preference > Appearance & Behavior > System Settings > Android SDK > SDK Tools

こちらで、SDK Toolsのチェックボックスを入れて、Applyを実行。
画面イメージはこちらが参考になる。

URL:https://nothing-behind.com/2021/09/13/flutter%E3%83%90%E3%83%BC%E3%82%B8%E3%83%A7%E3%83%B3%E3%82%A2%E3%83%83%E3%83%97%E6%99%82%E3%81%AEjava-lang-noclassdeffounderror%E3%81%AE%E5%AF%BE%E5%87%A6%E6%96%B9%E6%B3%95/

画像への直リンク：https://nothing-behind.com/wp-content/uploads/2021/09/%E3%82%B9%E3%82%AF%E3%83%AA%E3%83%BC%E3%83%B3%E3%82%B7%E3%83%A7%E3%83%83%E3%83%88-2021-09-13-9.47.47-1-1024x746.png

Applyを押下して、インストールをして、Android Studioをとりあえず抜ける。

## chromeのインストール。

chromeの正式サイトからdebパッケージを落としてきてdpkgコマンドで入れる。
以下。

   98  sudo dpkg -i google-chrome-stable_current_amd64.deb

結果、OK!

miyakz@flutter:~$ flutter doctor
Doctor summary (to see all details, run flutter doctor -v):
[✓] Flutter (Channel stable, 3.16.9, on Ubuntu 22.04 LTS 6.5.0-15-generic,
    locale ja_JP.UTF-8)
[✓] Android toolchain - develop for Android devices (Android SDK version 34.0.0)
[✓] Chrome - develop for the web
[✓] Linux toolchain - develop for Linux desktop
[✓] Android Studio (version 2023.1)
[✓] Connected device (2 available)
[✓] Network resources

• No issues found!
miyakz@flutter:~$ 









