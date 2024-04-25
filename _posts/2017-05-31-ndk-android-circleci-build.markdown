---
layout: post
title: "Android NDK project 用 Circleci 2.0 自動出 build"
date: 2017-05-31 00:00:00 +0800
categories: cantonese
image: /assets/images/2017-05-31-ndk-android-circleci-build/1.png
tags: android ndk devop circleci
---

Automated build of Android NDK native app in CircleCI 2.0

Originally posted on [Lakoo's medium](https://m.lakoo.com/android-ndk-project-%E7%94%A8-circleci-2-0-%E8%87%AA%E5%8B%95%E5%87%BA-build-ddfc48ff1119) on 2017-05-31

tl;dr 如果你咁啱都係用 compiled SDK 25，NDK 14b，build tool 25.03 嘅話，下面嘅 image+ config 可以直接攞去試用

{% highlight yaml %}
jobs:
build:
docker: - image: lakoo/android-ndk:25-25.0.3-r14b
working_directory: ~/app
environment:
TERM: dumb
steps: - checkout - run:
name: Assemble Stable Release
command: ./gradlew assembleStableRelease - store_artifacts:
path: app/build/outputs/apk/
destination: apks/
{% endhighlight %}

正文：

話説因爲前人對於效能嘅奇怪執着，Teon 嘅 Android 客戶端主要 code base 都係由 C 配合 JNI 組成，用 NDK compile 做原生嘅 .so 檔。Develop 嘅時候依靠 Android Studio 都叫勉強可以簡單 setup 到個開發環境，但係打算嘗試 CI 嘅時候，一係就 compile 嘅速度龜到暈，一係就唔係少哩樣就係少個樣，根本控制唔到裝咗咩，加上 NDK 嘅 toolchain 成 GB 咁大，build time 簡直係惡夢。

咁啱最近 circleci 出咗 2.0 版本，一次過解決曬所有問題：
1。 支援自製嘅 docker base image 作爲 ci 底，用啲咩工具自己係個 Dockerfile 加減就得，唔使晒時間下載無用嘅嘢，亦唔使等十世都先等到常用嘅工具加入 CI
2。 行 docker， compile clang 快到暈
3。 circleci 2.0 嘅反應快過舊版，重複類似嘅 build 都可以閒返唔少等 init vm 同 per step setup 嘅時間

假設你已經有個 github repo，簡單講下點樣可以 setup：

1。首先預備 compile android ndk 需要嘅 base image，以目前 Teon 用緊嘅
https://hub.docker.com/r/lakoo/android-ndk 做例子，

{% highlight docker %}
FROM openjdk:8-jdk
...
RUN cd /opt/android-sdk-linux && \ wget -q — output-document=sdk-tools.zip https://dl.google.com/android/repository/sdk-tools-linux-3859397.zip && \ unzip sdk-tools.zip && \ rm -f sdk-tools.zip && \ echo y | sdkmanager “build-tools;25.0.3” “platforms;android-25” && \ echo y | sdkmanager “extras;android;m2repository” “extras;google;m2repository” “extras;google;google_play_services”
RUN wget -q — output-document=android-ndk.zip https://dl.google.com/android/repository/android-ndk-r14b-linux-x86_64.zip && \ unzip android-ndk.zip && \ rm -f android-ndk.zip && \ mv android-ndk-r14b android-ndk-linux
{% endhighlight %}

Dockerfile 只安裝咗最低限度需要嘅 build tools，sdk，ndk 同 support library/google service 嘅 repo
留意用咗熱門嘅 library image openjdk:8-jdk 做底，可以令到 docker pull 嘅時候共享其它人嘅 cache 加快速度
另外就係同一個 docker image 如果 build machine 有 pull 過嘅都會有 cache 免去 download 步驟，所以上面嘅 image 啱使嘅話不妨攞去用，大家方便！

2。寫返啱個 config.yml，係 repo root 開個叫 .circleci 嘅資料夾放入去

{% highlight yaml %}
jobs:
build:
docker: - image: lakoo/android-ndk:25-25.0.3-r14b
working_directory: ~/app
environment:
TERM: dumb
steps: - checkout - run:
name: Assemble Stable Release
command: ./gradlew assembleStableRelease - store_artifacts:
path: app/build/outputs/apk/
destination: apks/
{% endhighlight %}

會重複不斷 build 嘅可以考慮加個 step cache 返 gradle 管理嘅 library

{% highlight yaml %}

- save_cache:
  key: teonclient-{{ checksum "build.gradle" }}-{{ checksum "app/build.gradle" }}
  paths: - "~/.gradle" - "~/.m2"
  restore 只要係 checkout 候加返同一個 key 嘅 restore 就得
- restore_cache:
  key: teonclient-{{ checksum "build.gradle" }}-{{ checksum "app/build.gradle" }}
  {% endhighlight %}

3。登入 circleci，揀返啱個 github project 開始第一個 build，成功之後嘅成品 apk 係 artifact 到下載得返

![成功嘅話 apk 就會係 artifacts 出現](/assets/images/2017-05-31-ndk-android-circleci-build/1.png)
成功嘅話 apk 就會係 artifacts 出現

同樣道理，proguard mapping 同 symbol 都可以用 step 儲成 artifact，但要留意 artifact 並唔係俾用家作永久保存用，放一排之後有可能會俾人 del！

4。有需要可以去 project setting 設定埋 Build forked pull requests，咁每個 pull request 就會自動觸發 circleci ，結果會自動係返 github 顯示

![private repo 嘅 Build forked pull requests 要手動開啟](/assets/images/2017-05-31-ndk-android-circleci-build/2.png)
private repo 嘅 Build forked pull requests 要手動開啟

![成功就會出綠剔](/assets/images/2017-05-31-ndk-android-circleci-build/3.png)
成功就會出綠剔

小結：目前 Teon 用咗 circleci 自動 build PR 後，無論係技術同事 review PR 定係 QA 同事做 feature QA 都方便唔少。其實 circleci 使用環境參數配合 gradle 嘅 build variant 設定可以做到配合唔同 workflow 嘅更多變化，有機會再寫
