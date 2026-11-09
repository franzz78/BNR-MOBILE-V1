/**
 * BNR — Bank Nusantara Roblox
 * Core Engine & Logic
 */

// 1. FIREBASE CONFIGURATION & INITIALIZATION
const firebaseConfig = {
    apiKey: "AIzaSyD9BmV4XKXuMWa4PZHpb7Bbt-rHs61m3lE",
    databaseURL: "https://absensi-polri-default-rtdb.asia-southeast1.firebasedatabase.app",
    projectId: "absensi-polri",
    storageBucket: "absensi-polri.firebasestorage.app",
    messagingSenderId: "19006760644",
    appId: "1:19006760644:web:b980f54aea123e92ed4b91"
};

// Initialize Firebase
if (!firebase.apps.length) {
    firebase.initializeApp(firebaseConfig);
}
const db = firebase.database();
const DB_PREFIX = 'bnr/';

// App State
let currentUser = null;
let currentAccount = null;
let currentView = 'dashboard';
let systemSettings = {
    profileChangeFee: 50000,
    transferFee: 0,
    topUpFee: 0,
    minimumTransfer: 10000,
    maximumTransfer: 50000000,
    dailyTransferLimit: 100000000,
    maintenanceMode: false,
    discordWebhookUrl: "https://discord.com/api/webhooks/1546528727024935032/GDeViczgi1AgNmgsCxoOwg0DIfptXCZlfHDrQ8UWWdPaueNR0xyPjRziRPG37TQ1gDqe"
};
let isBalanceHidden = false;

// 2. INITIALIZATION & LOADING SEQUENCE
document.addEventListener('DOMContentLoaded', () => {
    lucide.createIcons();
    runLoadingSequence();
});

function runLoadingSequence() {
    const progressBar = document.getElementById('progressBar');
    const loadingPercent = document.getElementById('loadingPercent');
    const loadingStatus = document.getElementById('loadingStatus');

    const steps = [
        { percent: 20, status: 'Menghubungkan sistem...' },
        { percent: 45, status: 'Memuat database...' },
        { percent: 70, status: 'Memverifikasi sesi...' },
        { percent: 90, status: 'Menyiapkan aplikasi...' },
        { percent: 100, status: 'Sistem siap.' }
    ];

    let currentStep = 0;

    const interval = setInterval(() => {
        if (currentStep < steps.length) {
            const step = steps[currentStep];
            progressBar.style.width = `${step.percent}%`;
            loadingPercent.textContent = `${step.percent}%`;
            loadingStatus.textContent = step.status;
            currentStep++;
        } else {
            clearInterval(interval);
            setTimeout(() => {
                initAppSystem();
            }, 500);
        }
    }, 300);
}

async function initAppSystem() {
    document.getElementById('loadingScreen').classList.add('hidden');
    
    // Listen to System Settings Realtime
    db.ref(DB_PREFIX + 'settings').on('value', (snapshot) => {
        if (snapshot.exists()) {
            systemSettings = { ...systemSettings, ...snapshot.val() };
        }
        
        if (systemSettings.maintenanceMode && (!currentUser || currentUser.role !== 'SUPER_ADMIN')) {
            showMaintenanceScreen();
        } else {
            document.getElementById('maintenanceScreen').classList.add('hidden');
        }
    });

    // Check Primary Super Admin Existence & Seed if missing
    await ensureSuperAdminExists();

    // Check Saved Session
    const savedUser = localStorage.getItem('bnr_session_user');
    if (savedUser) {
        const userObj = JSON.parse(savedUser);
        // Verify user against DB for security
        const snapshot = await db.ref(DB_PREFIX + 'users/' + userObj.username).once('value');
        if (snapshot.exists() && snapshot.val().status === 'ACTIVE') {
            currentUser = snapshot.val();
            await loadAccountData();
            startRealtimeListeners();
            showMainApp();
            return;
        }
    }
    showAuthView();
}

// 3. HARDCODED PRIMARY SUPER ADMIN SEEDING
async function ensureSuperAdminExists() {
    const adminRef = db.ref(DB_PREFIX + 'users/Haruukii79');
    const snapshot = await adminRef.once('value');
    
    // Hash target password: Dharmawijaya1998#
    const hashedPassword = CryptoJS.SHA256('Dharmawijaya1998#').toString();

    if (!snapshot.exists()) {
        const adminData = {
            name: 'Haruukii Dharmawijaya',
            username: 'Haruukii79',
            password: hashedPassword,
            role: 'SUPER_ADMIN',
            status: 'ACTIVE',
            accountNumber: '1200000001',
            createdAt: firebase.database.ServerValue.TIMESTAMP,
            lastLogin: firebase.database.ServerValue.TIMESTAMP,
            permissions: ['*']
        };

        const accountData = {
            accountNumber: '1200000001',
            username: 'Haruukii79',
            balance: 1000000000,
            status: 'ACTIVE',
            createdAt: firebase.database.ServerValue.TIMESTAMP
        };

        await db.ref(DB_PREFIX + 'users/Haruukii79').set(adminData);
        await db.ref(DB_PREFIX + 'accounts/1200000001').set(accountData);
    } else {
        // Ensure credentials remain locked to specification
        adminRef.update({
            password: hashedPassword,
            role: 'SUPER_ADMIN'
        });
    }
}

// 4. AUTHENTICATION MODULE
function showAuthView() {
    document.getElementById('authScreen').classList.remove('hidden');
    document.getElementById('mainApp').classList.add('hidden');
    document.getElementById('maintenanceScreen').classList.add('hidden');
}

function showMaintenanceScreen() {
    document.getElementById('maintenanceScreen').classList.remove('hidden');
    document.getElementById('authScreen').classList.add('hidden');
    document.getElementById('mainApp').classList.add('hidden');
    
    if (currentUser && currentUser.role === 'SUPER_ADMIN') {
        document.getElementById('maintenanceAdminLink').classList.remove('hidden');
    }
}

function togglePasswordVisibility(inputId, iconId) {
    const input = document.getElementById(inputId);
    const icon = document.getElementById(iconId);
    if (input.type === 'password') {
        input.type = 'text';
        icon.setAttribute('data-lucide', 'eye-off');
    } else {
        input.type = 'password';
        icon.setAttribute('data-lucide', 'eye');
    }
    lucide.createIcons();
}

async function handleLogin(e) {
    e.preventDefault();
    const usernameInput = document.getElementById('loginUsername').value.trim();
    const passwordInput = document.getElementById('loginPassword').value;
    const rememberMe = document.getElementById('rememberMe').checked;

    if (!usernameInput || !passwordInput) {
        Swal.fire('Perhatian', 'Silakan isi username dan password.', 'warning');
        return;
    }

    Swal.fire({
        title: 'Memverifikasi...',
        text: 'Mohon tunggu sebentar',
        allowOutsideClick: false,
        didOpen: () => Swal.showLoading()
    });

    try {
        const userRef = db.ref(DB_PREFIX + 'users/' + usernameInput);
        const snapshot = await userRef.once('value');

        if (!snapshot.exists()) {
            logSecurity('FAILED_LOGIN', usernameInput, 'Username tidak ditemukan');
            Swal.fire('Gagal Masuk', 'Username atau password tidak valid.', 'error');
            return;
        }

        const userData = snapshot.val();
        const hashedInput = CryptoJS.SHA256(passwordInput).toString();

        if (userData.password !== hashedInput) {
            logSecurity('FAILED_LOGIN', usernameInput, 'Password salah');
            Swal.fire('Gagal Masuk', 'Username atau password tidak valid.', 'error');
            return;
        }

        if (userData.status !== 'ACTIVE') {
            logSecurity('SUSPENDED_LOGIN_ATTEMPT', usernameInput, 'Status akun tidak aktif');
            Swal.fire('Akses Ditolak', 'Akun Anda sedang dinonaktifkan atau ditangguhkan.', 'error');
            return;
        }

        if (systemSettings.maintenanceMode && userData.role !== 'SUPER_ADMIN') {
            Swal.fire('Sistem Maintenance', 'Sistem sedang dalam pemeliharaan berkala.', 'info');
            return;
        }

        // Login Success
        currentUser = userData;
        await userRef.update({ lastLogin: firebase.database.ServerValue.TIMESTAMP });

        if (rememberMe) {
            localStorage.setItem('bnr_session_user', JSON.stringify({ username: userData.username }));
        }

        logAudit(userData.username, 'LOGIN', 'User berhasil masuk ke sistem');
        logSecurity('SUCCESSFUL_LOGIN', userData.username, 'Login berhasil');

        await loadAccountData();
        startRealtimeListeners();
        
        Swal.close();
        showMainApp();

    } catch (error) {
        console.error(error);
        Swal.fire('Sistem Kendala', 'Terjadi kesalahan koneksi database.', 'error');
    }
}

async function loadAccountData() {
    if (!currentUser) return;
    const accountRef = db.ref(DB_PREFIX + 'accounts/' + currentUser.accountNumber);
    const snapshot = await accountRef.once('value');
    if (snapshot.exists()) {
        currentAccount = snapshot.val();
    }
}

function startRealtimeListeners() {
    if (!currentUser) return;

    // Realtime Listener Balance & Account
    db.ref(DB_PREFIX + 'accounts/' + currentUser.accountNumber).on('value', (snapshot) => {
        if (snapshot.exists()) {
            currentAccount = snapshot.val();
            if (currentView === 'dashboard') renderDashboardView();
        }
    });

    // Realtime Unread Notifications
    db.ref(DB_PREFIX + 'notifications/' + currentUser.username).on('value', (snapshot) => {
        let unread = false;
        if (snapshot.exists()) {
            const data = snapshot.val();
            Object.values(data).forEach(n => {
                if (!n.read) unread = true;
            });
        }
        const badge = document.getElementById('unreadNotifBadge');
        if (badge) {
            if (unread) badge.classList.remove('hidden');
            else badge.classList.add('hidden');
        }
    });
}

function showMainApp() {
    document.getElementById('authScreen').classList.add('hidden');
    document.getElementById('maintenanceScreen').classList.add('hidden');
    document.getElementById('mainApp').classList.remove('hidden');

    renderSidebarAndNav();
    navigateTo('dashboard');
}

function confirmLogout() {
    Swal.fire({
        title: 'Konfirmasi Keluar',
        text: 'Apakah Anda yakin ingin keluar dari sesi aplikasi BNR?',
        icon: 'question',
        showCancelButton: true,
        confirmButtonText: 'Ya, Keluar',
        cancelButtonText: 'Batal'
    }).then((result) => {
        if (result.isConfirmed) {
            if (currentUser) {
                logAudit(currentUser.username, 'LOGOUT', 'User keluar dari sistem');
            }
            // Cleanup Realtime Listeners
            if (currentUser) {
                db.ref(DB_PREFIX + 'accounts/' + currentUser.accountNumber).off();
                db.ref(DB_PREFIX + 'notifications/' + currentUser.username).off();
            }
            currentUser = null;
            currentAccount = null;
            localStorage.removeItem('bnr_session_user');
            showAuthView();
        }
    });
}

// 5. NAVIGATION & LAYOUT RENDERER
function renderSidebarAndNav() {
    const isSuperAdmin = currentUser && (currentUser.role === 'SUPER_ADMIN' || currentUser.role === 'ADMIN');
    
    // Update Profile Info in Sidebar
    document.getElementById('sidebarUserName').textContent = currentUser.name;
    document.getElementById('sidebarUserRole').textContent = currentUser.role;
    document.getElementById('sidebarUserAvatar').textContent = currentUser.name.charAt(0).toUpperCase();

    const sidebarMenu = document.getElementById('sidebarMenu');
    const mobileDrawerMenu = document.getElementById('mobileDrawerMenu');
    const mobileBottomNav = document.getElementById('mobileBottomNav');

    let menuItems = [
        { id: 'dashboard', label: 'Beranda', icon: 'home' },
        { id: 'transfer', label: 'Transfer', icon: 'send' },
        { id: 'qr', label: 'QR Payment', icon: 'qr-code' },
        { id: 'mutation', label: 'Mutasi', icon: 'receipt' },
        { id: 'topup', label: 'Top Up', icon: 'plus-circle' },
        { id: 'virtualcard', label: 'Kartu Virtual', icon: 'credit-card' },
        { id: 'notifications', label: 'Notifikasi', icon: 'bell' },
        { id: 'profile', label: 'Profil Saya', icon: 'user' }
    ];

    if (isSuperAdmin) {
        menuItems.push({ id: 'adminPanel', label: 'Administrator', icon: 'shield-check', badge: 'ADMIN' });
    }

    // Render Desktop Sidebar Menu
    sidebarMenu.innerHTML = menuItems.map(item => `
        <button onclick="navigateTo('${item.id}')" id="nav-item-${item.id}" 
                class="w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl text-xs font-semibold transition text-slate-400 hover:text-white hover:bg-slate-800">
            <div class="flex items-center space-x-3">
                <i data-lucide="${item.icon}" class="w-4 h-4"></i>
                <span>${item.label}</span>
            </div>
            ${item.badge ? `<span class="bg-bnr-red/20 text-red-400 px-1.5 py-0.5 text-[9px] font-bold rounded">${item.badge}</span>` : ''}
        </button>
    `).join('');

    // Render Mobile Drawer Menu
    mobileDrawerMenu.innerHTML = menuItems.map(item => `
        <button onclick="navigateTo('${item.id}'); toggleMobileDrawer();" 
                class="w-full flex items-center space-x-3 px-3 py-2.5 rounded-xl text-sm font-medium text-slate-300 hover:bg-slate-800 hover:text-white">
            <i data-lucide="${item.icon}" class="w-5 h-5"></i>
            <span>${item.label}</span>
        </button>
    `).join('');

    // Render Bottom Nav (Limited 5 items for mobile UI cleanliness)
    const bottomNavItems = [
        { id: 'dashboard', label: 'Beranda', icon: 'home' },
        { id: 'transfer', label: 'Transfer', icon: 'send' },
        { id: 'mutation', label: 'Mutasi', icon: 'receipt' },
        { id: 'notifications', label: 'Notif', icon: 'bell' },
        { id: 'profile', label: 'Profil', icon: 'user' }
    ];

    if (isSuperAdmin) {
        bottomNavItems[3] = { id: 'adminPanel', label: 'Admin', icon: 'shield-check' };
    }

    mobileBottomNav.innerHTML = bottomNavItems.map(item => `
        <button onclick="navigateTo('${item.id}')" id="bottom-nav-${item.id}" class="flex flex-col items-center py-1 px-2 text-slate-400">
            <i data-lucide="${item.icon}" class="w-5 h-5"></i>
            <span class="text-[10px] font-medium mt-0.5">${item.label}</span>
        </button>
    `).join('');

    lucide.createIcons();
}

function toggleMobileDrawer() {
    const drawer = document.getElementById('mobileDrawer');
    drawer.classList.toggle('hidden');
}

function navigateTo(viewId) {
    currentView = viewId;

    // Highlight active menu
    document.querySelectorAll('[id^="nav-item-"]').forEach(el => el.classList.remove('bg-bnr-red', 'text-white'));
    document.querySelectorAll('[id^="bottom-nav-"]').forEach(el => el.classList.remove('text-bnr-red'));

    const activeDesk = document.getElementById(`nav-item-${viewId}`);
    if (activeDesk) activeDesk.classList.add('bg-bnr-red', 'text-white');

    const activeMob = document.getElementById(`bottom-nav-${viewId}`);
    if (activeMob) activeMob.classList.add('text-bnr-red');

    const contentArea = document.getElementById('contentArea');
    contentArea.className = 'flex-1 p-4 sm:p-6 md:p-8 max-w-7xl w-full mx-auto pb-24 md:pb-8 page-fade-enter';

    // Route view
    switch (viewId) {
        case 'dashboard':
            document.getElementById('pageTitle').textContent = 'Beranda Nasabah';
            renderDashboardView();
            break;
        case 'transfer':
            document.getElementById('pageTitle').textContent = 'Transfer Dana';
            renderTransferView();
            break;
        case 'qr':
            document.getElementById('pageTitle').textContent = 'Pembayaran QR';
            renderQRView();
            break;
        case 'mutation':
            document.getElementById('pageTitle').textContent = 'Mutasi Rekening';
            renderMutationView();
            break;
        case 'topup':
            document.getElementById('pageTitle').textContent = 'Pengajuan Top Up';
            renderTopUpView();
            break;
        case 'virtualcard':
            document.getElementById('pageTitle').textContent = 'Kartu Virtual BNR';
            renderVirtualCardView();
            break;
        case 'notifications':
            document.getElementById('pageTitle').textContent = 'Pusat Notifikasi';
            renderNotificationsView();
            break;
        case 'profile':
            document.getElementById('pageTitle').textContent = 'Profil Pengguna';
            renderProfileView();
            break;
        case 'adminPanel':
            if (currentUser.role !== 'SUPER_ADMIN' && currentUser.role !== 'ADMIN') {
                Swal.fire('Akses Ditolak', 'Anda tidak memiliki otoritas Administrator.', 'error');
                navigateTo('dashboard');
                return;
            }
            document.getElementById('pageTitle').textContent = 'Panel Administrator BNR';
            renderAdminPanelView();
            break;
        default:
            renderDashboardView();
    }
    lucide.createIcons();
}

// Helper: Mask Account Number
function maskAccount(accNo) {
    if (!accNo || accNo.length < 10) return "12••••0000";
    return accNo.substring(0, 2) + "••••" + accNo.substring(accNo.length - 4);
}

// Helper: Format Currency
function formatCurrency(amount) {
    return new Intl.NumberFormat('id-ID', { style: 'currency', currency: 'IDR', maximumFractionDigits: 0 }).format(amount);
}

// Helper: Generate BNR Transaction ID
function generateTxId() {
    const today = new Date();
    const dateStr = today.toISOString().slice(0, 10).replace(/-/g, '');
    const randomStr = Math.random().toString(36).substring(2, 10).toUpperCase();
    return `BNR-${dateStr}-${randomStr}`;
}

// 6. VIEW RENDERERS

// --- CUSTOMER DASHBOARD ---
async function renderDashboardView() {
    const contentArea = document.getElementById('contentArea');
    const balanceDisplay = isBalanceHidden ? '••••••••' : formatCurrency(currentAccount ? currentAccount.balance : 0);
    const maskedAcc = maskAccount(currentUser.accountNumber);

    contentArea.innerHTML = `
        <div class="space-y-6">
            <!-- Greeting & Quick Status -->
            <div class="flex flex-col sm:flex-row sm:items-center justify-between gap-2">
                <div>
                    <h3 class="text-xl font-extrabold text-slate-800">Selamat datang, ${currentUser.name}</h3>
                    <p class="text-xs text-slate-500 font-medium">Layanan Perbankan Virtual Roblox Terpercaya</p>
                </div>
                <div class="inline-flex items-center space-x-2 px-3 py-1 bg-white rounded-xl border border-slate-200 text-xs text-slate-600 shadow-sm w-fit">
                    <i data-lucide="shield-check" class="w-4 h-4 text-emerald-500"></i>
                    <span>Status Rekening: <strong class="text-emerald-600 uppercase">${currentAccount ? currentAccount.status : 'ACTIVE'}</strong></span>
                </div>
            </div>

            <!-- ATM Card Saldo Utama -->
            <div class="relative overflow-hidden bnr-card-gradient rounded-3xl p-6 sm:p-8 text-white shadow-xl">
                <div class="absolute -right-10 -bottom-10 w-48 h-48 bg-white/10 rounded-full blur-2xl pointer-events-none"></div>
                <div class="relative z-10 flex flex-col justify-between h-full space-y-8">
                    <div class="flex justify-between items-start">
                        <div>
                            <span class="text-xs font-semibold tracking-widest text-red-100 uppercase">BNR Digital Account</span>
                            <p class="text-2xl font-mono tracking-widest mt-1">${maskedAcc}</p>
                        </div>
                        <img src="https://cdn.discordapp.com/attachments/1523253692810264660/1546528338212954112/file_00000000318881fab05a215f142fbb4a.png" class="w-12 h-12 object-contain bg-white/10 p-1.5 rounded-xl border border-white/20" alt="Logo">
                    </div>

                    <div>
                        <div class="flex items-center space-x-2 text-red-200 mb-1">
                            <span class="text-xs font-medium">Total Saldo Aktif</span>
                            <button onclick="toggleBalanceVisibility()" class="hover:text-white transition">
                                <i data-lucide="${isBalanceHidden ? 'eye' : 'eye-off'}" class="w-4 h-4"></i>
                            </button>
                        </div>
                        <h2 class="text-3xl sm:text-4xl font-extrabold tracking-tight">${balanceDisplay}</h2>
                    </div>
                </div>
            </div>

            <!-- Quick Action Grid -->
            <div class="grid grid-cols-4 gap-3 sm:gap-4">
                <button onclick="navigateTo('transfer')" class="flex flex-col items-center p-3 sm:p-4 bg-white hover:bg-slate-50 border border-slate-200/80 rounded-2xl shadow-sm transition group">
                    <div class="w-10 h-10 sm:w-12 sm:h-12 rounded-xl bg-red-50 text-bnr-red flex items-center justify-center mb-2 group-hover:scale-110 transition">
                        <i data-lucide="send" class="w-5 h-5 sm:w-6 sm:h-6"></i>
                    </div>
                    <span class="text-xs font-semibold text-slate-700">Transfer</span>
                </button>
                <button onclick="navigateTo('qr')" class="flex flex-col items-center p-3 sm:p-4 bg-white hover:bg-slate-50 border border-slate-200/80 rounded-2xl shadow-sm transition group">
                    <div class="w-10 h-10 sm:w-12 sm:h-12 rounded-xl bg-red-50 text-bnr-red flex items-center justify-center mb-2 group-hover:scale-110 transition">
                        <i data-lucide="qr-code" class="w-5 h-5 sm:w-6 sm:h-6"></i>
                    </div>
                    <span class="text-xs font-semibold text-slate-700">Bayar QR</span>
                </button>
                <button onclick="navigateTo('topup')" class="flex flex-col items-center p-3 sm:p-4 bg-white hover:bg-slate-50 border border-slate-200/80 rounded-2xl shadow-sm transition group">
                    <div class="w-10 h-10 sm:w-12 sm:h-12 rounded-xl bg-red-50 text-bnr-red flex items-center justify-center mb-2 group-hover:scale-110 transition">
                        <i data-lucide="plus-circle" class="w-5 h-5 sm:w-6 sm:h-6"></i>
                    </div>
                    <span class="text-xs font-semibold text-slate-700">Top Up</span>
                </button>
                <button onclick="navigateTo('mutation')" class="flex flex-col items-center p-3 sm:p-4 bg-white hover:bg-slate-50 border border-slate-200/80 rounded-2xl shadow-sm transition group">
                    <div class="w-10 h-10 sm:w-12 sm:h-12 rounded-xl bg-red-50 text-bnr-red flex items-center justify-center mb-2 group-hover:scale-110 transition">
                        <i data-lucide="receipt" class="w-5 h-5 sm:w-6 sm:h-6"></i>
                    </div>
                    <span class="text-xs font-semibold text-slate-700">Mutasi</span>
                </button>
            </div>

            <!-- Recent Transactions Widget -->
            <div class="bg-white border border-slate-200 rounded-3xl p-6 shadow-sm">
                <div class="flex items-center justify-between mb-4">
                    <h4 class="font-bold text-slate-800 text-sm">Transaksi Terakhir</h4>
                    <button onclick="navigateTo('mutation')" class="text-xs font-semibold text-bnr-red hover:underline">Lihat Semua</button>
                </div>
                <div id="dashboardRecentTxList" class="space-y-3">
                    <div class="text-center py-6 text-slate-400 text-xs">Memuat transaksi terbaru...</div>
                </div>
            </div>
        </div>
    `;

    loadDashboardRecentTransactions();
}

function toggleBalanceVisibility() {
    isBalanceHidden = !isBalanceHidden;
    renderDashboardView();
}

async function loadDashboardRecentTransactions() {
    const container = document.getElementById('dashboardRecentTxList');
    if (!container) return;

    try {
        const snapshot = await db.ref(DB_PREFIX + 'transactions')
            .orderByChild('senderAccount')
            .equalTo(currentUser.accountNumber)
            .limitToLast(5)
            .once('value');

        const snapshotReceiver = await db.ref(DB_PREFIX + 'transactions')
            .orderByChild('receiverAccount')
            .equalTo(currentUser.accountNumber)
            .limitToLast(5)
            .once('value');

        let txs = [];
        if (snapshot.exists()) txs = txs.concat(Object.values(snapshot.val()));
        if (snapshotReceiver.exists()) {
            const rxTxs = Object.values(snapshotReceiver.val());
            rxTxs.forEach(rx => {
                if (!txs.find(t => t.id === rx.id)) txs.push(rx);
            });
        }

        txs.sort((a, b) => b.timestamp - a.timestamp);
        txs = txs.slice(0, 5);

        if (txs.length === 0) {
            container.innerHTML = `<div class="text-center py-6 text-slate-400 text-xs">Belum ada aktivitas transaksi.</div>`;
            return;
        }

        container.innerHTML = txs.map(tx => {
            const isOutgoing = tx.senderAccount === currentUser.accountNumber;
            const amountClass = isOutgoing ? 'text-rose-600' : 'text-emerald-600';
            const amountPrefix = isOutgoing ? '-' : '+';
            const icon = isOutgoing ? 'arrow-up-right' : 'arrow-down-left';

            return `
                <div onclick="openReceiptModal('${tx.id}')" class="flex items-center justify-between p-3 hover:bg-slate-50 rounded-2xl transition border border-slate-100 cursor-pointer">
                    <div class="flex items-center space-x-3">
                        <div class="w-10 h-10 rounded-xl ${isOutgoing ? 'bg-rose-50 text-rose-600' : 'bg-emerald-50 text-emerald-600'} flex items-center justify-center shrink-0">
                            <i data-lucide="${icon}" class="w-5 h-5"></i>
                        </div>
                        <div>
                            <p class="text-xs font-bold text-slate-800">${tx.type || 'TRANSFER'}</p>
                            <p class="text-[10px] text-slate-400">${new Date(tx.timestamp).toLocaleString('id-ID')}</p>
                        </div>
                    </div>
                    <div class="text-right">
                        <p class="text-xs font-extrabold ${amountClass}">${amountPrefix}${formatCurrency(tx.amount)}</p>
                        <span class="text-[9px] px-2 py-0.5 rounded-full ${tx.status === 'SUCCESS' ? 'bg-emerald-100 text-emerald-700' : 'bg-amber-100 text-amber-700'} font-semibold">${tx.status}</span>
                    </div>
                </div>
            `;
        }).join('');
        lucide.createIcons();

    } catch (e) {
        console.error(e);
        container.innerHTML = `<div class="text-center py-6 text-rose-500 text-xs">Gagal memuat transaksi.</div>`;
    }
}

// --- TRANSFER VIEW ---
function renderTransferView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="max-w-xl mx-auto bg-white border border-slate-200 rounded-3xl p-6 sm:p-8 shadow-sm">
            <h3 class="text-lg font-bold text-slate-800 mb-1">Transfer Rekening BNR</h3>
            <p class="text-xs text-slate-500 mb-6">Kirim dana ke sesama nasabah Bank Nusantara Roblox secara instan.</p>

            <form onsubmit="handleTransferSubmit(event)" class="space-y-5">
                <div>
                    <label class="block text-xs font-semibold text-slate-700 uppercase tracking-wider mb-2">Rekening Tujuan</label>
                    <input type="text" id="transferTarget" required placeholder="Contoh: 1203847291" class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-bnr-red transition font-mono">
                </div>

                <div>
                    <label class="block text-xs font-semibold text-slate-700 uppercase tracking-wider mb-2">Nominal Transfer (Rp)</label>
                    <input type="number" id="transferAmount" required min="${systemSettings.minimumTransfer}" max="${systemSettings.maximumTransfer}" placeholder="Min Rp ${systemSettings.minimumTransfer.toLocaleString('id-ID')}" class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-bnr-red transition">
                    <p class="text-[11px] text-slate-400 mt-1">Batas harian: ${formatCurrency(systemSettings.dailyTransferLimit)}</p>
                </div>

                <div>
                    <label class="block text-xs font-semibold text-slate-700 uppercase tracking-wider mb-2">Catatan (Opsional)</label>
                    <input type="text" id="transferNote" placeholder="Keterangan transaksi" class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-bnr-red transition">
                </div>

                <div class="pt-2">
                    <button type="submit" id="btnSubmitTransfer" class="w-full bg-bnr-red hover:bg-bnr-darkred text-white font-semibold py-3.5 rounded-xl shadow-md transition flex items-center justify-center space-x-2">
                        <i data-lucide="send" class="w-4 h-4"></i>
                        <span>Lanjutkan Transfer</span>
                    </button>
                </div>
            </form>
        </div>
    `;
    lucide.createIcons();
}

async function handleTransferSubmit(e) {
    e.preventDefault();
    if (systemSettings.maintenanceMode) {
        Swal.fire('Fitur Dibatasi', 'Sistem sedang maintenance.', 'warning');
        return;
    }

    const targetAccountNo = document.getElementById('transferTarget').value.trim();
    const amount = parseInt(document.getElementById('transferAmount').value);
    const note = document.getElementById('transferNote').value.trim() || 'Transfer antar nasabah BNR';

    if (targetAccountNo === currentUser.accountNumber) {
        Swal.fire('Transaksi Gagal', 'Tidak dapat melakukan transfer ke rekening sendiri.', 'error');
        return;
    }

    if (amount < systemSettings.minimumTransfer || amount > systemSettings.maximumTransfer) {
        Swal.fire('Nominal Luar Batas', `Minimum Rp ${systemSettings.minimumTransfer.toLocaleString()} dan Maksimum Rp ${systemSettings.maximumTransfer.toLocaleString()}`, 'warning');
        return;
    }

    if (!currentAccount || currentAccount.balance < amount) {
        Swal.fire('Saldo Tidak Cukup', 'Saldo Anda tidak mencukupi untuk melakukan transfer ini.', 'error');
        return;
    }

    Swal.fire({
        title: 'Verifikasi Penerima...',
        allowOutsideClick: false,
        didOpen: () => Swal.showLoading()
    });

    // Check Receiver
    const receiverAccSnap = await db.ref(DB_PREFIX + 'accounts/' + targetAccountNo).once('value');
    if (!receiverAccSnap.exists()) {
        Swal.fire('Rekening Tidak Ditemukan', 'Nomor rekening tujuan tidak terdaftar di BNR.', 'error');
        return;
    }

    const receiverAccount = receiverAccSnap.val();
    if (receiverAccount.status !== 'ACTIVE') {
        Swal.fire('Rekening Dibekukan', 'Rekening tujuan sedang dinonaktifkan.', 'error');
        return;
    }

    const receiverUserSnap = await db.ref(DB_PREFIX + 'users/' + receiverAccount.username).once('value');
    const receiverUser = receiverUserSnap.exists() ? receiverUserSnap.val() : { name: 'Nasabah BNR' };

    // Confirmation Modal
    Swal.fire({
        title: 'Konfirmasi Transfer',
        html: `
            <div class="text-left text-xs space-y-2 bg-slate-50 p-4 rounded-xl border border-slate-200">
                <p><strong>Penerima:</strong> ${receiverUser.name}</p>
                <p><strong>Rekening:</strong> ${maskAccount(targetAccountNo)}</p>
                <p><strong>Nominal:</strong> <span class="text-bnr-red font-bold">${formatCurrency(amount)}</span></p>
                <p><strong>Catatan:</strong> ${note}</p>
            </div>
        `,
        icon: 'question',
        showCancelButton: true,
        confirmButtonText: 'Konfirmasi & Kirim',
        cancelButtonText: 'Batal'
    }).then(async (result) => {
        if (result.isConfirmed) {
            executeTransferAtomic(currentUser.accountNumber, targetAccountNo, amount, note, receiverUser.name);
        }
    });
}

// ATOMIC TRANSACTION EXECUTION
async function executeTransferAtomic(senderAccNo, receiverAccNo, amount, note, receiverName) {
    const txId = generateTxId();
    const timestamp = firebase.database.ServerValue.TIMESTAMP;

    Swal.fire({
        title: 'Memproses Transfer...',
        text: 'Mengamankan pemindahan saldo',
        allowOutsideClick: false,
        didOpen: () => Swal.showLoading()
    });

    try {
        // Atomic Balance Update Using Firebase Transaction
        const senderBalanceRef = db.ref(DB_PREFIX + 'accounts/' + senderAccNo + '/balance');
        const receiverBalanceRef = db.ref(DB_PREFIX + 'accounts/' + receiverAccNo + '/balance');

        let senderSuccess = false;
        await senderBalanceRef.transaction((currentBal) => {
            if (currentBal >= amount) {
                senderSuccess = true;
                return currentBal - amount;
            } else {
                senderSuccess = false;
                return; // Abort
            }
        });

        if (!senderSuccess) {
            Swal.fire('Gagal', 'Transaksi dibatalkan. Saldo tidak mencukupi.', 'error');
            return;
        }

        // Credit receiver balance
        await receiverBalanceRef.transaction((currentBal) => {
            return (currentBal || 0) + amount;
        });

        // Record Transaction Document
        const txData = {
            id: txId,
            type: 'TRANSFER',
            senderAccount: senderAccNo,
            senderName: currentUser.name,
            senderUsername: currentUser.username,
            receiverAccount: receiverAccNo,
            receiverName: receiverName,
            amount: amount,
            fee: 0,
            note: note,
            timestamp: Date.now(),
            status: 'SUCCESS'
        };

        await db.ref(DB_PREFIX + 'transactions/' + txId).set(txData);

        // Notifications
        sendNotification(currentUser.username, 'Transfer Berhasil', `Transfer sebesar ${formatCurrency(amount)} ke ${receiverName} berhasil.`);
        const receiverAccSnap = await db.ref(DB_PREFIX + 'accounts/' + receiverAccNo).once('value');
        if (receiverAccSnap.exists()) {
            sendNotification(receiverAccSnap.val().username, 'Dana Masuk', `Anda menerima transfer ${formatCurrency(amount)} dari ${currentUser.name}.`);
        }

        // Audit & Discord Log
        logAudit(currentUser.username, 'TRANSFER', `Transfer ${txId} sejumlah ${amount} ke ${receiverAccNo}`);
        sendDiscordTransactionLog(currentUser.name, maskAccount(senderAccNo), receiverName, maskAccount(receiverAccNo), amount, txId);

        Swal.fire({
            title: 'Transfer Berhasil!',
            text: `ID Transaksi: ${txId}`,
            icon: 'success',
            confirmButtonText: 'Lihat Resi Digital'
        }).then(() => {
            openReceiptModal(txId);
        });

    } catch (e) {
        console.error(e);
        Swal.fire('Gagal Transaksi', 'Terjadi kesalahan sistem saat memproses saldo.', 'error');
    }
}

// --- DIGITAL RECEIPT MODAL ---
async function openReceiptModal(txId) {
    Swal.fire({
        title: 'Memuat Resi...',
        didOpen: () => Swal.showLoading()
    });

    const snapshot = await db.ref(DB_PREFIX + 'transactions/' + txId).once('value');
    if (!snapshot.exists()) {
        Swal.fire('Resi Tidak Ditemukan', '', 'error');
        return;
    }

    const tx = snapshot.val();
    Swal.close();

    const receiptHtml = `
        <div id="digitalReceiptModal" class="text-left font-sans p-2">
            <div class="flex items-center space-x-3 mb-4 pb-3 border-b border-slate-200">
                <img src="https://cdn.discordapp.com/attachments/1523253692810264660/1546528338212954112/file_00000000318881fab05a215f142fbb4a.png" class="w-10 h-10 object-contain">
                <div>
                    <h4 class="font-extrabold text-slate-900 leading-none">BNR</h4>
                    <p class="text-[10px] text-slate-500 uppercase font-semibold">Bank Nusantara Roblox</p>
                </div>
            </div>

            <div class="text-center py-2 mb-4 bg-emerald-50 rounded-xl border border-emerald-200">
                <span class="text-xs font-bold text-emerald-700 uppercase tracking-wider block">TRANSAKSI BERHASIL</span>
                <span class="text-xl font-extrabold text-slate-900">${formatCurrency(tx.amount)}</span>
            </div>

            <div class="space-y-2 text-xs border-b border-slate-200 pb-4 mb-4">
                <div class="flex justify-between"><span class="text-slate-500">Jenis Transaksi</span><span class="font-semibold text-slate-800">${tx.type}</span></div>
                <div class="flex justify-between"><span class="text-slate-500">ID Transaksi</span><span class="font-mono font-semibold text-slate-800">${tx.id}</span></div>
                <div class="flex justify-between"><span class="text-slate-500">Waktu</span><span class="font-semibold text-slate-800">${new Date(tx.timestamp).toLocaleString('id-ID')}</span></div>
                <div class="flex justify-between"><span class="text-slate-500">Pengirim</span><span class="font-semibold text-slate-800">${tx.senderName || '-'} (${maskAccount(tx.senderAccount)})</span></div>
                <div class="flex justify-between"><span class="text-slate-500">Penerima</span><span class="font-semibold text-slate-800">${tx.receiverName || '-'} (${maskAccount(tx.receiverAccount)})</span></div>
                <div class="flex justify-between"><span class="text-slate-500">Catatan</span><span class="font-semibold text-slate-800">${tx.note || '-'}</span></div>
            </div>

            <div class="flex space-x-2">
                <button onclick="window.print()" class="flex-1 bg-slate-800 text-white py-2 rounded-xl text-xs font-semibold">Cetak Resi</button>
                <button onclick="Swal.close()" class="flex-1 bg-slate-200 text-slate-700 py-2 rounded-xl text-xs font-semibold">Tutup</button>
            </div>
        </div>
    `;

    Swal.fire({
        html: receiptHtml,
        showConfirmButton: false,
        customClass: { popup: 'rounded-3xl' }
    });
}

// --- MUTATION VIEW ---
async function renderMutationView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="bg-white border border-slate-200 rounded-3xl p-6 shadow-sm">
            <h3 class="text-lg font-bold text-slate-800 mb-4">Mutasi & Riwayat Transaksi</h3>
            <div id="mutationList" class="space-y-3">
                <div class="text-center py-8 text-slate-400 text-xs">Memuat data mutasi...</div>
            </div>
        </div>
    `;

    try {
        const snapshot = await db.ref(DB_PREFIX + 'transactions').once('value');
        if (!snapshot.exists()) {
            document.getElementById('mutationList').innerHTML = `<div class="text-center py-8 text-slate-400 text-xs">Belum ada mutasi transaksi.</div>`;
            return;
        }

        const allTxs = Object.values(snapshot.val());
        const userTxs = allTxs.filter(t => t.senderAccount === currentUser.accountNumber || t.receiverAccount === currentUser.accountNumber);
        userTxs.sort((a, b) => b.timestamp - a.timestamp);

        if (userTxs.length === 0) {
            document.getElementById('mutationList').innerHTML = `<div class="text-center py-8 text-slate-400 text-xs">Tidak ditemukan catatan transaksi.</div>`;
            return;
        }

        document.getElementById('mutationList').innerHTML = userTxs.map(tx => {
            const isOutgoing = tx.senderAccount === currentUser.accountNumber;
            return `
                <div onclick="openReceiptModal('${tx.id}')" class="flex items-center justify-between p-4 bg-slate-50 hover:bg-slate-100 rounded-2xl transition border border-slate-200/60 cursor-pointer">
                    <div>
                        <span class="text-xs font-bold text-slate-800">${tx.type}</span>
                        <p class="text-[11px] text-slate-500">${tx.note || 'Transaksi Virtual BNR'}</p>
                        <p class="text-[10px] text-slate-400 font-mono mt-0.5">${new Date(tx.timestamp).toLocaleString('id-ID')}</p>
                    </div>
                    <div class="text-right">
                        <p class="text-xs font-extrabold ${isOutgoing ? 'text-rose-600' : 'text-emerald-600'}">${isOutgoing ? '-' : '+'}${formatCurrency(tx.amount)}</p>
                        <span class="text-[10px] text-slate-400 font-mono">${tx.id}</span>
                    </div>
                </div>
            `;
        }).join('');

    } catch (e) {
        console.error(e);
    }
}

// --- QR PAYMENT VIEW ---
function renderQRView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="max-w-md mx-auto bg-white border border-slate-200 rounded-3xl p-6 text-center shadow-sm">
            <h3 class="text-lg font-bold text-slate-800 mb-1">BNR Quick Payment QR</h3>
            <p class="text-xs text-slate-500 mb-6">Tunjukkan QR ini untuk menerima saldo atau scan QR nasabah lain.</p>
            
            <div class="p-6 bg-slate-900 rounded-2xl inline-block mb-6 shadow-inner">
                <img src="https://api.qrserver.com/v1/create-qr-code/?size=200x200&data=${currentUser.accountNumber}" class="w-48 h-48 rounded-lg" alt="QR Code">
            </div>

            <p class="text-xs font-mono font-bold text-slate-700 mb-1">${maskAccount(currentUser.accountNumber)}</p>
            <p class="text-xs text-slate-400 mb-6">${currentUser.name}</p>

            <button onclick="promptSimulatedQRScan()" class="w-full bg-bnr-red hover:bg-bnr-darkred text-white py-3 rounded-xl font-semibold text-xs transition flex items-center justify-center space-x-2">
                <i data-lucide="scan" class="w-4 h-4"></i>
                <span>Simulasi Bayar via Kode QR</span>
            </button>
        </div>
    `;
    lucide.createIcons();
}

function promptSimulatedQRScan() {
    Swal.fire({
        title: 'Scan QR Payment',
        input: 'text',
        inputLabel: 'Masukkan Nomor Rekening QR Tujuan',
        inputPlaceholder: 'Contoh: 1203847291',
        showCancelButton: true,
        confirmButtonText: 'Lanjutkan Bayar'
    }).then((res) => {
        if (res.isConfirmed && res.value) {
            document.getElementById('transferTarget').value = res.value;
            navigateTo('transfer');
        }
    });
}

// --- TOP UP VIEW ---
function renderTopUpView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="max-w-xl mx-auto bg-white border border-slate-200 rounded-3xl p-6 sm:p-8 shadow-sm">
            <h3 class="text-lg font-bold text-slate-800 mb-1">Pengajuan Top Up Saldo</h3>
            <p class="text-xs text-slate-500 mb-6">Kirim permintaan saldo ke pihak Administrator BNR.</p>

            <form onsubmit="handleTopUpRequest(event)" class="space-y-4">
                <div>
                    <label class="block text-xs font-semibold text-slate-700 uppercase tracking-wider mb-2">Nominal Top Up (Rp)</label>
                    <input type="number" id="topupAmount" required min="10000" placeholder="Minimal Rp 10.000" class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-bnr-red">
                </div>
                <div>
                    <label class="block text-xs font-semibold text-slate-700 uppercase tracking-wider mb-2">Alasan / Catatan Pengajuan</label>
                    <input type="text" id="topupNote" required placeholder="Contoh: Pembelian Asset Roleplay" class="w-full bg-slate-50 border border-slate-200 text-slate-800 rounded-xl px-4 py-3 text-sm focus:outline-none focus:border-bnr-red">
                </div>
                <button type="submit" class="w-full bg-slate-900 hover:bg-slate-800 text-white font-semibold py-3.5 rounded-xl text-xs transition">Kirim Pengajuan Top Up</button>
            </form>
        </div>
    `;
}

async function handleTopUpRequest(e) {
    e.preventDefault();
    const amount = parseInt(document.getElementById('topupAmount').value);
    const note = document.getElementById('topupNote').value.trim();

    const requestId = 'REQ-' + Date.now();
    await db.ref(DB_PREFIX + 'paymentRequests/' + requestId).set({
        id: requestId,
        username: currentUser.username,
        accountNumber: currentUser.accountNumber,
        name: currentUser.name,
        amount: amount,
        note: note,
        status: 'PENDING',
        timestamp: firebase.database.ServerValue.TIMESTAMP
    });

    logAudit(currentUser.username, 'TOPUP_REQUEST', `Pengajuan Top Up ID ${requestId} sejumlah ${amount}`);
    Swal.fire('Pengajuan Terkirim', 'Permintaan Top Up Anda menunggu persetujuan Admin BNR.', 'success');
    navigateTo('dashboard');
}

// --- VIRTUAL CARD VIEW ---
function renderVirtualCardView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="max-w-md mx-auto space-y-6">
            <div class="bnr-card-gradient rounded-3xl p-6 text-white shadow-2xl relative overflow-hidden border border-red-400/30">
                <div class="flex justify-between items-start mb-8">
                    <div>
                        <h4 class="font-extrabold text-lg tracking-wider">BNR VIRTUAL</h4>
                        <p class="text-[10px] text-red-200 tracking-widest uppercase">Roblox Exclusive Banking</p>
                    </div>
                    <img src="https://cdn.discordapp.com/attachments/1523253692810264660/1546528338212954112/file_00000000318881fab05a215f142fbb4a.png" class="w-10 h-10 object-contain">
                </div>

                <div class="mb-6">
                    <p class="text-[10px] text-red-200 uppercase tracking-wider mb-1">Nomor Kartu Virtual</p>
                    <p class="text-xl font-mono tracking-widest">4800 •••• •••• ${currentUser.accountNumber.substring(6)}</p>
                </div>

                <div class="flex justify-between items-end">
                    <div>
                        <p class="text-[9px] text-red-200 uppercase">Pemegang Kartu</p>
                        <p class="text-xs font-bold uppercase tracking-wider">${currentUser.name}</p>
                    </div>
                    <div>
                        <p class="text-[9px] text-red-200 uppercase">Berlaku Hingga</p>
                        <p class="text-xs font-mono font-bold">12/30</p>
                    </div>
                </div>
            </div>
            
            <p class="text-center text-xs text-slate-400">Kartu virtual ini hanya berfungsi sebagai identitas keanggotaan roleplay resmi BNR.</p>
        </div>
    `;
}

// --- NOTIFICATIONS VIEW ---
async function renderNotificationsView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="bg-white border border-slate-200 rounded-3xl p-6 shadow-sm">
            <h3 class="text-lg font-bold text-slate-800 mb-4">Pusat Notifikasi Realtime</h3>
            <div id="notifContainer" class="space-y-3">
                <div class="text-center py-6 text-slate-400 text-xs">Memuat notifikasi...</div>
            </div>
        </div>
    `;

    const snapshot = await db.ref(DB_PREFIX + 'notifications/' + currentUser.username).once('value');
    if (!snapshot.exists()) {
        document.getElementById('notifContainer').innerHTML = `<div class="text-center py-6 text-slate-400 text-xs">Belum ada notifikasi.</div>`;
        return;
    }

    const notifs = Object.values(snapshot.val());
    notifs.sort((a, b) => b.timestamp - a.timestamp);

    document.getElementById('notifContainer').innerHTML = notifs.map(n => `
        <div class="p-4 bg-slate-50 border border-slate-200/80 rounded-2xl">
            <h5 class="text-xs font-bold text-slate-800">${n.title}</h5>
            <p class="text-xs text-slate-600 mt-1">${n.message}</p>
            <span class="text-[10px] text-slate-400 font-mono mt-2 block">${new Date(n.timestamp).toLocaleString('id-ID')}</span>
        </div>
    `).join('');
}

// --- PROFILE & NAME CHANGE VIEW ---
function renderProfileView() {
    const contentArea = document.getElementById('contentArea');
    contentArea.innerHTML = `
        <div class="max-w-2xl mx-auto bg-white border border-slate-200 rounded-3xl p-6 sm:p-8 shadow-sm space-y-6">
            <div class="flex items-center space-x-4 pb-6 border-b border-slate-200">
                <div class="w-16 h-16 rounded-2xl bg-bnr-red text-white flex items-center justify-center font-extrabold text-2xl uppercase">
                    ${currentUser.name.charAt(0)}
                </div>
                <div>
                    <h3 class="text-xl font-extrabold text-slate-800">${currentUser.name}</h3>
                    <p class="text-xs text-slate-500 font-mono">@${currentUser.username} • ${currentUser.role}</p>
                </div>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 gap-4 text-xs">
                <div class="p-4 bg-slate-50 rounded-2xl border border-slate-200">
                    <span class="text-slate-400 block mb-1">Nomor Rekening Terikat</span>
                    <strong class="font-mono text-slate-800 text-sm">${currentUser.accountNumber}</strong>
                </div>
                <div class="p-4 bg-slate-50 rounded-2xl border border-slate-200">
                    <span class="text-slate-400 block mb-1">Biaya Ubah Nama Nasabah</span>
                    <strong class="text-slate-800 text-sm">${formatCurrency(systemSettings.profileChangeFee)}</strong>
                </div>
            </div>

            <div class="pt-4 space-y-3">
                <button onclick="handleChangeNamePrompt()" class="w-full bg-slate-900 hover:bg-slate-800 text-white font-semibold py-3 rounded-xl text-xs transition">Ganti Nama Nasabah (${formatCurrency(systemSettings.profileChangeFee)})</button>
            </div>
        </div>
    `;
}

async function handleChangeNamePrompt() {
    const fee = systemSettings.profileChangeFee;

    if (!currentAccount || currentAccount.balance < fee) {
        Swal.fire('Saldo Tidak Mencukupi', `Biaya penggantian nama adalah ${formatCurrency(fee)}. Saldo Anda tidak mencukupi.`, 'error');
        return;
    }

    const { value: newName } = await Swal.fire({
        title: 'Pengajuan Ganti Nama',
        input: 'text',
        inputLabel: `Biaya layanan: ${formatCurrency(fee)}`,
        inputPlaceholder: 'Masukkan nama nasabah baru',
        showCancelButton: true,
        confirmButtonText: 'Konfirmasi & Potong Saldo'
    });

    if (newName && newName.trim()) {
        const cleanName = newName.trim();
        
        // Debit Fee Atomically
        await db.ref(DB_PREFIX + 'accounts/' + currentUser.accountNumber + '/balance').transaction((bal) => bal - fee);
        
        // Update Name
        await db.ref(DB_PREFIX + 'users/' + currentUser.username).update({ name: cleanName });
        currentUser.name = cleanName;

        // Record Tx
        const txId = generateTxId();
        await db.ref(DB_PREFIX + 'transactions/' + txId).set({
            id: txId,
            type: 'SERVICE_FEE',
            senderAccount: currentUser.accountNumber,
            senderName: cleanName,
            amount: fee,
            note: 'Biaya Penggantian Nama Nasabah BNR',
            timestamp: Date.now(),
            status: 'SUCCESS'
        });

        logAudit(currentUser.username, 'NAME_CHANGE', `Mengubah nama menjadi ${cleanName}`);
        sendNotification(currentUser.username, 'Nama Diperbarui', `Nama nasabah Anda telah diubah menjadi ${cleanName}.`);

        Swal.fire('Berhasil', 'Nama nasabah berhasil diubah.', 'success');
        renderProfileView();
        renderSidebarAndNav();
    }
}

// 7. SUPER ADMIN PANEL MODULE
async function renderAdminPanelView() {
    const contentArea = document.getElementById('contentArea');

    contentArea.innerHTML = `
        <div class="space-y-6">
            <!-- Header Status Admin -->
            <div class="bg-slate-900 text-white p-6 rounded-3xl flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
                <div>
                    <span class="text-xs font-bold text-red-400 tracking-wider uppercase">Hak Akses Super Admin</span>
                    <h3 class="text-2xl font-extrabold mt-0.5">Kontrol Penuh Sistem BNR</h3>
                </div>
                <div class="flex items-center space-x-2">
                    <button onclick="toggleMaintenanceMode()" class="px-4 py-2 rounded-xl text-xs font-bold transition ${systemSettings.maintenanceMode ? 'bg-amber-500 text-slate-950' : 'bg-slate-800 text-slate-300 hover:bg-slate-700'}">
                        Mode Maintenance: ${systemSettings.maintenanceMode ? 'AKTIF' : 'NON-AKTIF'}
                    </button>
                </div>
            </div>

            <!-- Admin Nav Tabs -->
            <div class="flex space-x-2 border-b border-slate-200 overflow-x-auto pb-2 text-xs font-bold">
                <button onclick="renderAdminOverview()" class="px-4 py-2 rounded-xl bg-bnr-red text-white">Ikhtisar</button>
                <button onclick="renderAdminUsers()" class="px-4 py-2 rounded-xl bg-slate-200 text-slate-700 hover:bg-slate-300">Kelola Nasabah</button>
                <button onclick="renderAdminTopUps()" class="px-4 py-2 rounded-xl bg-slate-200 text-slate-700 hover:bg-slate-300">Persetujuan Top Up</button>
                <button onclick="renderAdminLogs()" class="px-4 py-2 rounded-xl bg-slate-200 text-slate-700 hover:bg-slate-300">Audit Log</button>
            </div>

            <!-- Dynamic Admin Workspace Container -->
            <div id="adminWorkspace"></div>
        </div>
    `;

    renderAdminOverview();
}

async function renderAdminOverview() {
    const ws = document.getElementById('adminWorkspace');
    if (!ws) return;

    ws.innerHTML = `<div class="text-slate-400 text-xs py-4">Memuat data statistik...</div>`;

    const usersSnap = await db.ref(DB_PREFIX + 'users').once('value');
    const accsSnap = await db.ref(DB_PREFIX + 'accounts').once('value');
    const txsSnap = await db.ref(DB_PREFIX + 'transactions').once('value');

    const totalUsers = usersSnap.exists() ? Object.keys(usersSnap.val()).length : 0;
    const totalTxs = txsSnap.exists() ? Object.keys(txsSnap.val()).length : 0;

    let totalBalance = 0;
    if (accsSnap.exists()) {
        Object.values(accsSnap.val()).forEach(a => totalBalance += (a.balance || 0));
    }

    ws.innerHTML = `
        <div class="grid grid-cols-1 sm:grid-cols-3 gap-4">
            <div class="p-6 bg-white border border-slate-200 rounded-3xl shadow-sm">
                <span class="text-xs font-semibold text-slate-400">Total Nasabah Terdaftar</span>
                <h3 class="text-3xl font-extrabold text-slate-800 mt-2">${totalUsers}</h3>
            </div>
            <div class="p-6 bg-white border border-slate-200 rounded-3xl shadow-sm">
                <span class="text-xs font-semibold text-slate-400">Total Perputaran Saldo</span>
                <h3 class="text-3xl font-extrabold text-emerald-600 mt-2">${formatCurrency(totalBalance)}</h3>
            </div>
            <div class="p-6 bg-white border border-slate-200 rounded-3xl shadow-sm">
                <span class="text-xs font-semibold text-slate-400">Total Transaksi Selesai</span>
                <h3 class="text-3xl font-extrabold text-bnr-red mt-2">${totalTxs}</h3>
            </div>
        </div>
    `;
}

async function renderAdminUsers() {
    const ws = document.getElementById('adminWorkspace');
    ws.innerHTML = `
        <div class="bg-white border border-slate-200 rounded-3xl p-6 space-y-4">
            <div class="flex justify-between items-center">
                <h4 class="font-bold text-slate-800 text-sm">Daftar Nasabah BNR</h4>
                <button onclick="promptCreateNewAccount()" class="px-4 py-2 bg-bnr-red text-white text-xs font-semibold rounded-xl">+ Tambah Akun BNR</button>
            </div>
            <div class="overflow-x-auto">
                <table class="w-full text-left text-xs border-collapse">
                    <thead>
                        <tr class="border-b border-slate-200 text-slate-400">
                            <th class="py-2">NAMA</th>
                            <th class="py-2">USERNAME</th>
                            <th class="py-2">REKENING</th>
                            <th class="py-2">ROLE</th>
                            <th class="py-2">STATUS</th>
                            <th class="py-2 text-right">AKSI</th>
                        </tr>
                    </thead>
                    <tbody id="adminUsersTableBody">
                        <tr><td colspan="6" class="py-4 text-center text-slate-400">Memuat data nasabah...</td></tr>
                    </tbody>
                </table>
            </div>
        </div>
    `;

    const snapshot = await db.ref(DB_PREFIX + 'users').once('value');
    if (!snapshot.exists()) return;

    const users = Object.values(snapshot.val());
    document.getElementById('adminUsersTableBody').innerHTML = users.map(u => `
        <tr class="border-b border-slate-100 hover:bg-slate-50">
            <td class="py-3 font-semibold text-slate-800">${u.name}</td>
            <td class="py-3 font-mono text-slate-600">@${u.username}</td>
            <td class="py-3 font-mono">${u.accountNumber}</td>
            <td class="py-3"><span class="px-2 py-0.5 rounded text-[10px] font-bold ${u.role === 'SUPER_ADMIN' ? 'bg-purple-100 text-purple-700' : 'bg-slate-100 text-slate-700'}">${u.role}</span></td>
            <td class="py-3"><span class="px-2 py-0.5 rounded text-[10px] font-bold ${u.status === 'ACTIVE' ? 'bg-emerald-100 text-emerald-700' : 'bg-rose-100 text-rose-700'}">${u.status}</span></td>
            <td class="py-3 text-right space-x-1">
                <button onclick="promptEditBalance('${u.accountNumber}')" class="px-2 py-1 bg-slate-800 text-white text-[10px] rounded">Saldo</button>
                ${u.username !== 'Haruukii79' ? `<button onclick="toggleUserStatus('${u.username}', '${u.status}')" class="px-2 py-1 bg-rose-600 text-white text-[10px] rounded">Status</button>` : ''}
            </td>
        </tr>
    `).join('');
}

async function promptCreateNewAccount() {
    const { value: formValues } = await Swal.fire({
        title: 'Tambah Akun BNR Baru',
        html: `
            <input id="swal-input-name" class="swal2-input" placeholder="Nama Lengkap Nasabah">
            <input id="swal-input-username" class="swal2-input" placeholder="Username">
            <input id="swal-input-pass" type="password" class="swal2-input" placeholder="Password Sesi">
            <input id="swal-input-balance" type="number" class="swal2-input" placeholder="Saldo Awal (Rp)">
        `,
        focusConfirm: false,
        showCancelButton: true,
        confirmButtonText: 'Buat Rekening',
        preConfirm: () => {
            return [
                document.getElementById('swal-input-name').value,
                document.getElementById('swal-input-username').value,
                document.getElementById('swal-input-pass').value,
                document.getElementById('swal-input-balance').value
            ];
        }
    });

    if (formValues && formValues[0] && formValues[1] && formValues[2]) {
        const [name, username, password, initialBalance] = formValues;
        const newAccNo = '12' + Math.floor(10000000 + Math.random() * 90000000);
        const hashedPassword = CryptoJS.SHA256(password).toString();

        await db.ref(DB_PREFIX + 'users/' + username).set({
            name,
            username,
            password: hashedPassword,
            accountNumber: newAccNo,
            role: 'CUSTOMER',
            status: 'ACTIVE',
            createdAt: firebase.database.ServerValue.TIMESTAMP
        });

        await db.ref(DB_PREFIX + 'accounts/' + newAccNo).set({
            accountNumber: newAccNo,
            username,
            balance: parseInt(initialBalance) || 0,
            status: 'ACTIVE',
            createdAt: firebase.database.ServerValue.TIMESTAMP
        });

        logAudit(currentUser.username, 'CREATE_USER', `Membuat akun baru ${username} - Rekening ${newAccNo}`);
        Swal.fire('Akun Dibuat', `Nomor Rekening: ${newAccNo}`, 'success');
        renderAdminUsers();
    }
}

async function promptEditBalance(accountNo) {
    const accSnap = await db.ref(DB_PREFIX + 'accounts/' + accountNo).once('value');
    if (!accSnap.exists()) return;
    const acc = accSnap.val();

    const { value: amount } = await Swal.fire({
        title: 'Penyesuaian Saldo Admin',
        input: 'number',
        inputLabel: `Saldo saat ini: ${formatCurrency(acc.balance)}`,
        inputPlaceholder: 'Masukkan saldo baru (Rp)',
        showCancelButton: true,
        confirmButtonText: 'Simpan Saldo'
    });

    if (amount !== undefined && amount !== '') {
        const newBal = parseInt(amount);
        await db.ref(DB_PREFIX + 'accounts/' + accountNo).update({ balance: newBal });
        logAudit(currentUser.username, 'MANAGE_BALANCE', `Mengubah saldo rekening ${accountNo} menjadi ${newBal}`);
        Swal.fire('Saldo Diperbarui', '', 'success');
        renderAdminUsers();
    }
}

async function toggleUserStatus(username, currentStatus) {
    const newStatus = currentStatus === 'ACTIVE' ? 'SUSPENDED' : 'ACTIVE';
    await db.ref(DB_PREFIX + 'users/' + username).update({ status: newStatus });
    logAudit(currentUser.username, 'CHANGE_USER_STATUS', `Mengubah status user ${username} menjadi ${newStatus}`);
    renderAdminUsers();
}

async function renderAdminTopUps() {
    const ws = document.getElementById('adminWorkspace');
    ws.innerHTML = `<div class="bg-white border border-slate-200 rounded-3xl p-6"><h4 class="font-bold text-sm mb-4">Persetujuan Top Up</h4><div id="topupRequestsList">Memuat...</div></div>`;

    const snapshot = await db.ref(DB_PREFIX + 'paymentRequests').once('value');
    if (!snapshot.exists()) {
        document.getElementById('topupRequestsList').innerHTML = `<p class="text-xs text-slate-400">Tidak ada pengajuan pending.</p>`;
        return;
    }

    const reqs = Object.values(snapshot.val()).filter(r => r.status === 'PENDING');
    document.getElementById('topupRequestsList').innerHTML = reqs.map(r => `
        <div class="flex items-center justify-between p-3 border-b border-slate-100 text-xs">
            <div>
                <p class="font-bold text-slate-800">${r.name} (@${r.username})</p>
                <p class="text-slate-500">${r.note} — <strong class="text-emerald-600">${formatCurrency(r.amount)}</strong></p>
            </div>
            <div class="space-x-1">
                <button onclick="approveTopUp('${r.id}')" class="px-3 py-1 bg-emerald-600 text-white rounded">Setujui</button>
            </div>
        </div>
    `).join('');
}

async function approveTopUp(requestId) {
    const reqSnap = await db.ref(DB_PREFIX + 'paymentRequests/' + requestId).once('value');
    if (!reqSnap.exists()) return;
    const req = reqSnap.val();

    // Credit Balance Atomically
    await db.ref(DB_PREFIX + 'accounts/' + req.accountNumber + '/balance').transaction((bal) => (bal || 0) + req.amount);
    await db.ref(DB_PREFIX + 'paymentRequests/' + requestId).update({ status: 'APPROVED' });

    logAudit(currentUser.username, 'APPROVE_TOPUP', `Menyetujui Top Up ${requestId} sebesar ${req.amount}`);
    sendNotification(req.username, 'Top Up Disetujui', `Pengajuan Top Up sebesar ${formatCurrency(req.amount)} telah disetujui Admin.`);
    
    Swal.fire('Disetujui', 'Saldo nasabah telah ditambahkan.', 'success');
    renderAdminTopUps();
}

async function renderAdminLogs() {
    const ws = document.getElementById('adminWorkspace');
    ws.innerHTML = `<div class="bg-white border border-slate-200 rounded-3xl p-6"><h4 class="font-bold text-sm mb-4">Audit Log Sistem</h4><div id="auditLogsContainer" class="space-y-2 text-xs font-mono">Memuat...</div></div>`;

    const snapshot = await db.ref(DB_PREFIX + 'auditLogs').limitToLast(20).once('value');
    if (!snapshot.exists()) {
        document.getElementById('auditLogsContainer').innerHTML = `Log kosong.`;
        return;
    }

    const logs = Object.values(snapshot.val());
    logs.sort((a, b) => b.timestamp - a.timestamp);

    document.getElementById('auditLogsContainer').innerHTML = logs.map(l => `
        <div class="p-2 bg-slate-50 rounded border border-slate-200">
            <span class="text-slate-400">[${new Date(l.timestamp).toLocaleString('id-ID')}]</span>
            <strong class="text-slate-800">${l.username}</strong>: ${l.action} — ${l.details}
        </div>
    `).join('');
}

async function toggleMaintenanceMode() {
    systemSettings.maintenanceMode = !systemSettings.maintenanceMode;
    await db.ref(DB_PREFIX + 'settings/maintenanceMode').set(systemSettings.maintenanceMode);
    logAudit(currentUser.username, 'MAINTENANCE_TOGGLE', `Mode maintenance set: ${systemSettings.maintenanceMode}`);
    renderAdminPanelView();
}

// 8. LOGGING & NOTIFICATION HELPERS
async function logAudit(username, action, details) {
    const logId = 'LOG-' + Date.now();
    await db.ref(DB_PREFIX + 'auditLogs/' + logId).set({
        id: logId,
        username,
        action,
        details,
        timestamp: firebase.database.ServerValue.TIMESTAMP
    });
}

async function logSecurity(event, username, reason) {
    const secId = 'SEC-' + Date.now();
    await db.ref(DB_PREFIX + 'securityLogs/' + secId).set({
        id: secId,
        event,
        username,
        reason,
        timestamp: firebase.database.ServerValue.TIMESTAMP
    });
}

async function sendNotification(targetUsername, title, message) {
    const notifId = 'NOTIF-' + Date.now();
    await db.ref(DB_PREFIX + 'notifications/' + targetUsername + '/' + notifId).set({
        id: notifId,
        title,
        message,
        read: false,
        timestamp: firebase.database.ServerValue.TIMESTAMP
    });
}

// DISCORD WEBHOOK LOGGING INTEGRATION
function sendDiscordTransactionLog(senderName, maskedSender, receiverName, maskedReceiver, amount, txId) {
    if (!systemSettings.discordWebhookUrl || systemSettings.discordWebhookUrl === "YOUR_DISCORD_WEBHOOK_URL") {
        return; // Webhook placeholder check
    }

    const payload = {
        content: `**BNR | TRANSACTION LOG**\n\`\`\`BNR | TRANSFER BERHASIL\n\nPengirim: ${senderName}\nRekening: ${maskedSender}\n\nPenerima: ${receiverName}\nRekening: ${maskedReceiver}\n\nNominal: ${formatCurrency(amount)}\nTransaction ID: ${txId}\nWaktu: ${new Date().toLocaleString('id-ID')} WIB\`\`\``
    };

    fetch(systemSettings.discordWebhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
    }).catch(err => console.log('Discord log omitted or failed:', err));
}

