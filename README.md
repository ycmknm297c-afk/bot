# bot
const express = require('express');
const bodyParser = require('body-parser');
const app = express();
const PORT = 3000;

app.use(bodyParser.json());
app.use(express.static('public')); // HTML 파일들을 담을 폴더

// 봇의 상태 및 데이터 저장소 (실제 서비스에서는 DB 사용 권장)
let botSettings = {
    isRunning: true // 봇 켜짐/꺼짐 상태
};

let userDatabase = {}; // { '유저닉네임': { warnings: 0, chatCount: 0 } }

// 1. 웹 대시보드용 HTML 반환
app.get('/', (req, res) => {
    res.send(`
        <!DOCTYPE html>
        <html>
        <head>
            <title>카카오톡 봇 관리자 패널</title>
            <style>
                body { font-family: Arial, sans-serif; margin: 40px; background: #f4f4f9; }
                .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 4px 8px rgba(0,0,0,0.1); margin-bottom: 20px; }
                button { padding: 10px 20px; font-size: 16px; cursor: pointer; border: none; border-radius: 4px; }
                .btn-on { background: #28a745; color: white; }
                .btn-off { background: #dc3545; color: white; }
                table { width: 100%; border-collapse: collapse; margin-top: 15px; }
                th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
                th { background-color: #f2f2f2; }
            </style>
        </head>
        <body>
            <h1>카카오톡 봇 제어 대시보드</h1>
            
            <div class="card">
                <h2>봇 전원 제어</h2>
                <p>현재 상태: <strong id="status">조회 중...</strong></p>
                <button id="toggleBtn" onclick="toggleBot()">토글</button>
            </div>

            <div class="card">
                <h2>경고 및 채팅 순위 (랭킹)</h2>
                <table>
                    <thead>
                        <tr>
                            <th>닉네임</th>
                            <th>채팅 횟수</th>
                            <th>누적 경고</th>
                            <th>상태</th>
                        </tr>
                    </thead>
                    <tbody id="userTable">
                        </tbody>
                </table>
            </div>

            <script>
                // 데이터 로드
                function loadData() {
                    fetch('/api/status')
                        .then(res => res.json())
                        .then(data => {
                            document.getElementById('status').innerText = data.isRunning ? 'RUNNING (켜짐)' : 'STOPPED (꺼짐)';
                            const toggleBtn = document.getElementById('toggleBtn');
                            toggleBtn.className = data.isRunning ? 'btn-off' : 'btn-on';
                            toggleBtn.innerText = data.isRunning ? '봇 끄기' : '봇 켜기';
                        });

                    fetch('/api/users')
                        .then(res => res.json())
                        .then(users => {
                            const tbody = document.getElementById('userTable');
                            tbody.innerHTML = '';
                            for (let username in users) {
                                const user = users[username];
                                const isKicked = user.warnings >= 4 ? '<span style="color:red; font-weight:bold;">강퇴 대상</span>' : '정상';
                                tbody.innerHTML += \`
                                    <tr>
                                        <td>\${username}</td>
                                        <td>\${user.chatCount}회</td>
                                        <td>\${user.warnings} / 4</td>
                                        <td>\${isKicked}</td>
                                    </tr>
                                \`;
                            }
                        });
                }

                function toggleBot() {
                    fetch('/api/toggle', { method: 'POST' })
                        .then(() => loadData());
                }

                setInterval(loadData, 2000); // 2초마다 갱신
                loadData();
            </script>
        </body>
        </html>
    `);
});

// 2. 봇 클라이언트 전용 API 엔드포인트들
app.get('/api/status', (req, res) => {
    res.json(botSettings);
});

app.post('/api/toggle', (req, res) => {
    botSettings.isRunning = !botSettings.isRunning;
    res.json(botSettings);
});

app.get('/api/users', (req, res) => {
    res.json(userDatabase);
});

// 메시지 수신 및 처리 API
app.post('/api/message', (req, res) => {
    const { sender, message } = req.body;

    if (!botSettings.isRunning) {
        return res.json({ success: false, reason: "Bot is turned off" });
    }

    // 유저 데이터 생성 혹은 가져오기
    if (!userDatabase[sender]) {
        userDatabase[sender] = { warnings: 0, chatCount: 0 };
    }

    // 채팅 횟수 증가
    userDatabase[sender].chatCount++;

    let responseMessage = null;
    let shouldKick = false;

    // 링크 감지 정규식 (http, https, 혹은 단축 URL 패턴)
    const linkRegex = /(https?:\/\/[^\s]+)/g;
    if (linkRegex.test(message)) {
        userDatabase[sender].warnings++;
        
        if (userDatabase[sender].warnings >= 4) {
            shouldKick = true;
            responseMessage = `⚠️ [${sender}]님은 경고 4회 누적으로 강퇴 대상입니다.`;
        } else {
            responseMessage = `⚠️ [${sender}]님, 오픈채팅방 내 링크 전송이 감지되었습니다. (경고 누적: ${userDatabase[sender].warnings}/4)`;
        }
    }

    res.json({
        success: true,
        reply: responseMessage,
        shouldKick: shouldKick,
        warnings: userDatabase[sender].warnings
    });
});

app.listen(PORT, () => {
    console.log(`서버가 포트 ${PORT}에서 작동 중입니다.`);
});
