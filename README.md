import React, { useState, useEffect, useMemo } from 'react';
import { 
  Leaf, 
  MapPin, 
  ShoppingBasket, 
  User, 
  Search, 
  ChevronRight, 
  Star, 
  Filter, 
  Clock, 
  ArrowLeft,
  X,
  Plus,
  Minus,
  CheckCircle,
  MessageSquare,
  Sparkles,
  Loader2,
  Package,
  Home
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  getDoc, 
  onSnapshot, 
  query, 
  addDoc, 
  updateDoc,
  orderBy
} from 'firebase/firestore';
import { 
  getAuth, 
  signInWithCustomToken, 
  signInAnonymously, 
  onAuthStateChanged 
} from 'firebase/auth';

// --- Firebase Configuration & Initialization ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'harvest-connect-v1';

// --- Mock Data ---
const CATEGORIES = ['All', 'Vegetables', 'Fruits', 'Dairy', 'Grains', 'Honey', 'Eggs'];

const FARMS = [
  {
    id: 'f1',
    name: 'Green Valley Organic',
    distance: '2.4 miles',
    rating: 4.8,
    reviews: 124,
    image: 'https://images.unsplash.com/photo-1500382017468-9049fed747ef?auto=format&fit=crop&q=80&w=400',
    tags: ['Certified Organic', 'Family Owned']
  },
  {
    id: 'f2',
    name: 'Sun-Kissed Orchards',
    distance: '5.1 miles',
    rating: 4.9,
    reviews: 89,
    image: 'https://images.unsplash.com/photo-1464226184884-fa280b87c399?auto=format&fit=crop&q=80&w=400',
    tags: ['Sustainable', 'BIPOC Owned']
  },
  {
    id: 'f3',
    name: 'Hilltop Apiary',
    distance: '1.2 miles',
    rating: 4.7,
    reviews: 56,
    image: 'https://images.unsplash.com/photo-1587049633562-ad3552a3f529?auto=format&fit=crop&q=80&w=400',
    tags: ['Beekeeping', 'Wildflower']
  }
];

const PRODUCTS = [
  { id: 'p1', farmId: 'f1', name: 'Heirloom Tomatoes', price: 4.50, unit: 'lb', category: 'Vegetables', image: 'https://images.unsplash.com/photo-1518977676601-b53f02bad673?auto=format&fit=crop&q=80&w=200', stock: 15 },
  { id: 'p2', farmId: 'f1', name: 'Curly Kale', price: 3.00, unit: 'bunch', category: 'Vegetables', image: 'https://images.unsplash.com/photo-1524179524329-5ca9cae62030?auto=format&fit=crop&q=80&w=200', stock: 20 },
  { id: 'p3', farmId: 'f2', name: 'Honeycrisp Apples', price: 6.00, unit: '3lb bag', category: 'Fruits', image: 'https://images.unsplash.com/photo-1567306226416-28f0efdc88ce?auto=format&fit=crop&q=80&w=200', stock: 12 },
  { id: 'p4', farmId: 'f3', name: 'Wildflower Honey', price: 12.00, unit: '12oz jar', category: 'Honey', image: 'https://images.unsplash.com/photo-1589733955941-5eeaf752f6dd?auto=format&fit=crop&q=80&w=200', stock: 8 },
  { id: 'p5', farmId: 'f1', name: 'Pasture-Raised Eggs', price: 7.50, unit: 'dozen', category: 'Eggs', image: 'https://images.unsplash.com/photo-1582722134903-b12687359ec1?auto=format&fit=crop&q=80&w=200', stock: 24 }
];

// --- Utilities ---
const fetchRecipeSuggestions = async (cartItems) => {
  const apiKey = "";
  const itemsText = cartItems.map(item => `${item.quantity} ${item.unit} of ${item.name}`).join(', ');
  const userQuery = `I have the following organic ingredients from local farmers: ${itemsText}. Can you suggest 2 simple, seasonal recipe ideas I could make with these? Keep descriptions concise and highlighting the fresh ingredients.`;
  const systemPrompt = "You are a farm-to-table chef expert. Provide 2 short recipe titles and brief cooking steps in a warm, encouraging tone.";

  try {
    const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        contents: [{ parts: [{ text: userQuery }] }],
        systemInstruction: { parts: [{ text: systemPrompt }] }
      })
    });
    const result = await response.json();
    return result.candidates?.[0]?.content?.parts?.[0]?.text || "No recipes found at this time.";
  } catch (error) {
    console.error("Gemini Error:", error);
    return "Could not load recipe suggestions.";
  }
};

// --- Main App Component ---
export default function App() {
  const [view, setView] = useState('home'); // home, farm-details, cart, profile, recipes
  const [user, setUser] = useState(null);
  const [selectedFarm, setSelectedFarm] = useState(null);
  const [cart, setCart] = useState([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [activeCategory, setActiveCategory] = useState('All');
  const [recipeData, setRecipeData] = useState(null);
  const [loadingRecipes, setLoadingRecipes] = useState(false);
  const [orders, setOrders] = useState([]);

  // --- Firebase Auth ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (u) => {
      if (u) setUser(u);
    });
    return () => unsubscribe();
  }, []);

  // --- Firestore Sync (Orders) ---
  useEffect(() => {
    if (!user) return;
    const ordersRef = collection(db, 'artifacts', appId, 'users', user.uid, 'orders');
    const unsubscribe = onSnapshot(ordersRef, (snapshot) => {
      const fetchedOrders = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      setOrders(fetchedOrders);
    }, (err) => console.error("Firestore Error:", err));
    return () => unsubscribe();
  }, [user]);

  // --- Actions ---
  const addToCart = (product) => {
    setCart(prev => {
      const existing = prev.find(item => item.id === product.id);
      if (existing) {
        return prev.map(item => 
          item.id === product.id ? { ...item, quantity: item.quantity + 1 } : item
        );
      }
      return [...prev, { ...product, quantity: 1 }];
    });
  };

  const removeFromCart = (productId) => {
    setCart(prev => prev.filter(item => item.id !== productId));
  };

  const updateQuantity = (productId, delta) => {
    setCart(prev => prev.map(item => {
      if (item.id === productId) {
        const newQty = Math.max(1, item.quantity + delta);
        return { ...item, quantity: newQty };
      }
      return item;
    }));
  };

  const checkout = async () => {
    if (cart.length === 0 || !user) return;
    
    const newOrder = {
      items: cart,
      total: cart.reduce((sum, item) => sum + (item.price * item.quantity), 0),
      timestamp: Date.now(),
      status: 'Processing'
    };

    try {
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'orders'), newOrder);
      setCart([]);
      setView('profile');
    } catch (err) {
      console.error("Checkout error:", err);
    }
  };

  const getRecipes = async () => {
    if (cart.length === 0) return;
    setLoadingRecipes(true);
    setView('recipes');
    const content = await fetchRecipeSuggestions(cart);
    setRecipeData(content);
    setLoadingRecipes(false);
  };

  // --- Filtered Content ---
  const filteredFarms = FARMS.filter(f => 
    f.name.toLowerCase().includes(searchQuery.toLowerCase())
  );

  const totalCartPrice = cart.reduce((sum, item) => sum + (item.price * item.quantity), 0);

  // --- Views ---
  const renderHome = () => (
    <div className="animate-in fade-in duration-500">
      <div className="relative h-64 bg-emerald-600 rounded-3xl overflow-hidden mb-8 group">
        <img 
          src="https://images.unsplash.com/photo-1542838132-92c53300491e?auto=format&fit=crop&q=80&w=1000" 
          className="w-full h-full object-cover opacity-60 transition-transform duration-700 group-hover:scale-105"
          alt="Farmer Market"
        />
        <div className="absolute inset-0 p-8 flex flex-col justify-end text-white">
          <h1 className="text-4xl font-bold mb-2">Farm Fresh, To Your Door</h1>
          <p className="text-emerald-50 text-lg">Direct access to local organic growers.</p>
        </div>
      </div>

      <div className="flex items-center gap-3 mb-8 overflow-x-auto pb-2 scrollbar-hide">
        {CATEGORIES.map(cat => (
          <button
            key={cat}
            onClick={() => setActiveCategory(cat)}
            className={`px-6 py-2 rounded-full whitespace-nowrap transition-all ${
              activeCategory === cat 
                ? 'bg-emerald-600 text-white shadow-lg' 
                : 'bg-white text-slate-600 hover:bg-slate-50 border border-slate-100'
            }`}
          >
            {cat}
          </button>
        ))}
      </div>

      <h2 className="text-2xl font-bold mb-6 text-slate-800">Nearby Farms</h2>
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
        {filteredFarms.map(farm => (
          <div 
            key={farm.id}
            onClick={() => { setSelectedFarm(farm); setView('farm-details'); }}
            className="bg-white rounded-2xl overflow-hidden border border-slate-100 shadow-sm hover:shadow-xl transition-all cursor-pointer group"
          >
            <div className="h-48 overflow-hidden relative">
              <img src={farm.image} alt={farm.name} className="w-full h-full object-cover group-hover:scale-110 transition-transform duration-500" />
              <div className="absolute top-4 left-4 bg-white/90 backdrop-blur px-3 py-1 rounded-full text-xs font-semibold text-emerald-700 flex items-center gap-1">
                <MapPin size={12} /> {farm.distance}
              </div>
            </div>
            <div className="p-5">
              <div className="flex justify-between items-start mb-2">
                <h3 className="text-lg font-bold text-slate-800">{farm.name}</h3>
                <div className="flex items-center text-amber-500 gap-1 font-medium text-sm">
                  <Star size={14} fill="currentColor" /> {farm.rating}
                </div>
              </div>
              <div className="flex flex-wrap gap-2">
                {farm.tags.map(tag => (
                  <span key={tag} className="text-[10px] uppercase tracking-wider font-bold text-slate-400 border border-slate-100 px-2 py-0.5 rounded">
                    {tag}
                  </span>
                ))}
              </div>
            </div>
          </div>
        ))}
      </div>
    </div>
  );

  const renderFarmDetails = () => (
    <div className="animate-in slide-in-from-right duration-300">
      <button onClick={() => setView('home')} className="mb-6 flex items-center gap-2 text-slate-500 hover:text-emerald-600 transition-colors">
        <ArrowLeft size={20} /> Back to Farms
      </button>

      <div className="flex flex-col md:flex-row gap-8 mb-12">
        <div className="w-full md:w-1/3">
          <img src={selectedFarm.image} className="w-full aspect-square object-cover rounded-3xl shadow-lg" alt={selectedFarm.name} />
          <div className="mt-6 p-6 bg-white rounded-2xl border border-slate-100 shadow-sm">
            <h3 className="font-bold text-slate-800 mb-4">Farm Info</h3>
            <div className="space-y-4">
              <div className="flex items-center gap-3 text-slate-600 text-sm">
                <Clock size={16} className="text-emerald-500" />
                <span>Next Delivery: Tomorrow, 9am - 12pm</span>
              </div>
              <div className="flex items-center gap-3 text-slate-600 text-sm">
                <MapPin size={16} className="text-emerald-500" />
                <span>123 Harvest Lane, Greenfield</span>
              </div>
              <div className="flex items-center gap-3 text-slate-600 text-sm">
                <MessageSquare size={16} className="text-emerald-500" />
                <span>Quick Responder (avg 10m)</span>
              </div>
            </div>
          </div>
        </div>

        <div className="flex-1">
          <div className="flex justify-between items-end mb-8">
            <div>
              <h1 className="text-3xl font-bold text-slate-800 mb-2">{selectedFarm.name}</h1>
              <div className="flex items-center gap-4 text-sm text-slate-500">
                <span className="flex items-center gap-1"><Star size={14} fill="#f59e0b" className="text-amber-500" /> {selectedFarm.rating} ({selectedFarm.reviews} reviews)</span>
                <span>•</span>
                <span>Member since 2021</span>
              </div>
            </div>
          </div>

          <h2 className="text-xl font-bold mb-6 text-slate-800">Fresh Produce</h2>
          <div className="grid grid-cols-1 sm:grid-cols-2 gap-4">
            {PRODUCTS.filter(p => p.farmId === selectedFarm.id).map(product => (
              <div key={product.id} className="bg-white p-4 rounded-2xl border border-slate-100 shadow-sm flex gap-4 hover:border-emerald-200 transition-colors">
                <img src={product.image} className="w-24 h-24 object-cover rounded-xl" alt={product.name} />
                <div className="flex-1 flex flex-col justify-between">
                  <div>
                    <h4 className="font-bold text-slate-800">{product.name}</h4>
                    <p className="text-sm text-slate-500">${product.price.toFixed(2)} / {product.unit}</p>
                  </div>
                  <div className="flex justify-between items-center mt-2">
                    <span className={`text-[10px] font-bold px-2 py-0.5 rounded-full ${product.stock < 10 ? 'bg-amber-100 text-amber-700' : 'bg-emerald-100 text-emerald-700'}`}>
                      {product.stock} left
                    </span>
                    <button 
                      onClick={() => addToCart(product)}
                      className="p-2 bg-emerald-600 text-white rounded-lg hover:bg-emerald-700 transition-colors shadow-sm"
                    >
                      <Plus size={18} />
                    </button>
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>
      </div>
    </div>
  );

  const renderCart = () => (
    <div className="max-w-2xl mx-auto animate-in slide-in-from-bottom duration-300">
      <h1 className="text-3xl font-bold mb-8 text-slate-800">Your Harvest Basket</h1>
      
      {cart.length === 0 ? (
        <div className="text-center py-20 bg-slate-50 rounded-3xl border border-dashed border-slate-200">
          <ShoppingBasket size={64} className="mx-auto text-slate-300 mb-4" />
          <h3 className="text-xl font-medium text-slate-500">Your basket is empty</h3>
          <button onClick={() => setView('home')} className="mt-4 text-emerald-600 font-bold hover:underline">Start browsing</button>
        </div>
      ) : (
        <div className="space-y-4">
          <div className="bg-emerald-50 border border-emerald-100 p-6 rounded-2xl mb-8 flex items-center justify-between">
            <div className="flex items-center gap-3">
              <Sparkles className="text-emerald-600" />
              <div>
                <p className="font-bold text-emerald-900">Chef AI Tip</p>
                <p className="text-sm text-emerald-700">I can create a personalized meal plan from your basket items!</p>
              </div>
            </div>
            <button 
              onClick={getRecipes}
              className="bg-emerald-600 text-white px-4 py-2 rounded-xl text-sm font-bold shadow-lg shadow-emerald-200 hover:bg-emerald-700 transition-all flex items-center gap-2"
            >
              Suggest Recipes
            </button>
          </div>

          {cart.map(item => (
            <div key={item.id} className="bg-white p-5 rounded-2xl border border-slate-100 shadow-sm flex items-center gap-4">
              <img src={item.image} className="w-16 h-16 object-cover rounded-xl" alt={item.name} />
              <div className="flex-1">
                <h4 className="font-bold text-slate-800">{item.name}</h4>
                <p className="text-sm text-slate-500">${item.price.toFixed(2)} / {item.unit}</p>
              </div>
              <div className="flex items-center gap-3 bg-slate-50 rounded-xl p-1 border border-slate-100">
                <button onClick={() => updateQuantity(item.id, -1)} className="p-1 hover:text-emerald-600 transition-colors"><Minus size={16} /></button>
                <span className="w-8 text-center font-bold text-slate-700">{item.quantity}</span>
                <button onClick={() => updateQuantity(item.id, 1)} className="p-1 hover:text-emerald-600 transition-colors"><Plus size={16} /></button>
              </div>
              <button onClick={() => removeFromCart(item.id)} className="p-2 text-slate-300 hover:text-red-500 transition-colors"><X size={18} /></button>
            </div>
          ))}

          <div className="mt-8 bg-white p-8 rounded-3xl border border-slate-100 shadow-md">
            <div className="flex justify-between items-center mb-6">
              <span className="text-slate-500 font-medium">Subtotal</span>
              <span className="text-2xl font-bold text-slate-800">${totalCartPrice.toFixed(2)}</span>
            </div>
            <button 
              onClick={checkout}
              className="w-full bg-emerald-600 text-white py-4 rounded-2xl font-bold text-lg hover:bg-emerald-700 shadow-xl shadow-emerald-200 transition-all active:scale-95"
            >
              Complete Secure Checkout
            </button>
          </div>
        </div>
      )}
    </div>
  );

  const renderProfile = () => (
    <div className="max-w-2xl mx-auto animate-in slide-in-from-top duration-300">
      <div className="flex items-center gap-6 mb-12">
        <div className="w-24 h-24 bg-emerald-100 text-emerald-600 rounded-full flex items-center justify-center border-4 border-white shadow-lg">
          <User size={40} />
        </div>
        <div>
          <h1 className="text-3xl font-bold text-slate-800">Organic Enthusiast</h1>
          <p className="text-slate-500">Member since March 2024 • ID: {user?.uid.slice(0, 8)}</p>
        </div>
      </div>

      <h2 className="text-xl font-bold mb-6 text-slate-800">Recent Orders</h2>
      {orders.length === 0 ? (
        <div className="p-12 text-center bg-slate-50 rounded-2xl border border-dashed border-slate-200">
          <Package size={40} className="mx-auto text-slate-300 mb-2" />
          <p className="text-slate-500">No orders yet. Your locally sourced items will appear here.</p>
        </div>
      ) : (
        <div className="space-y-4">
          {orders.map(order => (
            <div key={order.id} className="bg-white p-6 rounded-2xl border border-slate-100 shadow-sm">
              <div className="flex justify-between items-start mb-4">
                <div>
                  <p className="text-xs font-bold text-slate-400 uppercase tracking-widest">Order ID: {order.id.slice(0, 8)}</p>
                  <p className="text-slate-500 text-sm">{new Date(order.timestamp).toLocaleDateString()}</p>
                </div>
                <span className="bg-emerald-100 text-emerald-700 text-xs font-bold px-3 py-1 rounded-full">{order.status}</span>
              </div>
              <div className="flex -space-x-3 mb-4">
                {order.items.map((item, i) => (
                  <img key={i} src={item.image} className="w-10 h-10 rounded-full border-2 border-white object-cover shadow-sm" alt={item.name} />
                ))}
                {order.items.length > 4 && (
                  <div className="w-10 h-10 rounded-full border-2 border-white bg-slate-100 flex items-center justify-center text-[10px] font-bold text-slate-500">
                    +{order.items.length - 4}
                  </div>
                )}
              </div>
              <div className="flex justify-between items-center border-t border-slate-50 pt-4 mt-2">
                <span className=
