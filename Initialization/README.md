# Kernel initialization process

ここでは、kernel展開後の最初のステップから、kernelが自身で最初のプロセスを実行するまでの、kernelの初期化に関する全サイクルを説明します。

*注意* ここでは、kernelの全ての初期化ステップを説明しているわけではなく、割り込み処理やACPI、その他多くの部分を除く、汎用的なパートしか説明していません。これら欠けているパートに関しては、他の章にて後々説明します。

* [kernel展開後の最初のステップ](linux-initialization-1.md) - kernelの最初のステップを説明します。
* [早期割り込み(early interrupt)と例外処理](linux-initialization-2.md) - 早期割り込みの初期化と、早期ページフォルトハンドラ(early page fault handler)を説明します。
* [kernelのエントリポイント前の最後の準備](linux-initialization-3.md) - `start_kernel`をコールする前の最後の準備について説明します。
* [kernelのエントリポイント](linux-initialization-4.md) - kernelの汎用コード内の最初のステップについて説明します。
* [アーキテクチャ固有の初期化](linux-initialization-5.md) - アーキテクチャ固有の初期化について説明します。
* [アーキテクチャ固有の初期化の続き](linux-initialization-6.md) - アーキテクチャ固有の初期化処理について説明の続きです。
* [アーキテクチャ固有の初期化の終わり](linux-initialization-7.md) - `setup_arch`関連の終わりを説明します。
* [スケジューラの初期化](linux-initialization-8.md) - スケジューラの初期化前の準備と、初期化自体の説明をします。
* [RCUの初期化](linux-initialization-9.md) - [RCU](http://en.wikipedia.org/wiki/Read-copy-update)の初期化について説明します。
* [初期化の終わり](linux-initialization-10.md) - linux kernelの初期化に関する最後の部分について説明します。
