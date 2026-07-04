<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Neural Gesture Recognition System</title>
    <style>
        body {
            font-family: 'Orbitron', 'Arial Narrow', sans-serif;
            text-align: center;
            background: linear-gradient(135deg, #0f0c29, #302b63, #24243e);
            color: #00ffff;
            padding: 20px;
            margin: 0;
            min-height: 100vh;
            background-attachment: fixed;
        }
        h1 {
            font-size: 48px;
            color: #00ffff;
            text-shadow: 0 0 20px #00ffff, 0 0 40px #00ffff;
            margin-bottom: 10px;
        }
        .subtitle {
            font-size: 20px;
            color: #b388ff;
            text-shadow: 0 0 10px #b388ff;
            margin-bottom: 30px;
        }
        video {
            width: 720px;
            height: 540px;
            border: 3px solid #00ffff;
            border-radius: 15px;
            box-shadow: 0 0 30px #00ffff, inset 0 0 20px rgba(0, 255, 255, 0.3);
            margin: 20px auto;
            display: block;
        }
        canvas { display: none; }
        button {
            padding: 18px 50px;
            font-size: 24px;
            background: linear-gradient(45deg, #00ffff, #b388ff);
            color: #000;
            border: none;
            border-radius: 50px;
            cursor: pointer;
            box-shadow: 0 0 20px #00ffff;
            transition: all 0.3s;
            font-weight: bold;
        }
        button:hover {
            transform: scale(1.1);
            box-shadow: 0 0 40px #00ffff, 0 0 60px #b388ff;
        }
        #status {
            font-size: 40px;
            font-weight: bold;
            color: #ff00ff;
            text-shadow: 0 0 20px #ff00ff;
            margin: 40px;
            min-height: 80px;
        }
        .info {
            font-size: 18px;
            color: #b388ff;
            max-width: 800px;
            margin: 30px auto;
            background: rgba(0, 0, 0, 0.4);
            padding: 20px;
            border-radius: 15px;
            border: 1px solid #00ffff;
            box-shadow: 0 0 15px rgba(0, 255, 255, 0.5);
        }
        .grid-bg {
            position: fixed;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            background-image: 
                linear-gradient(rgba(0, 255, 255, 0.05) 1px, transparent 1px),
                linear-gradient(90deg, rgba(0, 255, 255, 0.05) 1px, transparent 1px);
            background-size: 50px 50px;
            pointer-events: none;
            z-index: -1;
        }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@500;700&display=swap" rel="stylesheet">
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/holistic@0.5/holistic.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
</head>
<body>
    <div class="grid-bg"></div>
    <h1>NEURAL GESTURE SYSTEM</h1>
    <p class="subtitle">Real-time Body Language to Speech Recognition</p>

    <div class="info">
        <strong>Active Gestures (English Voice):</strong><br>
        • Raise hand → "Hello"<br>
        • Wave hand → "Hello"<br>
        • Point to self → "Me"<br>
        • Thumbs Up → "Good"<br>
        • Thumbs Down → "Bad"<br>
        • OK sign → "Okay"<br>
        • Peace (V) → "Peace"<br>
        • Open palm → "Stop"
    </div>

    <video id="video" autoplay muted playsinline></video>
    <canvas id="canvas" width="720" height="540"></canvas>
    <br>
    <button onclick="startCamera()">⚡ ACTIVATE CAMERA</button>
    <p id="status">SYSTEM READY</p>

    <script>
        const video = document.getElementById('video');
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const status = document.getElementById('status');

        let lastGestureTime = 0;
        const gestureCooldown = 2000;
        let prevWristX = null;

        const holistic = new Holistic({
            locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/holistic@0.5/${file}`
        });

        holistic.setOptions({
            modelComplexity: 1,
            smoothLandmarks: true,
            minDetectionConfidence: 0.5,
            minTrackingConfidence: 0.5
        });

        holistic.onResults(onResults);

        const camera = new Camera(video, {
            onFrame: async () => {
                await holistic.send({ image: video });
            },
            width: 720,
            height: 540
        });

        function startCamera() {
            camera.start();
            status.textContent = "SYSTEM ONLINE - DETECTING GESTURES";
        }

        function speak(text) {
            const utterance = new SpeechSynthesisUtterance(text);
            utterance.volume = 1;
            utterance.rate = 1.0;
            utterance.pitch = 1.1;
            utterance.lang = 'en-US';
            speechSynthesis.speak(utterance);
        }

        function onResults(results) {
            ctx.save();
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            ctx.drawImage(results.image, 0, 0, canvas.width, canvas.height);

            // Neon landmark-lar
            if (results.rightHandLandmarks) {
                drawConnectors(ctx, results.rightHandLandmarks, HAND_CONNECTIONS, { color: '#00ffff', lineWidth: 6 });
                drawLandmarks(ctx, results.rightHandLandmarks, { color: '#ff00ff', lineWidth: 3 });
            }
            if (results.leftHandLandmarks) {
                drawConnectors(ctx, results.leftHandLandmarks, HAND_CONNECTIONS, { color: '#00ffff', lineWidth: 6 });
                drawLandmarks(ctx, results.leftHandLandmarks, { color: '#ff00ff', lineWidth: 3 });
            }
            if (results.poseLandmarks) {
                drawConnectors(ctx, results.poseLandmarks, POSE_CONNECTIONS, { color: '#b388ff', lineWidth: 5 });
                drawLandmarks(ctx, results.poseLandmarks, { color: '#00ffff', lineWidth: 3 });
            }

            let gesture = null;
            const currentTime = Date.now();

            if (results.rightHandLandmarks && results.poseLandmarks) {
                const hand = results.rightHandLandmarks;
                const pose = results.poseLandmarks;

                const thumbTip = hand[4];
                const indexTip = hand[8];
                const middleTip = hand[12];
                const ringTip = hand[16];
                const pinkyTip = hand[20];
                const wrist = hand[0];

                if (wrist.y < pose[12].y - 0.2) {
                    gesture = "Hello";
                } else if (prevWristX !== null && Math.abs(wrist.x - prevWristX) > 0.07) {
                    gesture = "Hello";
                }
                prevWristX = wrist.x;

                if (Math.abs(indexTip.x - pose[0].x) < 0.15 && indexTip.y > pose[0].y - 0.1) {
                    gesture = "Me";
                }

                if (thumbTip.y < hand[3].y && indexTip.y > hand[6].y && middleTip.y > hand[10].y && ringTip.y > hand[14].y && pinkyTip.y > hand[18].y) {
                    gesture = "Good";
                }

                if (thumbTip.y > hand[3].y && indexTip.y > hand[6].y && middleTip.y > hand[10].y) {
                    gesture = "Bad";
                }

                if (Math.hypot(thumbTip.x - indexTip.x, thumbTip.y - indexTip.y) < 0.08 && middleTip.y > hand[10].y && ringTip.y > hand[14].y && pinkyTip.y > hand[18].y) {
                    gesture = "Okay";
                }

                if (indexTip.y < hand[6].y && middleTip.y < hand[10].y && ringTip.y > hand[14].y && pinkyTip.y > hand[18].y && thumbTip.y > hand[2].y) {
                    gesture = "Peace";
                }

                if (indexTip.y < hand[6].y && middleTip.y < hand[10].y && ringTip.y < hand[14].y && pinkyTip.y < hand[18].y && thumbTip.y < hand[2].y) {
                    gesture = "Stop";
                }
            }

            if (gesture && (currentTime - lastGestureTime) > gestureCooldown) {
                speak(gesture);
                lastGestureTime = currentTime;
                status.textContent = `>> ${gesture.toUpperCase()} DETECTED <<`;
            } else if (!gesture) {
                status.textContent = "SCANNING FOR GESTURES...";
            }

            ctx.restore();
        }
    </script>
</body>
</html>
