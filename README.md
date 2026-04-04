<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>Byte Game App</title>
    <script src="https://telegram.org"></script>
    <style>
        :root {
            --bg: var(--tg-theme-bg-color, #fff);
            --text: var(--tg-theme-text-color, #000);
            --hint: var(--tg-theme-hint-color, #999);
            --btn: var(--tg-theme-button-color, #2481cc);
            --btn-text: var(--tg-theme-button-text-color, #fff);
        }

        body {
            background-color: var(--bg);
            color: var(--text);
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            margin: 0;
            padding-bottom: 70px; /* для меню */
            overflow-x: hidden;
        }

        /* Навигация */
        .tab-content { display: none; padding: 20px; animation: fadeIn 0.3s; }
        .active { display: block; }

        .nav-bar {
            position: fixed;
            bottom: 0;
            width: 100%;
            height: 60px;
            background: var(--tg-theme-secondary-bg-color, #efeff4);
            display: flex;
            justify-content: space-around;
            align-items: center;
            border-top: 1px solid #ccc;
            z-index: 100;
        }

        .nav-item { font-size: 12px; text-align: center; color: var(--hint); cursor: pointer; }
        .nav-item.active-nav { color: var(--btn); font-weight: bold; }

        /* Профиль */
        .profile-card {
            background: var(--tg-theme-secondary-bg-color, #f0f0f0);
            padding: 15px;
            border-radius: 15px;
            margin-bottom: 20px;
            text-align: center;
        }
        .balance-big { font-size: 32px; font-weight: bold; margin: 10px 0; }

        /* Колесо */
        .wheel-box { position: relative; width: 300px; height: 300px; margin: 20px auto; }
        canvas { width: 100%; height: 100%; border-radius: 50%; transition: transform 4s cubic-bezier(0.1, 0, 0.1, 1); }
        .arrow { position: absolute; top: -10px; left: 50%; transform: translateX(-50%); width: 0; height: 0; border-left: 15px solid transparent; border-right: 15px solid transparent; border-top: 25px solid red; z-index: 10; }

        .main-btn {
            width: 100%;
            padding: 15px;
            background: var(--btn);
            color: var(--btn-text);
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: bold;
            margin-top: 10px;
        }

        @keyframes fadeIn { from { opacity: 0; } to { opacity: 1; } }
    </style>
</head>
<body>

    <!-- Вкладка ИГРЫ -->
    <div id="games" class="tab-content active">
        <h2>Каталог игр</h2>
        <div class="profile-card" onclick="switchTab('wheel')">
            <h3>🎡 Wheel of Fortune</h3>
            <p>Испытай удачу и выиграй бонусы!</p>
            <button class="main-btn">ИГРАТЬ</button>
        </div>
    </div>

    <!-- Вкладка САМА ИГРА -->
    <div id="wheel" class="tab-content">
        <button onclick="switchTab('games')" style="background:none; border:none; color:var(--btn); cursor:pointer;">← Назад</button>
        <h2 style="text-align:center;">Колесо удачи</h2>
        <div class="wheel-box">
            <div class="arrow"></div>
            <canvas id="wheelCanvas" width="400" height="400"></canvas>
        </div>
        <p style="text-align:center;">Ставка: 100 🪙</p>
        <button class="main-btn" id="spinBtn" onclick="playWheel()">КРУТИТЬ (100)</button>
    </div>

    <!-- Вкладка КОШЕЛЕК -->
    <div id="wallet" class="tab-content">
        <h2>Кошелек</h2>
        <div class="profile-card">
            <p>Ваш баланс:</p>
            <div class="balance-big"><span id="userBalance">1000</span> 🪙</div>
        </div>
        <button class="main-btn" style="background:#4CAF50;" onclick="deposit()">Пополнить</button>
        <button class="main-btn" style="background:gray; margin-top:10px;" onclick="withdraw()">Вывести</button>
    </div>

    <!-- Вкладка ПРОФИЛЬ -->
    <div id="profile" class="tab-content">
        <h2>Профиль</h2>
        <div class="profile-card">
            <img id="userPhoto" src="" style="width:80px; border-radius:50%; display:none; margin: 0 auto 10px;">
            <div id="userName" style="font-weight:bold; font-size:20px;">Пользователь</div>
            <div id="userId" style="color:var(--hint);">ID: 000000</div>
        </div>
    </div>

    <!-- Нижнее меню -->
    <div class="nav-bar">
        <div class="nav-item active-nav" onclick="switchTab('games', this)">🎮 Игры</div>
        <div id="walletTab" class="nav-item" onclick="switchTab('wallet', this)">💰 Кошелек</div>
        <div class="nav-item" onclick="switchTab('profile', this)">👤 Профиль</div>
    </div>

    <script>
        const tg = window.Telegram.WebApp;
        tg.expand();

        let balance = 1000;
        let isSpinning = false;
        let currentRotation = 0;

        // Инициализация данных пользователя
        if (tg.initDataUnsafe?.user) {
            document.getElementById('userName').innerText = tg.initDataUnsafe.user.first_name;
            document.getElementById('userId').innerText = "ID: " + tg.initDataUnsafe.user.id;
            if (tg.initDataUnsafe.user.photo_url) {
                const img = document.getElementById('userPhoto');
                img.src = tg.initDataUnsafe.user.photo_url;
                img.style.display = 'block';
            }
        }

        // Переключение вкладок
        function switchTab(tabId, el) {
            document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
            document.getElementById(tabId).classList.add('active');
            
            if (el) {
                document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active-nav'));
                el.classList.add('active-nav');
            }
        }

        // Логика Колеса
        const canvas = document.getElementById('wheelCanvas');
        const ctx = canvas.getContext('2d');
        const sectors = [
            { label: "0", win: 0, color: "#444" },
            { label: "200", win: 200, color: "#FFBC03" },
            { label: "50", win: 50, color: "#FF5A10" },
            { label: "500", win: 500, color: "#E6007A" },
            { label: "0", win: 0, color: "#444" },
            { label: "1000", win: 1000, color: "#2196F3" }
        ];

        function drawWheel() {
            const arc = 2 * Math.PI / sectors.length;
            sectors.forEach((s, i) => {
                const angle = i * arc;
                ctx.beginPath();
                ctx.fillStyle = s.color;
                ctx.moveTo(200, 200);
                ctx.arc(200, 200, 200, angle, angle + arc);
                ctx.fill();
                ctx.save();
                ctx.translate(200, 200);
                ctx.rotate(angle + arc / 2);
                ctx.fillStyle = "#fff";
                ctx.font = "bold 20px sans-serif";
                ctx.fillText(s.label, 140, 10);
                ctx.restore();
            });
        }
        drawWheel();

        function playWheel() {
            if (isSpinning || balance < 100) {
                if (balance < 100) tg.showAlert("Недостаточно средств!");
                return;
            }

            balance -= 100;
            updateBalance();
            isSpinning = true;
            
            const extraDeg = Math.floor(Math.random() * 360) + 1800;
            currentRotation += extraDeg;
            canvas.style.transform = `rotate(${currentRotation}deg)`;

            setTimeout(() => {
                isSpinning = false;
                const actualDeg = currentRotation % 360;
                const sectorIdx = Math.floor((360 - actualDeg % 360) / (360 / sectors.length)) % sectors.length;
                const winAmount = sectors[sectorIdx].win;
                
                balance += winAmount;
                updateBalance();
                
                if (winAmount > 0) {
                    tg.showAlert(`Победа! Вы выиграли ${winAmount} 🪙`);
                    tg.HapticFeedback.notificationOccurred('success');
                } else {
                    tg.showAlert("Повезет в другой раз!");
                }
            }, 4000);
        }

        function updateBalance() {
            document.getElementById('userBalance').innerText = balance;
        }

        // Функции пополнения (заглушки)
        function deposit() {
            tg.showConfirm("Перейти к пополнению через @bytebot?", (ok) => {
                if (ok) tg.close(); // Здесь обычно переход по ссылке оплаты
            });
        }

        function withdraw() {
            tg.showAlert("Вывод доступен от 5000 🪙");
        }
    </script>
</body>
</html>
