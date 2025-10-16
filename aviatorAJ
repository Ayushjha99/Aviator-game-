<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Aviator Crash Simulator</title>
    <!-- Load Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;700;900&display=swap');
        body { font-family: 'Inter', sans-serif; }
        /* Custom styling for the crash graph background and grid lines */
        #gameCanvas {
            background-color: #1a1a2e; /* Dark blue background */
            border-radius: 12px;
            box-shadow: 0 4px 15px rgba(0, 0, 0, 0.5);
            touch-action: none; /* Disable default touch actions */
        }
        .text-neon {
            text-shadow: 0 0 5px rgba(255, 255, 255, 0.8), 0 0 15px #0ff;
        }
    </style>
</head>
<body class="bg-gray-900 text-white min-h-screen flex items-center justify-center p-4">

    <!-- Firebase SDK Imports (MANDATORY) -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, onSnapshot, getDoc, collection, setLogLevel } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Global variables provided by the Canvas environment
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-aviator-app-id';
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');

        // Set Firebase Log Level
        setLogLevel('Debug');

        // Initialize Firebase
        let app, db, auth, userId;
        try {
            app = initializeApp(firebaseConfig);
            db = getFirestore(app);
            auth = getAuth(app);
        } catch (error) {
            console.error("Firebase initialization failed:", error);
        }

        let isAuthReady = false;

        // Function to set up Firebase and Authentication
        async function setupFirebase() {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
                console.log("Signed in with custom token.");
            } else {
                await signInAnonymously(auth);
                console.log("Signed in anonymously.");
            }

            onAuthStateChanged(auth, (user) => {
                if (user) {
                    userId = user.uid;
                    isAuthReady = true;
                    console.log("User ID:", userId);
                    window.init(db, userId, appId); // Initialize the game with DB references
                } else {
                    console.log("No user signed in.");
                    // In a simulation, use a temp ID if auth fails
                    userId = crypto.randomUUID();
                    isAuthReady = true;
                    window.init(null, userId, appId);
                }
            });
        }

        // Expose setup to the global scope for the main script
        window.setupFirebase = setupFirebase;
        
    </script>


    <!-- Main Game Container -->
    <div id="app" class="w-full max-w-4xl bg-gray-800 p-6 md:p-8 rounded-xl shadow-2xl">
        <h1 class="text-4xl font-black text-center mb-6 text-yellow-400">AVIATOR CRASH SIMULATOR</h1>
        
        <!-- User Stats and History -->
        <div class="flex flex-col md:flex-row justify-between items-center mb-6 space-y-4 md:space-y-0">
            <div id="balanceDisplay" class="text-2xl font-bold p-3 rounded-lg bg-green-900 shadow-md">Balance: $1000.00</div>
            <div id="historyDisplay" class="text-lg text-gray-400 italic">Last Crashes: Loading...</div>
        </div>

        <!-- The Game Canvas -->
        <div class="relative mb-6">
            <canvas id="gameCanvas" width="800" height="400" class="w-full h-auto"></canvas>
            <!-- Multiplier Overlay -->
            <div id="multiplierOverlay" class="absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 text-center">
                <p id="currentMultiplier" class="text-7xl md:text-8xl font-black text-neon">1.00x</p>
                <p id="gameStateText" class="text-2xl font-bold mt-2 text-yellow-500">PLACE YOUR BET</p>
            </div>
            <!-- User ID Display (Mandatory for Multi-user apps) -->
            <div id="userIdDisplay" class="absolute top-2 left-2 text-xs text-gray-500 bg-gray-900/50 p-1 rounded">User ID: N/A</div>
        </div>

        <!-- Controls and Input -->
        <div class="grid grid-cols-2 gap-4">
            <!-- Bet Input Column -->
            <div class="flex flex-col space-y-3">
                <input type="number" id="betInput" value="10.00" min="1.00" step="1.00" placeholder="Bet Amount"
                       class="w-full p-3 rounded-lg bg-gray-700 text-white border-2 border-gray-600 focus:border-yellow-500 focus:ring-yellow-500 text-lg font-mono">
                
                <button id="placeBetBtn" onclick="window.placeBet()"
                        class="w-full py-4 text-xl font-bold rounded-xl shadow-lg transition duration-200 
                               bg-green-600 hover:bg-green-700 active:bg-green-800 disabled:bg-gray-500">
                    PLACE BET
                </button>
            </div>

            <!-- Cash Out Column -->
            <div class="flex flex-col space-y-3">
                <div id="potentialWinDisplay" class="w-full p-3 text-center rounded-lg bg-gray-700 text-white text-lg font-mono flex items-center justify-center">
                    Win: $0.00
                </div>

                <button id="cashOutBtn" onclick="window.cashOut()" disabled
                        class="w-full py-4 text-xl font-bold rounded-xl shadow-lg transition duration-200 
                               bg-red-600 hover:bg-red-700 active:bg-red-800 disabled:bg-gray-500">
                    CASH OUT
                </button>
            </div>
        </div>
    </div>

    <!-- JavaScript Game Logic and Firebase Persistence -->
    <script type="module">
        // Import firestore references from the imported module
        import { getFirestore, doc, setDoc, onSnapshot, getDoc, collection } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- Game State Variables ---
        let balance = 1000.00;
        let currentBet = 0.00;
        let isPlaying = false;
        let animationTime = 0; // Time in milliseconds since round start
        let animationFrameId = null;
        let targetCrashPoint = 1.00;
        let multiplier = 1.00;

        // --- DOM Elements ---
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const multiplierEl = document.getElementById('currentMultiplier');
        const balanceEl = document.getElementById('balanceDisplay');
        const betInputEl = document.getElementById('betInput');
        const placeBetBtn = document.getElementById('placeBetBtn');
        const cashOutBtn = document.getElementById('cashOutBtn');
        const winDisplayEl = document.getElementById('potentialWinDisplay');
        const stateTextEl = document.getElementById('gameStateText');
        const historyDisplayEl = document.getElementById('historyDisplay');
        const userIdDisplayEl = document.getElementById('userIdDisplay');

        // --- Firebase/Firestore Variables ---
        let db, currentUserId, currentAppId;
        let balanceDocRef;
        let historyCollectionRef;
        const HISTORY_LIMIT = 5;

        // --- Utility Functions ---

        // 1. Deterministic Crash Point Calculation (Simplified Provably Fair concept)
        // Uses a seed to determine the outcome before the round starts
        // C = (100 / (1 - R)) / 100. Most crash points are low.
        function calculateCrashPoint(seed) {
            // Using a simple deterministic calculation based on the seed
            const pseudoRand = (Math.sin(seed) * 10000) % 1;
            let R = Math.abs(pseudoRand);

            // House edge adjustment: Ensure R is never 1 (infinite multiplier)
            if (R >= 0.9999) R = 0.9999; 

            // Common crash game formula: Most results are low, but high results are possible
            let C = 100 / (1 - R); 
            
            // Apply a modest house edge (e.g., 1%) by ensuring minimum multiplier of 1.01x
            // And then limit high numbers for simulation stability
            let crashPoint = Math.max(101, Math.floor(C)) / 100;
            
            // Limit to a reasonable max for this simulation
            return Math.min(20.00, crashPoint); 
        }

        // 2. Real-time Multiplier Function (Exponential Growth)
        // This function determines the value of the multiplier at a given time (t)
        function getMultiplier(t) {
            // Formula for exponential curve: M = 1.00 + 0.0001 * t^2
            // This makes the growth accelerate slightly
            // t is in milliseconds. 
            const T_SECONDS = t / 1000;
            return 1.00 + 0.0005 * Math.pow(T_SECONDS, 2);
        }

        // --- Firestore Persistence Functions ---

        /** Initializes the game by setting up Firestore and loading/listening for balance. */
        window.init = (firestoreDb, userId, appId) => {
            currentUserId = userId;
            currentAppId = appId;
            db = firestoreDb;
            userIdDisplayEl.textContent = `User ID: ${userId}`;

            if (!db) {
                console.error("Firestore DB is not available. Using local balance.");
                updateUI();
                setTimeout(startNewRound, 3000); // Start round locally
                return;
            }
            
            // Private data path for user balance
            balanceDocRef = doc(db, `artifacts/${currentAppId}/users/${currentUserId}/data/aviator_balance/${currentUserId}`);
            // Public data path for global history
            historyCollectionRef = collection(db, `artifacts/${currentAppId}/public/data/aviator_history`);

            // 1. Set up real-time listener for user balance
            onSnapshot(balanceDocRef, (docSnap) => {
                if (docSnap.exists()) {
                    balance = docSnap.data().value || 1000.00;
                    console.log("Balance updated from Firestore:", balance);
                } else {
                    console.log("No balance found, initializing...");
                    // Initialize balance if document doesn't exist
                    setDoc(balanceDocRef, { value: 1000.00, lastUpdated: new Date() });
                }
                updateUI();
                // Only start the first round once balance is loaded
                if (!isPlaying) {
                     setTimeout(startNewRound, 3000); 
                }
            });

            // 2. Set up real-time listener for game history
            onSnapshot(historyCollectionRef, (snapshot) => {
                const history = [];
                snapshot.docs.forEach(doc => {
                    history.push(doc.data());
                });
                
                // Sort by timestamp and take the last few
                history.sort((a, b) => b.timestamp - a.timestamp);
                const recentHistory = history.slice(0, HISTORY_LIMIT).map(h => {
                    // Truncate to 2 decimal places
                    return Math.floor(h.crashPoint * 100) / 100;
                });

                historyDisplayEl.textContent = `Last Crashes: ${recentHistory.join('x, ')}x`;
            });
        };

        /** Updates the balance in Firestore. */
        async function saveBalance() {
            if (!db) return;
            try {
                await setDoc(balanceDocRef, { 
                    value: Math.floor(balance * 100) / 100, 
                    lastUpdated: new Date(),
                    userId: currentUserId
                });
            } catch (error) {
                console.error("Error saving balance:", error);
            }
        }

        /** Saves the round result to the public history collection. */
        async function saveCrashPoint(crashPoint) {
            if (!db) return;
            try {
                const newDocRef = doc(historyCollectionRef); // Firestore will auto-generate ID
                await setDoc(newDocRef, {
                    crashPoint: crashPoint,
                    timestamp: Date.now()
                });
            } catch (error) {
                console.error("Error saving crash point to history:", error);
            }
        }


        // --- Game Logic Functions ---

        /** Updates all UI elements */
        function updateUI() {
            balanceEl.innerHTML = `Balance: <span class="text-yellow-300">$${balance.toFixed(2)}</span>`;
            
            // Update potential win display
            if (currentBet > 0 && isPlaying) {
                const potentialWin = currentBet * multiplier;
                winDisplayEl.innerHTML = `Win: <span class="text-green-300">$${potentialWin.toFixed(2)}</span>`;
            } else {
                 winDisplayEl.innerHTML = `Win: <span class="text-gray-400">$0.00</span>`;
            }
        }

        /** Resets and prepares for a new round */
        function startNewRound() {
            if (isPlaying) return;
            
            // Reset state
            multiplier = 1.00;
            animationTime = 0;
            
            // Determine the next crash point (pseudo-randomly generated)
            // Use a new seed for each round (e.g., current timestamp)
            targetCrashPoint = calculateCrashPoint(Date.now() + Math.random());
            console.log(`Round starting. Target Crash Point: ${targetCrashPoint.toFixed(2)}x`);

            // Update UI for waiting state
            multiplierEl.textContent = '1.00x';
            multiplierEl.classList.remove('text-red-500', 'text-green-500');
            multiplierEl.classList.add('text-neon');
            stateTextEl.textContent = 'PLACE YOUR BET';

            // Enable controls
            placeBetBtn.disabled = false;
            cashOutBtn.disabled = true;
            betInputEl.disabled = false;
        }

        /** Called when the user clicks PLACE BET */
        window.placeBet = () => {
            const betAmount = parseFloat(betInputEl.value);

            if (isNaN(betAmount) || betAmount <= 0) {
                stateTextEl.textContent = "Enter a valid bet!";
                return;
            }

            if (betAmount > balance) {
                stateTextEl.textContent = "Insufficient Balance!";
                return;
            }

            // Deduct bet from balance
            balance -= betAmount;
            currentBet = betAmount;
            
            saveBalance();
            updateUI();
            
            // Lock controls
            placeBetBtn.disabled = true;
            betInputEl.disabled = true;
            cashOutBtn.disabled = false;
            
            // Start the game animation after a short delay
            stateTextEl.textContent = 'LIFT OFF...';
            setTimeout(runGame, 1000);
        }

        /** Starts the animation loop */
        function runGame() {
            isPlaying = true;
            stateTextEl.textContent = 'FLYING...';
            animationTime = 0;
            animationFrameId = requestAnimationFrame(animate);
        }

        /** Called when the user clicks CASH OUT */
        window.cashOut = () => {
            if (!isPlaying || currentBet <= 0) return;

            const winnings = currentBet * multiplier;
            balance += winnings;

            // Display success message
            stateTextEl.textContent = `CASHOUT @ ${multiplier.toFixed(2)}x! Win: $${winnings.toFixed(2)}`;
            multiplierEl.classList.remove('text-neon');
            multiplierEl.classList.add('text-green-500');
            
            endRound(true);

            // Save the balance
            saveBalance();
        }

        /** Ends the current round (either by crash or cashout) */
        function endRound(didCashOut) {
            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId);
            }
            isPlaying = false;
            cashOutBtn.disabled = true;
            currentBet = 0.00;
            updateUI();

            // The multiplier value at the moment of the crash is what gets saved to history
            const finalMultiplier = didCashOut ? multiplier : targetCrashPoint;
            saveCrashPoint(finalMultiplier);
            
            // Wait a moment and start a new round
            setTimeout(startNewRound, 5000);
        }


        // --- Animation and Canvas Drawing ---
        
        const graphStart = { x: 0, y: canvas.height };
        let points = []; // Stores historical points for the graph

        function drawGraph() {
            // Adjust canvas dimensions for responsiveness
            const containerWidth = canvas.clientWidth;
            canvas.width = containerWidth;
            canvas.height = containerWidth * 0.5; // Maintain 2:1 aspect ratio
            const W = canvas.width;
            const H = canvas.height;

            // Clear the canvas
            ctx.clearRect(0, 0, W, H);
            
            // Draw Grid Lines (simplified)
            ctx.strokeStyle = '#33334d'; // Darker grid color
            ctx.lineWidth = 1;

            // Draw X-axis lines (Time/Horizontal)
            for (let i = 0; i <= 10; i++) {
                let x = W / 10 * i;
                ctx.beginPath();
                ctx.moveTo(x, 0);
                ctx.lineTo(x, H);
                ctx.stroke();
            }

            // Draw Y-axis lines (Multiplier/Vertical) - Max 5.0x for visual clarity
            const maxGraphM = 5.0; 
            for (let i = 0; i <= 5; i++) {
                let y = H - (H / 5 * i);
                ctx.beginPath();
                ctx.moveTo(0, y);
                ctx.lineTo(W, y);
                ctx.stroke();
                
                // Draw Multiplier Labels
                ctx.fillStyle = '#6b7280';
                ctx.font = '14px Inter';
                ctx.fillText(`${i}.0x`, 5, y - 5);
            }
            
            // Draw the Multiplier Curve
            ctx.strokeStyle = '#0ff'; // Neon blue/cyan line
            ctx.lineWidth = 3;
            ctx.beginPath();
            ctx.moveTo(graphStart.x, graphStart.y);

            // Normalize the multiplier to the canvas height
            function getY(m) {
                // If multiplier exceeds maxGraphM, clamp it to the top
                const normalizedM = Math.min(m, maxGraphM); 
                return H - (normalizedM / maxGraphM) * H;
            }
            
            // Normalize time to the canvas width (Max graph time is 10 seconds for full width)
            const maxGraphTime = 10000; // 10 seconds

            // Calculate current graph position
            const timeRatio = Math.min(animationTime, maxGraphTime) / maxGraphTime;
            const currentX = timeRatio * W;
            const currentY = getY(multiplier);

            // Add new point
            if (isPlaying) {
                 points.push({ x: currentX, y: currentY });
            } else if (points.length === 0) {
                // Initial point
                points.push(graphStart);
            }
            
            // Draw the line from the points array
            points.forEach(point => {
                ctx.lineTo(point.x, point.y);
            });
            ctx.stroke();

            // Draw the current position (the 'plane')
            if (isPlaying) {
                // Draw the airplane marker (a simple circle or emoji)
                ctx.fillStyle = '#fff';
                ctx.beginPath();
                ctx.arc(currentX, currentY, 6, 0, Math.PI * 2);
                ctx.fill();

                // Draw the emoji (for fun)
                ctx.font = '24px Inter';
                ctx.fillText('✈️', currentX + 10, currentY - 10);
            }
            
            // Cleanup points if the round ended
            if (!isPlaying && points.length > 0 && points[points.length - 1].x !== 0) {
                points = []; // Clear for the next round
            }
        }

        /** Main animation loop */
        function animate(timestamp) {
            if (!animationTime) {
                animationTime = timestamp;
            }
            const deltaTime = timestamp - animationTime;
            animationTime = timestamp;

            // 1. Update Multiplier
            const newMultiplier = getMultiplier(deltaTime);
            
            // Check for crash condition
            if (newMultiplier >= targetCrashPoint) {
                multiplier = targetCrashPoint; // Set the multiplier exactly to the crash point
                multiplierEl.textContent = `${multiplier.toFixed(2)}x`;
                multiplierEl.classList.remove('text-neon');
                multiplierEl.classList.add('text-red-500');
                stateTextEl.textContent = `CRASHED @ ${multiplier.toFixed(2)}x! You Lost.`;
                endRound(false);
                return;
            }

            // If not crashed, update the multiplier and UI
            multiplier = newMultiplier;
            multiplierEl.textContent = `${multiplier.toFixed(2)}x`;
            
            // Update potential winnings
            updateUI();

            // 2. Draw Graph
            drawGraph();

            // 3. Loop
            animationFrameId = requestAnimationFrame(animate);
        }

        // --- Initialization ---
        
        // This is the starting point of the HTML file's script
        window.addEventListener('load', () => {
             // 1. Setup Firebase/Auth
             if (window.setupFirebase) {
                window.setupFirebase();
             } else {
                 console.error("Firebase setup function missing.");
                 // Fallback to local game logic if module setup failed
                 window.init(null, crypto.randomUUID(), 'default-id');
             }

             // Initial draw of the canvas
             drawGraph();
        });
    </script>
</body>
</html>
