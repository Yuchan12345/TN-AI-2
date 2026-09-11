# My Site Project<!DOCTYPE html>
<html lang="ko">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>⚽ 패널티킥 게임 – 키커 vs 키퍼</title>
  <style>
    * {
      box-sizing: border-box;
      user-select: none;
    }

    body {
      background: linear-gradient(145deg, #1a4a2a, #0e2b18);
      min-height: 100vh;
      display: flex;
      justify-content: center;
      align-items: center;
      font-family: 'Segoe UI', Roboto, system-ui, sans-serif;
      margin: 0;
      padding: 16px;
    }

    .game-wrapper {
      background: #2d6a3b;
      padding: 24px 30px 30px;
      border-radius: 64px 64px 48px 48px;
      box-shadow: 0 20px 30px rgba(0, 0, 0, 0.7), inset 0 2px 6px rgba(255, 255, 200, 0.3);
      border-bottom: 8px solid #1d4527;
    }

    .canvas-container {
      background: #1f4a2b;
      padding: 12px;
      border-radius: 48px;
      box-shadow: inset 0 0 0 2px #3f8a4e, inset 0 0 12px #0f2a14;
    }

    canvas {
      display: block;
      width: 600px;
      height: 400px;
      border-radius: 32px;
      background: radial-gradient(circle at 30% 40%, #6aaa7a, #2f6a3f);
      box-shadow: 0 0 0 3px #4e7a3e, 0 12px 28px rgba(0, 0, 0, 0.6);
      cursor: crosshair;
      touch-action: none;  /* 모바일 터치 스크롤 방지 */
    }

    .info-panel {
      display: flex;
      justify-content: space-between;
      align-items: center;
      margin-top: 20px;
      padding: 0 8px;
      color: #f8f0d5;
      text-shadow: 0 2px 3px #0f1f0b;
      font-weight: 600;
    }

    .score-box {
      background: #1d3922;
      padding: 8px 22px;
      border-radius: 40px;
      font-size: 1.5rem;
      letter-spacing: 1px;
      box-shadow: inset 0 0 0 2px #6f9f6a, 0 6px 0 #0e2a14;
      backdrop-filter: blur(2px);
    }

    .score-box span {
      display: inline-block;
      min-width: 2.2rem;
      text-align: center;
    }

    .btn-reset {
      background: #d9b650;
      border: none;
      color: #1b3a1a;
      font-size: 1.2rem;
      font-weight: 700;
      padding: 8px 28px;
      border-radius: 60px;
      cursor: pointer;
      box-shadow: 0 7px 0 #8f752b, 0 4px 12px rgba(0, 0, 0, 0.5);
      transition: 0.08s linear;
      letter-spacing: 0.5px;
      border-bottom: 2px solid #f9e68c;
    }

    .btn-reset:active {
      transform: translateY(5px);
      box-shadow: 0 2px 0 #6e5a20;
    }

    .btn-reset:hover {
      background: #e8c85c;
    }

    .hint {
      font-size: 0.95rem;
      background: #204d2b;
      padding: 6px 20px;
      border-radius: 50px;
      border-left: 4px solid #f3d06b;
      color: #f2efd0;
    }

    @media (max-width: 680px) {
      .game-wrapper {
        padding: 16px;
        border-radius: 40px;
      }
      canvas {
        width: 100%;
        height: auto;
        aspect-ratio: 600 / 400;
      }
      .info-panel {
        flex-wrap: wrap;
        gap: 12px;
        justify-content: center;
      }
      .score-box {
        font-size: 1.2rem;
        padding: 6px 18px;
      }
      .btn-reset {
        font-size: 1rem;
        padding: 6px 22px;
      }
    }
  </style>
</head>
<body>
  <div class="game-wrapper">
    <div class="canvas-container">
      <canvas id="penaltyCanvas" width="600" height="400"></canvas>
    </div>

    <div class="info-panel">
      <div class="score-box">
        ⚽ <span id="goalCount">0</span> : <span id="saveCount">0</span> 🧤
      </div>
      <div class="hint">👆 클릭해서 슛</div>
      <button class="btn-reset" id="resetBtn">↺ 재경기</button>
    </div>
  </div>

  <script>
    (function() {
      const canvas = document.getElementById('penaltyCanvas');
      const ctx = canvas.getContext('2d');

      // 점수 표시 엘리먼트
      const goalSpan = document.getElementById('goalCount');
      const saveSpan = document.getElementById('saveCount');

      // ---------- 게임 상태 ----------
      const state = {
        // 키커 위치 (고정)
        kickerX: 510,
        kickerY: 290,

        // 공 위치 (초기: 키커 발 근처)
        ballX: 490,
        ballY: 280,

        // 골대 영역 (골대는 왼쪽)
        goalLeft: 20,
        goalRight: 120,
        goalTop: 60,
        goalBottom: 340,

        // 골키퍼 (랜덤 다이빙)
        keeperX: 70,        // x 위치 (고정)
        keeperY: 200,       // 기본 중앙 (다이빙 후 변경)
        keeperTargetY: 200, // 다이빙 목표 y
        isDiving: false,    // 다이빙 애니메이션 중인지
        diveProgress: 0,    // 0~1
        diveDirection: 0,   // -1 위, 1 아래, 0 중앙

        // 공 애니메이션
        isKicking: false,
        kickProgress: 0,    // 0~1
        startBallX: 490,
        startBallY: 280,
        targetX: 0,
        targetY: 0,
        ballSpeed: 0.045,   // 진행 속도

        // 결과 표시
        showResult: false,
        resultText: '',
        resultColor: 'white',

        // 스코어
        goals: 0,
        saves: 0,

        // 잠금 (애니메이션 중 클릭 방지)
        locked: false,
      };

      // ---------- 초기화 ----------
      function resetGame() {
        state.goals = 0;
        state.saves = 0;
        updateScoreDisplay();
        resetPositions();
        state.showResult = false;
        state.isKicking = false;
        state.locked = false;
        drawCanvas();
      }

      // 위치/키퍼 초기화 (애니메이션 없이)
      function resetPositions() {
        state.ballX = 490;
        state.ballY = 280;
        state.keeperY = 200;
        state.keeperTargetY = 200;
        state.isDiving = false;
        state.diveProgress = 0;
        state.diveDirection = 0;
        state.isKicking = false;
        state.kickProgress = 0;
        state.showResult = false;
        state.locked = false;
        // 공 위치 동기화
        state.startBallX = 490;
        state.startBallY = 280;
      }

      // 점수 업데이트
      function updateScoreDisplay() {
        goalSpan.textContent = state.goals;
        saveSpan.textContent = state.saves;
      }

      // ---------- 골키퍼 랜덤 다이빙 (목표 Y 설정) ----------
      function randomDive() {
        // 0: 중앙, 1: 위, 2: 아래 (왼쪽/오른쪽은 없이 y축 다이빙)
        const dir = Math.floor(Math.random() * 3); // 0,1,2
        let targetY = 200;
        if (dir === 1) { // 위로 다이빙
          targetY = 100 + Math.random() * 40;  // 100~140
        } else if (dir === 2) { // 아래로 다이빙
          targetY = 260 + Math.random() * 40;  // 260~300
        } else { // 중앙 (살짝 랜덤)
          targetY = 180 + Math.random() * 40;   // 180~220
        }
        // 골대 범위 안에서만 (60~340)
        targetY = Math.min(340, Math.max(60, targetY));

        state.keeperTargetY = targetY;
        state.diveDirection = (targetY < state.keeperY) ? -1 : (targetY > state.keeperY ? 1 : 0);
        state.isDiving = true;
        state.diveProgress = 0;
      }

      // ---------- 슛 실행 (클릭한 좌표) ----------
      function shoot(clickX, clickY) {
        // 잠금, 킥 중, 결과 표시중이면 무시
        if (state.locked || state.isKicking || state.showResult) return;

        // 클릭 위치를 캔버스 좌표로 (canvas.getBoundingClientRect 이용)
        const rect = canvas.getBoundingClientRect();
        const scaleX = canvas.width / rect.width;   // canvas 물리 픽셀 비율
        const scaleY = canvas.height / rect.height;

        // 실제 캔버스 좌표
        const canvasX = (clickX - rect.left) * scaleX;
        const canvasY = (clickY - rect.top) * scaleY;

        // 골대 영역 밖 클릭 제한 (너무 벗어나면 무시)
        if (canvasX < 10 || canvasX > 180 || canvasY < 40 || canvasY > 360) {
          // 약간의 피드백 (무시)
          return;
        }

        // 목표 지점 (공이 날아갈 위치)
        state.targetX = Math.min(180, Math.max(10, canvasX));
        state.targetY = Math.min(355, Math.max(45, canvasY));

        // 키커가 차는 위치 (공 시작점)
        state.startBallX = 490;
        state.startBallY = 280;

        // 공 위치를 시작점으로
        state.ballX = state.startBallX;
        state.ballY = state.startBallY;

        // 골키퍼 랜덤 다이빙 (공이 날아가기 전에 결정)
        randomDive();

        // 킥 시작
        state.isKicking = true;
        state.kickProgress = 0;
        state.showResult = false;
        state.locked = true;  // 애니메이션 중 클릭 방지

        // draw는 requestAnimationFrame에서 계속
      }

      // ---------- 애니메이션 업데이트 (매 프레임) ----------
      function updateAnimation() {
        // 1. 골키퍼 다이빙 애니메이션 (킥과 별개로 진행)
        if (state.isDiving) {
          state.diveProgress += 0.04;
          if (state.diveProgress >= 1) {
            state.diveProgress = 1;
            state.isDiving = false;
            state.keeperY = state.keeperTargetY;
          } else {
            // easing: smooth step
            const t = state.diveProgress;
            const smooth = t * t * (3 - 2 * t);
            const startY = 200; // 다이빙 시작 Y는 항상 200 (중앙)
            state.keeperY = startY + (state.keeperTargetY - startY) * smooth;
          }
        }

        // 2. 공 이동 (킥 중)
        if (state.isKicking) {
          state.kickProgress += state.ballSpeed;
          if (state.kickProgress >= 1) {
            state.kickProgress = 1;
            state.isKicking = false;
            state.ballX = state.targetX;
            state.ballY = state.targetY;

            // ----- 결과 판정 (골/세이브) -----
            const goalX = state.targetX;
            const goalY = state.targetY;
            const keeperY = state.keeperY; // 다이빙 후 최종 위치

            // 골대 영역 안에 들어왔는지 확인 (왼쪽 골대)
            const inGoal = (goalX >= state.goalLeft && goalX <= state.goalRight &&
                            goalY >= state.goalTop && goalY <= state.goalBottom);

            // 키퍼가 막았는지: 공이 골대 영역 안 && 키퍼와의 거리 (반경 35px)
            const keeperDist = Math.hypot(goalX - 70, goalY - keeperY);
            const isSaved = inGoal && (keeperDist < 38); 

            let resultMsg = '';
            let resultColor = '#fff';

            if (isSaved) {
              state.saves += 1;
              resultMsg = '🧤 세이브!';
              resultColor = '#f7d44a';
            } else if (inGoal) {
              state.goals += 1;
              resultMsg = '⚽ 골!!';
              resultColor = '#b3ff9b';
            } else {
              // 골대 밖
              resultMsg = '❌ 빗나감';
              resultColor = '#ffb3b3';
            }

            state.resultText = resultMsg;
            state.resultColor = resultColor;
            state.showResult = true;
            updateScoreDisplay();

            // 0.8초 후 자동 리셋 (잠금 해제 + 위치 초기화)
            state.locked = false;  // 결과 표시 중에는 클릭 막지만, 리셋 타이머로 재시작 가능
            // 하지만 결과 표시 중에도 클릭을 막기 위해 showResult = true 상태에서 클릭 무시
            // 그리고 1.2초 뒤에 모든걸 리셋 (리셋은 결과가 사라지고 다시 슛 가능)
            if (window.resultResetTimer) clearTimeout(window.resultResetTimer);
            window.resultResetTimer = setTimeout(() => {
              // 결과 초기화 및 위치 리셋, 단 스코어는 유지
              state.showResult = false;
              state.isKicking = false;
              state.locked = false;
              // 키퍼와 공 위치 초기화 (단, 키퍼는 중앙으로)
              state.keeperY = 200;
              state.keeperTargetY = 200;
              state.isDiving = false;
              state.diveProgress = 0;
              state.ballX = 490;
              state.ballY = 280;
              state.startBallX = 490;
              state.startBallY = 280;
              drawCanvas();
            }, 1200);
          } else {
            // 공 위치 보간 (목표 지점으로)
            const t = state.kickProgress;
            // 약간의 호를 그리듯 (y축으로 살짝 커브) 
            const arcOffset = Math.sin(t * Math.PI) * 15; 
            const currentX = state.startBallX + (state.targetX - state.startBallX) * t;
            const currentY = state.startBallY + (state.targetY - state.startBallY) * t - arcOffset * 0.3;
            state.ballX = currentX;
            state.ballY = currentY;
          }
        }

        // 그리기
        drawCanvas();
      }

      // ---------- 렌더링 ----------
      function drawCanvas() {
        ctx.clearRect(0, 0, 600, 400);

        // ---- 잔디 느낌 ----
        ctx.fillStyle = '#3b8249';
        ctx.fillRect(0, 0, 600, 400);
        // 그라데이션 s
        const grad = ctx.createRadialGradient(180, 200, 30, 180, 200, 300);
        grad.addColorStop(0, '#58a86a');
        grad.addColorStop(1, '#1c5a2a');
        ctx.fillStyle = grad;
        ctx.fillRect(0, 0, 600, 400);

        // ---- 골대 (왼쪽) ----
        ctx.shadowColor = 'rgba(0,0,0,0.4)';
        ctx.shadowBlur = 10;

        // 골대 프레임 (네트 느낌)
        ctx.strokeStyle = '#f0f0e0';
        ctx.lineWidth = 6;
        ctx.strokeRect(state.goalLeft, state.goalTop, state.goalRight - state.goalLeft, state.goalBottom - state.goalTop);

        // 네트 라인 (십자)
        ctx.strokeStyle = 'rgba(220, 220, 200, 0.25)';
        ctx.lineWidth = 1.5;
        for (let y = state.goalTop + 20; y < state.goalBottom; y += 25) {
          ctx.beginPath();
          ctx.moveTo(state.goalLeft, y);
          ctx.lineTo(state.goalRight, y);
          ctx.stroke();
        }
        for (let x = state.goalLeft + 20; x < state.goalRight; x += 25) {
          ctx.beginPath();
          ctx.moveTo(x, state.goalTop);
          ctx.lineTo(x, state.goalBottom);
          ctx.stroke();
        }

        // 골대 기둥 (안쪽 선)
        ctx.shadowBlur = 4;
        ctx.strokeStyle = '#fcf9ea';
        ctx.lineWidth = 8;
        ctx.strokeRect(state.goalLeft-2, state.goalTop-2, state.goalRight - state.goalLeft + 4, state.goalBottom - state.goalTop + 4);

        // ---- 골키퍼 ----
        // 몸통 (점프/다이빙)
        ctx.shadowBlur = 12;
        ctx.shadowColor = '#0f1f0b';
        const kx = 70, ky = state.keeperY;
        // 키퍼 글러브/몸통
        ctx.fillStyle = '#e03a3a';
        ctx.beginPath();
        ctx.ellipse(kx, ky-5, 18, 28, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillStyle = '#f5d742';
        ctx.beginPath();
        ctx.ellipse(kx-5, ky-22, 10, 12, 0.2, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.ellipse(kx+5, ky-22, 10, 12, -0.2, 0, Math.PI * 2);
        ctx.fill();
        // 머리
        ctx.fillStyle = '#f7d9a0';
        ctx.beginPath();
        ctx.arc(kx, ky-34, 14, 0, Math.PI * 2);
        ctx.fill();
        // 글러브
        ctx.fillStyle = '#f0c040';
        ctx.beginPath();
        ctx.ellipse(kx-16, ky-8, 10, 14, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.ellipse(kx+16, ky-8, 10, 14, 0, 0, Math.PI * 2);
        ctx.fill();

        // ---- 키커 (오른쪽) ----
        ctx.shadowBlur = 8;
        ctx.fillStyle = '#2d4f9c';
        ctx.beginPath();
        ctx.ellipse(510, 280, 22, 30, 0, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillStyle = '#f7d9a0';
        ctx.beginPath();
        ctx.arc(510, 256, 16, 0, Math.PI * 2);
        ctx.fill();
        // 다리 (차는 모션)
        ctx.fillStyle = '#1d3a7a';
        ctx.beginPath();
        ctx.ellipse(492, 290, 12, 18, 0.2, 0, Math.PI * 2);
        ctx.fill();
        // 차는 발
        ctx.fillStyle = '#352211';
        ctx.beginPath();
        ctx.ellipse(484, 304, 10, 7, 0.2, 0, Math.PI * 2);
        ctx.fill();

        // ---- 공 ----
        ctx.shadowBlur = 20;
        ctx.shadowColor = '#111';
        ctx.beginPath();
        ctx.arc(state.ballX, state.ballY, 15, 0, Math.PI * 2);
        ctx.fillStyle = '#f5f5f0';
        ctx.fill();
        ctx.strokeStyle = '#333';
        ctx.lineWidth = 2;
        ctx.stroke();
        // 오각형 무늬
        ctx.fillStyle = '#444';
        ctx.beginPath();
        ctx.arc(state.ballX-4, state.ballY-4, 5, 0, Math.PI * 2);
        ctx.fill();
        ctx.beginPath();
        ctx.arc(state.ballX+5, state.ballY+3, 4, 0, Math.PI * 2);
        ctx.fill();

        // ---- 결과 메시지 ----
        if (state.showResult) {
          ctx.shadowBlur = 30;
          ctx.shadowColor = 'black';
          ctx.font = 'bold 46px "Segoe UI", system-ui, sans-serif';
          ctx.textAlign = 'center';
          ctx.textBaseline = 'middle';
          ctx.fillStyle = state.resultColor;
          ctx.strokeStyle = '#0f1f0b';
          ctx.lineWidth = 6;
          ctx.strokeText(state.resultText, 300, 80);
          ctx.fillText(state.resultText, 300, 80);
        }

        // 간단 안내문
        ctx.shadowBlur = 0;
        ctx.font = '16px sans-serif';
        ctx.fillStyle = '#e8f5da';
        ctx.textAlign = 'center';
        ctx.fillText('← 골대를 클릭해서 슛!', 300, 385);

        ctx.shadowBlur = 0;
      }

      // ---------- 이벤트 연결 ----------
      function handleCanvasClick(e) {
        e.preventDefault();
        const rect = canvas.getBoundingClientRect();
        const clientX = e.clientX || (e.touches && e.touches[0].clientX);
        const clientY = e.clientY || (e.touches && e.touches[0].clientY);
        if (clientX == null) return;
        shoot(clientX, clientY);
      }

      // 마우스 & 터치
      canvas.addEventListener('click', handleCanvasClick);
      canvas.addEventListener('touchstart', function(e) {
        e.preventDefauslt();
        const touch = e.touches[0];
        if (!touch) return;
        shoot(touch.clientX, touch.clientY);
      }, { passive: false });

      // 리셋 버튼
      document.getElementById('resetBtn').addEventListener('click', function() {
        if (window.resultResetTimer) {
          clearTimeout(window.resultResetTimer);
          window.resultResetTimer = null;
        }
        resetGame();
      });

      // 애니메이션 루프
      function gameLoop() {
        updateAnimation();
        requestAnimationFrame(gameLoop);
      }

      // 초기화 및 시작
      resetGame();
      gameLoop();
    })();
  </script>
</body>
</html>