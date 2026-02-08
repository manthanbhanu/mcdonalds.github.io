# mcdonalds.github.io
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>McDonald's Drive-Thru Manager (India Edition) - Multi-User</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body { margin: 0; overflow: hidden; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; }
        #canvas-container { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 1; }
        #ui-layer { position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 10; pointer-events: none; }
        .pointer-events-auto { pointer-events: auto; }
        .mcd-red { background-color: #DA291C; }
        .mcd-yellow { background-color: #FFC72C; }
        #joystick-container {
            position: absolute; bottom: 20px; left: 20px; width: 120px; height: 120px;
            background: rgba(255,255,255,0.2); border-radius: 50%; display: none; touch-action: none;
        }
    </style>
</head>
<body class="bg-gray-900">

<div id="canvas-container"></div>

<div id="ui-layer" class="flex flex-col">
    <!-- Role Selection -->
    <div id="role-selection" class="flex-1 flex items-center justify-center pointer-events-auto bg-black bg-opacity-80">
        <div class="bg-white p-8 rounded-2xl shadow-2xl max-w-2xl w-full text-center border-4 border-yellow-400">
            <img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/36/McDonald%27s_Golden_Arches.svg/1200px-McDonald%27s_Golden_Arches.svg.png" alt="Logo" class="w-20 mx-auto mb-4">
            <h1 class="text-4xl font-black mb-2 text-gray-800 uppercase">Multiplayer Drive-Thru</h1>
            <p class="text-red-600 font-bold mb-4">India Edition • Real-time Co-op</p>
            
            <div id="loading-auth" class="text-gray-500 italic mb-4">Connecting to servers...</div>
            
            <div id="role-buttons" class="hidden grid grid-cols-1 md:grid-cols-2 gap-6">
                <button onclick="selectRole('customer')" class="group bg-gray-100 p-6 rounded-xl border-2 border-transparent hover:border-red-600 transition-all">
                    <div class="text-5xl mb-4">🚗</div>
                    <h3 class="text-xl font-bold mb-2">Customer</h3>
                    <p class="text-sm text-gray-600">Drive and order in real-time.</p>
                </button>
                <button onclick="selectRole('employee')" class="group bg-gray-100 p-6 rounded-xl border-2 border-transparent hover:border-yellow-500 transition-all">
                    <div class="text-5xl mb-4">🧑‍🍳</div>
                    <h3 class="text-xl font-bold mb-2">Employee</h3>
                    <p class="text-sm text-gray-600">Manage kitchen and save metrics.</p>
                </button>
            </div>
            <div id="user-id-display" class="mt-4 text-[10px] text-gray-400 font-mono"></div>
        </div>
    </div>

    <!-- HUD -->
    <div id="hud" class="hidden p-4 flex justify-between items-start">
        <div class="flex flex-col gap-2">
            <div class="bg-white p-3 rounded-lg shadow-lg border-l-4 border-red-600 pointer-events-auto">
                <p id="role-display" class="text-xs font-bold text-gray-500 uppercase">Role: Initializing...</p>
                <p id="objective-display" class="text-sm font-medium">Objective: Drive to the first window</p>
            </div>
            <div id="stats-panel" class="bg-black bg-opacity-70 p-3 rounded-lg text-white text-xs space-y-1">
                <p>Lifetime Revenue: ₹<span id="stat-revenue">0</span></p>
                <p>Orders Served: <span id="stat-orders">0</span></p>
                <p>Players Online: <span id="online-count">1</span></p>
            </div>
        </div>
        
        <button onclick="location.reload()" class="bg-white p-2 rounded-full shadow-lg pointer-events-auto hover:bg-gray-100">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6 text-red-600" fill="none" viewBox="0 0 24 24" stroke="currentColor"><path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M4 4v5h.582m15.356 2A8.001 8.001 0 004.582 9m0 0H9m11 11v-5h-.581m0 0a8.003 8.003 0 01-15.357-2m15.357 2H15" /></svg>
        </button>
    </div>

    <!-- Modals -->
    <div id="interaction-modal" class="hidden fixed inset-0 flex items-center justify-center bg-black bg-opacity-50 pointer-events-auto p-4">
        <!-- POS / Ordering -->
        <div id="pos-ui" class="hidden bg-white rounded-2xl w-full max-w-4xl max-h-[90vh] overflow-hidden flex flex-col shadow-2xl">
            <div class="mcd-red p-4 text-white flex justify-between items-center">
                <h2 class="text-xl font-bold flex items-center gap-2"><img src="https://upload.wikimedia.org/wikipedia/commons/thumb/3/36/McDonald%27s_Golden_Arches.svg/1200px-McDonald%27s_Golden_Arches.svg.png" class="w-6"> POS Terminal</h2>
                <button onclick="closeInteraction()" class="hover:text-yellow-400">✕ Close</button>
            </div>
            <div class="flex-1 flex overflow-hidden">
                <div class="flex-1 p-4 overflow-y-auto">
                    <div id="menu-grid" class="grid grid-cols-2 md:grid-cols-3 gap-3"></div>
                </div>
                <div class="w-80 bg-gray-50 border-l p-4 flex flex-col">
                    <h3 class="font-bold border-b mb-2 pb-2">Checkout</h3>
                    <div id="order-items" class="flex-1 overflow-y-auto text-sm space-y-2 mb-4"></div>
                    <div class="border-t pt-2">
                        <div class="flex justify-between font-bold text-lg"><span>Total</span><span>₹<span id="order-total">0</span></span></div>
                        <button id="checkout-btn" onclick="submitOrder()" class="w-full mt-4 mcd-yellow text-red-800 font-bold py-3 rounded-xl disabled:opacity-50">SEND TO KITCHEN</button>
                    </div>
                </div>
            </div>
        </div>

        <!-- Kitchen KDS -->
        <div id="kitchen-ui" class="hidden bg-zinc-900 text-white rounded-xl p-6 w-full max-w-2xl shadow-2xl border-4 border-zinc-700">
            <h2 class="text-2xl font-black mb-4 flex items-center gap-2"><span class="bg-red-600 p-2 rounded uppercase">Kitchen Queue</span></h2>
            <div id="kitchen-queue" class="space-y-4 max-h-[60vh] overflow-y-auto"></div>
            <button onclick="closeInteraction()" class="mt-4 w-full bg-zinc-700 p-2 rounded font-bold">Close Station</button>
        </div>
    </div>
</div>

<script type="module">
    import { initializeApp } from 'https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js';
    import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js';
    import { getFirestore, doc, setDoc, getDoc, updateDoc, collection, onSnapshot, query, addDoc, deleteDoc, serverTimestamp } from 'https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js';

    // --- CONFIG & INIT ---
    const firebaseConfig = JSON.parse(__firebase_config);
    const appId = typeof __app_id !== 'undefined' ? __app_id : 'mcd-drive-thru-india';
    const app = initializeApp(firebaseConfig);
    const auth = getAuth(app);
    const db = getFirestore(app);

    const MCD_MENU = [
        { id: 'mt', name: 'McAloo Tikki', price: 85, emoji: '🍔' },
        { id: 'mv', name: 'McVeggie', price: 155, emoji: '🍔' },
        { id: 'mc', name: 'McChicken', price: 175, emoji: '🍗' },
        { id: 'msp', name: 'McSpicy Chicken', price: 245, emoji: '🌶️' },
        { id: 'fs', name: 'Large Fries', price: 110, emoji: '🍟' },
        { id: 'coke', name: 'Coca-Cola', price: 95, emoji: '🥤' }
    ];

    let state = {
        user: null,
        role: null,
        playerPos: { x: 0, z: 15 },
        carRotation: 0,
        playerVelocity: 0,
        keys: {},
        currentOrder: [],
        metrics: { revenue: 0, served: 0 },
        isInteracting: false,
        otherPlayers: {},
        activeOrders: []
    };

    let scene, camera, renderer, playerMesh;
    const remoteMeshes = {};

    // --- FIREBASE AUTH ---
    const initAuth = async () => {
        try {
            if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                await signInWithCustomToken(auth, __initial_auth_token);
            } else {
                await signInAnonymously(auth);
            }
        } catch (err) { console.error("Auth failed", err); }
    };

    onAuthStateChanged(auth, async (user) => {
        if (user) {
            state.user = user;
            document.getElementById('loading-auth').classList.add('hidden');
            document.getElementById('role-buttons').classList.remove('hidden');
            document.getElementById('user-id-display').innerText = `Logged in: ${user.uid}`;
            
            // Load persistent metrics
            const metricsDoc = await getDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'stats'));
            if (metricsDoc.exists()) {
                state.metrics = metricsDoc.data();
                updateHUD();
            }
            
            // Start sync listeners
            startSync();
        }
    });

    // --- SYNC ENGINE ---
    function startSync() {
        if (!state.user) return;

        // Sync Players
        const playersRef = collection(db, 'artifacts', appId, 'public', 'data', 'players');
        onSnapshot(playersRef, (snapshot) => {
            snapshot.docChanges().forEach(change => {
                const data = change.doc.data();
                const id = change.doc.id;
                if (id === state.user.uid) return;

                if (change.type === 'removed') {
                    if (remoteMeshes[id]) {
                        scene.remove(remoteMeshes[id]);
                        delete remoteMeshes[id];
                    }
                } else {
                    updateRemotePlayer(id, data);
                }
            });
            document.getElementById('online-count').innerText = snapshot.size;
        }, (err) => {});

        // Sync Orders
        const ordersRef = collection(db, 'artifacts', appId, 'public', 'data', 'orders');
        onSnapshot(ordersRef, (snapshot) => {
            state.activeOrders = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
            if (state.isInteracting && document.getElementById('kitchen-ui').offsetParent) {
                updateKitchenQueue();
            }
        }, (err) => {});
    }

    async function updateHeartbeat() {
        if (!state.user || !state.role) return;
        const playerRef = doc(db, 'artifacts', appId, 'public', 'data', 'players', state.user.uid);
        await setDoc(playerRef, {
            role: state.role,
            pos: state.playerPos,
            rot: state.carRotation,
            updated: serverTimestamp()
        }, { merge: true });
    }

    // --- 3D ENGINE ---
    function init3D() {
        scene = new THREE.Scene();
        scene.background = new THREE.Color(0xa0d2eb);
        camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
        renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        document.getElementById('canvas-container').appendChild(renderer.domElement);

        const ambient = new THREE.AmbientLight(0xffffff, 0.7);
        scene.add(ambient);
        const sun = new THREE.DirectionalLight(0xffffff, 0.8);
        sun.position.set(10, 20, 10);
        scene.add(sun);

        const floor = new THREE.Mesh(new THREE.PlaneGeometry(100, 100), new THREE.MeshStandardMaterial({ color: 0x444444 }));
        floor.rotation.x = -Math.PI / 2;
        scene.add(floor);

        // Building
        const shop = new THREE.Mesh(new THREE.BoxGeometry(15, 8, 20), new THREE.MeshStandardMaterial({ color: 0xDA291C }));
        shop.position.set(12, 4, 0);
        scene.add(shop);

        // Lane
        const lane = new THREE.Mesh(new THREE.PlaneGeometry(10, 80), new THREE.MeshStandardMaterial({ color: 0x222222 }));
        lane.rotation.x = -Math.PI/2;
        lane.position.set(-8, 0.05, 0);
        scene.add(lane);

        animate();
    }

    function updateRemotePlayer(id, data) {
        if (!remoteMeshes[id]) {
            const geo = data.role === 'customer' ? new THREE.BoxGeometry(2.5, 1.2, 4.5) : new THREE.BoxGeometry(1, 2, 0.5);
            const mat = new THREE.MeshStandardMaterial({ color: data.role === 'customer' ? 0x00ff00 : 0xffff00 });
            const mesh = new THREE.Mesh(geo, mat);
            scene.add(mesh);
            remoteMeshes[id] = mesh;
        }
        remoteMeshes[id].position.set(data.pos.x, data.role === 'customer' ? 0.6 : 1, data.pos.z);
        remoteMeshes[id].rotation.y = data.rot || 0;
    }

    window.selectRole = function(role) {
        state.role = role;
        document.getElementById('role-selection').classList.add('hidden');
        document.getElementById('hud').classList.remove('hidden');
        document.getElementById('role-display').innerText = `Role: ${role.toUpperCase()}`;
        
        const geo = role === 'customer' ? new THREE.BoxGeometry(2.5, 1.2, 4.5) : new THREE.BoxGeometry(1, 2, 0.5);
        const mat = new THREE.MeshStandardMaterial({ color: role === 'customer' ? 0x3366ff : 0x222222 });
        playerMesh = new THREE.Mesh(geo, mat);
        state.playerPos = role === 'customer' ? { x: -8, z: 30 } : { x: 8, z: 0 };
        scene.add(playerMesh);
        
        populateMenu();
        setInterval(updateHeartbeat, 100);
    };

    function populateMenu() {
        const grid = document.getElementById('menu-grid');
        grid.innerHTML = MCD_MENU.map(i => `
            <button onclick="addToOrder('${i.id}')" class="bg-white border p-4 rounded-xl flex flex-col items-center hover:bg-yellow-50">
                <span class="text-3xl">${i.emoji}</span>
                <span class="font-bold text-xs mt-1">${i.name}</span>
                <span class="text-red-600 font-bold">₹${i.price}</span>
            </button>
        `).join('');
    }

    window.addToOrder = function(id) {
        const item = MCD_MENU.find(i => i.id === id);
        state.currentOrder.push(item);
        updateOrderSummary();
    };

    function updateOrderSummary() {
        const container = document.getElementById('order-items');
        container.innerHTML = state.currentOrder.map(i => `<div class="flex justify-between text-xs"><span>${i.name}</span><b>₹${i.price}</b></div>`).join('');
        document.getElementById('order-total').innerText = state.currentOrder.reduce((a, b) => a + b.price, 0);
    }

    window.submitOrder = async function() {
        if (state.currentOrder.length === 0) return;
        const total = state.currentOrder.reduce((a, b) => a + b.price, 0);
        
        await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'orders'), {
            customerId: state.user.uid,
            items: state.currentOrder.map(i => i.name),
            total: total,
            status: 'pending',
            created: serverTimestamp()
        });
        
        state.currentOrder = [];
        closeInteraction();
        showMessage("Order Sent", "The kitchen has received your order!");
    };

    window.prepareOrder = async function(orderId, total) {
        // Remove from public queue
        await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'orders', orderId));
        
        // Update persistent stats
        state.metrics.revenue += total;
        state.metrics.served += 1;
        await setDoc(doc(db, 'artifacts', appId, 'users', state.user.uid, 'profile', 'stats'), state.metrics);
        
        updateHUD();
        updateKitchenQueue();
        showMessage("Served", `Collected ₹${total}! Stats updated.`);
    };

    function updateKitchenQueue() {
        const q = document.getElementById('kitchen-queue');
        if (state.activeOrders.length === 0) {
            q.innerHTML = '<p class="text-center text-zinc-500 py-10">No active orders</p>';
            return;
        }
        q.innerHTML = state.activeOrders.map(o => `
            <div class="bg-zinc-800 p-4 rounded border-l-4 border-yellow-500 flex justify-between items-center">
                <div><div class="font-bold text-yellow-500">New Order</div><div class="text-xs text-zinc-400">${o.items.join(', ')}</div></div>
                <button onclick="prepareOrder('${o.id}', ${o.total})" class="bg-green-600 px-4 py-2 rounded text-xs font-bold">SERVE (₹${o.total})</button>
            </div>
        `).join('');
    }

    function checkInteractions() {
        if (!state.role) return;
        const p = state.playerPos;
        if (state.role === 'customer') {
            if (p.x < -4 && p.x > -12 && p.z < 10 && p.z > 2) openInteraction('order');
        } else {
            if (p.x > 4 && p.x < 12 && p.z < 5 && p.z > -5) openInteraction('kitchen');
        }
    }

    function openInteraction(type) {
        if (state.isInteracting) return;
        state.isInteracting = true;
        document.getElementById('interaction-modal').classList.remove('hidden');
        if (type === 'order') {
            document.getElementById('pos-ui').classList.remove('hidden');
            document.getElementById('kitchen-ui').classList.add('hidden');
        } else {
            document.getElementById('pos-ui').classList.add('hidden');
            document.getElementById('kitchen-ui').classList.remove('hidden');
            updateKitchenQueue();
        }
    }

    window.closeInteraction = () => {
        state.isInteracting = false;
        document.getElementById('interaction-modal').classList.add('hidden');
    };

    function showMessage(title, text) {
        const div = document.createElement('div');
        div.className = "fixed top-20 left-1/2 transform -translate-x-1/2 bg-white p-4 rounded-xl shadow-2xl z-50 border-b-4 border-yellow-500";
        div.innerHTML = `<b class="text-red-600 block">${title}</b><p class="text-xs">${text}</p>`;
        document.body.appendChild(div);
        setTimeout(() => div.remove(), 3000);
    }

    function updateHUD() {
        document.getElementById('stat-revenue').innerText = state.metrics.revenue;
        document.getElementById('stat-orders').innerText = state.metrics.served;
    }

    function animate() {
        requestAnimationFrame(animate);
        if (state.role && !state.isInteracting) {
            const moveSpeed = state.role === 'customer' ? 0.25 : 0.15;
            if (state.role === 'customer') {
                if (state.keys['w'] || state.keys['ArrowUp']) state.playerVelocity = Math.min(state.playerVelocity + 0.01, 0.4);
                else if (state.keys['s'] || state.keys['ArrowDown']) state.playerVelocity = Math.max(state.playerVelocity - 0.01, -0.2);
                else state.playerVelocity *= 0.95;
                if (Math.abs(state.playerVelocity) > 0.01) {
                    if (state.keys['a'] || state.keys['ArrowLeft']) state.carRotation += 0.04;
                    if (state.keys['d'] || state.keys['ArrowRight']) state.carRotation -= 0.04;
                }
                state.playerPos.x += Math.sin(state.carRotation) * state.playerVelocity;
                state.playerPos.z -= Math.cos(state.carRotation) * state.playerVelocity;
                playerMesh.position.set(state.playerPos.x, 0.6, state.playerPos.z);
                playerMesh.rotation.y = state.carRotation;
                camera.position.set(state.playerPos.x + Math.sin(state.carRotation) * 12, 8, state.playerPos.z + Math.cos(state.carRotation) * 12);
                camera.lookAt(state.playerPos.x, 1, state.playerPos.z);
            } else {
                let dx = 0, dz = 0;
                if (state.keys['w'] || state.keys['ArrowUp']) dz -= 1;
                if (state.keys['s'] || state.keys['ArrowDown']) dz += 1;
                if (state.keys['a'] || state.keys['ArrowLeft']) dx -= 1;
                if (state.keys['d'] || state.keys['ArrowRight']) dx += 1;
                if (dx !== 0 || dz !== 0) {
                    const angle = Math.atan2(dx, dz);
                    state.playerPos.x += Math.sin(angle) * moveSpeed;
                    state.playerPos.z += Math.cos(angle) * moveSpeed;
                    playerMesh.rotation.y = angle;
                }
                state.playerPos.x = Math.max(4, Math.min(state.playerPos.x, 19));
                state.playerPos.z = Math.max(-9, Math.min(state.playerPos.z, 9));
                playerMesh.position.set(state.playerPos.x, 1, state.playerPos.z);
                camera.position.set(10, 18, 12);
                camera.lookAt(8, 0, 0);
            }
            checkInteractions();
        }
        renderer.render(scene, camera);
    }

    window.addEventListener('keydown', e => state.keys[e.key] = true);
    window.addEventListener('keyup', e => state.keys[e.key] = false);
    window.onload = () => { init3D(); initAuth(); };
</script>
</body>
</html>
