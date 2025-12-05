import React, { useState, useEffect, useCallback, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, collection, doc, setDoc, query, where, getDocs, onSnapshot, orderBy, runTransaction, Timestamp } from 'firebase/firestore';
import { Menu, Search, ShoppingCart, DollarSign, Archive, LogOut, Calendar, Plus, X, BarChart2, TrendingDown } from 'lucide-react';

// --- Konfigurasi Firebase & Inisialisasi ---
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-warmindo-app-id';
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

// Helper untuk format Rupiah
const formatRupiah = (number) => new Intl.NumberFormat('id-ID', {
  style: 'currency',
  currency: 'IDR',
  minimumFractionDigits: 0,
}).format(number);

// Helper untuk format tanggal
const formatDate = (date) => new Intl.DateTimeFormat('id-ID', {
  year: 'numeric', month: 'long', day: 'numeric'
}).format(date);

// Daftar Menu Lengkap
const menuItems = [
  // Makanan
  { id: 'M1', name: 'Indomie Goreng', price: 8000, category: 'Makanan', variants: ['Original', 'Rendang', 'Aceh'] },
  { id: 'M2', name: 'Indomie Rebus', price: 8000, category: 'Makanan', variants: ['Soto', 'Ayam Bawang'] },
  { id: 'M3', name: 'Nasi Goreng', price: 15000, category: 'Makanan', variants: ['Pedas', 'Tidak Pedas'] },
  { id: 'M4', name: 'Bubur Ayam', price: 10000, category: 'Makanan', variants: ['Original'] },
  { id: 'M5', name: 'Bubur Kacang Ijo', price: 8000, category: 'Makanan', variants: ['Campur Ketan', 'Tidak Campur Ketan'] },
  { id: 'M6', name: 'Bubur Ketan', price: 8000, category: 'Makanan', variants: ['Original'] },
  { id: 'M7', name: 'Magelangan', price: 15000, category: 'Makanan', variants: ['Pedas', 'Tidak Pedas'] },
  { id: 'M8', name: 'Mie Tek Tek', price: 15000, category: 'Makanan', variants: ['Pedas', 'Tidak Pedas'] },
  // Minuman
  { id: 'B1', name: 'Hot Kopi', price: 5000, category: 'Minuman', variants: ['Gooday', 'Kapal Api', 'Indocafe'] },
  { id: 'B2', name: 'Ice Kopi', price: 8000, category: 'Minuman', variants: ['Gooday Freez', 'ABC Ice'] },
  { id: 'B3', name: 'Teh Tarik', price: 8000, category: 'Minuman', variants: ['Original'] },
  { id: 'B4', name: 'Susu Jahe', price: 5000, category: 'Minuman', variants: ['Original'] },
  { id: 'B5', name: 'Tea Jus', price: 3000, category: 'Minuman', variants: ['Apel', 'Gula Batu'] },
  { id: 'B6', name: 'Nutrisari', price: 4000, category: 'Minuman', variants: ['Jeruk Peras', 'Sweet Orange'] },
  { id: 'B7', name: 'Susu', price: 5000, category: 'Minuman', variants: ['Putih', 'Coklat'] },
];

const App = () => {
  // State Autentikasi & Database
  const [db, setDb] = useState(null);
  const [auth, setAuth] = useState(null);
  const [userId, setUserId] = useState(null);
  const [isAuthReady, setIsAuthReady] = useState(false);

  // State Aplikasi
  const [activeTab, setActiveTab] = useState('kasir'); // 'kasir', 'dashboard', 'inventory', 'expenses'
  const [cart, setCart] = useState([]);
  const [searchTerm, setSearchTerm] = useState('');
  const [selectedItem, setSelectedItem] = useState(null);
  const [modal, setModal] = useState(null); // 'payment', 'variant', 'expense-add', 'stock-add', 'receipt'
  const [transactions, setTransactions] = useState([]);
  const [expenses, setExpenses] = useState([]);
  const [inventory, setInventory] = useState([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);
  
  // State Baru untuk Mobile (Sidebar)
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);

  // --- 1. Inisialisasi Firebase & Autentikasi ---
  useEffect(() => {
    try {
      if (Object.keys(firebaseConfig).length === 0) {
        console.error("Firebase config is missing. Please check __firebase_config.");
        return;
      }

      const app = initializeApp(firebaseConfig);
      const firestore = getFirestore(app);
      const userAuth = getAuth(app);
      setDb(firestore);
      setAuth(userAuth);

      const unsubscribe = onAuthStateChanged(userAuth, (user) => {
        if (user) {
          setUserId(user.uid);
        } else {
          // Coba login anonim jika tidak ada token, atau sign-in dengan custom token
          if (initialAuthToken) {
            signInWithCustomToken(userAuth, initialAuthToken)
              .then(userCredential => setUserId(userCredential.user.uid))
              .catch(err => {
                console.error("Custom token sign-in failed. Falling back to anonymous.", err);
                signInAnonymously(userAuth).then(userCredential => setUserId(userCredential.user.uid));
              });
          } else {
            signInAnonymously(userAuth).then(userCredential => setUserId(userCredential.user.uid));
          }
        }
        setIsAuthReady(true);
      });

      return () => unsubscribe();
    } catch (e) {
      console.error("Error initializing Firebase:", e);
      setError("Gagal menginisialisasi layanan. Cek konfigurasi.");
    }
  }, []);

  // --- 2. Real-time Listeners Firestore (Transactions, Expenses, Inventory) ---
  useEffect(() => {
    if (!db || !userId) return;

    // A. Listener Transaksi (Publik/Shared)
    const transactionsCollectionRef = collection(db, 'artifacts', appId, 'public', 'data', 'transactions');
    const qTransactions = query(transactionsCollectionRef, orderBy('timestamp', 'desc'));

    const unsubscribeTransactions = onSnapshot(qTransactions, (snapshot) => {
      const transList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setTransactions(transList);
    }, (e) => console.error("Error listening to transactions:", e));

    // B. Listener Pengeluaran (Privat/User Specific)
    const expensesCollectionRef = collection(db, 'artifacts', appId, 'users', userId, 'expenses');
    const qExpenses = query(expensesCollectionRef, orderBy('timestamp', 'desc'));

    const unsubscribeExpenses = onSnapshot(qExpenses, (snapshot) => {
      const expenseList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setExpenses(expenseList);
    }, (e) => console.error("Error listening to expenses:", e));

    // C. Listener Inventaris (Privat/User Specific)
    const inventoryCollectionRef = collection(db, 'artifacts', appId, 'users', userId, 'inventory');

    const unsubscribeInventory = onSnapshot(inventoryCollectionRef, (snapshot) => {
      const invList = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setInventory(invList);
    }, (e) => console.error("Error listening to inventory:", e));

    return () => {
      unsubscribeTransactions();
      unsubscribeExpenses();
      unsubscribeInventory();
    };
  }, [db, userId]);

  // --- 3. Logika Kasir (Keranjang & Menu) ---

  // Menampilkan menu yang sudah difilter
  const filteredMenu = useMemo(() => {
    if (!searchTerm) return menuItems;
    const lowerCaseSearch = searchTerm.toLowerCase();
    return menuItems.filter(item =>
      item.name.toLowerCase().includes(lowerCaseSearch) ||
      item.category.toLowerCase().includes(lowerCaseSearch)
    );
  }, [searchTerm]);

  // Menambah item ke keranjang
  const handleAddToCart = (item, variant) => {
    const cartItem = {
      ...item,
      variant,
      cartId: `${item.id}-${variant}-${Date.now()}`, // ID unik untuk item di keranjang
      quantity: 1,
      subtotal: item.price,
    };
    setCart(prevCart => {
      // Cek apakah item (dengan varian yang sama) sudah ada
      const existingItemIndex = prevCart.findIndex(i => i.id === item.id && i.variant === variant);
      if (existingItemIndex > -1) {
        const newCart = [...prevCart];
        newCart[existingItemIndex].quantity += 1;
        newCart[existingItemIndex].subtotal = newCart[existingItemIndex].quantity * item.price;
        return newCart;
      }
      return [...prevCart, cartItem];
    });
    setModal(null); // Tutup modal varian setelah ditambahkan
    setSelectedItem(null);
  };

  // Mengubah kuantitas di keranjang
  const updateQuantity = (cartId, delta) => {
    setCart(prevCart => prevCart.map(item => {
      if (item.cartId === cartId) {
        const newQty = item.quantity + delta;
        if (newQty <= 0) return null; // Akan dihapus oleh filter
        return { ...item, quantity: newQty, subtotal: newQty * item.price };
      }
      return item;
    }).filter(Boolean));
  };

  // Total Transaksi
  const totalAmount = useMemo(() => cart.reduce((sum, item) => sum + item.subtotal, 0), [cart]);

  // --- 4. Logika Pembayaran (Checkout & Receipt) ---

  const getInventoryItemRef = (name, variant) => {
    // Membuat ID dokumen unik berdasarkan Nama dan Varian
    return doc(collection(db, 'artifacts', appId, 'users', userId, 'inventory'), `${name}-${variant}`);
  };

  const handleCheckout = () => {
    if (cart.length === 0) {
      setError("Keranjang masih kosong.");
      return;
    }
    setError(null);
    setModal('payment');
  };

  const processPayment = async (paidAmount, changeAmount) => {
    if (loading || !db || !userId) return;

    setLoading(true);
    setError(null);

    const newTransaction = {
      timestamp: Timestamp.now(),
      total: totalAmount,
      paid: paidAmount,
      change: changeAmount,
      items: cart.map(({ cartId, ...rest }) => rest), // Hilangkan cartId unik
      kasirId: userId,
      status: 'Completed',
    };
    
    // Siapkan referensi transaksi baru
    const transRef = doc(collection(db, 'artifacts', appId, 'public', 'data', 'transactions'));

    try {
      await runTransaction(db, async (transaction) => {
        
        // =========================================================
        // FIX KRUSIAL: 1. SEMUA BACA (READS) HARUS DI AWAL
        // =========================================================
        const inventoryCollectionRef = collection(db, 'artifacts', appId, 'users', userId, 'inventory');
        const inventoryReads = [];
        const inventoryRefs = [];

        for (const item of cart) {
            const invRef = getInventoryItemRef(item.name, item.variant);
            inventoryRefs.push({ ref: invRef, item: item });
            // Kumpulkan promise dari transaction.get()
            inventoryReads.push(transaction.get(invRef));
        }

        // Tunggu semua operasi baca selesai
        const inventoryDocs = await Promise.all(inventoryReads);

        // =========================================================
        // 2. SEMUA TULIS (WRITES) SETELAH SEMUA BACA
        // =========================================================

        // A. Tulis Transaksi (Write)
        transaction.set(transRef, newTransaction);
        
        // B. Tulis Update Stok (Writes)
        inventoryDocs.forEach((invDoc, index) => {
            const { ref: invRef, item } = inventoryRefs[index];
            const stockChange = -item.quantity; // Selalu negatif karena ini penjualan

            if (invDoc.exists()) {
                const currentStock = invDoc.data().stock || 0;
                const newStock = Math.max(0, currentStock + stockChange);
                
                // Lakukan update stok
                transaction.update(invRef, { stock: newStock, lastUpdated: Timestamp.now() });
            } else {
                // Jika item di cart tidak ada di inventory, buat item baru dengan stok 0
                console.warn(`Item ${item.name} (${item.variant}) tidak ditemukan di inventaris untuk dikurangi. Membuat item dengan stok 0.`);
                transaction.set(invRef, {
                    name: item.name,
                    variant: item.variant,
                    stock: 0, 
                    lastUpdated: Timestamp.now(),
                    createdAt: Timestamp.now(),
                });
            }
        });
        
      }); // runTransaction ends

      // Transaksi berhasil, tampilkan struk
      setModal('receipt');
      setCart([]);

    } catch (e) {
      console.error("Gagal memproses pembayaran dan update stok:", e);
      setError(`Transaksi gagal diproses: ${e.message}.`);
    } finally {
      setLoading(false);
    }
  };

  // --- 5. Logika Dashboard (Perhitungan Omset) ---
  const calculateDailyOmset = useCallback((dateString) => {
    const today = new Date(dateString);
    today.setHours(0, 0, 0, 0);
    const tomorrow = new Date(today);
    tomorrow.setDate(tomorrow.getDate() + 1);

    return transactions.reduce((sum, t) => {
      // Pastikan t.timestamp adalah Firestore Timestamp
      const transDate = t.timestamp && typeof t.timestamp.toDate === 'function' ? t.timestamp.toDate() : new Date(0);
      
      if (transDate >= today && transDate < tomorrow) {
        return sum + t.total;
      }
      return sum;
    }, 0);
  }, [transactions]);

  const calculateOmsetRange = useCallback((startDate, endDate) => {
    const start = new Date(startDate);
    start.setHours(0, 0, 0, 0);
    const end = new Date(endDate);
    end.setHours(23, 59, 59, 999);

    return transactions.reduce((sum, t) => {
      // Pastikan t.timestamp adalah Firestore Timestamp
      const transDate = t.timestamp && typeof t.timestamp.toDate === 'function' ? t.timestamp.toDate() : new Date(0);
      
      if (transDate >= start && transDate <= end) {
        return sum + t.total;
      }
      return sum;
    }, 0);
  }, [transactions]);

  const dailyOmsetToday = useMemo(() => {
    const todayStr = new Date().toISOString().substring(0, 10);
    return calculateDailyOmset(todayStr);
  }, [transactions, calculateDailyOmset]);


  const totalExpenses = useMemo(() => expenses.reduce((sum, exp) => exp.amount && exp.amount > 0 ? sum + exp.amount : sum, 0), [expenses]);
  
  const netProfitToday = dailyOmsetToday - expenses.filter(e => {
    const expenseDate = e.timestamp && typeof e.timestamp.toDate === 'function' ? e.timestamp.toDate().toDateString() : '';
    return expenseDate === new Date().toDateString();
  }).reduce((sum, e) => e.amount && e.amount > 0 ? sum + e.amount : sum, 0);


  // --- 6. Logika Pengeluaran (Expenses) ---
  const addExpense = async (name, amount) => {
    if (loading || !db || !userId) return;
    setLoading(true);
    setError(null);

    const newExpense = {
      name,
      amount: Number(amount),
      timestamp: Timestamp.now(),
      userId: userId,
    };

    try {
      const expensesCollectionRef = collection(db, 'artifacts', appId, 'users', userId, 'expenses');
      await setDoc(doc(expensesCollectionRef), newExpense);
      setModal(null);
    } catch (e) {
      console.error("Gagal menambah pengeluaran:", e);
      setError(`Gagal menambah pengeluaran: ${e.message}`);
    } finally {
      setLoading(false);
    }
  };

  // --- 7. Logika Stok/Inventaris (Inventory) ---

  // Menambah atau mengupdate stok
  const addOrUpdateStock = async (name, variant, stockChange) => {
    if (loading || !db || !userId) return;
    setLoading(true);
    setError(null);

    const invRef = getInventoryItemRef(name, variant);

    try {
      await runTransaction(db, async (transaction) => {
        
        // 1. Baca (Read)
        const invDoc = await transaction.get(invRef);
        
        // 2. Tulis (Write)
        const stockToAdd = Number(stockChange);

        if (invDoc.exists()) {
          const currentStock = invDoc.data().stock || 0;
          const newStock = currentStock + stockToAdd;
          if (newStock < 0) throw new Error("Stok tidak boleh menjadi negatif.");
          transaction.update(invRef, { stock: newStock, lastUpdated: Timestamp.now() });
        } else {
          if (stockToAdd < 0) throw new Error("Tidak dapat mengurangi stok item yang belum ada.");
          transaction.set(invRef, {
            name,
            variant,
            stock: stockToAdd,
            lastUpdated: Timestamp.now(),
            createdAt: Timestamp.now(),
          });
        }
      });
      setModal(null);
    } catch (e) {
      console.error("Gagal mengupdate stok:", e);
      setError(`Gagal mengupdate stok: ${e.message}`);
    } finally {
      setLoading(false);
    }
  };


  // --- Komponen UI ---

  // Sidebar Navigasi
  const Sidebar = ({ isMobileOpen, setIsMobileOpen }) => (
    <div className={`fixed inset-y-0 left-0 z-50 w-64 bg-gray-900 text-white flex flex-col p-4 shadow-2xl transition-transform transform ${isMobileOpen ? 'translate-x-0' : '-translate-x-full'} lg:relative lg:translate-x-0 lg:flex lg:h-full`}>
      <div className='flex justify-between items-center'>
        <div className="text-2xl font-bold mb-8 text-yellow-400">Warmindo POS</div>
        <button onClick={() => setIsMobileOpen(false)} className='lg:hidden text-white mb-8'>
            <X size={24} />
        </button>
      </div>
      <div className='text-sm mb-4 p-2 bg-gray-800 rounded-lg'>
        <p className='text-xs text-gray-400'>User ID:</p>
        <p className='font-mono break-all'>{userId || "Loading..."}</p>
      </div>
      <nav className="flex-grow">
        <NavItem icon={<ShoppingCart />} label="Kasir (POS)" tab="kasir" />
        <NavItem icon={<BarChart2 />} label="Dashboard Omset" tab="dashboard" />
        <NavItem icon={<Archive />} label="Inventaris (Stok)" tab="inventory" />
        <NavItem icon={<TrendingDown />} label="Pengeluaran" tab="expenses" />
      </nav>
      <div className="pt-4 border-t border-gray-700">
        <button className="flex items-center w-full p-3 text-red-400 hover:bg-gray-700 rounded-lg transition duration-150">
          <LogOut size={20} className="mr-3" /> Logout
        </button>
      </div>
    </div>
  );

  const NavItem = ({ icon, label, tab }) => (
    <button
      onClick={() => {setActiveTab(tab); setIsSidebarOpen(false);}} // Tutup sidebar di mobile setelah klik
      className={`flex items-center w-full p-3 my-1 rounded-lg transition duration-150 ${activeTab === tab ? 'bg-yellow-600 text-white font-semibold' : 'hover:bg-gray-700 text-gray-300'}`}
    >
      {icon}
      <span className="ml-3">{label}</span>
    </button>
  );

  // Panel Menu (Kasir)
  const MenuPanel = () => (
    // Menghapus lg:overflow-y-auto di sini agar scroll ditangani oleh kontainer utama
    <div className="flex flex-col w-full lg:flex-1 p-6 bg-gray-50">
      <h2 className="text-3xl font-extrabold mb-6 text-gray-800 border-b pb-2">Daftar Menu Warmindo</h2>

      <div className="mb-6 flex items-center bg-white border border-gray-200 rounded-xl shadow-sm p-3">
        <Search className="text-gray-400 mr-3" size={20} />
        <input
          type="text"
          placeholder="Cari menu (Indomie, Kopi, Bubur...)"
          value={searchTerm}
          onChange={(e) => setSearchTerm(e.target.value)}
          className="w-full text-lg p-1 focus:outline-none bg-white placeholder-gray-500"
        />
      </div>

      <div className="grid grid-cols-2 sm:grid-cols-3 lg:grid-cols-4 xl:grid-cols-5 gap-4">
        {filteredMenu.map(item => (
          <div
            key={item.id}
            onClick={() => {
              setSelectedItem(item);
              setModal('variant');
            }}
            className="bg-white p-4 rounded-xl shadow-md hover:shadow-lg transition duration-200 cursor-pointer border border-gray-100 transform hover:scale-[1.02]"
          >
            <div className="text-xl font-bold text-gray-800 mb-1 truncate">{item.name}</div>
            <div className="text-sm text-yellow-600 font-medium">{item.category}</div>
            <div className="text-2xl font-extrabold text-green-700 mt-2">
              {formatRupiah(item.price)}
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  // Panel Keranjang (Kasir)
  const CartPanel = () => (
    // Penyesuaian class: w-full di mobile, w-96 di desktop. Border dipindahkan ke desktop.
    <div className="w-full lg:w-96 bg-white flex flex-col p-6 shadow-xl border-t lg:border-t-0 lg:border-l">
      <h2 className="text-3xl font-extrabold mb-6 text-gray-800 border-b pb-2">Keranjang <span className='text-sm text-gray-500'>({cart.length} item)</span></h2>

      <div className="flex-grow overflow-y-auto space-y-3 pr-2">
        {cart.length === 0 ? (
          <div className="text-center py-10 text-gray-500 italic">Keranjang kosong. Tambahkan menu!</div>
        ) : (
          cart.map(item => (
            <div key={item.cartId} className="flex justify-between items-center border-b pb-3 pt-1">
              <div className="flex-1">
                <div className="font-semibold text-gray-800 truncate">{item.name}</div>
                <div className="text-xs text-gray-500 italic">{item.variant}</div>
                <div className="text-sm text-green-700 font-bold">{formatRupiah(item.price)}</div>
              </div>
              <div className="flex items-center space-x-2">
                <button
                  onClick={() => updateQuantity(item.cartId, -1)}
                  className="bg-red-500 text-white w-7 h-7 rounded-full text-lg hover:bg-red-600 transition"
                >-</button>
                <span className="font-bold text-lg w-5 text-center">{item.quantity}</span>
                <button
                  onClick={() => updateQuantity(item.cartId, 1)}
                  className="bg-green-500 text-white w-7 h-7 rounded-full text-lg hover:bg-green-600 transition"
                >+</button>
              </div>
            </div>
          ))
        )}
      </div>

      <div className="mt-6 pt-4 border-t-2 border-dashed border-gray-300">
        <div className="flex justify-between items-center mb-4">
          <span className="text-2xl font-semibold text-gray-700">TOTAL</span>
          <span className="text-4xl font-extrabold text-red-600">{formatRupiah(totalAmount)}</span>
        </div>
        <button
          onClick={handleCheckout}
          disabled={cart.length === 0 || loading}
          className="w-full bg-yellow-500 text-gray-900 font-bold py-4 rounded-xl text-xl shadow-lg hover:bg-yellow-600 transition duration-200 disabled:bg-gray-400"
        >
          {loading ? 'Memproses...' : 'CHECKOUT & BAYAR'}
        </button>
      </div>
    </div>
  );

  // Modal Pilihan Varian
  const VariantModal = ({ item }) => (
    <Modal title={`Pilih Varian: ${item.name}`}>
      <div className="text-xl font-bold mb-4 text-green-700">{formatRupiah(item.price)}</div>
      <div className="grid grid-cols-2 gap-3">
        {item.variants.map(variant => (
          <button
            key={variant}
            onClick={() => handleAddToCart(item, variant)}
            className="p-3 bg-yellow-100 border-2 border-yellow-500 text-gray-800 font-medium rounded-lg hover:bg-yellow-500 hover:text-white transition"
          >
            {variant}
          </button>
        ))}
      </div>
    </Modal>
  );

  // Modal Pembayaran
  const PaymentModal = () => {
    const [paid, setPaid] = useState('');
    const paidAmount = paid ? parseInt(paid, 10) : 0;
    const changeAmount = paidAmount - totalAmount;

    return (
      <Modal title="Pembayaran Transaksi">
        <div className="mb-6 p-4 bg-gray-100 rounded-xl">
          <div className="text-xl font-medium text-gray-600">Total Tagihan:</div>
          <div className="text-5xl font-extrabold text-red-600">{formatRupiah(totalAmount)}</div>
        </div>

        <div className="mb-4">
          <label className="block text-gray-700 font-medium mb-2">Uang Dibayar Customer (Rp)</label>
          <input
            type="number"
            value={paid}
            onChange={(e) => setPaid(e.target.value)}
            className="w-full text-3xl p-4 border-2 border-yellow-400 rounded-xl focus:border-yellow-600"
            placeholder="0"
          />
        </div>

        <div className="flex flex-wrap gap-2 mb-6">
          {[10000, 20000, 50000, 100000, totalAmount].map(nominal => (
            <button
              key={nominal}
              onClick={() => setPaid(String(nominal))}
              className="px-4 py-2 bg-gray-200 rounded-lg hover:bg-gray-300 font-semibold text-sm transition"
            >
              {nominal === totalAmount ? 'Uang Pas' : formatRupiah(nominal)}
            </button>
          ))}
        </div>

        <div className={`p-4 rounded-xl font-bold text-center ${changeAmount >= 0 ? 'bg-green-100 text-green-700' : 'bg-red-100 text-red-600'}`}>
          <div className="text-xl">KEMBALIAN</div>
          <div className="text-4xl">{formatRupiah(changeAmount)}</div>
        </div>

        <button
          onClick={() => processPayment(paidAmount, changeAmount)}
          disabled={paidAmount < totalAmount || loading}
          className="w-full bg-green-600 text-white font-bold py-4 rounded-xl text-xl mt-6 hover:bg-green-700 transition disabled:bg-gray-400"
        >
          {loading ? 'Memproses...' : 'BAYAR SELESAI'}
        </button>
      </Modal>
    );
  };

  // Modal Struk/Nota (Printable)
  const ReceiptModal = () => {
    // Ambil transaksi terakhir yang baru saja disimpan
    const lastTransaction = transactions[0];
    if (!lastTransaction) return <Modal title="Struk Transaksi" />;

    const handlePrint = () => {
      // Sembunyikan elemen non-struk sebelum print
      const mainContent = document.getElementById('main-app-content');
      const receiptElement = document.getElementById('receipt-print-area');
      
      if (mainContent && receiptElement) {
        // Menyimpan status tampilan asli
        const originalDisplay = mainContent.style.display;
        const originalScroll = window.scrollY;
        
        // Hanya tampilkan area struk
        mainContent.style.display = 'none';
        receiptElement.style.display = 'block';

        // Panggil fungsi print bawaan browser
        window.print();

        // Kembalikan tampilan asli
        mainContent.style.display = originalDisplay;
        receiptElement.style.display = 'none';
        window.scrollTo(0, originalScroll);
      }
      
      // Setelah print, tutup modal
      setModal(null);
    };

    return (
      <Modal title="Struk Transaksi (Siap Cetak)">
        {/* Area yang akan dicetak, disembunyikan secara default kecuali saat print */}
        <div id="receipt-print-area" className="w-full max-w-sm mx-auto bg-white p-6 shadow-2xl print:w-full print:p-0 print:block hidden">
          <div className="text-center pb-4 border-b border-dashed border-gray-400 mb-4">
            <h3 className="text-2xl font-bold text-gray-900">WARMINDO {appId.toUpperCase().substring(0, 8)}</h3>
            <p className="text-sm">Jalan Kenangan No. 17</p>
            <p className="text-sm">Kasir: {lastTransaction.kasirId.substring(0, 8)}</p>
            {/* Memastikan t.timestamp adalah Firestore Timestamp sebelum memanggil toDate() */}
            <p className="text-sm">{lastTransaction.timestamp && typeof lastTransaction.timestamp.toDate === 'function' ? formatDate(lastTransaction.timestamp.toDate()) : 'Tanggal Tidak Tersedia'}</p>
          </div>

          <div className="space-y-2 mb-4">
            {lastTransaction.items.map((item, index) => (
              <div key={index} className="flex justify-between text-sm">
                <div>
                  {item.quantity}x {item.name} ({item.variant})
                </div>
                <div>{formatRupiah(item.subtotal)}</div>
              </div>
            ))}
          </div>

          <div className="pt-4 border-t border-dashed border-gray-400">
            <div className="flex justify-between font-bold text-lg mb-2">
              <span>TOTAL</span>
              <span>{formatRupiah(lastTransaction.total)}</span>
            </div>
            <div className="flex justify-between text-sm">
              <span>BAYAR</span>
              <span>{formatRupiah(lastTransaction.paid)}</span>
            </div>
            <div className="flex justify-between font-bold text-lg text-green-600">
              <span>KEMBALIAN</span>
              <span>{formatRupiah(lastTransaction.change)}</span>
            </div>
          </div>
          <p className="text-center text-xs mt-4 pt-2 border-t border-dashed border-gray-400">Terima kasih atas kunjungannya!</p>
        </div>
        
        {/* Pratinjau Struk di Modal */}
        <div className="p-4 border border-gray-200 rounded-lg">
             <h4 className='text-lg font-semibold mb-2'>Pratinjau Struk:</h4>
             <div className="max-h-64 overflow-y-auto p-2 border border-gray-100 bg-white">
                {/* Tampilkan konten struk yang sama di pratinjau */}
                <div className="text-center pb-2 border-b border-dashed border-gray-300 mb-2">
                    <h3 className="text-lg font-bold">WARMINDO {appId.toUpperCase().substring(0, 8)}</h3>
                    <p className="text-xs">{lastTransaction.timestamp && typeof lastTransaction.timestamp.toDate === 'function' ? formatDate(lastTransaction.timestamp.toDate()) : 'Tanggal Tidak Tersedia'}</p>
                </div>
                {lastTransaction.items.map((item, index) => (
                    <div key={index} className="flex justify-between text-xs">
                        <div className='truncate'>{item.quantity}x {item.name}</div>
                        <div>{formatRupiah(item.subtotal)}</div>
                    </div>
                ))}
                 <div className="pt-2 border-t border-dashed border-gray-300 mt-2">
                    <div className="flex justify-between font-bold text-sm">
                        <span>TOTAL</span>
                        <span>{formatRupiah(lastTransaction.total)}</span>
                    </div>
                </div>
             </div>
        </div>

        <button
          onClick={handlePrint}
          className="w-full bg-blue-600 text-white font-bold py-3 rounded-xl text-lg mt-6 hover:bg-blue-700 transition print:hidden"
        >
          Cetak Struk (PDF/Printer)
        </button>
        <button
          onClick={() => setModal(null)}
          className="w-full bg-gray-200 text-gray-800 font-bold py-3 rounded-xl text-lg mt-2 hover:bg-gray-300 transition print:hidden"
        >
          Selesai
        </button>
      </Modal>
    );
  };

  // Dashboard Omset
  const DashboardPanel = () => {
    // State untuk kalender dan rentang waktu
    const [selectedDate, setSelectedDate] = useState(new Date().toISOString().substring(0, 10));
    const [viewRange, setViewRange] = useState('daily');
    const [startDate, setStartDate] = useState(new Date().toISOString().substring(0, 10));
    const [endDate, setEndDate] = useState(new Date().toISOString().substring(0, 10));

    const totalOmsetDaily = useMemo(() => calculateDailyOmset(selectedDate), [selectedDate, calculateDailyOmset]);
    const totalOmsetRange = useMemo(() => calculateOmsetRange(startDate, endDate), [startDate, endDate, calculateOmsetRange]);
    
    // Perhitungan Total Pengeluaran untuk Range yang dipilih
    const totalExpenseRange = useMemo(() => {
        const start = new Date(startDate);
        start.setHours(0, 0, 0, 0);
        const end = new Date(endDate);
        end.setHours(23, 59, 59, 999);

        return expenses.reduce((sum, exp) => {
            const expenseDate = exp.timestamp && typeof exp.timestamp.toDate === 'function' ? exp.timestamp.toDate() : new Date(0);
            if (expenseDate >= start && expenseDate <= end) {
                return sum + exp.amount;
            }
            return sum;
        }, 0);
    }, [startDate, endDate, expenses]);
    
    const netProfitRange = totalOmsetRange - totalExpenseRange;

    // Menghitung omset berdasarkan viewRange
    const displayOmset = viewRange === 'daily' ? totalOmsetDaily : totalOmsetRange;
    const displayExpenses = viewRange === 'daily' ? expenses.filter(e => {
        const expenseDate = e.timestamp && typeof e.timestamp.toDate === 'function' ? e.timestamp.toDate().toDateString() : '';
        return expenseDate === new Date(selectedDate).toDateString();
    }).reduce((sum, e) => e.amount && e.amount > 0 ? sum + e.amount : sum, 0) : totalExpenseRange;
    const displayProfit = displayOmset - displayExpenses;


    return (
      <div className="p-6 bg-gray-50 overflow-y-auto w-full">
        <h2 className="text-3xl font-extrabold mb-6 text-gray-800 border-b pb-2">Dashboard Penjualan & Omset</h2>

        {/* Info Ringkas Hari Ini */}
        <div className="grid grid-cols-1 md:grid-cols-3 gap-6 mb-8">
          <Card icon={<DollarSign size={24} />} title="Omset Hari Ini" value={dailyOmsetToday} color="bg-yellow-500" />
          <Card icon={<TrendingDown size={24} />} title="Pengeluaran Hari Ini" value={totalExpenses} color="bg-red-500" />
          <Card icon={<BarChart2 size={24} />} title="Profit Bersih Hari Ini" value={netProfitToday} color="bg-green-500" />
        </div>

        {/* Filter Omset Berdasarkan Waktu */}
        <div className="bg-white p-6 rounded-xl shadow-lg mb-8">
          <h3 className="text-xl font-bold mb-4 text-gray-800">Cek Omset Berdasarkan Periode</h3>

          <div className="flex space-x-2 mb-4">
            {['daily', 'weekly', 'monthly', 'yearly'].map(v => (
              <button
                key={v}
                onClick={() => setViewRange(v)}
                className={`px-4 py-2 rounded-full font-semibold transition ${viewRange === v ? 'bg-yellow-500 text-white' : 'bg-gray-200 text-gray-700 hover:bg-gray-300'}`}
              >
                {v === 'daily' ? 'Harian' : v === 'weekly' ? 'Mingguan' : v === 'monthly' ? 'Bulanan' : 'Tahunan'}
              </button>
            ))}
          </div>

          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            {viewRange === 'daily' && (
              <div>
                <label className="block text-sm font-medium text-gray-700 mb-1">Pilih Tanggal</label>
                <input
                  type="date"
                  value={selectedDate}
                  onChange={(e) => setSelectedDate(e.target.value)}
                  className="w-full p-2 border rounded-lg"
                />
              </div>
            )}
            {viewRange !== 'daily' && (
              <>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Tanggal Mulai</label>
                  <input
                    type="date"
                    value={startDate}
                    onChange={(e) => setStartDate(e.target.value)}
                    className="w-full p-2 border rounded-lg"
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-gray-700 mb-1">Tanggal Akhir</label>
                  <input
                    type="date"
                    value={endDate}
                    onChange={(e) => setEndDate(e.target.value)}
                    className="w-full p-2 border rounded-lg"
                  />
                </div>
              </>
            )}
          </div>
          
          <div className="mt-6 p-4 bg-yellow-50 border-2 border-yellow-300 rounded-xl">
            <p className="text-xl font-medium text-gray-700">Omset Penjualan:</p>
            <p className="text-4xl font-extrabold text-yellow-700">{formatRupiah(displayOmset)}</p>
          </div>
          <div className="mt-4 p-4 bg-red-50 border-2 border-red-300 rounded-xl">
            <p className="text-xl font-medium text-gray-700">Total Pengeluaran:</p>
            <p className="text-4xl font-extrabold text-red-700">{formatRupiah(displayExpenses)}</p>
          </div>
          <div className="mt-4 p-4 bg-green-50 border-2 border-green-300 rounded-xl">
            <p className="text-xl font-medium text-gray-700">Profit Bersih:</p>
            <p className="text-4xl font-extrabold text-green-700">{formatRupiah(displayProfit)}</p>
          </div>

        </div>

        {/* Tabel Riwayat Transaksi */}
        <div className="bg-white p-6 rounded-xl shadow-lg">
          <h3 className="text-xl font-bold mb-4 text-gray-800">Riwayat Transaksi Terakhir</h3>
          <div className="overflow-x-auto">
            <table className="min-w-full divide-y divide-gray-200">
              <thead>
                <tr className='bg-gray-100'>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Waktu</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Total</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Items</th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {transactions.slice(0, 10).map(t => (
                  <tr key={t.id}>
                    {/* Memastikan t.timestamp adalah Firestore Timestamp sebelum memanggil toDate() */}
                    <td className="px-4 py-4 whitespace-nowrap text-sm text-gray-900">{t.timestamp && typeof t.timestamp.toDate === 'function' ? formatDate(t.timestamp.toDate()) : 'N/A'}</td>
                    <td className="px-4 py-4 whitespace-nowrap text-sm font-bold text-green-600">{formatRupiah(t.total)}</td>
                    <td className="px-4 py-4 text-xs text-gray-500">
                      {t.items.map(item => `${item.quantity}x ${item.name} (${item.variant})`).join(', ')}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>

      </div>
    );
  };

  const Card = ({ icon, title, value, color }) => (
    <div className={`p-5 rounded-xl shadow-md flex items-center space-x-4 text-white ${color}`}>
      <div className={`p-3 rounded-full bg-opacity-30 ${color.replace('bg-', 'bg-')}`}>{icon}</div>
      <div>
        <div className="text-sm font-medium">{title}</div>
        <div className="text-2xl font-extrabold">{formatRupiah(value)}</div>
      </div>
    </div>
  );

  // Panel Pengeluaran
  const ExpensesPanel = () => {
    const [name, setName] = useState('');
    const [amount, setAmount] = useState('');

    const ExpenseAddModal = () => (
      <Modal title="Tambah Pengeluaran Baru">
        <label className="block text-gray-700 font-medium mb-1">Nama Pengeluaran (e.g., Beli Gas, Listrik)</label>
        <input
          type="text"
          value={name}
          onChange={(e) => setName(e.target.value)}
          className="w-full p-3 border rounded-lg mb-4"
          placeholder="Nama Pengeluaran"
        />
        <label className="block text-gray-700 font-medium mb-1">Jumlah (Rp)</label>
        <input
          type="number"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
          className="w-full p-3 border rounded-lg mb-6"
          placeholder="0"
        />
        <button
          onClick={() => addExpense(name, amount)}
          disabled={!name || !amount || loading}
          className="w-full bg-red-500 text-white font-bold py-3 rounded-xl hover:bg-red-600 transition disabled:bg-gray-400"
        >
          {loading ? 'Menyimpan...' : 'Simpan Pengeluaran'}
        </button>
      </Modal>
    );

    return (
      <div className="p-6 bg-gray-50 overflow-y-auto w-full">
        <h2 className="text-3xl font-extrabold mb-6 text-gray-800 border-b pb-2">Manajemen Pengeluaran</h2>

        <div className="flex justify-between items-center mb-6 bg-white p-4 rounded-xl shadow">
          <div className="text-xl font-medium">Total Pengeluaran Sepanjang Waktu:</div>
          <div className="text-3xl font-extrabold text-red-600">{formatRupiah(totalExpenses)}</div>
        </div>

        <button
          onClick={() => { setModal('expense-add'); setName(''); setAmount(''); }}
          className="flex items-center bg-red-500 text-white font-bold py-3 px-6 rounded-xl hover:bg-red-600 transition mb-6 shadow-md"
        >
          <Plus size={20} className="mr-2" /> Tambah Pengeluaran
        </button>

        <div className="bg-white p-6 rounded-xl shadow-lg">
          <h3 className="text-xl font-bold mb-4 text-gray-800">Riwayat Pengeluaran</h3>
          <div className="overflow-x-auto">
            <table className="min-w-full divide-y divide-gray-200">
              <thead>
                <tr className='bg-gray-100'>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Waktu</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Jumlah</th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {expenses.map(exp => (
                  <tr key={exp.id}>
                    {/* Memastikan exp.timestamp adalah Firestore Timestamp sebelum memanggil toDate() */}
                    <td className="px-4 py-4 whitespace-nowrap text-sm text-gray-500">{exp.timestamp && typeof exp.timestamp.toDate === 'function' ? formatDate(exp.timestamp.toDate()) : 'N/A'}</td>
                    <td className="px-4 py-4 whitespace-nowrap text-sm font-medium text-gray-900">{exp.name}</td>
                    <td className="px-4 py-4 whitespace-nowrap text-sm font-bold text-red-600">{formatRupiah(exp.amount)}</td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
        {modal === 'expense-add' && <ExpenseAddModal />}
      </div>
    );
  };

  // Panel Inventaris/Stok
  const InventoryPanel = () => {
    const [stockName, setStockName] = useState('');
    const [stockVariant, setStockVariant] = useState('');
    const [stockAmount, setStockAmount] = useState('');
    const [selectedItemForStock, setSelectedItemForStock] = useState(null); // Item yang dipilih dari tabel untuk diubah stoknya

    const StockAddModal = () => (
      <Modal title="Tambah/Ubah Stok Barang">
        <label className="block text-gray-700 font-medium mb-1">Nama Barang</label>
        <input
          type="text"
          value={stockName}
          onChange={(e) => setStockName(e.target.value)}
          className="w-full p-3 border rounded-lg mb-4 bg-gray-100"
          placeholder="Nama Barang (e.g. Indomie Original)"
          disabled={!!selectedItemForStock}
        />
        <label className="block text-gray-700 font-medium mb-1">Varian/Keterangan</label>
        <input
          type="text"
          value={stockVariant}
          onChange={(e) => setStockVariant(e.target.value)}
          className="w-full p-3 border rounded-lg mb-4 bg-gray-100"
          placeholder="Varian (e.g. Putih, Rendang)"
          disabled={!!selectedItemForStock}
        />
        <label className="block text-gray-700 font-medium mb-1">Perubahan Jumlah Stok (+/-)</label>
        <input
          type="number"
          value={stockAmount}
          onChange={(e) => setStockAmount(e.target.value)}
          className="w-full p-3 border rounded-lg mb-6"
          placeholder="Contoh: 10 (tambah 10), -5 (kurangi 5)"
        />
        <button
          onClick={() => addOrUpdateStock(stockName, stockVariant, stockAmount)}
          disabled={!stockName || !stockVariant || !stockAmount || loading}
          className="w-full bg-blue-500 text-white font-bold py-3 rounded-xl hover:bg-blue-600 transition disabled:bg-gray-400"
        >
          {loading ? 'Menyimpan...' : 'Simpan Perubahan Stok'}
        </button>
      </Modal>
    );

    const handleOpenStockModal = (item = null) => {
      if (item) {
        setSelectedItemForStock(item);
        setStockName(item.name);
        setStockVariant(item.variant);
        setStockAmount('');
      } else {
        setSelectedItemForStock(null);
        setStockName('');
        setStockVariant('');
        setStockAmount('');
      }
      setModal('stock-add');
    };

    const sortedInventory = useMemo(() => {
        return [...inventory].sort((a, b) => b.stock - a.stock);
    }, [inventory]);

    return (
      <div className="p-6 bg-gray-50 overflow-y-auto w-full">
        <h2 className="text-3xl font-extrabold mb-6 text-gray-800 border-b pb-2">Manajemen Inventaris (Stok)</h2>

        <button
          onClick={() => handleOpenStockModal()}
          className="flex items-center bg-blue-500 text-white font-bold py-3 px-6 rounded-xl hover:bg-blue-600 transition mb-6 shadow-md"
        >
          <Plus size={20} className="mr-2" /> Tambah Item Baru / Update Stok
        </button>

        <div className="bg-white p-6 rounded-xl shadow-lg">
          <h3 className="text-xl font-bold mb-4 text-gray-800">Daftar Stok Barang</h3>
          <div className="overflow-x-auto">
            <table className="min-w-full divide-y divide-gray-200">
              <thead>
                <tr className='bg-gray-100'>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Nama Barang</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Varian</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Stok Saat Ini</th>
                  <th className="px-4 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Aksi</th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {sortedInventory.map(item => (
                  <tr key={item.id} className={item.stock < 5 ? 'bg-red-50' : ''}>
                    <td className="px-4 py-4 whitespace-nowrap text-sm font-medium text-gray-900">{item.name}</td>
                    <td className="px-4 py-4 whitespace-nowrap text-sm text-gray-500">{item.variant}</td>
                    <td className={`px-4 py-4 whitespace-nowrap text-lg font-bold ${item.stock < 5 ? 'text-red-600 animate-pulse' : 'text-green-700'}`}>
                      {item.stock}
                    </td>
                    <td className="px-4 py-4 whitespace-nowrap text-sm">
                        <button
                            onClick={() => handleOpenStockModal(item)}
                            className="text-blue-600 hover:text-blue-800 font-semibold"
                        >
                            Ubah Stok
                        </button>
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
        {modal === 'stock-add' && <StockAddModal />}
      </div>
    );
  };


  // Komponen Modal Universal
  const Modal = ({ title, children }) => (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black bg-opacity-50 backdrop-blur-sm p-4">
      <div className="bg-white rounded-2xl shadow-2xl w-full max-w-lg p-6 relative transform transition-all">
        <button
          onClick={() => setModal(null)}
          className="absolute top-4 right-4 text-gray-500 hover:text-gray-800 transition"
        >
          <X size={24} />
        </button>
        <h3 className="text-2xl font-extrabold mb-5 border-b pb-2 text-gray-800">{title}</h3>
        {children}
      </div>
    </div>
  );

  // Loading dan Error
  if (!isAuthReady) {
    return (
      <div className="flex h-screen items-center justify-center bg-gray-100">
        <div className="text-xl font-semibold text-yellow-600">
          Menginisialisasi Warmindo POS...
        </div>
      </div>
    );
  }

  // Render Utama Aplikasi
  return (
    <div className="flex h-screen w-screen font-inter">
      <style>
        {`
        /* CSS untuk Struk Print */
        @media print {
            body * {
                visibility: hidden;
            }
            /* Hanya tampilkan area struk saat mencetak */
            #receipt-print-area, #receipt-print-area * {
                visibility: visible;
            }
            #receipt-print-area {
                position: absolute;
                left: 0;
                top: 0;
                width: 100%;
                max-width: none;
                margin: 0;
                padding: 0;
                font-family: monospace; /* Font monospace untuk kesan struk */
            }
            .print\\:hidden {
                display: none !important;
            }
            /* Pastikan elemen cetak tidak memiliki padding/margin yang mengganggu */
            .print\\:w-full {
                width: 100% !important;
            }
            .print\\:p-0 {
                padding: 0 !important;
            }
            .print\\:block {
                display: block !important;
            }
        }
        `}
      </style>

      {/* Sidebar Navigasi (Responsive: Overlay di mobile, Statis di desktop) */}
      {isSidebarOpen && (
        <div className="fixed inset-0 z-40 bg-black bg-opacity-50 lg:hidden" onClick={() => setIsSidebarOpen(false)}></div>
      )}
      <Sidebar isMobileOpen={isSidebarOpen} setIsMobileOpen={setIsSidebarOpen} />

      {/* Kontainer Utama yang menangani SCROLL untuk seluruh konten */}
      <div id="main-app-content" className="flex flex-1 flex-col overflow-y-auto">
        
        {/* Mobile Header/Toggle (Hidden on desktop) */}
        <div className="lg:hidden flex justify-between items-center p-4 bg-gray-900 text-white shadow-lg">
          <button onClick={() => setIsSidebarOpen(true)} className="p-2">
            <Menu size={24} />
          </button>
          <div className="text-xl font-bold text-yellow-400">Warmindo POS</div>
          <div className="text-sm font-medium p-2 rounded bg-gray-700">
            {activeTab.toUpperCase()}
          </div>
        </div>
        
        {/* Konten Utama Aplikasi */}
        <div className="flex flex-1"> 
          {activeTab === 'kasir' && (
            // FIX: Gunakan flex-col di mobile (stacking) dan lg:flex-row di desktop (side-by-side)
            <div className="flex flex-col lg:flex-row flex-1">
              <MenuPanel />
              <CartPanel />
            </div>
          )}
          {activeTab === 'dashboard' && <DashboardPanel />}
          {activeTab === 'inventory' && <InventoryPanel />}
          {activeTab === 'expenses' && <ExpensesPanel />}
        </div>
      </div>

      {/* Modals */}
      {modal === 'variant' && selectedItem && <VariantModal item={selectedItem} />}
      {modal === 'payment' && <PaymentModal />}
      {modal === 'receipt' && <ReceiptModal />}

      {/* Notifikasi Error */}
      {error && (
        <div className="fixed bottom-4 right-4 p-4 bg-red-600 text-white rounded-lg shadow-xl flex items-center">
          <X size={20} className="mr-2" />
          <p className="font-medium">{error}</p>
          <button onClick={() => setError(null)} className="ml-4 font-bold">Tutup</button>
        </div>
      )}
    </div>
  );
};

export default App;

