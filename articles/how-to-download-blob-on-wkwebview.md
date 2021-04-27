---
title: "iOS 14.5のWKWebViewでBlobオブジェクトをダウンロードする"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["iOS", "Swift", "WKWebView"]
published: true
---
# Webブラウザアプリ開発者待望の新機能

iOS 14.5未満のWKWebViewは`blob:`ではじまるURLのダウンロード、つまりJavaScriptで生成されたBlobオブジェクトのダウンロードにネイティブ対応していませんでした。BlobオブジェクトはWKWebView上のメモリに存在するため、単純にURLを指定するだけではダウンロードできず、以下のようにWKWebViewと連携するJavaScriptを実行してData URLに変換するなどの工夫が必要でした。

- https://luke-1220.github.io/post/webview-dl/
- https://stackoverflow.com/a/61703086

2021年4月27日にリリースされたiOS 14.5から、WKWebViewや関連クラスにダウンロード機能を実装するためのAPIがいくつか追加されました。それらを使うことで、Blobオブジェクトのダウンロードもネイティブコードのみで実装可能になりました。

# Blobオブジェクトをダウンロードするための手順

以下がXcode 12.5とSwift 5.4で実装する手順です。わかりやすいようにSwiftUIではなくUIKitベースで実装しています。また、各デリゲートメソッドの実装はWKWebViewをサブビューに持つViewControllerクラスで行っています。

なお、ここではBlobオブジェクトのダウンロード元サイトとして[Scratch 3.0](https://scratch.mit.edu/)のエディター画面を使っています。ファイルメニューから「コンピューターに保存する」を選択すると、プロジェクトファイルを`sb3`の拡張子を持つBlobオブジェクトとしてダウンロードできます。

エディター画面: https://scratch.mit.edu/projects/editor/ 

![Scratch 3.0のエディター画面](https://storage.googleapis.com/zenn-user-upload/xl06iu636fo3yzglbwlqf6ykvkca)

## 手順１. WKNavigationDelegateの実装

WKNavigationDelegateプロトコルの`webView(_:decidePolicyFor:preferences:decisionHandler:)`を以下のように実装します。`blob:`ではじまるURLがリクエストされたときに、`decisionHandler()`で新しく追加された`.download`を返すことでダウンロードの準備がはじまります。

```swift
extension ViewController: WKNavigationDelegate {
    
    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        if navigationAction.request.url?.scheme == "blob" {
            decisionHandler(.download)
        } else {
            decisionHandler(.allow)
        }
    }
...
```

さらに、新しく追加されたデリゲートメソッドである`webView(_:navigationAction:didBecome:)`も実装します。`WKDownload`というWKWebView上でのダウンロードを担当するクラスのインスタンスが渡されるので、その`delegate`プロパティを指定してあげます。これにより、ダウンロード先の指定などができるようになります。

```swift
...
    func webView(_ webView: WKWebView, navigationAction: WKNavigationAction, didBecome download: WKDownload) {
        download.delegate = self
    }
}
```

## 手順２. WKDownloadDelegateの実装

WKDownload用に新規追加されたWKDownloadDelegateプロトコルの`download(_:decideDestinationUsing:suggestedFilename:completionHandler:)`を以下のように実装します。`suggestedFilename`によって提示されたファイル名からダウンロード先のURLを決定し、`completionHandler()`に渡してあげます。ここでは`tmp`ディレクトリ直下にそのまま保存するようにしてみました。

```swift
extension ViewController: WKDownloadDelegate {

    func download(_ download: WKDownload, decideDestinationUsing response: URLResponse, suggestedFilename: String, completionHandler: @escaping (URL?) -> Void) {
        let url = FileManager.default.temporaryDirectory.appendingPathComponent(suggestedFilename)
        completionHandler(url)
    }
}
```

## 手順３. WKWebViewでダウンロード元のURLを開く

あとはこれまでのWKWebViewの使い方と同じです。`delegate`プロパティを指定し、ダウンロード元のURLをリクエストします。その後、Blobオブジェクトのダウンロードがはじまる操作（今回は「コンピューターに保存する」）をすると、ファイルのダウンロードがはじまり、指定したURLにファイルが保存されます。

```swift
import UIKit
import WebKit

class ViewController: UIViewController {
    
    @IBOutlet weak var webView: WKWebView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        webView.navigationDelegate = self
        
        let url = URL(string: "https://scratch.mit.edu/projects/editor/")!
        webView.load(URLRequest(url: url))
    }
}
```

# おまけ１: Blobオブジェクト以外もダウンロードできるようにする

WKNavigationActionに新しく`shouldPerformDownload`というプロパティが追加されているので、それを使うとダウンロード対象のリクエストかどうかを判断することができます。`blob:`のときにはこれが`true`になるので、`webView(_:decidePolicyFor:preferences:decisionHandler:)`を以下のように実装することもできます。

```swift
    func webView(_ webView: WKWebView, decidePolicyFor navigationAction: WKNavigationAction, decisionHandler: @escaping (WKNavigationActionPolicy) -> Void) {
        if navigationAction.shouldPerformDownload {
            decisionHandler(.download)
        } else {
            decisionHandler(.allow)
        }
    }
```

# おまけ２: 任意のURLをダウンロードする

WKWebViewに追加された`startDownload(using:completionHandler:)`を使って、任意のURLをダウンロードすることもできます。ダウンロードの準備ができると`completionHandler`に指定したクロージャーが呼ばれるので、そこに渡されるWKDownloadのインスタンスに`delegate`を指定してあげれば、WKDownloadDelegateプロトコルの各デリゲートメソッドが呼ばれるようになります。

```swift
webView.startDownload(using: URLRequest(url: URL(string: "https://scratch.mit.edu/")!)) { download in
    download.delegate = self
}
```
