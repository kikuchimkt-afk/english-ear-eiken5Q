# 高品質・無料 Web Speech API (TTS) 実装ガイド

このガイドでは、Google Cloud TTSやYoudaoなどの外部APIキーを使用せず、ブラウザ標準の機能（Web Speech API）だけで**高品質かつ人間らしい音声**を再生する方法を解説します。

## 概要

通常、`window.speechSynthesis` をそのまま使うと「昔ながらのロボット音」になりがちですが、本手法では**EdgeやChromeに隠されている高品質音声（Natural Voice / Google Voice）**をプログラム側で優先的に選択することで、無料で劇的に品質を向上させます。

### メリット
- **完全無料**: APIキー不要、課金リスクなし。
- **高信頼性**: 外部サーバーダウンの影響を受けない。
- **高品質**: 特にWindows上のEdgeブラウザでは、人間と変わらないレベルの音声が利用可能。

---

## 実装のポイント

### 1. ベストな音声を探すロジック

ブラウザが持っている音声リストから、以下の優先順位で音声を探し出します。

1.  `Google US English` (Chrome向け・高品質)
2.  `Microsoft ... Natural` (Edge向け・超高品質)
3.  `en-US` (標準的な英語)

```javascript
let cachedVoice = null;

function getBestVoice() {
    if (cachedVoice) return cachedVoice;

    // 音声リストを取得
    const voices = window.speechSynthesis.getVoices();
    if (voices.length === 0) return null;

    // 優先順位リスト（上にある条件ほど優先される）
    const priorities = [
        name => name.includes("Google US English"),           // Chrome高音質
        name => name.includes("Microsoft") && name.includes("Natural") && name.includes("English"), // Edge Natural
        name => name.includes("Google") && name.includes("English"),
        name => name.includes("en-US")
    ];

    // 優先順位に従って検索
    for (const check of priorities) {
        const found = voices.find(v => check(v.name));
        if (found) {
            console.log("Selected Voice:", found.name);
            cachedVoice = found;
            return found;
        }
    }

    // 見つからなければ英語のどれかを使う
    return voices.find(v => v.lang.startsWith('en')) || voices[0];
}

// ブラウザによっては音声リストのロードが非同期なのでイベント監視が必要
if (window.speechSynthesis) {
    window.speechSynthesis.onvoiceschanged = () => {
        getBestVoice(); // キャッシュしておく
    };
}
```

### 2. 再生関数の実装

速度変更をリアルタイムに反映させるため、`rate` は再生直前に設定します。

```javascript
function speakText(text, speedRate = 1.0, onEndCallback) {
    // ブラウザが対応しているか確認
    if (!window.speechSynthesis) {
        console.error("TTS not supported");
        if (onEndCallback) onEndCallback();
        return;
    }

    const u = new SpeechSynthesisUtterance(text);
    
    // 音声を設定
    const voice = getBestVoice();
    if (voice) u.voice = voice;
    
    u.lang = 'en-US';
    u.rate = speedRate; // 0.5 ~ 2.0 程度推奨

    // 終了時のコールバック（連続再生に必須）
    u.onend = () => {
        if (onEndCallback) onEndCallback();
    };

    // エラーハンドリング
    u.onerror = (e) => {
        console.warn("TTS Error:", e);
        // エラー起きても止まらないようにコールバックを呼ぶのがコツ
        if (onEndCallback) onEndCallback();
    };

    window.speechSynthesis.speak(u);
}
```

### 3. 連続再生（推奨パターン）

`setTimeout` などの遅延を入れると、スマホのブラウザでは自動再生ポリシーに引っかかり途中で止まることがあります。
`onend` コールバックを使って「数珠つなぎ」に再生するのが最も安定します。

```javascript
const playlist = ["Hello.", "How are you?", "This is a free TTS demo."];
let currentIndex = 0;

function playNext() {
    if (currentIndex >= playlist.length) {
        console.log("All done");
        return;
    }

    const text = playlist[currentIndex];
    
    // 次の再生をする関数をコールバックとして渡す
    speakText(text, 1.0, () => {
        currentIndex++;
        playNext(); // 再帰的に次を呼ぶ
    });
}

// 開始
playNext();
```

---

## 補足: Android / iOS 対応について

モバイル端末では、ユーザーのタップ操作（クリックなど）の直後に音声を再生し始めないとブロックされることがあります。
対策として、最初の「再生開始ボタン」のクリックイベント内で、無音の音声を一瞬流すか、`speechSynthesis.speak` を一度だけ呼び出しておくと安定します。
