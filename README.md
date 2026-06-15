<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>OmniVision AI - Motion, Symmetry & Face Analytics</title>
    
    <script src="https://cdn.jsdelivr.net/npm/@tailwindcss/browser@4"></script>
    
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/pose/pose.js" crossorigin="anonymous"></script>
    
    <script defer src="https://cdn.jsdelivr.net/npm/@vladmandic/face-api/dist/face-api.js"></script>

    <style>
        .glass {
            background: rgba(255, 255, 255, 0.05);
            backdrop-filter: blur(10px);
            border: 1px solid rgba(255, 255, 255, 0.1);
        }
        body {
            background: radial-gradient(circle at center, #111827 0%, #030712 100%);
        }
    </style>
</head>
<body class="text-gray-100 min-h-screen font-sans antialiased">

    <header class="border-b border-gray-800 bg-gray-950/50 backdrop-blur px-6 py-4 flex justify-between items-center sticky top-0 z-50">
        <div class="flex items-center gap-3">
            <div class="h-4 w-4 rounded-full bg-emerald-500 animate-pulse"></div>
            <h1 class="text-xl font-bold tracking-wider bg-gradient-to-r from-emerald-400 to-cyan-400 bg-clip-text text-transparent">OMNIVISION AI</h1>
        </div>
        <div id="status-badge" class="text-xs bg-amber-500/10 text-amber-400 px-3 py-1 rounded-full border border-amber-500/20 font-medium">
            Initializing AI Models...
        </div>
    </header>

    <main class="max-w-7xl mx-auto p-4 lg:p-6 grid grid-cols-1 lg:grid-cols-12 gap-6">
        
        <div class="lg:col-span-7 flex flex-col gap-4">
            <div class="relative rounded-2xl overflow-hidden shadow-2xl border border-gray-800 bg-black aspect-video flex items-center justify-center">
                <video id="webcam" autoplay playsinline muted class="hidden"></video>
                <canvas id="output_canvas" class="w-full h-full object-cover"></canvas>
                
                <div id="loading-overlay" class="absolute inset-0 bg-gray-950/90 flex flex-col items-center justify-center gap-4 transition-opacity duration-500">
                    <div class="w-12 h-12 border-4 border-emerald-500/30 border-t-emerald-400 rounded-full animate-spin"></div>
                    <p class="text-gray-400 text-sm font-medium">Downloading neural weights (~5MB)...</p>
                </div>
            </div>
            
            <div class="glass rounded-xl p-4 flex gap-4 justify-between items-center">
                <div>
                    <h3 class="text-sm font-semibold text-gray-300">Camera Controls</h3>
                    <p class="text-xs text-gray-500">Ensure good lighting for better feature extraction.</p>
                </div>
                <button id="toggle-cam" disabled class="px-5 py-2.5 bg-emerald-600 hover:bg-emerald-500 disabled:bg-gray-800 disabled:text-gray-600 font-medium rounded-lg text-sm transition-all shadow-lg shadow-emerald-900/20 cursor-pointer">
                    Start Camera
                </button>
            </div>
        </div>

        <div class="lg:col-span-5 flex flex-col gap-6">
            
            <div class="glass rounded-2xl p-5 flex flex-col gap-4">
                <h2 class="text-sm font-bold text-emerald-400 uppercase tracking-widest border-b border-gray-800 pb-2">Motion & Symmetry</h2>
                <div class="grid grid-cols-2 gap-4">
                    <div class="bg-gray-900/40 p-3 rounded-xl border border-gray-800/60">
                        <span class="text-xs text-gray-400 block mb-1">Motion Delta</span>
                        <span id="motion-val" class="text-xl font-mono font-bold text-cyan-400">0.00%</span>
                        <div class="w-full bg-gray-800 h-1.5 rounded-full mt-2 overflow-hidden">
                            <div id="motion-bar" class="bg-cyan-400 h-full w-0 transition-all duration-100"></div>
                        </div>
                    </div>
                    <div class="bg-gray-900/40 p-3 rounded-xl border border-gray-800/60">
                        <span class="text-xs text-gray-400 block mb-1">Body Alignment</span>
                        <span id="symmetry-val" class="text-xl font-mono font-bold text-purple-400">Perfect</span>
                    </div>
                </div>
                <div class="space-y-2 text-xs text-gray-400">
                    <div class="flex justify-between"><p>Shoulder Tilt Alignment:</p><span id="shoulder-tilt" class="font-mono text-gray-200">0°</span></div>
                    <div class="flex justify-between"><p>Hip Base Alignment:</p><span id="hip-tilt" class="font-mono text-gray-200">0°</span></div>
                </div>
            </div>

            <div class="glass rounded-2xl p-5 flex flex-col gap-4">
                <h2 class="text-sm font-bold text-cyan-400 uppercase tracking-widest border-b border-gray-800 pb-2">Facial Specs & Demographics</h2>
                <div class="grid grid-cols-2 gap-4">
                    <div class="bg-gray-900/40 p-3 rounded-xl border border-gray-800/60">
                        <span class="text-xs text-gray-400 block mb-1">Estimated Age</span>
                        <span id="age-val" class="text-2xl font-bold text-emerald-400">--</span>
                        <span class="text-[10px] text-gray-500 block mt-1">±5 years variance</span>
                    </div>
                    <div class="bg-gray-900/40 p-3 rounded-xl border border-gray-800/60">
                        <span class="text-xs text-gray-400 block mb-1">Primary Expression</span>
                        <span id="expression-val" class="text-lg font-bold text-amber-400 capitalize">--</span>
                    </div>
                </div>

                <div class="space-y-3">
                    <h3 class="text-xs font-semibold text-gray-400 uppercase tracking-wider">Expression Probabilities</h3>
                    <div id="expression-bars" class="space-y-2">
                        <div class="text-xs text-gray-500 italic">Awaiting face mesh detection...</div>
                    </div>
                </div>
            </div>

        </div>
    </main>

    <script>
        const videoElement = document.getElementById('webcam');
        const canvasElement = document.getElementById('output_canvas');
        const canvasCtx = canvasElement.getContext('2d');
        
        // UI Elements
        const statusBadge = document.getElementById('status-badge');
        const toggleCamBtn = document.getElementById('toggle-cam');
        const loadingOverlay = document.getElementById('loading-overlay');
        
        let poseModel, faceModelsLoaded = false;
        let lastLandmarks = null;

        // Initialize Face-API Models from Public Unpkg CDN CDN
        async function loadFaceAPI() {
            const MODEL_URL = 'https://cdn.jsdelivr.net/npm/@vladmandic/face-api/model/';
            try {
                await faceapi.nets.tinyFaceDetector.loadFromUri(MODEL_URL);
                await faceapi.nets.faceLandmark68Net.loadFromUri(MODEL_URL);
                await faceapi.nets.faceExpressionNet.loadFromUri(MODEL_URL);
                await faceapi.nets.faceRecognitionNet.loadFromUri(MODEL_URL);
                await faceapi.nets.ageGenderNet.loadFromUri(MODEL_URL);
                faceModelsLoaded = true;
                checkAllModelsReady();
            } catch (err) {
                console.error("Face-API failed to load: ", err);
                statusBadge.innerText = "Error loading Face Metrics";
            }
        }

        // Initialize MediaPipe Pose Model
        function initMediaPipePose() {
            poseModel = new Pose({
                locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/pose/${file}`
            });
            poseModel.setOptions({
                modelComplexity: 1,
                smoothLandmarks: true,
                enableSegmentation: false,
                smoothSegmentation: false,
                minDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });
            poseModel.onResults(onPoseResults);
            checkAllModelsReady();
        }

        function checkAllModelsReady() {
            if(faceModelsLoaded && poseModel) {
                statusBadge.innerText = "System Ready";
                statusBadge.className = "text-xs bg-emerald-500/10 text-emerald-400 px-3 py-1 rounded-full border border-emerald-500/20 font-medium";
                loadingOverlay.classList.add('opacity-0');
                setTimeout(() => loadingOverlay.remove(), 500);
                toggleCamBtn.disabled = false;
            }
        }

        // Calculate Alignment & Symmetry Metrics
        function calculateSymmetry(landmarks) {
            // Landmarks 11, 12 are Left/Right shoulders. 23, 24 are Left/Right hips
            const leftShoulder = landmarks[11];
            const rightShoulder = landmarks[12];
            const leftHip = landmarks[23];
            const rightHip = landmarks[24];

            // Calculate tilt angle in degrees using basic trigonometry 
            const shoulderTilt = Math.abs(Math.atan2(rightShoulder.y - leftShoulder.y, rightShoulder.x - leftShoulder.x) * (180 / Math.PI));
            const hipTilt = Math.abs(Math.atan2(rightHip.y - leftHip.y, rightHip.x - leftHip.x) * (180 / Math.PI));

            document.getElementById('shoulder-tilt').innerText = `${shoulderTilt.toFixed(1)}°`;
            document.getElementById('hip-tilt').innerText = `${hipTilt.toFixed(1)}°`;

            if (shoulderTilt > 5 || hipTilt > 5) {
                document.getElementById('symmetry-val').innerText = "Imbalanced";
                document.getElementById('symmetry-val').className = "text-xl font-mono font-bold text-rose-400";
            } else {
                document.getElementById('symmetry-val').innerText = "Aligned";
                document.getElementById('symmetry-val').className = "text-xl font-mono font-bold text-emerald-400";
            }
        }

        // Calculate frame-by-frame movement differential
        function calculateMotion(currentLandmarks) {
            if (!lastLandmarks) {
                lastLandmarks = currentLandmarks;
                return;
            }
            let totalDiff = 0;
            let activePoints = 0;

            // Compute distance variance across critical frame markers
            currentLandmarks.forEach((point, idx) => {
                if (point.visibility > 0.5) {
                    const prev = lastLandmarks[idx];
                    const diff = Math.sqrt(Math.pow(point.x - prev.x, 2) + Math.pow(point.y - prev.y, 2));
                    totalDiff += diff;
                    activePoints++;
                }
            });

            if (activePoints > 0) {
                const motionPercentage = Math.min((totalDiff / activePoints) * 1000, 100);
                document.getElementById('motion-val').innerText = `${motionPercentage.toFixed(2)}%`;
                document.getElementById('motion-bar').style.width = `${motionPercentage}%`;
            }
            lastLandmarks = currentLandmarks;
        }

        // Pipeline callback for Pose Tracking
        function onPoseResults(results) {
            // Resize canvas context dynamically based on output
            if(canvasElement.width !== videoElement.videoWidth) {
                canvasElement.width = videoElement.videoWidth;
                canvasElement.height = videoElement.videoHeight;
            }

            canvasCtx.save();
            canvasCtx.clearRect(0, 0, canvasElement.width, canvasElement.height);
            
            // Draw mirror image webcam viewport background frame
            canvasCtx.drawImage(results.image, 0, 0, canvasElement.width, canvasElement.height);

            if (results.poseLandmarks) {
                calculateMotion(results.poseLandmarks);
                calculateSymmetry(results.poseLandmarks);

                // Draw skeletons using custom tracking markers
                results.poseLandmarks.forEach((landmark) => {
                    if(landmark.visibility > 0.5) {
                        canvasCtx.beginPath();
                        canvasCtx.arc(landmark.x * canvasElement.width, landmark.y * canvasElement.height, 4, 0, 2 * Math.PI);
                        canvasCtx.fillStyle = '#10b981';
                        canvasCtx.fill();
                    }
                });
            }
            canvasCtx.restore();
        }

        // Face Analytics Frame Loop Function
        async function runFaceAnalysis() {
            if (!videoElement.paused && !videoElement.ended) {
                const detections = await faceapi.detectAllFaces(videoElement, new faceapi.TinyFaceDetectorOptions())
                    .withFaceLandmarks()
                    .withFaceExpressions()
                    .withAgeAndGender();

                if (detections && detections.length > 0) {
                    const primaryFace = detections[0];
                    
                    // Render age mapping metrics
                    document.getElementById('age-val').innerText = `${Math.round(primaryFace.age)} yrs`;
                    
                    // Sort expressions by confidence rating
                    const expressions = primaryFace.expressions;
                    const sorted = Object.entries(expressions).sort((a,b) => b[1] - a[1]);
                    document.getElementById('expression-val').innerText = sorted[0][0];

                    // Render expression strength dynamic bars
                    const barsContainer = document.getElementById('expression-bars');
                    barsContainer.innerHTML = '';
                    sorted.slice(0, 4).forEach(([expr, val]) => {
                        const pct = (val * 100).toFixed(0);
                        barsContainer.innerHTML += `
                            <div>
                                <div class="flex justify-between text-xs text-gray-400 mb-1">
                                    <span class="capitalize">${expr}</span>
                                    <span>${pct}%</span>
                                </div>
                                <div class="w-full bg-gray-800/80 h-1 rounded-full overflow-hidden">
                                    <div class="bg-cyan-500 h-full" style="width: ${pct}%"></div>
                                </div>
                            </div>
                        `;
                    });
                }
            }
            // Continuous scheduling inside microtask execution schedule loop
            setTimeout(() => runFaceAnalysis(), 200);
        }

        // Handle Webcam Thread Hooks
        async function startWebcam() {
            try {
                const stream = await navigator.mediaDevices.getUserMedia({
                    video: { width: 640, height: 480, frameRate: { ideal: 30 } }
                });
                videoElement.srcObject = stream;
                
                // Initialize MediaPipe active loop processor instance
                const camera = new Camera(videoElement, {
                    onFrame: async () => {
                        await poseModel.send({image: videoElement});
                    },
                    width: 640,
                    height: 480
                });
                
                camera.start();
                videoElement.play();
                runFaceAnalysis(); // Start face analysis execution thread block

                toggleCamBtn.innerText = "System Online";
                toggleCamBtn.className = "px-5 py-2.5 bg-cyan-600 font-medium rounded-lg text-sm transition-all text-white cursor-not-allowed shadow-md";
                toggleCamBtn.disabled = true;
            } catch (err) {
                console.error("Webcam streaming access denied:", err);
                alert("Camera permission denied. Please allow access to use the AI engine trackers.");
            }
        }

        // Event Trigger Handlers
        toggleCamBtn.addEventListener('click', startWebcam);

        // Lifecycle Ignition Entry Point Execution Hook
        window.addEventListener('DOMContentLoaded', () => {
            initMediaPipePose();
            loadFaceAPI();
        });
    </script>
</body>
</html>
